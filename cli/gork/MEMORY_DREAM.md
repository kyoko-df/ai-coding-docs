# Grok Build 记忆写路径与 Dream 巩固管线解析

> 本文说明 grok 记忆的**写侧**：每轮对话如何被提取成观察（capture）、观察如何被离线巩固成主题记忆（Dream/consolidation）、以及这套管线怎么做崩溃恢复。
> 姊妹篇：`CONTEXT_AND_COMPACTION.md` 讲了 memory 的**读路径**（检索、注入、压缩后补注）；本文讲数据是怎么进来的、怎么被整理的。
> 核心结论：**这是一套数据库思维的 LLM 管线——所有状态转换先落 SQLite/文件系统再调用模型，模型输出永远经过确定性校验才提交，每个阶段都设计成"进程崩在任意一步，下一个 worker 能确定性 roll forward"。**

## 1. 总览：两代记忆、四条写管线

```
                        ┌─────────── legacy 管线 ───────────┐
                        │  turn → flush(压缩前摘要) →        │
                        │  sessions/*.md → Dream(周期闸门)  │
                        │  → MEMORY.md(单文件)              │
                        └──────────┬──────────────────────┘
                                   │ 一次性迁移
                                   ▼ carryover
   ┌───────────── v2 管线 ─────────────────────────────────────────────┐
   │ 每轮结束: enqueue_v2_turn_capture                                 │
   │   → capture worker(租约+schema 提取) → observations/_inbox/*.md    │
   │ 巩固:   evaluate_v2_automatic_dream(≥20 条 或 ≥24h)                │
   │   → 默认: V2Consolidation(claim→模型plan→确定性commit)             │
   │   → batch: run_batch_dream(catalog+notes→actions/plan 循环→commit) │
   │ 结果:   topics/*.md + MEMORY.md(生成的清单) + lexical index        │
   └──────────────────────────────────────────────────────────────────┘
```

v2 数据布局（`xai-grok-memory/src/v2.rs::initialize_scope`）：

```
<scope>/                      global 与 workspace 两个 scope 同构
├── MEMORY.md                 生成的主题清单（原子替换，不是手写文件）
├── topics/                   主题文件，每个是一个 markdown 记忆单元
├── observations/             已归档观察
│   └── _inbox/               待巩固的观察（不可变，content-hash 命名）
├── archive/                  无法安置/退役的观察
└── state.db (schema v4)      跨进程串行点：job、cursor、lease、claim、plan
```

关键不变式：**模型从不直接写文件系统**。模型只产出结构化草稿（observation draft / plan），由 `V2CaptureStore`/`BatchDreamStore`/`V2ConsolidationStore` 在 SQLite 事务里校验并提交。

## 2. Capture：每轮结束的观察提取

### 2.1 入队与租约

轮次结束时（`run_loop.rs` / `cancel.rs` 的 `enqueue_v2_turn_capture`）：

1. `V2CaptureStore::ensure_session` + 读 cursors，`missing_range(requested, completed)` 算出尚未捕获的轮次区间；
2. `enqueue_for_prompt_with_visibility` 入队（memory 处于 hidden/shadow 状态时产出的观察标记为 hidden，后期 `promote_v2_hidden_observations` 再转正）；
3. `start_v2_capture_worker` 起后台 worker（`memory.capture_worker_is_running` 保证单例）。

worker 先做 `reconcile()`（崩溃恢复，见 §6），再 `claim` 拿一个 **5 分钟租约**（`CAPTURE_LEASE`），owner 为 `pid-uuid`。

### 2.2 转录压缩（`capture_transcript.rs`）

提取的输入是本轮对话转录，但有硬预算（`TranscriptBudget`，**每多一次重试预算减半**，下限兜底）：

```
stage 1: 剥 reasoning.encrypted_content（加密推理对提取器无意义）
stage 2: 剥图片、tool body
stage 3: 砍 tool loop 中段（保留首尾）
stage 4: 连用户文本都截——size 上永不失败
```

