# Codex Memory 系统深度分析

## 概述

Codex Memory 系统是一个智能的长期记忆机制，旨在让 AI 代理在多次会话中持续学习和改进。它通过自动提取、整合和检索历史会话中的有价值信息，帮助未来的代理：

- 深入理解用户偏好和工作模式
- 复用已验证的工作流程和检查清单
- 避免已知的陷阱和失败模式
- 减少工具调用和推理令牌消耗

---

## 系统架构

### 核心目录结构

```
codex-rs/core/
├── src/memories/                    # 核心实现 (2,617 行代码)
│   ├── mod.rs                       # 模块入口，常量定义
│   ├── start.rs                     # 启动入口函数
│   ├── phase1.rs                    # Phase 1: Rollout 提取
│   ├── phase2.rs                    # Phase 2: 全局整合
│   ├── storage.rs                   # 文件系统存储管理
│   ├── prompts.rs                   # 提示词构建器
│   ├── usage.rs                     # 访问指标追踪
│   └── tests.rs                     # 测试套件
│
└── templates/memories/              # Prompt 模板
    ├── stage_one_system.md          # Phase 1 系统提示词
    ├── consolidation.md             # Phase 2 整合提示词
    └── read_path.md                 # Memory 工具注入指令
```

### 运行时文件布局

```
~/.codex/memories/
├── memory_summary.md               # 始终加载到系统提示中
├── MEMORY.md                       # 可搜索的知识手册
├── raw_memories.md                 # 临时文件：合并的 Phase 1 输出
├── rollout_summaries/              # 每个 rollout 的详细摘要
│   ├── 2026-02-11T15-35-19-jqmb.md
│   └── ...
└── skills/                         # 可复用的技能包
    ├── skill-name/
    │   ├── SKILL.md               # 入口文件
    │   ├── scripts/               # 辅助脚本
    │   ├── templates/             # 模板文件
    │   └── examples/              # 示例输出
    └── ...
```

---

## 两阶段处理流程

### Phase 1: Rollout 提取（并行处理）

**位置**: `codex-rs/core/src/memories/phase1.rs`

Phase 1 负责从每个会话的 rollout（对话历史记录）中提取有价值的记忆。

#### 核心配置常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MODEL` | `gpt-5.1-codex-mini` | 使用的模型 |
| `REASONING_EFFORT` | Low | 推理强度 |
| `CONCURRENCY_LIMIT` | 8 | 最大并行任务数 |
| `DEFAULT_STAGE_ONE_ROLLOUT_TOKEN_LIMIT` | 150,000 | Rollout 截断令牌限制 |
| `CONTEXT_WINDOW_PERCENT` | 70% | 上下文窗口使用比例 |
| `JOB_LEASE_SECONDS` | 3,600 | 任务租约时长（秒） |
| `THREAD_SCAN_LIMIT` | 5,000 | 最大扫描线程数 |

#### 任务选择规则

Phase 1 只处理符合以下条件的 rollout：

1. **交互式会话来源**: 仅限 CLI 和 VSCode（非子代理）
2. **年龄窗口**: 在配置的天数内（默认 30 天）
3. **空闲时间**: 已空闲足够长时间（默认 6 小时）
4. **未被占用**: 没有其他正在进行的 Phase 1 任务
5. **数量限制**: 每次启动最多处理指定数量（默认 16 个）

#### 输出 Schema

```json
{
  "raw_memory": "详细的 Markdown 记忆内容",
  "rollout_summary": "紧凑的摘要行",
  "rollout_slug": "文件系统安全的标识符"
}
```

#### 高价值记忆标准

Phase 1 模型会识别并提取：

1. **验证过的复现方案** - 成功案例的可重复步骤
2. **失败防护** - 症状 → 原因 → 修复 + 验证 + 停止规则
3. **决策触发器** - 防止浪费探索的判断依据
4. **仓库/任务地图** - 入口点、配置、命令的位置
5. **工具特性和快捷方式** - 已验证的命令和标志
6. **稳定的用户偏好** - 跨任务保持不变的约束

#### No-Op 门控机制

当 rollout 没有有价值的内容时，Phase 1 会返回空字段：

```
{"rollout_summary":"","rollout_slug":"","raw_memory":""}
```

这避免了存储无意义的记录。

---

### Phase 2: 全局整合（串行处理）

**位置**: `codex-rs/core/src/memories/phase2.rs`

Phase 2 将所有 Phase 1 的输出整合为一致的知识库。

