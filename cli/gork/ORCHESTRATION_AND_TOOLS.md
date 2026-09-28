# Grok Build 编排体系与工具能力解析

> 本文说明 grok 的「编排」（orchestration：子代理、目标模式、工作流、调度器）与「工具能力」（tool surface：工具分类、注册表、MCP、终端、权限与沙箱）。
> 姊妹篇 `INNER_LOOP.md` 讲单轮采样-工具循环；本文讲循环之上的多轮/多代理协作与工具栈本身。

## 1. 编排总览

grok 的编排分两层：

| 层 | 性质 | 代表机制 |
|----|------|----------|
| **模型驱动的即兴编排** | 模型在轮次里自己调用工具来 fan-out | `task`（子代理）、`scheduler`（定时）、`monitor`（事件流）、`wait_tasks`/`send_subagent_message`（收束与转向） |
| **harness 驱动的结构化编排** | 框架在轮次间注入指令/校验，不依赖模型自觉 | `/goal` 目标模式（planner/evaluator/classifier-skeptic/strategist 角色代理）、workflow 引擎（Rhai 脚本）、hooks、stop gate、todo gate |

两层共用同一套「注入点」：所有编排产物（子代理回报、goal directive、定时器触发、监视事件）最终都作为一条 `ConversationItem`/用户消息进入 `pending_inputs` 队列或直接 `push_user_message`，下一次 `build_request` 时自然进入上下文——**编排 = 往对话历史里排队带元信息的输入**。

```
        ┌────────── 模型驱动编排 ──────────┐      ┌─────── harness 驱动编排 ───────┐
        │  task → 子代理                    │      │  /goal: planner → worker     │
        │  scheduler → 定时触发              │      │         → evaluator →        │
        │  monitor → 事件流唤醒             │      │         skeptic panel →      │
        │  wait_tasks → 收束                │      │         strategist（停滞时）  │
        │  send_subagent_message → 转向     │      │  workflow: Rhai 脚本驱动      │
        └──────────────┬───────────────────┘      └──────────────┬───────────────┘
                       └────────────┬────────────────────────────┘
                                    ▼
                   pending_inputs 队列 / push_user_message
                                    ▼
                   SessionActor 命令循环 → 轮次内循环（INNER_LOOP.md）
```

## 2. 子代理编排（Task / SubagentCoordinator）

### 2.1 两个世界的分工

子代理系统横跨两个 crate，职责切得很干净：

- **`xai-grok-tools/.../grok_build/task/`**：通用基础设施。
  - `coordinator.rs`：**单写者协调器 actor**（`SubagentCoordinator`），拥有 admission 队列、pending/active/completed 三张表、阻塞等待者、前台超时、取消、派发终态。文件头注释明确「本模块无共享可变状态」。
  - `backend.rs`/`ChannelBackend`：宿主侧 API——所有操作都是对 coordinator 通道发 `SubagentEvent`，宿主之间只差 `ChildRunner` 实现。
  - `admission.rs`/`coordinator/`：并发准入（`max_concurrent` + 超限策略）、spawn 队列、`SpawnGraph` 记录父子关系。
  - `types.rs`：`SubagentRequest`/`SubagentResult`、capability mode、isolation mode、前台/后台语义。

- **`xai-grok-shell/src/agent/subagent/`**：shell 专属运行时。
  - `child_runtime.rs`：子会话的构造与执行（`run_shell_child`），实现 `ChildRunner`。
  - `subagent_spawn.rs`（`agent/mvp_agent/`）：把父会话状态快照成 `SubagentSpawnContext`——模型、cwd、`chat_state_handle`、`cmd_tx`、hunk tracker、yolo、`subagent_depth`。子代理**共享父的文件系统、终端、env 后端**，所以编辑和 bash 走的是同一条通道。
  - `spawn.rs`：协调器 spawn 入口、ACP 呈现、子代理事件流。
  - `prompt_turn_receipt.rs`/`prompt_turn_result.rs`：把子代理终态转成父会话可消费的输入（`TaskCompleted`/`SubagentCompleted` 来源消息）。

### 2.2 一次 spawn 的路径

