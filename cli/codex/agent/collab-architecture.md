# Codex Collab 多Agent协作架构分析

## 概述

`Collab` 是 Codex 项目中的多Agent协作功能模块，允许主Agent生成多个子Agent并行处理任务，提升工作效率。该功能目前处于 **Experimental（实验性）** 阶段。

## 功能定义

### Feature Flag 配置

位置：`codex-rs/core/src/features.rs:573-582`

```rust
FeatureSpec {
    id: Feature::Collab,
    key: "multi_agent",  // 别名: "collab"
    stage: Stage::Experimental {
        name: "Multi-agents",
        menu_description: "Ask Codex to spawn multiple agents to parallelize the work and win in efficiency.",
        announcement: "NEW: Multi-agents can now be spawned by Codex. Enable in /experimental and restart Codex!",
    },
    default_enabled: false,
},
```

### 启用方式

1. **TUI 界面**: 使用 `/experimental` 命令菜单启用
2. **配置文件**: 在 `config.toml` 中添加：
   ```toml
   [features]
   multi_agent = true
   # 或使用别名
   collab = true
   ```
3. **命令行**: 使用 `--enable multi_agent` 参数

## 核心架构

### 1. AgentControl 控制平面

位置：`codex-rs/core/src/agent/control.rs`

`AgentControl` 是多Agent操作的核心控制句柄：

```rust
#[derive(Clone, Default)]
pub(crate) struct AgentControl {
    /// 弱引用回全局线程注册表/状态
    manager: Weak<ThreadManagerState>,
    /// 保护机制（线程限制、深度控制）
    state: Arc<Guards>,
}
```

**核心方法**：

| 方法 | 功能 | 返回值 |
|------|------|--------|
| `spawn_agent()` | 创建新Agent线程并提交初始提示 | `ThreadId` |
| `resume_agent_from_rollout()` | 从rollout文件恢复Agent | `ThreadId` |
| `send_input()` | 向Agent发送用户输入 | `String` (submission_id) |
| `interrupt_agent()` | 中断Agent当前任务 | `String` |
| `shutdown_agent()` | 关闭Agent线程 | `String` |
| `get_status()` | 获取Agent状态 | `AgentStatus` |
| `subscribe_status()` | 订阅Agent状态更新 | `watch::Receiver<AgentStatus>` |

### 2. 多Agent工具集

位置：`codex-rs/core/src/tools/handlers/multi_agents.rs`

#### 2.1 spawn_agent

创建新的子Agent。

**参数**：
```typescript
{
  message?: string,      // 文本提示（与items二选一）
  items?: UserInput[],   // 结构化输入项
  agent_type?: string    // 角色类型（如 "explorer"）
}
```

**返回**：
```typescript
{
  agent_id: string  // 新创建的Agent ID
}
```

**流程**：
1. 解析输入参数（message 或 items）
2. 检查深度限制 (`exceeds_thread_spawn_depth_limit`)
3. 应用角色配置 (`apply_role_to_config`)
4. 通过 `AgentControl.spawn_agent()` 创建Agent
5. 发送 `CollabAgentSpawnBeginEvent` 和 `CollabAgentSpawnEndEvent` 事件

#### 2.2 send_input

向现有Agent发送输入。

**参数**：
```typescript
{
  id: string,            // Agent ID
  message?: string,      // 文本提示
  items?: UserInput[],   // 结构化输入项
  interrupt?: boolean    // 是否先中断当前任务
}
```

**返回**：
```typescript
{
  submission_id: string
}
```

#### 2.3 resume_agent

恢复已关闭的Agent。

**参数**：
```typescript
{
  id: string  // Agent ID
}
```

**返回**：
```typescript
{
  status: AgentStatus
}
```

**功能**：
- 如果Agent仍在运行，返回当前状态
- 如果Agent已关闭，尝试从rollout文件恢复

#### 2.4 wait

等待Agent完成。

**参数**：
```typescript
{
  ids: string[],        // Agent ID列表
  timeout_ms?: number   // 超时时间（毫秒）
}
```

**超时限制**：
- 最小值：10,000ms（防止CPU空转）
- 默认值：30,000ms
- 最大值：300,000ms

**返回**：
```typescript
{
  status: HashMap<ThreadId, AgentStatus>,
  timed_out: boolean
}
```

#### 2.5 close_agent

关闭Agent。

**参数**：
```typescript
{
  id: string  // Agent ID
}
```

**返回**：
```typescript
{
  status: AgentStatus
}
```

## 状态管理

### AgentStatus 枚举

```rust
pub enum AgentStatus {
    PendingInit,          // 等待初始化
    Running,              // 运行中
    Completed(Option<String>),  // 已完成（带最后消息）
    Errored(String),      // 错误
    Shutdown,             // 已关闭
    NotFound,             // 未找到
}
```

### 状态转换

```
PendingInit -> Running -> Completed/Errored/Shutdown
                        ^
                        |
                    resume_agent
```

## 保护机制

### 1. 线程数量限制

**配置**：`agent_max_threads`（默认值：6）

```rust
pub(crate) const DEFAULT_AGENT_MAX_THREADS: Option<usize> = Some(6);
```

**实现**：使用 `Guards` 结构体管理：

```rust
pub(crate) struct Guards {
    /// 活跃线程ID集合
    active_threads: Mutex<HashSet<ThreadId>>,
    /// 当前活跃线程数（原子操作）
    active_count: AtomicUsize,
}
```