#### 核心配置常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MODEL` | `gpt-5.3-codex` | 使用更强大的模型 |
| `REASONING_EFFORT` | Medium | 中等推理强度 |
| `JOB_LEASE_SECONDS` | 3,600 | 任务租约时长 |
| `JOB_HEARTBEAT_SECONDS` | 90 | 心跳间隔 |

#### 运行模式

1. **INIT 模式**: 首次构建所有记忆文件
2. **INCREMENTAL UPDATE 模式**: 增量更新现有文件

#### Watermark 机制

Phase 2 使用水印追踪整合进度：

- `input_watermark`: 任务开始时最新输入的时间戳
- `completion_watermark`: 成功完成后存储的时间戳
- 如果没有新数据，Phase 2 会跳过（`SkippedNotDirty`）

#### 子代理配置

Phase 2 启动一个受限的子代理：

- **审批策略**: `Never`（无需用户审批）
- **协作功能**: 禁用（防止递归委托）
- **沙箱策略**: `WorkspaceWrite`
  - 可写目录: 仅 codex_home
  - 网络访问: 禁用
- **工作目录**: Memory 根目录

---

## 配置系统

### MemoriesConfig 结构体

```rust
pub struct MemoriesConfig {
    pub max_raw_memories_for_global: usize,    // 默认: 1,024 (最大 4,096)
    pub max_rollout_age_days: i64,             // 默认: 30 (范围: 0-90)
    pub max_rollouts_per_startup: usize,       // 默认: 16 (最大 128)
    pub min_rollout_idle_hours: i64,           // 默认: 6 (范围: 1-48)
    pub phase_1_model: Option<String>,         // 可选覆盖
    pub phase_2_model: Option<String>,         // 可选覆盖
}
```

### TOML 配置示例

```toml
[memories]
max_raw_memories_for_global = 512
max_rollout_age_days = 42
max_rollouts_per_startup = 9
min_rollout_idle_hours = 24
phase_1_model = "gpt-5-mini"
phase_2_model = "gpt-5"
```

---

## 记忆文件格式

### MEMORY.md 格式

这是持久的、面向检索的知识手册：

```markdown
# Task Group: <项目/工作流/任务族>

scope: <覆盖范围和使用边界>

## Task 1: <任务描述, 结果>

### rollout_summary_files
- rollout_summaries/file.md (cwd=<路径>, updated_at=<时间戳>, thread_id=<ID>)

### keywords
- keyword1, keyword2, keyword3, ...

### learnings
- <经验教训>
- <用户偏好>
- <失败防护: 症状 → 原因 → 修复>
- <范围边界/防漂移说明>

## General Tips
- <跨任务指导，去重后的通用建议> [Task 1]
```

### memory_summary.md 格式

这是始终加载到系统提示中的高层摘要：

```markdown
## User Profile
<用户的生动描述，帮助未来的代理更好地协作>

## General Tips
<几乎每次运行都有用的信息>
- 协作偏好
- 工作流和环境
- 决策启发式
- 工具使用习惯
- 验证习惯
- 陷阱和修复

## What's in Memory
### <最近的记忆日期: YYYY-MM-DD>
- <主题>: <关键词1>, <关键词2>, ...
  - desc: <描述和何时使用>
  - learnings: <最近的关键发现>

### <第二近的记忆日期>
...

### Older Memory Topics
- <主题>: <关键词>
  - desc: <紧凑描述>
```

### SKILL.md 格式

可复用的技能包：

```markdown
---
name: skill-name
description: 1-2 行描述，包含具体触发条件
argument-hint: "[可选参数]"
disable-model-invocation: false
user-invocable: true
allowed-tools: ["Read", "Glob", "Bash"]
---

When to use:
- <触发条件>

Inputs / context:
- <所需输入>

Procedure:
1. <步骤>
2. ...

Efficiency plan:
- <效率提升策略>

Pitfalls and fixes:
- 症状 → 原因 → 修复

Verification:
- <验证清单>
```

---

## 记忆注入机制

### 入口点

**位置**: `codex-rs/core/src/codex.rs`

```rust
if let Some(memory_prompt) =
    build_memory_tool_developer_instructions(&turn_context.config.codex_home).await
    && turn_context.features.enabled(Feature::MemoryTool)
{
    // 注入到开发者指令中
}
```

### 注入流程

1. 读取 `memory_summary.md`
2. 截断至 5,000 令牌（如需要）
3. 渲染模板，替换：
   - `{{ base_path }}`: Memory 根目录
   - `{{ memory_summary }}`: 截断后的摘要内容

### 决策边界

代理根据以下规则决定是否使用 memory：

**跳过 memory 的情况**:
- 当前时间/日期查询
- 简单翻译
- 简单单行命令
- 简单格式化

