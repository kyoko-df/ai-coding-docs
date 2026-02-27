# Rollout Reconstruction 机制

### Rollout 是什么

Rollout 是会话记录文件，采用 JSONL 格式存储在 `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`。它记录了完整的对话历史，支持会话恢复、分叉和重放。

### 数据结构

RolloutItem 枚举定义了五种记录类型：

```rust
pub enum RolloutItem {
    SessionMeta(SessionMetaLine),      // 会话元数据
    ResponseItem(ResponseItem),        // 用户/助手消息
    Compacted(CompactedItem),          // 压缩摘要
    TurnContext(TurnContextItem),      // 转向上下文快照
    EventMsg(EventMsg),                // 系统事件
}
```

会话初始化有三种方式：

```rust
pub enum InitialHistory {
    New,                         // 新会话
    Resumed(ResumedHistory),     // 恢复已有会话
    Forked(Vec<RolloutItem>),    // 从其他会话分叉
}
```

### Reconstruction 触发场景

**Resume（恢复会话）**

用户继续之前的对话时，系统从 rollout 文件读取 `RolloutItem` 列表，调用 `reconstruct_history_from_rollout()` 在内存中重建对话历史。

**Fork（会话分叉）**

用户从某个会话的特定点创建新会话。系统复制截至该点的所有 rollout items，重建历史作为新会话的基础，然后立即持久化新会话文件。

**Compact（历史压缩）**

长对话通过压缩减少上下文长度。压缩后的历史需要在 reconstruction 时正确还原。

### 核心实现

```rust
async fn reconstruct_history_from_rollout(
    &self,
    turn_context: &TurnContext,
    rollout_items: &[RolloutItem],
) -> Vec<ResponseItem>
```

处理逻辑逐项遍历 rollout items：

- `ResponseItem` → 直接加入历史
- `Compacted` → 用压缩后的内容替换历史
- `EventMsg(ThreadRolledBack)` → 删除最后 N 个用户回合
- 其他类型 → 跳过

Resume 和 Fork 都调用这个函数重建历史。区别在于 Fork 重建后会添加新的初始上下文，并立即写入新文件；Resume 则延迟到第一个 turn 开始时才持久化。

### Compaction 处理

压缩时，模型生成长对话摘要，创建 `CompactedItem`：

```rust
pub struct CompactedItem {
    pub message: String,                              // 压缩摘要
    pub replacement_history: Option<Vec<ResponseItem>>, // 预计算历史
}
```

Reconstruction 时，如果 `replacement_history` 存在就直接使用；否则需要重新构建——收集用户消息，调用 `build_compacted_history()` 生成压缩后的历史。

### Rollout 文件示例

```jsonl
{"timestamp":"2025-01-01T00:00:00.000Z","type":"session_meta","payload":{"id":"uuid"}}
{"timestamp":"...","type":"response_item","item":{"type":"message","role":"user","content":[...]}}
{"timestamp":"...","type":"response_item","item":{"type":"message","role":"assistant","content":[...]}}
{"timestamp":"...","type":"event_msg","payload":{"type":"thread_rolled_back","num_turns":1}}
{"timestamp":"...","type":"compacted","item":{"message":"summary..."}}
```

### ContextManager 方法

- `record_items()` - 添加响应到历史，应用截断策略
- `replace()` - 完全替换历史（用于 compaction）
- `drop_last_n_user_turns()` - 删除最后 N 个用户回合（用于 rollback）

### 小结

Rollout Reconstruction 从 JSONL 文件重建对话历史，支持 resume、fork、compact 三种操作。核心是逐项处理 `RolloutItem` 序列，应用压缩和回滚等修改，最终生成模型可用的 `Vec<ResponseItem>`。