**RAII 模式**：`SpawnReservation` 确保槽位正确释放：

```rust
pub(crate) struct SpawnReservation {
    guards: Arc<Guards>,
    slot_reserved: bool,
    thread_id: Option<ThreadId>,
}
```

### 2. 深度限制

**配置**：`agent_max_depth`（默认值：1）

```rust
pub(crate) const DEFAULT_AGENT_MAX_DEPTH: i32 = 1;
```

**目的**：防止Agent无限嵌套生成子Agent。

**检查逻辑**：
```rust
if exceeds_thread_spawn_depth_limit(child_depth, turn.config.agent_max_depth) {
    return Err(FunctionCallError::RespondToModel(
        "Agent depth limit reached. Solve the task yourself.".to_string(),
    ));
}
```

### 3. 自动禁用嵌套

当达到深度限制时，自动禁用子Agent的Collab功能：

```rust
fn apply_spawn_agent_overrides(config: &mut Config, child_depth: i32) {
    config.permissions.approval_policy = Constrained::allow_only(AskForApproval::Never);
    if exceeds_thread_spawn_depth_limit(child_depth + 1, config.agent_max_depth) {
        config.features.disable(Feature::Collab);
    }
}
```

## 内置角色

### explorer 角色

位置：`codex-rs/core/src/agent/builtins/explorer.toml`

```toml
model = "gpt-5.1-codex-mini"
model_reasoning_effort = "medium"
```

**用途**：轻量级探索任务，使用较小的模型以节省成本。

## 事件协议

### 协议事件类型

定义在 `codex-rs/protocol/src/protocol.rs`：

| 事件类型 | 触发时机 |
|---------|---------|
| `CollabAgentSpawnBeginEvent` | 开始创建Agent |
| `CollabAgentSpawnEndEvent` | Agent创建完成 |
| `CollabAgentInteractionBeginEvent` | 开始发送输入 |
| `CollabAgentInteractionEndEvent` | 输入发送完成 |
| `CollabWaitingBeginEvent` | 开始等待Agent |
| `CollabWaitingEndEvent` | 等待结束 |
| `CollabCloseBeginEvent` | 开始关闭Agent |
| `CollabCloseEndEvent` | Agent关闭完成 |
| `CollabResumeBeginEvent` | 开始恢复Agent |
| `CollabResumeEndEvent` | Agent恢复完成 |

### 事件结构示例

```rust
pub struct CollabAgentSpawnEndEvent {
    pub call_id: String,
    pub sender_thread_id: ThreadId,
    pub new_thread_id: Option<ThreadId>,
    pub prompt: String,
    pub status: AgentStatus,
}
```

## TUI 渲染

位置：`codex-rs/tui/src/multi_agents.rs`

TUI 组件负责渲染多Agent事件，显示格式包括：
- Agent 状态（颜色编码）
- 提示预览（截断显示）
- 等待状态汇总
- 完成消息预览

```rust
fn status_span(status: &AgentStatus) -> Span<'static> {
    match status {
        AgentStatus::PendingInit => Span::from("pending init").dim(),
        AgentStatus::Running => Span::from("running").cyan().bold(),
        AgentStatus::Completed(_) => Span::from("completed").green(),
        AgentStatus::Errored(_) => Span::from("errored").red(),
        AgentStatus::Shutdown => Span::from("shutdown").dim(),
        AgentStatus::NotFound => Span::from("not found").red(),
    }
}
```

## 完整配置示例

```toml
# config.toml

[features]
multi_agent = true

[agents]
max_threads = 6    # 最大并发Agent数
max_depth = 1      # 最大嵌套深度

# 自定义角色映射
[agents.agent_roles]
explorer = "explorer"
worker = "default"
```

## 使用流程图

```
┌─────────────────┐
│   主 Agent      │
│  (用户会话)     │
└────────┬────────┘
         │ spawn_agent()
         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   子 Agent 1    │    │   子 Agent 2    │    │   子 Agent N    │
│  (并行执行)     │    │  (并行执行)     │    │  (并行执行)     │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    wait() 等待所有Agent完成                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    收集结果并继续主流程                       │
└─────────────────────────────────────────────────────────────┘
```

## 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `codex-rs/core/src/features.rs` | Feature flag 定义 |
| `codex-rs/core/src/tools/handlers/multi_agents.rs` | 多Agent工具处理器 |
| `codex-rs/core/src/agent/control.rs` | Agent控制平面 |
| `codex-rs/core/src/agent/guards.rs` | 线程限制保护 |
| `codex-rs/core/src/agent/role.rs` | 角色配置系统 |
| `codex-rs/core/src/agent/builtins/explorer.toml` | Explorer角色定义 |
| `codex-rs/tui/src/multi_agents.rs` | TUI渲染组件 |
| `codex-rs/protocol/src/protocol.rs` | 协议事件定义 |

## 最佳实践

1. **合理设置线程限制**：根据系统资源和任务复杂度调整 `max_threads`
2. **控制嵌套深度**：默认深度为1，避免过度嵌套导致资源耗尽
3. **使用合适的角色**：探索任务使用 `explorer` 角色，节省成本
4. **处理超时**：`wait()` 工具支持超时，避免无限等待
5. **清理资源**：任务完成后使用 `close_agent()` 释放资源
