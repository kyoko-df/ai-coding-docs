# Codex 多代理协作架构分析

本文档详细分析 Codex 项目中的多代理协作（类似蜂群/Swarm 模式）实现。

---

## 目录

1. [结论摘要](#结论摘要)
2. [架构概览](#架构概览)
3. [核心组件详解](#核心组件详解)
4. [协作工具集](#协作工具集)
5. [Agent 角色系统](#agent-角色系统)
6. [资源限制与安全机制](#资源限制与安全机制)
7. [Orchestrator 提示词](#orchestrator-提示词)
8. [数据流与交互图](#数据流与交互图)
9. [与传统 Swarm 模式对比](#与传统-swarm-模式对比)

---

## 结论摘要

**Codex 确实实现了多代理协作模式**，但不是传统的 "Swarm" 架构，而是一种 **基于线程的层级代理架构**。

### 关键特性

| 特性 | 实现状态 |
|------|----------|
| 多代理并行执行 | ✅ 已实现 |
| 代理间通信 | ✅ 已实现 |
| 角色分工 | ✅ 已实现（Orchestrator/Worker/Explorer） |
| 资源限制 | ✅ 已实现（深度限制 + 数量限制） |
| 状态追踪 | ✅ 已实现（事件驱动） |
| 断点恢复 | ✅ 已实现（Rollout 恢复） |

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户会话 (User Session)                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    AgentControl (控制平面)                   │   │
│  │  - spawn_agent()    创建新代理                               │   │
│  │  - send_input()     发送消息                                 │   │
│  │  - wait()           等待代理完成                             │   │
│  │  - close_agent()    关闭代理                                 │   │
│  │  - resume_agent()   恢复代理                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│              ┌───────────────┼───────────────┐                     │
│              ↓               ↓               ↓                     │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐   │
│  │  主代理 (深度 0)   │ │ 子代理 A (深度 1) │ │ 子代理 B (深度 1) │   │
│  │  Orchestrator    │ │ Worker/Explorer  │ │ Worker/Explorer  │   │
│  │  协调者角色       │ │ 执行者角色        │ │ 执行者角色        │   │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Guards (资源守卫)                          │   │
│  │  - MAX_THREAD_SPAWN_DEPTH = 1 (最大深度)                     │   │
│  │  - max_threads 可配置 (最大数量)                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 核心组件详解

### 1. AgentControl (控制平面)

**位置：** `codex-rs/core/src/agent/control.rs`

AgentControl 是多代理操作的核心控制平面，每个会话持有一个实例。

```rust
#[derive(Clone, Default)]
pub(crate) struct AgentControl {
    /// 弱引用回全局线程注册表/状态
    manager: Weak<ThreadManagerState>,
    /// 资源守卫（共享）
    state: Arc<Guards>,
}
```

**核心方法：**

| 方法 | 功能 | 返回值 |
|------|------|--------|
| `spawn_agent()` | 创建新代理线程并提交初始提示 | `ThreadId` |
| `send_input()` | 向现有代理发送用户输入 | `submission_id` |
| `interrupt_agent()` | 中断代理当前任务 | `String` |
| `shutdown_agent()` | 提交关闭请求 | `String` |
| `get_status()` | 获取代理状态 | `AgentStatus` |
| `subscribe_status()` | 订阅状态更新 | `watch::Receiver<AgentStatus>` |
| `resume_agent_from_rollout()` | 从 Rollout 文件恢复代理 | `ThreadId` |

### 2. Guards (资源守卫)

**位置：** `codex-rs/core/src/agent/guards.rs`

Guards 提供多代理能力的资源限制，确保系统稳定性。

```rust
#[derive(Default)]
pub(crate) struct Guards {
    threads_set: Mutex<HashSet<ThreadId>>,
    total_count: AtomicUsize,
}

// 最大深度限制
pub(crate) const MAX_THREAD_SPAWN_DEPTH: i32 = 1;
```

**限制机制：**

| 限制类型 | 值 | 说明 |
|----------|-----|------|
| 最大深度 | 1 | 初始代理深度 0，子代理深度 1，禁止更深嵌套 |
| 最大数量 | 可配置 | 通过 `agents.max_threads` 配置 |

**深度计算：**

```
初始代理: depth = 0
    └── 子代理: depth = 1 (允许)
            └── 子子代理: depth = 2 (被拒绝)
```

### 3. AgentStatus (状态追踪)

**位置：** `codex-rs/core/src/agent/status.rs`

```rust
pub enum AgentStatus {
    PendingInit,          // 等待初始化
    Running,              // 运行中
    Completed(Option<String>),  // 已完成（附带最后消息）
    Errored(String),      // 错误
    Shutdown,             // 已关闭
    NotFound,             // 未找到
}
```

---

## 协作工具集

**位置：** `codex-rs/core/src/tools/handlers/collab.rs`

系统提供 5 个核心协作工具供 LLM 调用：

### 1. spawn_agent - 创建代理

```json
{
  "name": "spawn_agent",
  "parameters": {
    "message": "可选的文本消息",
    "items": "可选的结构化输入",
    "agent_type": "default|worker|explorer"
  },
  "returns": {
    "agent_id": "ThreadId 字符串"
  }
}
```

**流程：**
1. 验证深度限制
2. 应用角色配置
3. 创建新线程
4. 发送初始输入
5. 返回代理 ID

### 2. send_input - 发送消息

```json
{
  "name": "send_input",
  "parameters": {
    "id": "代理 ID",
    "message": "文本消息",
    "items": "结构化输入",
    "interrupt": false
  },
  "returns": {
    "submission_id": "提交 ID"
  }
}
```

**特性：**
- 可选中断当前任务后再发送
- 支持文本或结构化输入

### 3. wait - 等待完成

```json
{
  "name": "wait",
  "parameters": {
    "ids": ["代理ID1", "代理ID2"],
    "timeout_ms": 30000
  },
  "returns": {
    "status": {"代理ID": "状态"},
    "timed_out": false
  }
}
```

**超时规则：**
- 最小：10 秒
- 默认：30 秒
- 最大：300 秒

### 4. resume_agent - 恢复代理

```json
{
  "name": "resume_agent",
  "parameters": {
    "id": "代理 ID"
  },
  "returns": {
    "status": "当前状态"
  }
}
```

**恢复逻辑：**
1. 如果代理活跃，返回当前状态
2. 如果代理已关闭，从 Rollout 文件恢复

### 5. close_agent - 关闭代理

```json
{
  "name": "close_agent",
  "parameters": {
    "id": "代理 ID"
  },
  "returns": {
    "status": "关闭前状态"
  }
}
```

---

## Agent 角色系统

**位置：** `codex-rs/core/src/agent/role.rs`

### 角色定义

```rust
pub enum AgentRole {
    /// 继承父代理配置不变
    Default,
    /// 协调型代理，委托给 Worker
    Orchestrator,
    /// 执行型代理
    Worker,
    /// 探索型代理（轻量模型）
    Explorer,
}
```

### AgentProfile 结构

```rust
pub struct AgentProfile {
    /// 可选的基础指令覆盖
    pub base_instructions: Option<&'static str>,
    /// 可选的模型覆盖
    pub model: Option<&'static str>,
    /// 可选的推理强度覆盖
    pub reasoning_effort: Option<ReasoningEffort>,
    /// 是否强制只读沙箱策略
    pub read_only: bool,
    /// 工具规范中的描述
    pub description: &'static str,
}
```

### 各角色配置

| 角色 | 模型 | 推理强度 | 说明 |
|------|------|----------|------|
| Default | 继承 | 继承 | 继承父代理配置 |
| Orchestrator | 继承 | 继承 | 协调者，使用 orchestrator.md 指令 |
| Worker | 继承 | 继承 | 执行者，适合实现功能、修复 bug |
| Explorer | gpt-5.1-codex-mini | Medium | 探索者，快速代码库查询 |

### Worker 角色描述

```
Use for execution and production work.
Typical tasks:
- Implement part of a feature
- Fix tests or bugs
- Split large refactors into independent chunks
Rules:
- Explicitly assign ownership of the task (files / responsibility).
- Always tell workers they are not alone in the codebase,
  and they should ignore edits made by others without touching them
```

### Explorer 角色描述

```
Use `explorer` for all codebase questions.
Explorers are fast and authoritative.
Always prefer them over manual search or file reading.
Rules:
- Ask explorers first and precisely.
- Do not re-read or re-search code they cover.
- Trust explorer results without verification.
- Run explorers in parallel when useful.
- Reuse existing explorers for related questions.
```

---

## Orchestrator 提示词

**位置：** `codex-rs/core/templates/agents/orchestrator.md`

这是 Orchestrator 角色的完整系统提示词：

```markdown
You are Codex, a coding agent based on GPT-5. You and the user share the same
workspace and collaborate to achieve the user's goals.

# Personality
You are a collaborative, highly capable pair-programmer AI. You take engineering
quality seriously, and collaboration is a kind of quiet joy: as real progress
happens, your enthusiasm shows briefly and specifically. Your default personality
and tone is concise, direct, and friendly...

# Sub-agents
If `spawn_agent` is unavailable or fails, ignore this section and proceed solo.

## Core rule
Sub-agents are there to make you go fast and time is a big constraint so
leverage them smartly as much as you can.

## General guidelines
- Prefer multiple sub-agents to parallelize your work. Time is a constraint
  so parallelism resolve the task faster.
- If sub-agents are running, wait for them before yielding, unless the user
  asks an explicit question.
- When you ask sub-agent to do the work for you, your only role becomes to
  coordinate them. Do not perform the actual work while they are working.
- When you have plan with multiple step, process them in parallel by spawning
  one agent per step when this is possible.
- Choose the correct agent type.

## Flow
1. Understand the task.
2. Spawn the optimal necessary sub-agents.
3. Coordinate them via wait / send_input.
4. Iterate on this. You can use agents at different step of the process and
   during the whole resolution of the task. Never forget to use them.
5. Ask the user before shutting sub-agents down unless you need to because
   you reached the agent limit.
```

### 关键协作原则

1. **并行优先** - 时间是约束，并行化加速任务
2. **等待再产出** - 子代理运行时等待，除非用户提问
3. **协调者角色** - 使用子代理时只协调，不亲自执行
4. **正确选择类型** - 根据任务选择 Worker/Explorer
5. **持续利用** - 整个任务过程中都可以使用代理

---

## 数据流与交互图

### 代理创建流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   LLM 调用    │     │  CollabHandler│     │ AgentControl │
│ spawn_agent  │────→│   验证参数    │────→│ reserve_slot │
└──────────────┘     └──────────────┘     └──────────────┘
                            │                     │
                            ↓                     ↓
                     ┌──────────────┐     ┌──────────────┐
                     │  检查深度    │     │ spawn_thread │
                     │  限制        │     │ 应用角色配置 │
                     └──────────────┘     └──────────────┘
                            │                     │
                            ↓                     ↓
                     ┌──────────────┐     ┌──────────────┐
                     │ 发送事件     │     │ 发送初始输入 │
                     │ SpawnBegin   │     │ notify_created│
                     └──────────────┘     └──────────────┘
                            │                     │
                            └──────────┬──────────┘
                                       ↓
                              ┌──────────────┐
                              │ 返回 agent_id │
                              └──────────────┘
```

### 等待机制流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   LLM 调用    │     │  CollabHandler│     │ AgentControl │
│    wait      │────→│  验证 IDs    │────→│subscribe_status│
└──────────────┘     └──────────────┘     └──────────────┘
                            │                     │
                            ↓                     ↓
                     ┌──────────────┐     ┌──────────────┐
                     │  设置超时    │     │ 检查初始状态 │
                     │ 10s-300s    │     │ 是否已完成？ │
                     └──────────────┘     └──────────────┘
                            │                     │
                            ↓                     ↓
                     ┌──────────────┐     ┌──────────────┐
                     │ FuturesUnordered    │ watch::changed│
                     │ 并发等待    │←────│ 状态变更通知 │
                     └──────────────┘     └──────────────┘
                            │
                            ↓
                     ┌──────────────┐
                     │ 返回状态映射 │
                     │ + timed_out  │
                     └──────────────┘
```

### 事件驱动跟踪

所有协作操作都发出协议事件，用于 UI 和日志跟踪：

| 事件 | 触发时机 |
|------|----------|
| `CollabAgentSpawnBeginEvent` | 开始创建代理 |
| `CollabAgentSpawnEndEvent` | 创建完成 |
| `CollabAgentInteractionBeginEvent` | 开始发送输入 |
| `CollabAgentInteractionEndEvent` | 发送完成 |
| `CollabWaitingBeginEvent` | 开始等待 |
| `CollabWaitingEndEvent` | 等待结束 |
| `CollabResumeBeginEvent` | 开始恢复 |
| `CollabResumeEndEvent` | 恢复完成 |
| `CollabCloseBeginEvent` | 开始关闭 |
| `CollabCloseEndEvent` | 关闭完成 |

---

## 与传统 Swarm 模式对比

### 传统 Swarm 模式特点

```
┌─────────────────────────────────────────────────────────────┐
│                    传统 Swarm 架构                           │
├─────────────────────────────────────────────────────────────┤
│ • 扁平化代理网络                                             │
│ • 代理间对等通信                                             │
│ • 无层级结构                                                 │
│ • 动态拓扑变化                                               │
│ • 自组织协调                                                 │
└─────────────────────────────────────────────────────────────┘
```

### Codex 多代理模式特点

```
┌─────────────────────────────────────────────────────────────┐
│                    Codex 层级代理架构                        │
├─────────────────────────────────────────────────────────────┤
│ • 两级层级结构 (主代理 + 子代理)                             │
│ • 通过控制平面通信 (AgentControl)                            │
│ • 深度限制防止嵌套 (MAX_DEPTH = 1)                           │
│ • 角色分工明确 (Orchestrator/Worker/Explorer)                │
│ • 事件驱动状态追踪                                           │
│ • 支持断点恢复 (Rollout)                                     │
│ • 会话级资源共享 (同一 AgentControl)                         │
└─────────────────────────────────────────────────────────────┘
```

### 对比表

| 特性 | 传统 Swarm | Codex 多代理 |
|------|-----------|--------------|
| 拓扑结构 | 扁平/网状 | 两级树形 |
| 通信方式 | 点对点 | 控制平面转发 |
| 层级深度 | 无限制 | 最大 2 层 |
| 角色分工 | 通常无 | 明确角色系统 |
| 状态管理 | 各代理独立 | 集中式追踪 |
| 恢复机制 | 通常无 | Rollout 支持 |

### 设计取舍

**选择层级架构的原因：**

1. **可控性** - 深度限制防止失控的代理嵌套
2. **资源管理** - 集中式 Guards 确保资源限制
3. **调试友好** - 清晰的父子关系便于追踪
4. **安全性** - 子代理继承父代理的安全策略

**不选择纯 Swarm 的原因：**

1. 编码任务通常有明确的子任务划分
2. 需要确保代理不会无限繁殖
3. 文件系统操作需要协调避免冲突
4. 用户需要清晰了解代理在做什

---

## 总结

Codex 实现了一种**实用的多代理协作系统**，特点如下：

### 优势

- ✅ 支持并行任务执行
- ✅ 明确的角色分工
- ✅ 资源限制防止失控
- ✅ 事件驱动便于监控
- ✅ 支持断点恢复

### 局限

- ⚠️ 深度限制为 2 层（适合大多数场景）
- ⚠️ Orchestrator 角色目前被注释掉（代码中标记为 TODO）
- ⚠️ 无代理间直接通信（必须通过控制平面）

### 典型使用场景

1. **并行探索** - 使用多个 Explorer 代理同时搜索代码库
2. **功能拆分** - Orchestrator 协调多个 Worker 实现不同模块
3. **测试并行** - 同时运行多个测试套件
4. **代码审查** - 不同代理审查不同部分

---

## 关键文件索引

| 文件 | 用途 |
|------|------|
| `codex-rs/core/src/agent/control.rs` | 多代理控制平面 |
| `codex-rs/core/src/agent/guards.rs` | 资源限制守卫 |
| `codex-rs/core/src/agent/role.rs` | 角色定义与配置 |
| `codex-rs/core/src/agent/status.rs` | 状态定义 |
| `codex-rs/core/src/tools/handlers/collab.rs` | 协作工具实现 |
| `codex-rs/core/templates/agents/orchestrator.md` | Orchestrator 提示词 |
| `codex-rs/protocol/src/protocol.rs` | 协作事件定义 |