```
模型发起 tool call "task"
  → prepare_tool_call（INNER_LOOP §4.5：参数解析、PreToolUse、权限）
  → TaskTool::execute → ChannelBackend::spawn
  → SubagentCoordinator：admission → pending →（并发槽可用）→ active
  → ChildRunner = run_shell_child：独立子会话，自己的上下文窗口，跑自己的内循环
  → 完成后 coordinator 生成 SubagentCompleted 唤醒消息
  → 进入父会话 pending_inputs，父会话下一轮转历史中看到结果
```

### 2.3 父代理可用的编排原语（模型侧工具）

| 工具 | 作用 |
|------|------|
| `task`（`TaskTool`） | spawn 子代理。参数：`subagent_type`（agent 类型，省略即 `general-purpose`）、`run_in_background`（后台跑不阻塞轮次）、`await_to_completion`、`resume_from`（恢复既有子代理）、`capability_mode`（read-only / read-write / execute / all）、`isolation: "worktree"`（隔离工作区，见 §2.5） |
| `task_output` / `get_task_output` | 取后台任务/子代理的输出 |
| `wait_tasks` | 阻塞等待指定任务完成；可中断——`select!` 上挂父级 steer/用户插话，提前返回继续轮次（`tool_dispatch.rs` 的 wait-aware dispatch） |
| `kill_task` | 终止后台任务 |
| `send_subagent_message` | 给**仍在运行**的子代理发转向消息（`ActiveAgentMessage`）——这是唯一的"打断-注入"通道，配合 coordinator 的 `active_message` ingress 与配额（`MAX_ACTIVE_MESSAGE_ADMISSIONS`） |
| `update_goal` | 见 §3.5 |

后台子代理完成时会通过 `TaskCompletionReminder` 提醒 + `ScheduledTaskFired` 类通知自动唤醒父会话（`surface_completion`），不一定需要父代理显式 wait。

### 2.4 子代理类型与能力投影

- **agent type**（`.grok/agents/*.md` 或内置 `general-purpose`/`explore`/`plan`）：决定 system prompt、工具集、harness 风格。`explore`/`plan` 是只读变体（查/搜/grep 但不写）。
- **persona**（`[subagents.personas]`）：行为叠加层，注入 `<system-reminder>`，只调语气/输出格式/输入输出契约，不动工具集。
- **capability mode**：`child_tool_projection.rs` 按模式裁剪工具集（read-only 砍掉所有非只读 kind；并清理由此变成孤儿的 `get_task_output`/`kill_task`——`prune_orphaned_background_task_tools`）。
- **允许清单**：`allowed_subagent_types` + `subagent_toggle` 在 `build_subagent_validation_context` 时校验类型合法性。
- **嵌套深度**：`subagent_depth` 快照传给子代理，限制子代理再生孙代理。

### 2.5 Worktree 隔离

`task` 的 `isolation: "worktree"` 让子代理在 `xai-fast-worktree` 建的 CoW git worktree 中执行（`session/worktree.rs`、`session/worktree_cleanup.rs`）：

- 子代理的编辑不影响父工作区；
- 完成后 worktree 保留，路径随结果返回；
- 池化版本（`session/worktree_pool.rs`）已下线，只保留进程死后的残留清理。

配合 `FOREGROUND_BLOCK_BUDGET`（终端前台命令 15s 自动转后台）和 coordinator 的 deadline，子代理编排天然适合"并行探路 + 汇报"形态。

## 3. 目标模式编排（/goal）

`/goal <objective>` 开启一个跨多轮的自主执行模式。核心是**每个轮次结束时由 harness 决定要不要续跑**，而不是等用户。

### 3.1 状态机（`goal_tracker.rs`，纯状态机无 async）

```
GoalPhase:   Idle → Planning → Executing → (Idle)
GoalStatus:  Active
             ├─ UserPaused        Ctrl+C / /goal pause
             ├─ BackOffPaused     classifier 运行次数到 cap
             ├─ NoProgressPaused  验证者连续看到相同 gap 指纹（stall 早停）
             ├─ InfraPaused       轮次基础设施错误
             ├─ Blocked           需要用户动作的外部前置
             ├─ BudgetLimited     token 预算耗尽
             └─ Complete
```

历史事件环（`GoalEvent`，上限 64 条）+ token 基线/用量 + `live_subagent_tokens`/`live_context_pct`/`live_turn_count` 通过 `GoalUpdated` ACP 更新实时推给客户端（`goal_orchestrator.rs` 的 `build_goal_updated`）。