**使用 memory 的情况**:
- 查询涉及 memory_summary 中提到的仓库/路径/文件
- 用户要求保持一致性或参考之前决策
- 任务模糊，可能依赖早期选择
- 与 memory 主题相关的非平凡任务

### 引用要求

如果使用了 memory 文件，必须在回复末尾添加：

```xml
<memory_citation>
<citation_entries>
MEMORY.md:234-236|note=[代码指针引用]
rollout_summaries/2026-02-17T21-23-02-LN3m.md:10-12|note=[报告格式]
</citation_entries>
<thread_ids>
019c6e27-e55b-73d1-87d8-4e01f1f75043
</thread_ids>
</memory_citation>
```

---

## 安全机制

### 秘密脱敏

所有 memory 输出都经过秘密脱敏处理：

- API 密钥、令牌、密码被替换为 `[REDACTED_SECRET]`
- 使用 `codex_secrets` crate 进行模式检测
- 应用于 `raw_memory` 和 `rollout_summary` 字段

### 沙箱隔离

Phase 2 子代理运行在受限环境中：

- 无法访问网络
- 只能写入 memory 根目录
- 无用户审批能力
- 禁用协作模式（防止递归）

### 令牌预算

- Phase 1 rollout 截断至模型上下文窗口的 70%
- Memory summary 注入限制在 5,000 令牌
- 防止上下文溢出

---

## 可观测性

### 指标追踪

```rust
// Phase 1
codex.memory.phase1              // 按状态分组的任务数
codex.memory.phase1.e2e_ms       // 端到端延迟
codex.memory.phase1.output       // 产生的原始记忆数
codex.memory.phase1.token_usage  // 令牌使用直方图

// Phase 2
codex.memory.phase2              // 按状态分组的任务数
codex.memory.phase2.e2e_ms       // 端到端延迟
codex.memory.phase2.input        // Stage-1 输入数
codex.memory.phase2.token_usage  // 令牌使用直方图

// Memory 访问
codex.memories.usage             // 访问指标
                                // 标签: kind, tool, success
```

---

## 关键设计模式

### 1. 两阶段架构

- **Phase 1**: 可并行扩展，每个 rollout 独立处理
- **Phase 2**: 串行整合，防止文件冲突

### 2. 租约协调

- 基于数据库的所有权令牌防止重复工作
- 失败后的退避延迟（3,600 秒）
- 心跳机制保持长时间运行的任务活跃

### 3. 水印进度追踪

- 追踪"脏"状态（上次整合后的新数据）
- 支持安全的增量更新
- 允许跳过无需整合的情况

### 4. No-Op 门控

- Phase 1: 无有效信号时返回空字段
- Phase 2: 无新信号时不做更改
- 防止无意义的数据积累

### 5. 特性开关

- `Feature::MemoryTool` 控制 memory 功能
- 可全局或按会话禁用
- 仅在功能启用时注入 memory

---

## 启动流程

```
会话启动
    │
    ├─ 检查条件:
    │  ├─ 非临时会话
    │  ├─ MemoryTool 功能启用
    │  ├─ 非子代理会话
    │  └─ State DB 可用
    │
    └─ 异步启动:
       │
       ├─ Phase 1 (并行)
       │  ├─ 从 State DB 声明任务
       │  ├─ 并行处理 rollout
       │  ├─ 提取记忆
       │  └─ 存储到 State DB
       │
       └─ Phase 2 (串行)
          ├─ 声明全局锁
          ├─ 加载 Stage-1 输出
          ├─ 同步文件系统
          ├─ 启动整合子代理
          └─ 标记完成
```

---

## 最近更新 (2026-02-23 ~ 2026-02-24)

根据 git 历史记录，最近的改进包括：

### Memory 查找指导加强

- 收紧 memory 使用决策边界
- 使快速 memory 搜索更加明确和有界
- 添加结构化 `<memory_citation>` 要求

### Consolidation 提示词强化

- 加强 Phase 2 consolidation 提示词
- 改进 `MEMORY.md` 模式清理
- 更好的排序行为（实用性和时效性）
- 重写 `## What's in Memory` 作为更清晰的路由索引

---

## 总结

Codex Memory 系统是一个精心设计的长期学习机制，通过：

1. **自动化提取** - 无需用户干预，自动从对话中学习
2. **分层存储** - 从详细 rollout 到紧凑摘要的多级结构
3. **渐进式披露** - 按需访问不同详细程度的信息
4. **安全隔离** - 秘密脱敏和沙箱保护
5. **高效检索** - 关键词驱动的快速查找

这个系统让 AI 代理能够真正"记住"过去的经验，在后续会话中表现得越来越好。
