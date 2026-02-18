# Codex Skill 系统深度分析文档

## 目录

1. [概述](#概述)
2. [核心数据模型](#核心数据模型)
3. [加载与发现机制](#加载与发现机制)
4. [Prompt 系统架构](#prompt-系统架构)
5. [工程架构设计](#工程架构设计)
6. [文件格式规范](#文件格式规范)
7. [触发与调用机制](#触发与调用机制)
8. [权限与安全模型](#权限与安全模型)
9. [性能优化策略](#性能优化策略)
10. [关键文件索引](#关键文件索引)

---

## 概述

Skill 是 Codex 项目中用于扩展 AI Agent 能力的模块化指令系统。它允许用户创建自定义的"技能"，为 Agent 提供特定领域的专业知识、工作流程和工具集成。

### 核心设计理念

1. **渐进式披露 (Progressive Disclosure)**: 三级加载系统，按需加载内容
2. **多层作用域隔离**: Repo > User > System > Admin 优先级
3. **上下文高效利用**: 元数据常驻，内容按需，脚本可执行而不读入
4. **声明式配置**: YAML frontmatter 定义元数据，Markdown 编写指令

### 架构概览图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Skill 系统架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │ SkillsManager │←→│   Loader    │←→│   Parser    │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                   │                │
│         ▼                  ▼                   ▼                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │   Cache     │    │  Discovery  │    │  Validator  │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  Injection  │    │   Render    │    │ Permissions │        │
│  └──────┬──────┘    └──────┬──────┘    └─────────────┘        │
│         │                  │                                    │
│         ▼                  ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              LLM Context (User Message)                  │  │
│  │  ┌─────────────────┐    ┌─────────────────────────────┐ │  │
│  │  │ Skills Section  │    │  Individual Skill Content   │ │  │
│  │  │  (Discovery)    │    │  (XML Wrapped Instructions) │ │  │
│  │  └─────────────────┘    └─────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心数据模型

### 1. SkillMetadata - 运行时表示

位置: `codex-rs/core/src/skills/model.rs`

```rust
pub struct SkillMetadata {
    pub name: String,                          // 技能名称 (≤64字符)
    pub description: String,                   // 技能描述 (≤1024字符)
    pub short_description: Option<String>,     // 短描述 (≤1024字符)
    pub interface: Option<SkillInterface>,     // UI 元数据
    pub dependencies: Option<SkillDependencies>, // 依赖配置
    pub policy: Option<SkillPolicy>,           // 调用策略
    pub permissions: Option<Permissions>,      // 权限配置
    pub path: PathBuf,                         // SKILL.md 文件路径
    pub scope: SkillScope,                     // 作用域级别
}
```

### 2. SkillInterface - UI 展示元数据

```rust
pub struct SkillInterface {
    pub display_name: Option<String>,          // 显示名称
    pub short_description: Option<String>,     // 短描述
    pub icon_small: Option<PathBuf>,           // 小图标路径
    pub icon_large: Option<PathBuf>,           // 大图标路径
    pub brand_color: Option<String>,           // 品牌颜色
    pub default_prompt: Option<String>,        // 默认提示词 (≤1024字符)
}
```

### 3. SkillPolicy - 调用策略

```rust
pub struct SkillPolicy {
    pub allow_implicit_invocation: Option<bool>, // 是否允许隐式触发
}
```

### 4. SkillDependencies - 依赖管理

```rust
pub struct SkillDependencies {
    pub tools: Vec<SkillToolDependency>,
}

pub struct SkillToolDependency {
    pub r#type: String,           // 类型: "env_var", "command" 等
    pub value: String,            // 值
    pub description: Option<String>, // 描述
    pub transport: Option<String>,   // 传输方式
    pub command: Option<String>,     // 执行命令
    pub url: Option<String>,         // URL
}
```

### 5. SkillLoadOutcome - 加载结果

```rust
pub struct SkillLoadOutcome {
    pub skills: Vec<SkillMetadata>,           // 成功加载的技能
    pub errors: Vec<SkillError>,              // 加载错误
    pub disabled_paths: HashSet<PathBuf>,     // 被禁用的技能路径
}
```

### 6. SkillScope - 作用域枚举

```rust
pub enum SkillScope {
    Repo,    // 项目级 - 最高优先级 (0)
    User,    // 用户级 (1)
    System,  // 系统级 (2)
    Admin,   // 管理员级 - 最低优先级 (3)
}
```

---

## 加载与发现机制

### 1. 多层级 Skill 根目录

```
优先级从高到低:
┌─────────────────────────────────────────────────────────────┐
│ 1. Project Layer (SkillScope::Repo)                         │
│    位置: {project}/.agents/skills                           │
│    用途: 项目特定的技能                                      │
├─────────────────────────────────────────────────────────────┤
│ 2. User Layer (SkillScope::User)                            │
│    位置 1: $HOME/.agents/skills (推荐)                      │
│    位置 2: $CODEX_HOME/skills (已弃用，向后兼容)             │
│    用途: 用户个人技能                                        │
├─────────────────────────────────────────────────────────────┤
│ 3. System Layer (SkillScope::System)                        │
│    位置: $CODEX_HOME/skills/.system                         │
│    用途: 系统内置技能 (由 Codex 管理)                        │
├─────────────────────────────────────────────────────────────┤
│ 4. Admin Layer (SkillScope::Admin)                          │
│    位置: /etc/codex/skills (Unix)                           │
│    用途: 管理员全局技能                                      │
└─────────────────────────────────────────────────────────────┘
```

### 2. 发现流程

位置: `codex-rs/core/src/skills/loader.rs`

```
load_skills(config)
    │
    ▼
skill_roots(config) ──────────────────┐
    │                                  │
    ▼                                  ▼
load_skills_from_roots()      构建多层根目录列表
    │
    ▼
discover_skills_under_root()
    │
    ├── 广度优先搜索 (BFS)
    ├── 最大深度: 6 层
    ├── 每个根目录最多 2000 个技能
    │
    ▼
parse_skill_file()
    │
    ├── 读取 SKILL.md
    ├── 提取 YAML frontmatter
    ├── 加载 agents/openai.yaml (可选)
    ├── 验证字段长度
    │
    ▼
返回 SkillLoadOutcome
```

### 3. 关键常量

```rust
const SKILLS_FILENAME: &str = "SKILL.md";           // 技能文件名
const SKILLS_METADATA_FILENAME: &str = "openai.yaml"; // UI 元数据文件
const MAX_SCAN_DEPTH: usize = 6;                    // 最大扫描深度
const MAX_SKILLS_DIRS_PER_ROOT: usize = 2000;       // 单根目录最大技能数
const MAX_NAME_LEN: usize = 64;                     // 名称最大长度
const MAX_DESCRIPTION_LEN: usize = 1024;            // 描述最大长度
```

### 4. 去重和排序

```rust
// 1. 按路径去重 (同一路径只保留第一个)
// 2. 按作用域排序 (Repo > User > System > Admin)
// 3. 按名称排序
// 4. 按路径排序
```

---

## Prompt 系统架构

### 1. 分层 Prompt 构建

```
┌─────────────────────────────────────────────────────────────┐
│                    完整 LLM 上下文                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Base System Prompt (prompt.md)                          │
│     └── Codex 核心能力和行为准则                            │
│                                                             │
│  2. Project Instructions (AGENTS.md)                        │
│     └── 项目特定的指令和上下文                              │
│                                                             │
│  3. Skills Section (render_skills_section)                  │
│     ├── 可用技能列表                                        │
│     ├── 如何使用技能指南                                    │
│     └── 渐进式披露规则                                      │
│                                                             │
│  4. Individual Skill Content (SkillInstructions)            │
│     └── XML 包装的完整技能内容                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. Skills Section 模板

位置: `codex-rs/core/src/skills/render.rs`

```markdown
## Skills
A skill is a set of local instructions to follow that is stored in a `SKILL.md` file.
Below is the list of skills that can be used...

### Available skills
- {name}: {description} (file: {path})
- ...

### How to use skills
- Discovery: ...
- Trigger rules: ...
- Missing/blocked: ...
- How to use a skill (progressive disclosure):
  1) After deciding to use a skill, open its `SKILL.md`...
  2) Resolve relative paths relative to skill directory...
  3) Load only specific files needed...
  4) Prefer running scripts over code blocks...
  5) Reuse templates and assets...
- Coordination and sequencing: ...
- Context hygiene: ...
- Safety and fallback: ...
```

### 3. Skill 内容注入格式

位置: `codex-rs/core/src/instructions/user_instructions.rs`

**UserInstructions 格式:**
```xml
# AGENTS.md instructions for {directory}

<INSTRUCTIONS>
{project_doc_content}
{skills_section}
{child_agents_message}
</INSTRUCTIONS>
```

**SkillInstructions 格式:**
```xml
<skill>
<name>{skill_name}</name>
<path>{skill_path}</path>
{skill_contents_from_SKILL.md}
</skill>
```

### 4. 关键设计: 用户消息而非系统消息

```rust
impl From<SkillInstructions> for ResponseItem {
    fn from(si: SkillInstructions) -> Self {
        ResponseItem::Message {
            role: "user".to_string(),  // 注意: 是用户消息!
            content: vec![ContentItem::InputText {
                text: format!(
                    "<skill>\n<name>{}</name>\n<path>{}</path>\n{}\n</skill>",
                    si.name, si.path, si.contents
                ),
            }],
            // ...
        }
    }
}
```

**设计原因:**
- 保留指令在对话历史中
- 允许每轮动态改变技能
- 与轮次交互模型自然集成
- 避免系统提示过长

---

## 工程架构设计

### 1. 模块结构

```
codex-rs/core/src/skills/
├── mod.rs                      # 模块入口，导出公共 API
├── model.rs                    # 数据模型定义
├── manager.rs                  # SkillsManager - 生命周期管理
├── loader.rs                   # 技能发现和加载 (~2000+ 行)
├── injection.rs                # 内容注入到上下文
├── permissions.rs              # 权限编译和验证
├── system.rs                   # 系统内置技能安装
├── env_var_dependencies.rs     # 环境变量依赖解析
├── remote.rs                   # 远程技能下载
├── render.rs                   # 技能渲染为文本指令
└── assets/samples/             # 内置示例技能
    ├── skill-creator/
    └── skill-installer/
```

### 2. SkillsManager - 生命周期管理器

位置: `codex-rs/core/src/skills/manager.rs`

```rust
pub struct SkillsManager {
    codex_home: PathBuf,
    cache_by_cwd: RwLock<HashMap<PathBuf, SkillLoadOutcome>>,
}

impl SkillsManager {
    // 创建管理器，安装系统技能
    pub fn new(codex_home: PathBuf) -> Self

    // 从现有配置加载技能，缓存结果
    pub fn skills_for_config(&self, config: &Config) -> SkillLoadOutcome

    // 从工作目录异步加载技能
    pub async fn skills_for_cwd(&self, cwd: &Path, force_reload: bool) -> SkillLoadOutcome

    // 加载额外的用户根目录
    pub async fn skills_for_cwd_with_extra_user_roots(...) -> SkillLoadOutcome

    // 清理缓存
    pub fn clear_cache(&self)
}
```

### 3. 缓存策略

```rust
// 按工作目录缓存
cache_by_cwd: RwLock<HashMap<PathBuf, SkillLoadOutcome>>

// 缓存命中条件:
// 1. 工作目录匹配
// 2. force_reload = false

// 缓存失效:
// 1. force_reload = true
// 2. clear_cache() 调用
// 3. 新的 extra_user_roots
```

### 4. 依赖关系图

```
┌─────────────┐
│   Config    │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│SkillsManager│────→│   Loader    │
└──────┬──────┘     └──────┬──────┘
       │                   │
       │            ┌──────┴──────┐
       │            ▼             ▼
       │     ┌──────────┐  ┌──────────┐
       │     │ Discovery│  │  Parser  │
       │     └──────────┘  └──────────┘
       │
       ▼
┌─────────────┐
│   Cache     │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│  Injection  │────→│   Render    │
└──────┬──────┘     └──────┬──────┘
       │                   │
       ▼                   ▼
┌─────────────────────────────────┐
│        LLM Context              │
└─────────────────────────────────┘
```

### 5. 与其他系统集成

| 系统 | 集成点 | 文件 |
|------|--------|------|
| 配置系统 | 禁用技能配置 | `manager.rs:154-186` |
| 会话系统 | SkillsManager 作为 Arc | `codex.rs` |
| 文件监视器 | 监视技能目录变化 | `file_watcher.rs` |
| MCP 依赖 | 自动安装 MCP 服务器 | `skill_dependencies.rs` |
| 分析系统 | 技能调用追踪 | `injection.rs` |
| OTEL | 性能指标 | `injection.rs` |

---

## 文件格式规范

### 1. SKILL.md 结构

```markdown
---
name: skill-name
description: What the skill does and when to use it. This is the PRIMARY trigger.
metadata:
  short-description: Brief description for UI
---

# Skill Title

Instructions and procedures...

## How to use
Step-by-step guide...

## References
- [Detailed Guide](references/guide.md)
- [API Docs](references/api.md)
```

### 2. 目录结构

```
skill-name/
├── SKILL.md                    # 必需 - 主指令文件
├── agents/
│   └── openai.yaml            # 推荐 - UI 元数据
├── scripts/                    # 可选 - 可执行代码
│   ├── process.py
│   └── transform.sh
├── references/                 # 可选 - 参考文档
│   ├── schema.md
│   └── api.md
└── assets/                     # 可选 - 输出资源
    ├── template.html
    └── logo.png
```

### 3. openai.yaml 格式

```yaml
interface:
  display_name: "Skill Display Name"
  short_description: "Brief description for UI chips"
  icon_small: "./assets/icon-small.svg"
  icon_large: "./assets/icon-large.png"
  brand_color: "#FF5733"
  default_prompt: "Default prompt when user clicks skill"
```

### 4. 字段验证规则

| 字段 | 最大长度 | 必需 | 说明 |
|------|---------|------|------|
| name | 64 | 是 | 小写字母、数字、连字符 |
| description | 1024 | 是 | 触发描述，包含何时使用 |
| short_description | 1024 | 否 | UI 短描述 |
| default_prompt | 1024 | 否 | 默认提示词 |
| dependency_type | 64 | 否 | 依赖类型 |
| dependency_value | 1024 | 否 | 依赖值 |

---

## 触发与调用机制

### 1. 显式触发方式

**方式一: `$skill-name` 语法**
```
用户输入: "请使用 $skill-creator 创建一个新技能"
```

**方式二: 链接语法**
```
用户输入: "请使用 [$skill-creator](/path/to/skill-creator/SKILL.md)"
```

**方式三: 结构化输入**
```rust
UserInput::Skill {
    name: "skill-creator".to_string(),
    path: PathBuf::from("/path/to/skill-creator/SKILL.md"),
}
```

### 2. 隐式触发

当任务描述与技能描述匹配时，Agent 自动选择使用。

条件:
- `allow_implicit_invocation = true` (默认)
- 技能未被禁用
- 技能描述与任务匹配

### 3. 消歧逻辑

位置: `codex-rs/core/src/skills/injection.rs:collect_explicit_skill_mentions`

```rust
// 1. 结构化输入优先 (UserInput::Skill)
// 2. 链接路径匹配次之 (精确路径)
// 3. 纯名称匹配最后 (需要唯一)

// 名称消歧规则:
// - 如果多个技能同名，跳过纯名称匹配
// - 如果存在同名 MCP 连接器，跳过纯名称匹配
// - 只有唯一匹配时才使用纯名称
```

### 4. 调用流程

```
用户输入
    │
    ▼
collect_explicit_skill_mentions()
    │
    ├── 解析 $skill-name 标记
    ├── 解析链接语法
    ├── 处理结构化输入
    │
    ▼
build_skill_injections()
    │
    ├── 读取 SKILL.md 内容
    ├── 包装为 XML 格式
    ├── 发送 OTEL 指标
    ├── 追踪分析事件
    │
    ▼
注入到 LLM 上下文 (User Message)
```

---

## 权限与安全模型

### 1. 权限清单结构

位置: `codex-rs/core/src/skills/permissions.rs`

```rust
struct SkillManifestPermissions {
    network: bool,                                    // 网络访问
    file_system: SkillManifestFileSystemPermissions, // 文件系统
    macos: SkillManifestMacOsPermissions,            // macOS 特定
}

struct SkillManifestFileSystemPermissions {
    read: Vec<String>,   // 可读路径
    write: Vec<String>,  // 可写路径
}
```

### 2. 沙箱策略编译

```rust
fn compile_permission_profile(
    skill_dir: &Path,
    permissions: Option<SkillManifestPermissions>,
) -> Option<Permissions>

// 转换规则:
// - 无写入权限 → SandboxPolicy::ReadOnly
// - 有写入权限 → SandboxPolicy::WorkspaceWrite
// - 网络访问 → 启用网络策略
```

### 3. 禁用机制

配置文件 (`~/.codex/config.toml`):
```toml
[skills.config]
[[skills.config]]
path = "/path/to/skill"
enabled = false
```

---

## 性能优化策略

### 1. 渐进式加载

```
Level 1: 元数据 (name + description)  ~100 字, 常驻上下文
Level 2: SKILL.md body               <5k 字, 触发时加载
Level 3: Bundled resources           按需, 脚本可执行而不读入
```

### 2. 缓存策略

- 按工作目录缓存 `SkillLoadOutcome`
- 使用 `RwLock` 实现并发安全
- 支持强制重新加载

### 3. 扫描限制

```rust
const MAX_SCAN_DEPTH: usize = 6;              // 最大深度
const MAX_SKILLS_DIRS_PER_ROOT: usize = 2000; // 单根目录上限
```

### 4. 符号链接处理

- User/Admin/Repo 作用域: 跟随符号链接
- System 作用域: 不跟随 (由 Codex 管理)

---

## 关键文件索引

| 文件 | 功能 | 行数 |
|------|------|------|
| `skills/mod.rs` | 模块导出 | ~23 |
| `skills/model.rs` | 数据结构定义 | ~96 |
| `skills/loader.rs` | 加载和解析逻辑 | ~2000+ |
| `skills/manager.rs` | 缓存管理 | ~200 |
| `skills/injection.rs` | 内容注入 | ~400 |
| `skills/render.rs` | 系统提示渲染 | ~44 |
| `skills/permissions.rs` | 权限编译 | ~150 |
| `skills/system.rs` | 系统 skill 部署 | ~100 |
| `instructions/user_instructions.rs` | 指令包装 | ~170 |

---

## 总结

Codex 的 Skill 系统是一个设计精良的模块化指令扩展框架，其核心特点包括:

1. **三层渐进式披露**: 有效管理上下文窗口使用
2. **多作用域隔离**: 支持项目、用户、系统、管理员四级优先级
3. **灵活的触发机制**: 支持显式提及和隐式匹配
4. **用户消息注入**: 保持对话历史的完整性
5. **完善的权限系统**: 支持沙箱隔离和细粒度控制
6. **高性能缓存**: 按目录缓存，支持并发访问

这套系统使得 Codex 能够从通用 AI Agent 转变为具备特定领域专业知识的专家系统，同时保持上下文效率和安全隔离。