### 3.2 角色阵容（`session/goal_*.rs`）

| 角色 | 文件 | 职责 | 失败语义 |
|------|------|------|----------|
| **Planner** | `goal_planner.rs` | goal 启动时生成结构化 `plan.md`（写盘到 tracker.plan_path），后续验证的"合同" | **fail-CLOSED**：失败即暂停 goal |
| **Worker** | 主会话自身 | 按 directive 推进实际工作 | — |
| **Evaluator** | `goal_evaluator.rs` | 每轮结束时看 transcript（截到 32KB）判 `continue`/`candidate_complete`/`blocked`；JSON schema 校验 + `blocker_key` 稳定性追踪 | 重试仍失败 → `InfraPaused` |
| **Classifier / Skeptic 面板** | `goal_classifier.rs` | `candidate_complete` 后并行 spawn N 个对抗性验证子代理（默认 3，1–5），各自独立看 diff（≤256KB）+ 工作区证据出 JSON verdict；面板详情写 `goal-classifier-*-attempt.md` | 失败同样计入尝试次数 |
| **Strategist** | `goal_strategist.rs` | 连续 N 次 `NotAchieved` 后触发，诊断结构性卡点写 strategy note（**不写 plan.md**——`PlanGuard` RAII 快照并在 spawn 后恢复字节），建议注入下一条 directive | **fail-OPEN**：失败仅记日志，goal 继续 |
| **Stop detector** | `goal_stop_detector.rs` | 正则面板识别"弃疗式"收尾话术（giving up / stopping here / check back later / ready for review / please do X for me …共 9 类），命中则下发针对性 nudge | — |
| **Laziness classifier** | `acp_session_impl/laziness*.rs` | 闲置超时（默认 10s）后用会话模型自评"对话是否停滞"，置信度 ≥0.7 才注入 nudge；仅 goal 激活时生效 | 非 goal 模式 Suppressed |
| **Summarizer** | `goal_summarizer.rs` | 阶段摘要 | — |

角色的模型/工具面可独立配置（`GoalRoleModel` 池 + `resolve_goal_role_override`）；skeptic 索引与池按 round-robin 稳定分配（`expand_skeptic_assignment`），保证 resume 后同编号 skeptic 用同一模型。

### 3.3 轮次间的续跑机制

`handle_turn_input_inner` 的外层循环（`turn.rs`）每轮结束调用 `run_goal_round_end`（`acp_session_impl/goal.rs`）：

```
轮次完成（Completed + EndTurn）
  → goal 激活？
    → laziness_injection_active 检查 → evaluator 评判（限流重试）
    → enforce_goal_token_budget（超限 → BudgetLimited 暂停）
    → decision:
        continue            → record progress，prepare_goal_continuation 出 directive
        candidate_complete  → verify_goal_candidate → skeptic 面板投票
        blocked             → blocker streak ≥3 → auto_pause（Verification）
  → directive = "Evaluator next step: ..." + 可选 strategist 建议 + 可选 premature-stop nudge
  → inject_goal_continuation_message：goal_summary 用户消息入历史 → continue 外层循环 → 下一轮
```

两个细节：

- `prune_prior_goal_continuation_directives`：注入前先把历史里旧的 continuation directive 标记清理，避免指令堆积；
- `maybe_queue_goal_continuation`（非轮内路径）：轮次整体完成回到命令循环后，把 directive 作为普通 `InputItem` 推进 `pending_inputs`（origin=`GoalSummary`），走队列而不是直插——因此 goal 续跑天然复用队列的 merge/dedup 规则，且代码里双检重防止重复注入。

### 3.4 停滞与预算的多层防线

| 防线 | 位置 | 触发 |
|------|------|------|
| Evaluator `blocked` streak | `goal.rs` | 同一 `blocker_key` 连续 ≥3 → 暂停 + 提示用户动作 |
| Skeptic 面板 gap 指纹 | `goal_tracker.rs` | 连续 2 次相同 gap 指纹 → `NoProgressPaused`（strategist 触发后放宽到 5） |
| Strategist cap bonus | `goal_tracker.rs` | strategist 触发后 +3 次 classifier 额度 |
| Token budget | `goal.rs` `enforce_goal_token_budget` | 超 `token_budget` → `BudgetLimited` |
| Classifier max runs | `goal_classifier.rs` | `GOAL_CLASSIFIER_MAX_RUNS_DEFAULT=10`（可调）→ `BackOffPaused` |
| 连续失败退避 | `handle_turn_end`（`turn.rs`） | 轮次 `Err` 连续计数 → 自动暂停 |

