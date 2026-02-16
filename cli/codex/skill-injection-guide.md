# Codex Skill 注入机制详解

本文档详细说明 Codex 项目中 Skill 是如何注入到用户提示词中的完整流程。

---

## 目录

1. [核心文件结构](#核心文件结构)
2. [完整数据流程](#完整数据流程)
3. [阶段一：Skill 文件结构](#阶段一skill-文件结构)
4. [阶段二：Skill 加载与注册](#阶段二skill-加载与注册)
5. [阶段三：Skill 引用检测](#阶段三skill-引用检测)
6. [阶段四：Skill 选择算法](#阶段四skill-选择算法)
7. [阶段五：Skill 注入到提示词](#阶段五skill-注入到提示词)
8. [阶段六：集成到请求流程](#阶段六集成到请求流程)
9. [实战案例](#实战案例)
10. [关键设计细节](#关键设计细节)

---

## 核心文件结构

### Skill 系统核心文件

| 文件路径 | 用途 |
|----------|------|
| `codex-rs/core/src/skills/mod.rs` | 公共 API 导出 |
| `codex-rs/core/src/skills/loader.rs` | SKILL.md 解析与发现 |
| `codex-rs/core/src/skills/manager.rs` | SkillsManager 生命周期管理 |
| `codex-rs/core/src/skills/model.rs` | SkillMetadata、SkillLoadOutcome 数据结构 |
| `codex-rs/core/src/skills/injection.rs` | 引用检测、选择、构建 |
| `codex-rs/core/src/skills/render.rs` | Skills 段落 markdown 渲染 |
| `codex-rs/core/src/skills/system.rs` | 系统 skills 安装 |
| `codex-rs/core/src/instructions/user_instructions.rs` | SkillInstructions XML 转换 |
| `codex-rs/core/src/codex.rs` | 主流程集成 (~第 4264 行) |
| `codex-rs/protocol/src/user_input.rs` | UserInput::Skill 枚举定义 |

---

## 完整数据流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户输入 (UserInput enum)                     │
├────────────────────┬────────────────────┬───────────────────────┤
│ Text("$skill-name")│ Skill{name, path}  │ Mention{name, path}   │
└────────────────────┴────────────────────┴───────────────────────┘
                              ↓
          ┌──────────────────────────────────────┐
          │  引用检测层                           │
          │  extract_tool_mentions() 扫描文本     │
          │  检测: $names 和 [$names](paths)      │
          └──────────────────────────────────────┘
                              ↓
              ┌───────────────────────────────────┐
              │ Skill 选择层                       │
              │ collect_explicit_skill_mentions() │
              │ 1. 结构化输入优先                  │
              │ 2. 文本链接 (显式路径)             │
              │ 3. 简单名称 (无歧义时)             │
              │ 4. 过滤禁用的 skills               │
              └───────────────────────────────────┘
                              ↓
                  ┌──────────────────────────────┐
                  │  被引用的 Skills (Vec)        │
                  │  (去重、有序)                 │
                  └──────────────────────────────┘
                    │                  │
         ┌──────────┘                  └──────────┐
         ↓                                        ↓
   ┌──────────────────────┐    ┌──────────────────────────┐
   │ 隐式 Skills          │    │ Skill 注入               │
   │ (所有启用且          │    │ build_skill_injections() │
   │  allow_implicit=true)│    │ - 异步读取磁盘           │
   │                      │    │ - 创建 XML 项目          │
   │ render_skills_       │    │ - 跟踪分析事件           │
   │  section() →         │    │ - 错误处理               │
   │ Markdown 列表        │    └──────────────────────────┘
   │ 在系统指令中         │                ↓
   └──────────────────────┘  ┌──────────────────────────────┐
                              │  ResponseItem::Message       │
                              │  (XML 包装的 skills)         │
                              │  <skill>...</skill>          │
                              └──────────────────────────────┘
                              ↓
                     ┌─────────────────────────┐
                     │  对话历史记录           │
                     │  record_conversation_   │
                     │  items()                │
                     └─────────────────────────┘
                              ↓
                     ┌─────────────────────────┐
                     │  LLM 模型输入           │
                     │  (完整上下文)           │
                     └─────────────────────────┘
```

---

## 阶段一：Skill 文件结构

### 文件位置

```
~/.codex/skills/[skill-name]/SKILL.md
```

### SKILL.md 格式

```markdown
---
name: demo-skill
description: This is a demo skill
metadata:
  short-description: Optional brief description
interface:
  display_name: Optional display name
  default_prompt: Optional default prompt
dependencies:
  tools:
    - Read
    - Write
policy:
  allow_implicit_invocation: true  # 默认: true
permissions:
  allowedTools:
    - Bash
    - Read
---

# Skill 指令内容

这里是实际传递给 LLM 的指令...
```

### SkillMetadata 数据结构

```rust
pub struct SkillMetadata {
    pub name: String,                    // 来自 frontmatter
    pub description: String,             // 来自 frontmatter
    pub short_description: Option<String>,
    pub interface: Option<InterfaceConfig>,
    pub dependencies: Option<SkillDependencies>,
    pub policy: Option<SkillPolicy>,
    pub permissions: Option<SkillPermissions>,
    pub path: PathBuf,                   // SKILL.md 文件路径
    pub scope: SkillScope,               // System 或 User
}
```

---

## 阶段二：Skill 加载与注册

### SkillsManager

**位置：** `manager.rs`

```rust
pub struct SkillsManager {
    codex_home: PathBuf,
    cache_by_cwd: RwLock<HashMap<PathBuf, Arc<SkillLoadOutcome>>>,
}
```

### 加载流程

```
SkillsManager::skills_for_cwd()
        ↓
    检查 CWD 缓存
        ↓ (若未缓存)
    加载配置层 (用户配置、工作区配置)
        ↓
    skill_roots_from_layer_stack_with_agents()
    确定 skill 根目录位置
        ↓
    load_skills_from_roots()
    扫描目录并解析 SKILL.md
        ↓
    加载禁用的 skill 列表
        ↓
    缓存结果
```

### Skill 根目录来源

| 来源 | 路径示例 |
|------|----------|
| 系统缓存 | `~/.codex/skills/.system/` |
| 用户层 | `~/.codex/skills/` |
| 项目层 | `.codex/skills/` |
| 自定义根目录 | `extra_user_roots` 配置 |

### 禁用 Skills

```yaml
# 配置文件
skills:
  config:
    - path: /path/to/SKILL.md
      enabled: false
```

禁用路径存储在 `SkillLoadOutcome.disabled_paths` (HashSet<PathBuf>)

---

## 阶段三：Skill 引用检测

### 支持的引用格式

#### 1. 简单文本引用

```text
请使用 $demo-skill 处理代码
```

模式：`$[a-zA-Z0-9_-]+`

#### 2. 链接引用

```markdown
使用 [$demo-skill](/path/to/SKILL.md)
使用 [$research](skill://team/research)
```

#### 3. 结构化输入 (TUI/API)

```rust
UserInput::Skill {
    name: "demo-skill",
    path: PathBuf::from("/path/to/SKILL.md")
}
```

### 引用解析逻辑

**函数：** `extract_tool_mentions()` (injection.rs)

```rust
pub struct ToolMentions<'a> {
    pub names: Vec<&'a str>,       // 所有被引用的工具名称
    pub paths: Vec<&'a str>,       // 所有链接资源路径
    pub plain_names: Vec<&'a str>, // 无显式路径的名称
}
```

**解析过程：**
1. 逐字节扫描文本，查找 `$` 字符 (O(T) 时间复杂度)
2. 识别有效引用名称字符：`[a-zA-Z0-9_-]`
3. 解析链接语法：`[$name](path)`
4. 过滤常见环境变量

### 过滤的环境变量

| 变量 | 说明 |
|------|------|
| `PATH` | 系统路径 |
| `HOME` | 用户主目录 |
| `USER` | 用户名 |
| `SHELL` | 当前 shell |
| `PWD` | 当前目录 |
| `TMPDIR`, `TEMP`, `TMP` | 临时目录 |
| `LANG` | 语言设置 |
| `TERM` | 终端类型 |
| `XDG_CONFIG_HOME` | XDG 配置目录 |

### 路径类型分类

```rust
pub enum ToolMentionKind {
    App,    // "app://" 前缀
    Mcp,    // "mcp://" 前缀
    Skill,  // "skill://" 前缀 或 文件名为 "SKILL.md"
    Other,  // 其他
}
```

---

## 阶段四：Skill 选择算法

**函数：** `collect_explicit_skill_mentions()` (injection.rs)

### 选择优先级

```
┌─────────────────────────────────────────────────────────────┐
│ 优先级 1: 结构化输入 (最高)                                 │
│   - UserInput::Skill { name, path }                         │
│   - 验证路径匹配可用 skill                                  │
│   - 检查未被禁用                                            │
│   - 阻止该 skill 的简单名称回退                             │
├─────────────────────────────────────────────────────────────┤
│ 优先级 2: 基于路径的引用                                    │
│   - 匹配链接路径与 skill 路径                               │
│   - 优先精确 SKILL.md 路径匹配                              │
│   - 规范化 "skill://" 前缀                                  │
├─────────────────────────────────────────────────────────────┤
│ 优先级 3: 简单名称回退 (仅当无歧义)                         │
│   - 跳过: 名称在 skills 列表中出现多次                      │
│   - 跳过: 名称与 connector 冲突                             │
│   - 跳过: 被结构化输入阻止                                  │
│   - 选择条件: count == 1 且无 connector 匹配                │
└─────────────────────────────────────────────────────────────┘
```

### 去重与排序

- **去重：** 按 skill 路径去重
- **排序：** 保持文件系统发现的原始顺序

---

## 阶段五：Skill 注入到提示词

### 构建 Skill 注入

**函数：** `build_skill_injections()` (injection.rs)

```rust
pub struct SkillInjections {
    pub items: Vec<ResponseItem>,    // XML 格式的 skill 项目
    pub warnings: Vec<String>,       // 警告消息
}
```

**流程：**

```
对于每个被引用的 skill:
    ↓
异步读取 SKILL.md 文件内容
    ↓
创建 SkillInstructions { name, path, contents }
    ↓
转换为 ResponseItem
    ↓
记录分析事件
    ↓
返回 SkillInjections
```

### XML 格式转换

**位置：** `user_instructions.rs`

```rust
impl From<SkillInstructions> for ResponseItem {
    fn from(si: SkillInstructions) -> Self {
        ResponseItem::Message {
            id: None,
            role: "user".to_string(),
            content: vec![ContentItem::InputText {
                text: format!(
                    "<skill>\n<name>{}</name>\n<path>{}</path>\n{}\n</skill>",
                    si.name, si.path, si.contents
                ),
            }],
            end_turn: None,
            phase: None,
        }
    }
}
```

### 最终注入格式

```xml
<skill>
<name>demo-skill</name>
<path>/canonical/path/to/SKILL.md</path>
[完整的 SKILL.md 内容 - frontmatter 后的 markdown]
</skill>
```

### 隐式 Skills 渲染

**函数：** `render_skills_section()` (render.rs)

在系统指令中渲染为 Markdown 列表：

```markdown
## Skills
A skill is a set of local instructions to follow...

### Available skills
- demo-skill: Description here (file: /path/to/SKILL.md)
- another-skill: Another description (file: /path/to/another/SKILL.md)

### How to use skills
[关于 skill 发现、触发规则、使用模式的说明...]
```

---

## 阶段六：集成到请求流程

### 主入口

**位置：** `codex.rs` 的 `submit_turn()` 函数 (~第 4264-4346 行)

### 步骤详解

```rust
// 1. 加载 skills (按 CWD 缓存)
let skills_outcome = skills_manager.skills_for_cwd(...)?;

// 2. 检测用户输入中的引用
let mentioned_skills = collect_explicit_skill_mentions(
    &input,
    &outcome.skills,
    &outcome.disabled_paths,
    &connector_slug_counts
);

// 3. 解析依赖 (如果功能启用)
let env_var_dependencies = collect_env_var_dependencies(&mentioned_skills);
resolve_skill_dependencies_for_turn(...).await;

// 4. 构建 skill 注入 (异步读取文件)
let SkillInjections { items: skill_items, warnings } =
    build_skill_injections(&mentioned_skills, otel, analytics, tracking).await;

// 5. 记录到对话历史
sess.record_conversation_items(&turn_context, &skill_items).await;

// 6. 从 skills 中提取 app 引用
collect_explicit_app_ids_from_skill_items(&skill_items, ...);

// 7. 完整注入到提示词
// - 用户指令 (隐式 skills 渲染为 markdown 列表)
// - 显式 skill 项目 (XML 格式的 SkillInstructions)
// - 两者作为系统 + 用户消息上下文发送给模型
```

---

## 实战案例

### 案例 1：简单引用

**用户输入：**
```
请使用 $my-formatter 处理这段代码
```

**流程：**

| 步骤 | 操作 |
|------|------|
| 1. 检测 | `extract_tool_mentions()` 找到 "my-formatter" |
| 2. 选择 | 若无歧义且已启用，添加到 `mentioned_skills` |
| 3. 加载 | 异步读取 `my-formatter/SKILL.md` |
| 4. 格式化 | 包装为 `<skill>...</skill>` XML |
| 5. 注入 | 作为 `ResponseItem::Message` 添加 |
| 6. 记录 | 通过 `record_conversation_items()` 添加到历史 |
| 7. 提示词 | 构建模型请求时，skill 出现在历史中 |

**模型收到的内容：**
```xml
<skill>
<name>my-formatter</name>
<path>.claude/skills/my-formatter/SKILL.md</path>

## Formatting Rules
- Use 2 spaces for indentation
- Maximum line length: 100 characters
...
</skill>
```

### 案例 2：链接引用

**用户输入：**
```
参考 [$custom-linter](./lint-rules/SKILL.md) 检查代码
```

**特点：**
- 路径直接从链接提取
- 优先匹配路径（不依赖名称匹配）
- 即使有同名歧义也能正确选择

### 案例 3：UI 结构化选择

**用户操作：** 通过斜杠命令菜单选择 skill

**输入结构：**
```rust
UserInput::Skill {
    name: "security-review",
    path: ".claude/skills/security/SKILL.md"
}
```

**特点：**
- 最高优先级
- 不受歧义影响
- 阻止同名的简单引用回退

### 案例 4：歧义处理

**场景：** 存在两个同名 skill

```
.claude/skills/team-a/test-skill/SKILL.md
.claude/skills/team-b/test-skill/SKILL.md
```

**用户输入：**
```
使用 $test-skill
```

**结果：**
- `skill_name_counts["test-skill"] = 2`
- 由于 count != 1，不选择任何 skill
- 用户需要使用完整路径或链接语法消除歧义

### 案例 5：Connector 冲突

**场景：** 存在同名的 MCP connector

```
# connector 名称: "alpha-skill"
# skill 名称: "alpha-skill"
```

**用户输入：**
```
使用 $alpha-skill
```

**结果：**
- 检测到 connector 冲突
- 跳过该 skill 的简单名称匹配
- 用户需要使用显式路径

---

## 关键设计细节

### 缓存策略

| 层级 | 实现 | 说明 |
|------|------|------|
| CWD 缓存 | `SkillsManager::cache_by_cwd` | 按工作目录缓存 |
| 失效条件 | `force_reload: true` | 强制重新加载 |
| 扫描限制 | 最多 2000 目录/根 | 防止性能问题 |
| 目录深度 | 最多 6 层 | 限制递归深度 |

### 歧义解决优先级

```
1. 路径匹配 (最高)     → 链接引用中的显式路径
2. 结构化输入 (次高)   → UI 选择阻止简单回退
3. 名称唯一性要求      → 只选择无歧义名称 (count == 1)
4. Connector 冲突防止  → 有同名 connector 时跳过 skill
```

### 引用解析规则

| 模式 | 正则表达式 | 说明 |
|------|------------|------|
| 简单引用 | `$[a-zA-Z0-9_-]+` | `$` 后跟有效名称字符 |
| 链接引用 | `[$name](path)` | Markdown 链接语法 |

### 分析跟踪

| 指标/事件 | 说明 |
|-----------|------|
| `codex.skill.injected` | 计数器，包含 status (ok/error) 和 skill name |
| `SkillInvocation` | 事件，包含 skill_name, skill_scope, skill_path |

### 错误处理

| 情况 | 处理方式 |
|------|----------|
| 文件读取失败 | 添加警告，不阻塞执行 |
| 禁用的 skills | 静默跳过（无警告） |
| 缺失结构化输入 | 警告消息后继续 |
| 歧义简单名称 | 静默跳过（歧义解决失败） |

### 复杂度分析

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| 引用检测 | O(T + N·S) | T=文本长度, N=输入数, S=skills数 |
| Skill 选择 | O((N_s + N_t) × S) | 路径匹配 + 去重 |
| 文件注入 | O(M) | M=被引用的 skills 数量 |

---

## 总结

Codex 的 Skill 注入系统通过精心设计的多阶段流程，实现了：

### 功能特性

- ✅ 从磁盘自动发现 skills
- ✅ 双重注入模式（隐式 + 显式）
- ✅ 文本和结构化引用检测
- ✅ 歧义解决和冲突处理
- ✅ 缓存和性能优化
- ✅ 分析事件和错误跟踪
- ✅ XML 包装确保 LLM 安全

### 设计原则

1. **渐进式披露：** 隐式 skills 列表 → 显式 skill 内容
2. **确定性选择：** 明确的优先级规则避免随机选择
3. **容错设计：** 错误不阻塞主流程
4. **性能优化：** 缓存 + 异步 IO