注释原话：*"Stages remove what the extractor does not need before touching what it does... Never fails on size."*

### 2.3 Schema 提取（`v2_capture.rs::extraction_schema`）

提取调用**无工具**：固定 system + 转录 JSON + 严格 JSON schema，模型只能返回：

```json
{"outcome": "noop" | "observations",
 "observations": [{
    "type": "user | feedback | project | reference",
    "topic_hint": "string|null",
    "statement": "≤1KB 的一句话事实",
    "keywords": ["≤16 个"],
    "aliases": ["≤16 个"],
    "body": "≤8KB 的正文|null"
 }]}
```

`parse_model_outcome` 解析后由 `V2CaptureStore::commit` 校验落盘：观察文件先发布到 `observations/_inbox/`（不可变、hash 命名），**再**写 outcome 行、**再**推进 indexed cursor——这个顺序是崩溃恢复设计的一半（§6）。

### 2.4 完成后跟进

`on_v2_capture_completed` 按 `v2_capture_followups` 配置依次：

1. **维护 GC**（`V2MaintenanceStore::gc_now`）：按 `RetentionPolicy` 清过期 archived 观察与 terminal job；
2. **Dream 资格评估**（`evaluate_v2_automatic_dream`）：`V2ConsolidationStore::on_capture_completed` 查 pending 观察，**≥20 条**或最老的**超 24h** → `Ready` → 触发 `run_v2_dream_with_cancel(Automatic)`；
3. Shadow rollout 下走 `on_capture_completed_shadow`（只评估不真巩固）。

## 3. Batch Dream：LLM 驱动的批量巩固

当 `batch_dream_enabled` 且 rollout=`Active` 时，`execute_v2_dream` 走批量路径（`shell/session/batch_dream.rs::run_batch_dream` + `xai-grok-memory/src/batch_dream_*.rs`）。

### 3.1 运行骨架

```
open BatchDreamStore（挂共享巩固租约，与默认 Dream 互斥）
  → recover()（上一轮的半途状态 roll-forward/回滚）
  → TopicCatalog::build（全量主题目录，≤256KB）
loop {  claim 一批 notes（≤20 条/≤256KB，排除本run已尝试的）
      → BatchDreamSession → run_batch（多轮模型对话）
      → commit 或 release（notes 回 inbox 下次再试）}
```

`BatchDreamControl` 带 deadline（默认 `max_run_time`），watcher 到点 `cancel()`；`LEASE_GRACE` 额外 +120s 让压哨 commit 收尾。

### 3.2 提示协议：actions 或 plan，二选一

模型每轮收到固定 system prompt + `<topic_catalog>` + `<notes>` + 本轮预算声明。回复只能是**一个** JSON 对象（`response_schema` 结构化输出）：

- `{"actions": [...]}` 查资料（≤12 个/次）：`read_topic`（≤16KB 整文，大了返回带字节偏移的大纲）、`read_range`（≤32KB）、`search`（≤8 个 Rust regex，行级命中带偏移，游标翻页）、`list`（目录翻页）。
- `{"plan": {...}}` 结束本批：≤32 个 edits（`patch`/`insert`/`replace_section`/`create`/`update_description`）+ 每条 note 一个 outcome（`applied`/`no_change`/`deferred`）。

### 3.3 防幻觉编辑：hash 绑定读标签

这是整套设计里最精妙的一处：**每次 read 返回一个标签 R1/R2/…，标签绑定读取时刻的文件 hash**。`plan` 里的 `patch{read, old_text, new_text}` 必须引用某个读标签，`session.check(&plan)` 校验：

- 文件当前 hash 必须等于读取时的 hash——中途被并发改过则该读失效（stale），
- `old_text` 必须在该读文本里精确出现一次——防止模型"凭印象"编辑没看过的文本。

校验失败 → `repair_message`：列出错误 + 对 stale 路径**重新下发新鲜读取**（`session.refresh`），模型重发修正 plan；`MAX_REPAIRS = 2` 次后或调用预算耗尽时，用最后一次 check 的 commit 兜底。