### 3.5 模型侧的参与：`update_goal` 工具

模型可用 `update_goal` 主动汇报（`completed` / `blocked_reason` / `message`）——**这不是旁路**：工具经 oneshot 等 `SessionActor` 回执，tool result 反映真实编排结果（classifier 判定/状态迁移/拒绝）。`drain_goal_updates` 在轮次中攒这些信号。

## 4. Workflow 编排（Rhai 脚本引擎）

`xai-workflow` crate + `session/workflow/` 提供**代码级编排**：把多代理流程写成 Rhai 脚本，由引擎驱动。

### 4.1 引擎能力（`xai-workflow/src/engine.rs`）

脚本可用的 host 函数：

| 函数 | 作用 |
|------|------|
| `agent(prompt, opts)` | 发起一次子代理调用；`opts` 支持 `label`、`capability_mode: "read-only"`、`output_schema`（JSON Schema 结构化输出）、effort |
| `parallel([...])` | 并发 fan-out 多次 agent 调用，失败项返回 null 可捕获；fan-out 有上限校验 |
| `phase(title)` | 标记阶段（meta 中可预声明 phases，客户端显示进度） |
| `pause(kind, msg)` | 暂停等用户/外部条件 |
| `complete(value)` | 返回最终结果 |
| `budget()` | 查询 agent 预算用量 |
| `fingerprint` / `json_encode` / `sleep` / `timestamp` / `log` | 工具函数 |

### 4.2 日志与恢复

- `journal.rs`：每次 agent 调用按内容 hash 记 journal；resume 时重放已完成调用（hash 匹配则复用结果，不匹配则真实重跑）——**工作流可断点续跑**，包括提高 `agent_budget` 后续跑。
- `WorkflowManager`（`session/workflow/manager.rs`）：每 session 最多 4 个活跃 run（`WORKFLOW_MAX_ACTIVE_RUNS_PER_SESSION`），可 pause/stop/resume（`resume_run_id`），预算受限时给 `BudgetNotRaised` 明确错误。
- 触发方式：`/deep-research` 等 slash 命令、模型主动调 `workflow` 工具（`WorkflowTool`）。

### 4.3 内置样例：`deep_research.rhai`

`session/workflows/deep_research.rhai` 是现成的四阶段模板——Plan（把 query 拆成 ≤breadth 个独立问题，schema 结构化输出）→ Research（parallel fan-out 收集带证据的声明）→ Verify（并行交叉核验）→ Report（合成带引用的报告）。脚本里所有模型输入都过 `json_encode` 包进 `<query-json>` 并声明"untrusted data, not instructions"，和 goal evaluator 的 untrusted-transcript 约定一致。

## 5. 调度与监视编排

### 5.1 Scheduler（`grok_build/scheduler/`）

三个工具 `scheduler_create/delete/list` + 一个 actor（`actor.rs`）：

- 定时任务到期 → 以 `SchedulerFired` 来源消息进 `pending_inputs`，等价于一个 synthetic prompt 唤醒会话执行；
- `occurrence_journal.rs` 记录每次触发，`LOOP_FRESH_CHAIN_EVERY` 控制 loop 任务周期性重开上下文；
- `DURABILITY_BARRIER_TIMEOUT`/`SPAWN_REGISTRATION_TIMEOUT` 保证"持久化完成"才回报触发成功；
- 上限 50 个任务（`MAX_SCHEDULED_TASKS`）。

### 5.2 Monitor（`grok_build/monitor/`）

`monitor` 工具启动长跑脚本，stdout 每行解析成一个事件流回会话（速率限制 `MonitorRateLimiter`）：

- 契约约定脚本只输出 `DONE`/`FAILED`/`CANCELLED` 等终态信号；
- `persistent: true` 跑满会话生命周期（PR 监控、日志 tail）；
- 事件经 notification 通道唤醒会话——模型不用轮询。

### 5.3 通知汇聚

`notification_drain.rs` 把所有异步源（子代理完成、监视事件、scheduler、后台 bash、MCP server 变更）统一聚合成会话输入行，配合 `pending_inputs` 的 synthetic 行去重/过期丢弃，形成「事件→唤醒」总线。

