# Grok Build 内循环实现解析

> 本文说明 `grok` 的 Agent 内循环：一次用户输入如何驱动「采样 → 工具执行 → 结果回填 → 再采样」直到本轮结束。
> 代码集中在 `crates/codegen/xai-grok-shell/src/session/acp_session_impl/` 与 `crates/codegen/xai-grok-sampler/`。

## 1. 三层循环总览

内循环不是一个 `loop`，而是三层嵌套循环 + 若干旁路注入机制：

```
┌─ L0 会话命令循环 run_session ─────────────────────────────────────┐
│  tokio::select! 派发 SessionCommand / SessionEvent / 定时器        │
│  Prompt → 入队 pending_inputs → maybe_start_running_task          │
│         → spawn turn task（run_task → handle_turn_input）          │
│  completion_rx ← TurnCompletionMsg → handle_completion → 队列推进  │
│                                                                  │
│  └─ L1 轮次（turn）外层循环 handle_turn_input_inner ────────────┐  │
│    loop {                                                      │  │
│      process_conversation_turn_with_recovery()  ── 一轮完整对话  │  │
│      ├─ completion_requirement 未满足 → 注入恢复提示 → 重跑      │  │
│      ├─ goal 模式 → run_goal_round_end → Continue(directive)     │  │
│      │           → push_user_message(goal directive) → continue  │  │
│      └─ stop gate → Stop hooks 判 KeepWorking → 注入反馈 → 续跑  │  │
│    }                                                           │  │
│    │                                                           │  │
│    └─ L2 采样-工具内循环 process_conversation_turn_inner ──────┐ │  │
│      loop {                                                   │ │  │
│        loop_index += 1                                        │ │  │
│        ① 停滞检测 / interjection drain / reminder 注入          │ │  │
│        ② 压缩预触发 + auto-compact 检查                          │ │  │
│        ③ build_request（chat-state 组装请求）                    │ │  │
│        ④ run_turn_via_sampler → 模型流式采样                    │ │  │
│        ⑤ 有 tool_calls → execute_tool_calls → 结果回填 → 续循环  │ │  │
│           无 tool_calls → （TodoGate/interject 再确认）→ 结束     │ │  │
│      }                                                        │ │  │
│    └──────────────────────────────────────────────────────────┘ │  │
└──────────────────────────────────────────────────────────────────┘
```

| 层 | 代码位置 | 粒度 |
|----|----------|------|
| L0 命令循环 | `acp_session_impl/run_loop.rs` `run_session` | 会话生命周期，单线程 LocalSet |
| L1 轮次外层 | `acp_session_impl/turn.rs` `handle_turn_input_inner` | 一次用户 prompt 的完整生命周期 |
| L2 采样-工具循环 | `acp_session_impl/turn.rs` `process_conversation_turn_inner` | 一次模型往返（一轮 sample + 一批 tool calls） |
| 请求级重试 | `xai-grok-sampler/src/actor/request_task.rs` `run_request_task` | 单次 HTTP 请求的传输/doom-loop 重试 |

## 2. L0：会话命令循环与轮次调度

`run_session`（`run_loop.rs`）是 `SessionActor` 的主循环，`tokio::select!` 在四类来源间多路复用：

- `cmd_rx`：全部 `SessionCommand`（Prompt / Cancel / Compact / SetModel / 队列操作 …）
- `chat_state_event_rx`：ChatStateActor 事件（`ConversationReset`、图片预算等）
- `event_rx`：`SessionEvent`（FlushReplay 定时器、内部信号）
- `completion_rx`：轮次任务完成通知 `TurnCompletionMsg`

调度规则：

1. `SessionCommand::Prompt` → `queue_input()` 把 `InputItem` 推进 `state.pending_inputs`（`prompt_queue.rs`）；`send_now` 会先 `cancel_turn_for_send_now` 取消在跑的轮次。
2. `maybe_start_running_task`（`notification_drain.rs`）做队首提升：丢过期 synthetic 行、跳过 edit-hold 行、可选合并连续 prompt，然后 `AgentTask::new_prompt` spawn 出轮次 task。同一时刻只有一个 `running_task`。
3. 轮次 task 跑 `run_task` → `handle_turn_input`，结束后把 `TurnCompletionMsg` 发回 `completion_tx`。
4. 命令循环收到 completion → `handle_completion`（记账、stop reason）→ `handle_turn_end`（goal 续跑排队 / 连续失败退避暂停）→ `flush_stranded_interjections` → 再 `maybe_start_running_task` 取下一条 → 空闲则 `emit_session_idle_if_idle`。