### 3.4 截断与容量退让

- 回复被输出上限截断：保留头 2KB + `[cut off]` 标记回灌，指示模型发更短的 patch；`MAX_TRUNCATIONS = 2`。
- 模型可以 `deferred{reason: "batch too large"}` 退让——这个精确理由**不计入** note 的失败次数。
- 一条 note 被 defer 满 `MAX_NOTE_DEFERRALS = 3` 次 → 归档为 unplaceable，不再反复触发永远安置不了它的运行。
- commit 本身有 `MAX_COMMIT_TEXT_BYTES = 256KB` 新文本上限、≤32 splices/文件。

### 3.5 KV-cache 亲和

每个 batch 共用同一个 `x_grok_conv_id`；每次请求重复 system + 完整 catalog——刻意让 provider 命中前缀缓存，只有 actions/plan 部分变化。目录分层降级（`CatalogTier`: Full → ShortDescriptions → TitlesOnly → Partial）保证超大主题库也有目录可发。

## 4. 默认 V2 巩固与 Legacy Dream

### 4.1 默认 V2 consolidation（`v2_consolidation.rs`）

batch 未启用时的路径：claim 在 SQLite 里固化一份 inbox 快照（capture 可继续并发发布新文件）；**模型工作在事务外**，最终产出一个确定性 plan，topic/archive/index/manifest 提交可从 plan 重放；每个变更入口都验 lease generation——"fenced"语义：租约被抢走后旧 worker 的提交会被拒。

### 4.2 Legacy Dream（`dream.rs`）

老管线：每隔 `min_hours`（默认 24）且累积 ≥`min_sessions`（默认 5）个 session log 才开门（`check_dream_gates`）。`DreamLock` 是 `O_EXCL` 原子创建的互斥文件，内容为 `<pid> <nonce>`——pid 用于存活回收，nonce 让 guard 删锁前确认仍是自己那把；`.dream-consolidated` 标记**只在 commit 成功后才盖**，崩溃则门保持开着。

`DREAM_SYSTEM_PROMPT` 是一整段巩固规范：合并相关项、以新事实覆盖旧事实、相对日期转绝对、丢弃寒暄/工具噪音/状态性内容、保留决策与根因、把任务叙事泛化成可复用机制、只抄录输入中逐字出现的标识符与数值（**禁止无依据细节**）。输出必须含 markdown `##` 头，否则与 `NO_REPLY` 一样视为无可巩固。输入封顶 `MAX_DREAM_INPUT_CHARS = 32K`，超出的 session 不进 `processed_stems`、不会被清掉，留给下一梦。

### 4.3 Carryover：legacy → v2 一次性迁移

会话启动时（`memory_carryover.rs` + `v2_carryover.rs`）：读旧 `MEMORY.md`（只读不改），每个 `##` section 变成一个 v2 topic；scaffold 样板和空文件跳过；>24 个 section 折叠成一个 topic 防目录爆预算；源文件 hash 记进 scope 的 meta 表——**幂等**，除非旧客户端又改写了文件否则不再迁。

### 4.4 外部编辑感知（`watcher.rs`）

`notify` watcher 盯 memory 目录：`.md` 增删改记进脏路径集（`ArcSwap` RCU 无锁）；搜索路径 `is_dirty` 为真时才同步索引——created/modified 重索引、deleted 清 stale chunk。**用户/其他进程手改记忆文件不会丢**。

## 5. 统一的设计约束

| 约束 | 体现 |
|------|------|
| 一切有界 | 每条 note ≤16KB、批 ≤256KB、读 ≤32KB/次、计划 ≤32 edits、回复 ≤256KB、转录按尝试减半 |
| 模型零信任 | schema 校验 → hash 绑定读标签 → check → 确定性 commit；模型失败只回滚自己那批 |
| 崩溃可恢复 | 文件先于元数据发布、outcome 先于 cursor、plan 可重放、lease 防 fencing |
| 跨进程互斥 | capture/consolidation lease 存在 state.db；DreamLock 文件锁管 legacy |
| 成本意识 | KV-cache 亲和 conv_id、catalog 分层、read 预算共享、deferral 记账防死循环 |