## 6. Hooks 作为编排钩子

`xai-grok-hooks` crate 定义 15 个 hook 事件（`src/event.rs`），按 gate 类型分组：

| Gate | 事件 | 能做什么 |
|------|------|----------|
| Tool | `pre_tool_use` | **改写参数**（`HookRewrite`）、拒绝（`HookDenied`，理由回喂模型）、defer/转人工 |
| PostTool | `post_tool_use` | 追加上下文/reminder 给模型 |
| Stop | `stop` / `subagent_stop` | 判 `KeepWorking`——把反馈文本合成 user 消息强制续跑（stop gate，上限 8 次/轮）；`prevent_continuation` 反向强停 |
| Prompt | `user_prompt_submit` | `decision: "block"` 拒绝用户输入（面向用户，不进模型上下文） |
| Observe | `session_start`、`notification`、`permission_denied`、`stop_failure/cancelled`、`subagent_start`、`pre/post_compact`、`session_end` | 只观察 |

hooks 来源三层：全局 `~/.grok/hooks/`、项目 `.grok/hooks/`、插件包（`GROK_PLUGIN_ROOT`/`GROK_PLUGIN_DATA` 注入）；managed policy hook 即使被列进 disabled 也仍在 stop-gate 热路径生效。别名系统兼容 camelCase 及 `beforeShellExecution` 等 per-operation 写法。

## 7. 工具能力（Tool Surface）

### 7.1 分类学：ToolKind 统一语义层

`types/tool.rs` 的 `ToolKind`（36 个变体）把"工具叫什么"和"工具是什么"解耦：同一语义在不同 harness 里可以有不同 wire 名字，但都映射到同一个 kind——权限、UI 展示（`presentation_name`）、goal 角色模板（`{READ_TOOL}` 等占位符）、能力裁剪都只认 kind。`#[serde(other)]` 保证旧版本消费者遇到新 kind 降级为 `Other` 而不是报错。

### 7.2 注册表与 harness 风味（`registry/types.rs`）

`ToolRegistryBuilder::new()` **一次性注册所有风味**的内置工具，再由 `ToolServerConfig` 按 agent 类型/模型选启用子集：

| 风味 | 特点 |
|------|------|
| `grok_build` | 主工具集：`BashTool`、`ReadFileTool`、`SearchReplaceTool`、`GrepTool`、`ListDirTool`、`TodoWriteTool`、`TaskTool`、`WebSearch/Fetch`、`LspTool`、媒体生成、`Monitor`、scheduler 三件套、`EnterPlanMode/ExitPlanMode`、`AskUserQuestion`… |
| `grok_build_concise` | Read/Edit/Bash 的精简描述变体（省 token） |
| `grok_build_hashline` | 行哈希锚点读写：read 输出带行 hash，edit 以 hash 为锚，偏移/过期时有限范围恢复（`grok_build_hashline`） |
| `codex` | 移植 codex 的 `apply_patch`/`list_dir`/`grep_files`/`read_file` |
| `opencode` | 移植 opencode 的 bash/read/edit/write/grep/glob/todo/skill |
| 通用 | `memory`（memory_search/get）、`skills`（模型调用用户 skill）、`search_tool`/`use_tool`（MCP 网关） |

工具的 `description_template` 是 Tera 风格模板，`${{ tools.by_kind.task }}` 这类占位符在 finalize 时替换为**该 toolset 实际启用的工具名**——同一个 prompt 模板自动适配不同命名方案（`TemplateRenderer` + `param_map`）。

### 7.3 核心工具面（grok_build 主集）