取消路径（`cancel.rs`）：用户 Esc / `SessionCommand::Cancel` → `cancel_running_task` → `abort_turn_task` 直接 `JoinHandle::abort()` 掉轮次 future；`turn_stream_drained` 里的 request 所有权登记配合 `TurnReport`/`FinalizationGate` 保证「取消后迟到的事件/结果不会误记到下一个轮次」。

## 3. L1：轮次外层循环（`handle_turn_input_inner`）

`turn.rs` 中 `handle_turn_input` → `handle_turn_input_inner` 负责一个 prompt 的完整生命周期：

1. **前置**：计 turn span、打开 subagent spawn admission、写盘检查、direct-bash 短路（`!cmd` 不走模型）。
2. **用户消息提交**：按 `PromptOrigin` 构造不同 `ConversationItem`（User / TaskCompleted / SubagentCompleted / GoalSummary / SchedulerFired …），`push_user_message_and_ack` 提交到 ChatStateActor 并落盘（`mark_front_message_committed`）。
3. **外层 `loop`**（turn.rs ~1312 起）：每迭代跑一轮完整的 `process_conversation_turn_with_recovery`，然后根据结果决定是否续跑：
   - 非 `Completed`（Cancelled / MaxTurnsReached / Err）→ 直接 break；
   - `Completed + Refusal` → goal 自动暂停后 break；
   - **goal 模式**：`run_goal_round_end` 让 evaluator 判一轮 → `Continue(directive)` 就把 directive 作为 `goal_summary` 用户消息推回并 `continue`（这就是"目标驱动长任务"的外层续跑机制）；
   - **stop gate**：`run_stop_gate`（`stop_gate.rs`）派发 `Stop`/`SubagentStop` hooks——hook 返回 block 时把反馈文本作为合成 user 消息推回，`continue` 再跑一轮（上限 `MAX_STOP_HOOK_CONTINUATIONS_PER_TURN = 8`）；hook 的 `prevent_continuation` 可反向强制结束。
4. **收尾**：`finalize_turn_bookkeeping`、rewind point、usage 冻结、`TurnCompleted`/`AfterTurn` 事件、turn telemetry。

`process_conversation_turn_with_recovery`（turn.rs ~1976）再包一层 **completion-requirement 自动恢复**：agent 定义若要求必须调用某工具（如汇报类 subagent 的 `StructuredOutput`/汇报工具），本轮没调到就注入 `recovery_prompt` 并以指数退避重跑整个内循环，直到 `max_retries` 耗尽。

## 4. L2：采样-工具内循环（核心）

`process_conversation_turn_inner`（`turn.rs` ~2605）的 `loop`（~2737）是通常意义上的"agent 内循环"。`loop_index` 计轮，每次迭代 = 一次模型往返。

### 4.1 循环顶部的检查与注入

每次采样前先做一批" housekeeping "：

- **动作停滞检测**（`IdenticalToolCallRun`）：按 `step_signature`（参数 canonicalize 后的签名）统计连续相同的工具步。
  - 达到 nudge 阈值（普通 8 / Read·Plan 类 4）→ `push_system_reminder` 提醒模型换路子；
  - 达到硬停阈值（12 / 8 / true-noop 4）→ 返回 `TurnOutcome::StationarityEnded` 结束轮次。
- **interjection drain**：`drain_interjections_at_safe_point`（`interjection.rs`）把轮次中途到达的用户插队消息（`pending_interjections` 缓冲）以 `<user_query>` 信封包装推入历史；`follow_up_steer` 开启时还会把队列里可编辑的人类消息提升为 interjection。
- **reminder 注入**：首轮 memory reminder、MCP server 变更 delta/full reminder、长推理提醒（reasoning token 超阈值时提示模型收尾）、skill reminder。
- **压缩**：
  - 两阶段压缩**预触发**：估算 tokens 达到 `threshold - lead` 时后台跑 pass1（~95% 历史 → NOTE₁ 缓存），`should_prefire_two_pass` / `run_prefire_pass1`（`compaction.rs`、`two_pass.rs`）；
  - **auto-compact**：`check_auto_compact_needed` 达到阈值 → `run_compact_only`（two-pass 或单遍）；
  - 模型切换且窗口变小时 `maybe_compact_on_model_switch` 提前压。
- **工具面准备**：`prepare_tool_definitions_timed` 等 MCP 就绪；`turn_base_tool_specs` + subagent 场景经 `child_safe_tool_specs` 投影裁剪工具集；`--json-schema` 时追加 `StructuredOutput` 工具或 native schema。

### 4.2 组装请求

`chat_state_handle.build_request(...)`（`xai-chat-state/src/actor/request_builder.rs`）在 ChatStateActor 内组装 `ConversationRequest`：

