# Codex 多 Agent 功能指南

本文档介绍如何在 Codex 中启用和使用多 Agent（Multi-Agent）功能。

---

## 目录

- [功能概述](#功能概述)
- [启用方法](#启用方法)
- [配置选项](#配置选项)
- [可用工具](#可用工具)
- [内置角色](#内置角色)
- [使用示例](#使用示例)
- [保护机制](#保护机制)
- [注意事项](#注意事项)

---

## 功能概述

多 Agent 功能允许 Codex 创建多个子 agent 并行处理任务，提高工作效率。主要特性：

- **并行执行**：多个 agent 同时处理独立任务
- **角色分工**：不同类型的 agent 专注不同领域
- **层级控制**：防止无限嵌套的资源保护机制
- **状态追踪**：实时监控 agent 运行状态

---

## 启用方法

### 方法一：`/experimental` 菜单（推荐）

在 Codex CLI 中执行：

```
/experimental
```

选择 **"Multi-agents"** 选项：

```
┌─────────────────────────────────────────────────────────┐
│ Multi-agents                                             │
│ Ask Codex to spawn multiple agents to parallelize       │
│ the work and win in efficiency.                         │
└─────────────────────────────────────────────────────────┘
```

重启 Codex 后生效。

### 方法二：配置文件

编辑 `~/.codex/config.toml`：

```toml
[features]
multi_agent = true
```

> 配置键 `collab` 是 `multi_agent` 的别名，两者等效。

### 方法三：命令行参数

```bash
codex --enable multi_agent
```

---

## 配置选项

在 `~/.codex/config.toml` 中配置 Agent 参数：

```toml
[features]
multi_agent = true

[agents]
# 最大并发 agent 线程数（默认: 6）
max_threads = 6

# 最大嵌套深度（默认: 1，子 agent 无法再创建孙 agent）
max_depth = 1

# 自定义角色
[agents.researcher]
description = "研究专注型 agent"
config_file = "./agents/researcher.toml"

[agents.tester]
description = "测试专用 agent"
config_file = "./agents/tester.toml"
```

### 配置验证规则

| 配置项 | 约束 |
|--------|------|
| `max_threads` | 必须 >= 1 |
| `max_depth` | 必须 >= 1 |
| `config_file` | 必须指向存在的文件 |

---

## 可用工具

启用多 Agent 功能后，以下工具会注册到模型：

### `spawn_agent` - 创建新 Agent

```typescript
// 参数
{
  message?: string,      // 发送给新 agent 的文本消息
  items?: UserInput[],   // 结构化输入（与 message 二选一）
  agent_type?: string    // 角色类型
}

// 返回
{ agent_id: string }  // 新 agent 的唯一标识
```

**示例**：
```json
{
  "message": "分析 src 目录的代码结构",
  "agent_type": "explorer"
}
```

### `send_input` - 向 Agent 发送消息

```typescript
{
  id: string,           // agent ID
  message?: string,     // 文本消息
  items?: UserInput[],  // 结构化输入
  interrupt?: boolean   // 是否先中断当前任务（默认 false）
}
```

**示例**：
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "继续分析 tests 目录",
  "interrupt": true
}
```

### `resume_agent` - 恢复已关闭的 Agent

```typescript
{
  id: string  // 已关闭 agent 的 ID
}
```

从 rollout 文件恢复之前关闭的 agent，保留历史上下文。

### `wait` - 等待 Agent 完成

```typescript
{
  ids: string[],       // agent ID 列表
  timeout_ms?: number  // 超时时间（10s-300s，默认 30s）
}

// 返回
{
  status: HashMap<ThreadId, AgentStatus>,
  timed_out: boolean
}
```

**超时范围**：
- 最小：10 秒
- 最大：300 秒
- 默认：30 秒

### `close_agent` - 关闭 Agent

```typescript
{
  id: string  // agent ID
}

// 返回
{ status: AgentStatus }  // 关闭前的状态
```

---

## 内置角色

| 角色 | 描述 | 特殊配置 |
|------|------|----------|
| **default** | 默认 agent | 无特殊配置 |
| **explorer** | 代码库探索专家 | `gpt-5.1-codex-mini` 模型，中等推理强度 |
| **worker** | 执行和生产工作 | 无特殊配置 |

### explorer 角色详情

```toml
# 内置配置
model = "gpt-5.1-codex-mini"
model_reasoning_effort = "medium"
```

使用场景：
- 代码库结构分析
- 文件搜索和定位
- 快速问答

**最佳实践**：
```
- 优先使用 explorer 查询代码库问题
- 信任 explorer 结果，无需重复验证
- 相关问题复用同一 explorer
- 并行运行多个 explorer 提高效率
```

### 自定义角色

创建自定义角色配置文件：

```toml
# ~/.codex/agents/researcher.toml
model = "gpt-5-codex"
model_reasoning_effort = "high"

# 可覆盖其他配置
```

在 `config.toml` 中注册：

```toml
[agents.researcher]
description = "深度研究 agent，适合复杂分析"
config_file = "./agents/researcher.toml"
```

---

## 使用示例

### 场景一：并行代码分析

```
用户: 分析这个项目的架构，包括核心模块、测试和文档

模型:
1. spawn_agent("分析 src/core 架构", agent_type="explorer")
   → agent_id: "agent-1"

2. spawn_agent("分析测试覆盖情况", agent_type="explorer")
   → agent_id: "agent-2"

3. spawn_agent("检查文档完整性", agent_type="explorer")
   → agent_id: "agent-3"

4. wait([agent-1, agent-2, agent-3], timeout_ms=60000)

5. close_agent(agent-1)
   close_agent(agent-2)
   close_agent(agent-3)
```

### 场景二：任务分发执行

```
用户: 实现 API 端点，包括用户认证、数据验证和错误处理

模型:
1. spawn_agent("实现用户认证模块", agent_type="worker")
   → agent_id: "worker-1"

2. spawn_agent("实现数据验证逻辑", agent_type="worker")
   → agent_id: "worker-2"

3. wait([worker-1, worker-2])

4. 发送后续指令：
   send_input(worker-1, {message: "添加单元测试"})

5. 最终清理：
   close_agent(worker-1)
   close_agent(worker-2)
```

### 场景三：恢复中断的任务

```
# 之前的 agent 已关闭
模型:
resume_agent("agent-xxx")
→ 状态: {status: "PendingInit"}

send_input("agent-xxx", {message: "继续之前的工作"})
```

---

## 保护机制

### 线程数量限制

- 默认最多 **6 个**并发 agent
- 达到限制时返回 `AgentLimitReached` 错误
- 使用原子操作确保线程安全

```
用户会话
├── agent-1 (活跃)
├── agent-2 (活跃)
├── agent-3 (活跃)
├── agent-4 (活跃)
├── agent-5 (活跃)
├── agent-6 (活跃) ← 达到上限
└── spawn_agent → ERROR: AgentLimitReached
```

### 嵌套深度限制

```
主 agent (depth=0)
└── 子 agent (depth=1)  ← 默认 max_depth=1
    └── spawn_agent → 禁止（Collab 功能自动禁用）
```

**深度控制逻辑**：
- 子 agent 的审批策略自动设为 `Never`
- 接近深度限制时禁用 Collab 功能
- 防止资源耗尽和无限递归

### Agent 状态

| 状态 | 说明 |
|------|------|
| `PendingInit` | 初始化中 |
| `Running` | 运行中 |
| `Completed` | 正常完成 |
| `Errored` | 错误终止 |
| `Shutdown` | 已关闭 |
| `NotFound` | 不存在/已清理 |

---

## 注意事项

### 实验性功能

多 Agent 功能目前处于 **Experimental** 阶段：
- 功能可能不稳定
- API 可能发生变化
- 不建议用于生产环境

### 资源消耗

- 每个 agent 独立消耗 API 配额
- 并发 agent 过多可能影响响应速度
- 建议根据任务复杂度调整 `max_threads`

### 调试建议

启用详细日志：
```bash
RUST_LOG=codex::agent=debug codex
```

查看 agent 状态：
```
/status
```

### 已知限制

1. 子 agent 无法创建子 agent（单层嵌套）
2. 跨 agent 的上下文不共享
3. agent 间通信通过消息传递，非共享内存

---

## 故障排除

### 问题：spawn_agent 返回 "Agent depth limit reached"

**原因**：当前 agent 已达到最大嵌套深度

**解决**：
- 在主 agent 中创建子 agent
- 或增加 `max_depth` 配置值

### 问题：spawn_agent 返回 "AgentLimitReached"

**原因**：并发 agent 数量达到上限

**解决**：
- 先 `close_agent` 释放资源
- 或增加 `max_threads` 配置值

### 问题：agent 状态一直是 "Running"

**原因**：任务长时间执行或卡死

**解决**：
- 使用 `send_input` 配合 `interrupt: true` 中断
- 使用 `close_agent` 强制关闭

---

## 参考资料

- 源码位置：`codex-rs/core/src/tools/handlers/multi_agents.rs`
- 配置定义：`codex-rs/core/src/config/mod.rs`
- Feature 定义：`codex-rs/core/src/features.rs`
- Agent 控制：`codex-rs/core/src/agent/control.rs`

---

*文档生成日期：2026-02-20*