| 域 | 工具 | 备注 |
|----|------|------|
| 文件 | `read_file`、`search_replace`、`list_dir`、`grep`、`apply_patch`(codex) | read 有 bytes/tokens 双帽；edit 走 search/replace 或 hashline 锚 |
| 终端 | `run_terminal_cmd`/`bash`、`kill_task`、`get_task_output` | 见 §7.5 |
| 搜索 | `grep`、`web_search`、`web_fetch`、`lsp`（定义/引用） | lsp 复用 `implementations/lsp` |
| 计划 | `todo_write`、`enter_plan_mode`、`exit_plan_mode` | todo 状态喂给 TodoGate |
| 代理 | `task`、`task_output`、`wait_tasks`、`send_subagent_message`、`update_goal` | §2 |
| 调度 | `scheduler_create/delete/list`、`monitor`、`workflow` | §4–5 |
| 记忆 | `memory_search`、`memory_get` | v2 dream/capture/carryover 在 session 层 |
| 交互 | `ask_user_question`、`send_feedback` | 前者走 ACP 反向请求 |
| 媒体 | `image_gen`、`image_edit`、`image_to_video`、`reference_to_video`、`video_gen`、`deploy_app`/`init_or_update_app`(stub) | `media_gen_limits` 限每步批量、每轮总帽；超帽触发丢弃重采样（INNER_LOOP §4.4） |
| 生态 | `skill`、`search_tool`、`use_tool` | skill = 用户 markdown 提示词；后两者见 §7.4 |

### 7.4 MCP 网关：search_tool + use_tool

MCP 工具不全量塞进 tool definition（工具多了会撑爆 schema），而是走发现-调用两段式（`implementations/search_tool`、`use_tool`）：

- `search_tool`：对 MCP server 工具做 BM25 关键词检索（本地 `ToolIndex`），返回候选；server 指纹 `(count, desc_hash, names_hash)`（FNV-1a）跟踪变化并广播 reminder；
- `use_tool`：按名派发；`native_tool_correction` 识别误投给它的本地工具名并返回纠正性错误；支持文件参数（`parse_arguments_file`，`file_input_supported` 由宿主能力决定）；输出过 `truncate_tool_output` 和 `render_structured_content`；
- `mcp_elicitation/` 处理 MCP server 反向向用户请求输入；`mcp_file_input.rs` 把文件参数预读注入。

### 7.5 终端/计算后端（`computer/local/terminal.rs`）

`LocalTerminalActor` 是 actor 模式的终端后端，所有可变状态归 actor 所有：

- **前台预算**：前台命令最多阻塞 15s（`FOREGROUND_BLOCK_BUDGET`），超时自动转后台不杀死——长命令不卡轮次；
- **生命周期**：后台任务最长 10h，完成后记录保留 5min，输出落盘；SIGTERM → 1s 宽限 → SIGKILL；
- **资源约束**：可选 cgroup 内存限（`CgroupGuard`+`MemoryMonitor`，OOM 特定退出码）；
- **持久 shell**：`shell_state.rs` 让 bash 工具跨调用保留 cwd/env（fd 4 写状态 dump）；
- **流式输出**：默认 100ms 推 chunk 通知，大头输出 front/back 截断。

### 7.6 权限、信任与沙箱

工具执行前有完整门禁链（`xai-grok-workspace/src/permission/` + `folder_trust.rs`）：

- **AccessKind**：按 kind 推断权限级别（Edit/Read/Bash/MCP/WebFetch…），`is_read_only` 由 kind 默认给出；
- **权限模式**：`ask`（发 ACP permission 弹窗等用户）/ `auto`（auto_mode 分类器判风险）/ `yolo`（全放行）；模式可由 user/项目配置与 folder trust 共同解析（`resolution.rs`）；
- **folder trust**：未信任目录收紧默认权限；
- **规则系统**：`rules.rs`/`grants.rs`/`managed_policy` 支持托管策略（不可被本地禁用覆盖）；
- **沙箱**：`xai-grok-sandbox` + `nono`——macOS Seatbelt、Linux Landlock；网络由 seccomp 按进程拦截；`exec_risk.rs`/`bash_command_splitting.rs` 做 bash 命令拆分与风险判定；
- **文件锁**：同文件编辑在 dispatch 前串行化（`tool_dispatch.rs` `lock_path_for_args`）。

### 7.7 提醒系统（reminder as cross-cutting capability）

`types/tool.rs` 的 `Reminder` trait 让提醒逻辑独立于工具：工具有自己的 per-tool reminder（如空文件提示），注册表上挂跨工具 reminder——`LspDiagnosticsReminder`（编辑后报 LSP 诊断）、`TaskCompletionReminder`（后台任务完成）、`SkillDiscoveryReminder`。提醒追加进 `ToolRunResult.prompt_text`，随 tool result 一起进上下文。

## 8. 编排全景小结