- 会话历史已在写入侧做过完整性修复（dangling tool_call / dedup）；
- 超阈值时 `prune_conversation` 裁剪旧轮次（保护最近若干轮）；
- memory reminder 以 `<memory_context>` 段落 upsert 进 system prompt；
- 图片预算超限先驱逐/压缩图片。

### 4.3 采样：`run_turn_via_sampler`

`sampler_turn.rs` 把一次采样包成三层重试：

```
run_turn_via_sampler                          submit_turn_request                    sampler actor
─────────────────────                         ───────────────────                    ─────────────
rate-limit 等待预算                            sampling_gate 信号量                    run_request_task:
loop { submit → 429/5xx?                      → submit_and_collect_with_metadata      loop {
  budget.decide → sleep(backoff) 重试           （附带 per-request oneshot 完成通知）      run_one_attempt（SSE 流）
}                                             ↓                                       传输错误/空响应 → 重试分类器
错误 → handle_sampling_failure：               SamplingEvent 流经 event_tx             doom-loop 信号 → 独立预算
  ├─ 上下文溢出 → CompactAndResubmit           → drainer 边收边推 ACP 增量给 UI          resample（append_recovery_context）
  ├─ 401 → RefreshAuthAndResubmit              ↓                                       }
  ├─ transient → RetryTransient（退避续循环）   Ok(Response) → 返回内循环
  └─ 其余 → Err 结束轮次
```

要点：

- **流式与结果解耦**：`SamplerHandle::submit_and_collect_with_metadata` 提交后等 oneshot 拿最终结果；同时所有 `SamplingEvent`（text/thinking/tool-delta chunk）经共享 `event_tx` 由 spawn 时挂的 drainer task（`spawn.rs` ~2087 → `handle_sampling_event`）实时转成 ACP `SessionUpdate` 推给客户端——UI 边出字，循环只看终态。
- **doom-loop**：后端在 SSE 里下发 `doom_loop_check` 信号，`DoomLoopSignalCollector` 在流中实时累积，置信信号可**中止当前流**并触发独立预算的重采样（`doom_loop_recovery.rs`，把失败响应保留 + 追加指导重新采样）。
- **rate-limit 等待预算**：`RateLimitWaitBudget` 允许在预算内循环 sleep+重试而不杀轮次；耗尽才进 `handle_sampling_failure`。
- **取消即撤销**：request 完成后先过 `turn_stream_drained` 屏障——若期间轮次被取消/rewind，已收的结果判 `revoked` 丢弃，不污染新轮次。

### 4.4 响应处理与长度抢救

拿到 `ConversationResponse` 后：

- 记账（usage/TTFT/TPS/telemetry），`record_response_items` 把 assistant item（含 tool_calls）写入历史；
- **media-gen 超帽**：单步媒体生成调用超上限 2x → 丢弃整次响应，`push_system_reminder` 后 resample（`MAX_MEDIA_GEN_OVER_CAP_RESAMPLES = 1`）；
- **length salvage**（`length_salvage.rs`）：`stop_reason = Length` 且无 tool_calls → `SalvageStep::Continue` 注入"继续写"reminder 并 `continue`（在 continue 态下请求带 `LengthPolicy::CompletePartial`，把截断输出续写拼接）；预算耗尽则以 `CompletedStop::MaxTokens` 结束。带 tool_calls 的 Length 走 `LengthSalvageStreak` 防连续截断死循环；
- content-filter 拒答 → 发说明 chunk，按 `Refusal` 收尾。

### 4.5 工具执行：`execute_tool_calls`

`tool_calls.rs`：

1. **准备阶段**（串行）：每个 call 过 `prepare_tool_call`——
   - 解析参数（容忍拼接 JSON、MCP `use_tool` 包装、MCP 文件输入）；
   - `PreToolUse` hooks：可改写参数（`HookRewrite`）、deny（`HookDenied`，把理由喂回模型继续跑）、defer 或转人工；
   - **权限门**：按 `AccessKind`（Edit/Read/Bash/MCP/WebFetch…）走权限策略——yolo 全自动、auto 走分类器、ask 发 `permission` 反向请求等用户；拒绝 → `PermissionReject`（整轮 Cancelled），用户在权限弹窗里改输入 → `FollowupMessage`（消息转新 user 轮次）；
   - plan mode 下的编辑门禁、media-gen 批限额（超出的 call 直接回 tool_result 拒执）。