## 6. 崩溃恢复语义

v2 capture 的文件头注释把协议写得很清楚：

```
观察文件先于 outcome 行发布 → 崩溃可能留下孤儿文件；
每个文件带 job id + 期望文件数 → reconcile() 只收养完整且 hash 一致的集合。
outcome 先于索引和 cursor 推进提交 → reconcile() 确定性重放第二个崩溃窗口。
```

worker 三个恢复点：`reconcile()`（收养/清孤儿）→ `release_retryable`（租约被夺可安全释放，`StaleLease` 忽略）→ `CaptureLeaseGuard::Drop`（abort/unwind 兜底，detached blocking release）。Batch Dream 同理：`STAGE_PREFIX = ".batch-dream-"` 的 staging 文件 + 先持久化 commit plan 再动 topic 字节，中断后 apply 只会 roll forward。

## 7. 调度与生命周期

- capture worker：turn 结束触发，单例（`capture_worker_is_running`），`CaptureActivity`（queued/running/completed/no_op/retry/failed）全量上报 telemetry。
- Dream 触发：`Automatic`（pending ≥20 或 ≥24h）、`Manual`（`/memory dream` → `run_v2_dream_slash_command`）；`run_v2_dream_with_cancel` 循环跑直到无 coalesced 触发；`V2DreamCancellationGuard` drop 即取消。
- `dream_workers` 统一 track 所有后台 memory 任务；会话停止/memory 关闭时 `cancel_and_join`。
- 寿命管理：`V2MaintenanceStore` GC archived 观察与 terminal job；`RetentionPolicy` 由 `archived_retention_days`/`job_retention_days` 配置。

## 8. 与读路径的衔接

巩固产物如何回到上下文（详见 `CONTEXT_AND_COMPACTION.md` §4）：

- `regenerate_scope_manifest` 把 topics 渲染成 `MEMORY.md` 清单 → 首轮注入 `<memory_context>`；
- `memory_search`/`memory_get` 工具按 need-to-know 取细节；
- lexical index（`index.rs`）+ watcher 脏同步保证检索面跟上写面；
- 压缩后 reminder 的 memory Top-3 补注搜索的正是这批 topics。

## 9. 源码索引

| 主题 | 路径 |
|------|------|
| v2 scope/布局/manifest | `crates/codegen/xai-grok-memory/src/v2.rs` |
| capture store/租约/恢复 | `crates/codegen/xai-grok-memory/src/v2_capture.rs` |
| 转录压缩 | `crates/codegen/xai-grok-shell/src/session/memory/capture_transcript.rs` |
| 提取 schema/outcome 解析 | `crates/codegen/xai-grok-shell/src/session/memory/v2_capture.rs` |
| capture 驱动（shell 侧） | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/memory_capture.rs` |
| batch dream store/session/plan | `crates/codegen/xai-grok-memory/src/batch_dream{,_catalog,_session,_plan,_outline,_io,_control,_commit,_recovery}.rs` |
| batch dream 模型循环 | `crates/codegen/xai-grok-shell/src/session/batch_dream.rs`、`acp_session_impl/batch_memory_dream.rs` |
| 默认巩固 | `crates/codegen/xai-grok-memory/src/v2_consolidation.rs` |
| legacy dream/锁 | `crates/codegen/xai-grok-memory/src/{dream.rs,dream_lock.rs,flush.rs}` |
| v2 dream 调度 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/v2_memory_dream.rs` |
| legacy flush | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/memory_dream.rs` |
| carryover | `crates/codegen/xai-grok-memory/src/v2_carryover.rs` + `acp_session_impl/memory_carryover.rs` |
| watcher | `crates/codegen/xai-grok-memory/src/watcher.rs` |
| 生命周期/控制 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/memory_control.rs`、`session/memory_state.rs` |