```
                        ┌──────────────────────────────────────────┐
                        │               用户 / 客户端               │
                        └──────────────────┬───────────────────────┘
              /goal, /deep-research,      │      prompt, slash cmd
              subagent takeover, hooks    │
   ┌──────────────────────────────────────▼───────────────────────────┐
   │                 SessionActor 命令循环 (L0)                        │
   │   pending_inputs: Human/GoalSummary/SchedulerFired/              │
   │                   SubagentCompleted/MonitorEvent/Notif…          │
   └───────┬──────────────────────────────┬──────────────────────────┘
           │ turn (L1)                    │ turn (L1)
   ┌───────▼──────────┐         ┌─────────▼──────────┐
   │ goal 模式外层循环  │         │ workflow run        │
   │ evaluator→directive│        │ Rhai 脚本 → agent() │
   │ skeptic 面板      │         │ → parallel()        │
   │ strategist        │         │ (journal 可 resume) │
   └───────┬──────────┘         └─────────┬──────────┘
           │ spawn SubagentRequest        │ spawn SubagentRequest
           └──────────────┬───────────────┘
                  ┌───────▼────────┐
                  │ SubagentCoord. │  admission/queue/graph/cancel
                  └───────┬────────┘
                  ┌───────▼────────┐   共享父 FS/terminal/env
                  │  子会话内循环    │   isolation=worktree 可选
                  │  (同 L2 机制)   │   capability_mode 裁剪工具
                  └───────────────┘
```

关键设计取舍：

1. **一切编排归约为"往队列/历史里放带 origin 的输入"**——无专属消息总线，复用轮次机制；
2. **编排判定与工作分离**：evaluator/skeptic/strategist 都是独立子代理，读 transcript/diff 但（除 planner 写 plan.md 外）不碰工作区；planner fail-closed、strategist fail-open、skeptic 计入 cap，不同失败语义对应不同关键度；
3. **子代理即一等会话**：同一套内循环、同一套权限/工具栈，只是换了 agent definition + 裁剪后的工具集 + 独立上下文；
4. **可观测性内置**：goal token 记账、skeptic 详情文件、classifier 尝试次数、premature-stop pattern 标签全部走 `GoalUpdated`/telemetry；
5. **安全纵深**：admission 并发帽、嵌套深度、capability mode、worktree 隔离、folder trust、permission、hooks、托管策略层层叠加——即兴编排再自由也出不了门禁。

## 9. 相关文件速查

| 主题 | 文件 |
|------|------|
| 子代理协调器 | `xai-grok-tools/src/implementations/grok_build/task/{coordinator.rs,coordinator/,admission.rs,backend.rs,types.rs}` |
| 子代理 shell 运行时 | `xai-grok-shell/src/agent/subagent/`、`agent/mvp_agent/subagent_spawn.rs` |
| Goal 状态机/角色 | `xai-grok-shell/src/session/goal_tracker.rs`、`goal_{planner,evaluator,classifier,strategist,stop_detector,summarizer,role_tools,orchestrator}.rs` |
| Goal 续跑 glue | `xai-grok-shell/src/session/acp_session_impl/goal.rs`（`run_goal_round_end`、`maybe_queue_goal_continuation`、`inject_goal_continuation_message`） |
| Workflow 引擎 | `xai-workflow/src/{engine,host,journal,run}.rs`、`xai-grok-shell/src/session/workflow/`、`session/workflows/*.rhai` |
| Scheduler / Monitor | `grok_build/{scheduler,monitor}/` |
| Hooks | `xai-grok-hooks/src/{event,dispatcher,discovery,config}.rs`、`acp_session_impl/{hook_dispatch,stop_gate,turn_end_hooks}.rs` |
| 工具注册/分类 | `xai-grok-tools/src/{registry/types.rs,types/tool.rs,tool_taxonomy.rs,bridge.rs}` |
| MCP 网关 | `xai-grok-tools/src/implementations/{search_tool,use_tool}/`、`mcp_elicitation/`、`acp_session_impl/mcp*.rs` |
| 终端后端 | `xai-grok-tools/src/computer/local/terminal.rs`、`computer/types.rs` |
| 权限/沙箱 | `xai-grok-workspace/src/permission/`、`folder_trust.rs`、`xai-grok-sandbox` |
| 隔离工作区 | `xai-fast-worktree`、`xai-grok-shell/src/session/worktree*.rs` |
| 用户文档 | `crates/codegen/xai-grok-pager/docs/user-guide/`（hooks、subagents、sandbox、plugins 各章） |