2. **派发阶段**（并行）：approved 集合先按 `lock_path_for_args` 计算目标文件锁（同一文件的写串行），`FuturesUnordered` 并发跑 `dispatch_observed` → `WorkspaceOps::call_tool_with_context` → `toolset().call_with_context` → `xai-tool-runtime` 的 `ToolDispatch`/`Tool::execute` 流（本地工具与 MCP 工具同一注册表，本地同名优先）。可中断的 wait 类工具（`wait_tasks` 等）外层套 `select!`，父级 steer/用户插话可提前打断。
3. **结果收尾**（串行 drain `dispatch_rx`）：每个结果经 PostToolUse hook / PostToolUseFailure hook，成功走 `handle_bridge_tool_success` → 渲染 `prompt_text` → **`chat_state_handle.push_tool_result(ConversationItem::tool_result(call_id, text))`**——这就是"工具结果回填对话"的那一步；失败/未执行走 `handle_tool_error`/`handle_tool_not_executed`，同样落成 tool_result（错误信息也进上下文，供模型自我修正）。

返回 `ToolLoop::Continue` 后，内循环检查 `max_turns` 上限和 `check_preflight_overflow`（工具输出把估算 tokens 推过窗口就先压缩），然后 `continue` 进入下一次采样。

### 4.6 退出条件（`TurnOutcome`）

| 出口 | 触发 |
|------|------|
| `Completed{EndTurn}` | 响应无 tool_calls：先过 TodoGate（有未完成 todo → nudge 续跑，封顶后放行）、interjection/迟到消息 drain 确认无误后才落地 |
| `Completed{MaxTokens}` | length salvage 预算耗尽 / parked 续写空响应 |
| `Completed{Refusal}` | content filter |
| `Completed{structured_output}` | `StructuredOutput` 工具调用成功（native schema 校验后收尾） |
| `Cancelled` | PermissionReject / 用户取消 / HookDenied 链路（注意 HookDenied 本身不终止，只有用户取消权限弹窗才终止） |
| `MaxTurnsReached{limit}` | `tool_turn_count` 超 `max_turns` |
| `StationarityEnded` | 相同工具签名连跑硬停阈值 |
| `Err` | 不可恢复的采样/落盘/权限错误 |

## 5. 关键并发与一致性设计

- **单线程 actor + spawn_local**：`SessionActor` 是 `!Send`，全部会话状态在 LocalSet 上串行演化；轮次 task、压缩预触发、drainer 都以 `spawn_local` 挂进同一线程，靠 `state` mutex + `FinalizationGate`/`TurnReport`/`turn_stream_drained` 处理取消竞态。
- **队列化输入**：人类 prompt、subagent/任务完成唤醒、goal continuation、scheduler、notification drain 全部统一进 `pending_inputs` 队列，由 origin 决定 `ConversationItem` 类型与注入策略；synthetic 行有过期丢弃逻辑。
- **中途转向**：interjection（用户边跑边追加指令）、send-now（取消重发）、permission 弹窗内改写（`FollowupMessage`）、Stop hook 强制续跑，都通过"在安全点 drain → 推 user 消息 → `continue`"的方式注入内循环，不需要打断采样 HTTP 流。
- **每个 continue 都是新一轮 build_request**：所有注入（tool_result、reminder、interjection）先写进 ChatStateActor 的历史，下一轮采样重建请求自然带上——上下文永远以持久化历史为唯一事实源。

## 6. 相关文件速查

| 主题 | 文件 |
|------|------|
| 命令循环/调度 | `xai-grok-shell/src/session/acp_session_impl/run_loop.rs` |
| 队列与提升 | `…/prompt_queue.rs`、`…/notification_drain.rs`（`maybe_start_running_task`） |
| 轮次外层 + 内循环本体 | `…/turn.rs`（`handle_turn_input_inner`、`process_conversation_turn_inner`） |
| 采样调用链 | `…/sampler_turn.rs`、`xai-grok-sampler/src/{handle.rs, actor/request_task.rs, doom_loop*.rs}` |
| 工具准备/权限/执行 | `…/tool_calls.rs`、`…/tool_dispatch.rs`、`xai-grok-workspace/src/workspace_ops.rs`、`xai-tool-runtime` |
| 结果落历史 | `xai-chat-state/src/{handle.rs, actor/request_builder.rs, actor/mutations.rs}` |
| 循环保护 | 停滞检测（`turn.rs` `IdenticalToolCallRun`）、length salvage（`length_salvage.rs`）、stop gate（`stop_gate.rs`）、todo gate（`turn.rs` `evaluate_todo_gate`）、completion recovery（`turn.rs` `process_conversation_turn_with_recovery`） |
| 目标驱动外循环 | `…/goal.rs`（`run_goal_round_end`、`maybe_queue_goal_continuation`）、`session/goal_*.rs` |
| 压缩 | `session/compaction.rs`、`session/two_pass.rs`、`xai-grok-compaction` |
| 取消/插队 | `…/cancel.rs`、`…/interjection.rs` |
