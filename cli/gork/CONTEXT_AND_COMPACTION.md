# Grok Build 上下文组装与压缩流程解析

> 本文说明 grok 的上下文系统：每个请求怎么拼装出来（retrieval & assembly）、上下文膨胀后怎么缩水（pruning → offload → flush → compaction）。
> 姊妹篇：`INNER_LOOP.md`（单轮采样-工具循环）、`ORCHESTRATION_AND_TOOLS.md`（编排与工具）、`SANDBOX.md`（安全体系）。
> 核心结论先行：**这不是一个"满了就总结"的单一功能，而是一条多级流水线——每轮请求组装时先做廉价的请求内裁剪，越接近窗口上限动用的手段越重，直到 LLM 压缩替换整段历史；压缩本身又是带重试、降级、指纹校验和状态重建的子系统。**

## 1. 总览：上下文的生命周期

```
写入路径（每轮）                     读出路径（每次采样前）
────────────────────────────     ─────────────────────────────────────────
system + skill listing ──┐
user prefix（规则/AGENTS） ──┤        ┌─ build_request (ChatStateActor) ────────┐
用户消息 → push_user ────┤        │  1. 完整性修复（孤儿 tool_result / 悬挂调用）│
memory 首轮注入 ────────┼───────▶│  2. memory reminder upsert/persist          │──▶ ConversationRequest
工具结果 → push_tool_result ┤        │  3. 图片字节预算（hysteresis 逐出）         │      │
assistant → record_response ┤        │  4. tool_result 软裁剪/硬清除（>50%窗口）  │      ▼
插队/reminder → 同上    ──┘        │  5. 尾部半截 tool_call 截断 + 角色标注      │   sampler
                                  └──────────────────────────────────────────┘
                                             ▲
                          上下文压力增大（estimated tokens ↗）
        ┌───────────────────┬─────────────────┼─────────────────┬────────────────┐
        ▼                   ▼                 ▼                 ▼                ▼
  prompt offload      tool pruning      memory flush      两段压缩          全量替换压缩
  （单条大输入）       （>50% 窗口）      （≈阈值-余量）      （prefire 95%）   （≈85% 窗口）
  → 存文件+摘录         → 软裁/硬清       → 提取要点入记忆    → NOTE₁ 缓存      → summary+reminder
```

设计主线三条：

- **历史是唯一事实源**：所有注入物（规则、memory、reminder、插队消息、工具结果）都先落 `ChatStateActor` 的 item 历史，`build_request` 每次从历史重建请求——压缩时同一份历史既是要被替换的对象，又是摘要的输入。
- **先裁剪后总结**：便宜的请求内手段（prune、图片逐出、offload）每轮都跑；昂贵的 LLM 总结只在阈值附近动用，且本身有输入阶梯、重试和失败抑制。
- **KV-cache 稳定压倒一切**：leading system message、memory 块、AGENTS 段一旦写入就冻结复用，绝不在每轮重排——否则整个下游对话的 prompt cache 全部失效。

## 2. 上下文有哪些来源

每个发给模型的请求由以下成分拼成，各有独立生命周期：

| 成分 | 写入时机 | 载体 | 关键点 |
|------|----------|------|--------|
| system prompt | 会话初始化 `initialize` | `ConversationItem::system` | 常驻第 0 项，压缩后原样保留 |
| skill/workflow listing | 初始化 + 发现变化时 | `system_reminder` user item | `inject_baseline_skill_reminder` 幂等重写，避免重复 |
| 用户消息前缀 | 首个用户输入时，MCP 握手宽限后 | `user_meta` | 见 §3，workspace/user 规则分桶渲染 |
| memory context | 首轮一次（atomic guard） | `system_reminder` 内 `<memory_context>` | 见 §4，复用不重排 |
| 真实用户轮次 | `push_user_message` | `ConversationItem::user` | 写边界会做完整性修复 |
| assistant/tool | 采样循环 | `assistant`/`tool_result` | 带 `prompt_index`，压缩锚点用 |
| 各类 reminder | 合成 user item `SyntheticReason` | interjection、todo、监控、调度器、stop gate | `is_real_user_turn` 过滤后不会污染"最后真实 query" |
| MCP/技能动态公告 | 变化时 | `system_reminder` | 变化才注入，不是每轮重发 |

`turn.rs` 的轮首顺序：drain interjection → drain pending reminders → 首轮 memory 注入 → two-pass prefire 触发判定 → auth 刷新 → `check_auto_compact_needed` → 长推理 reminder → `build_request` → `run_turn_via_sampler`。

## 3. 用户消息前缀与规则分桶

`prompt_build.rs::build_user_message_prefix` 把每次用户输入包成结构化前缀：

- **模板渲染失败回退**：配置模板渲染失败时退回 legacy 前缀格式，保证会话不会死于模板错误。
- **规则分桶**（`gather_partitioned_rules` → `partition_rules_by_scope`）：规则文件按来源分成 workspace 桶和 user 桶，分开渲染、分开标注权限边界。`AGENTS.md`/`CLAUDE.md` 等已知指令路径与用户 glob 配置都参与分类——不是无脑拼接。
- **追加发现**：`agents_md_user_reminder`（`xai-grok-agent/prompt/context.rs`）在模板已内嵌规则时返回 `None`，避免 AGENTS.md 双份注入。
- **前缀的压缩稳定性**：前缀作为 `user_meta` 存进历史，压缩重建时原样保留（§8 组装顺序第 2、3 项）。

## 4. Memory 检索与注入

### 4.1 两套 pipeline

`xai-grok-memory` 同时存在两代：

- **legacy**：sqlite 持久化，FTS5 BM25 + 可选 sqlite-vec KNN 混合搜索，预压缩 flush 把要点写成 session log。
- **v2**：文件系统 topics + observation inbox + manifest + 词法索引 + dream 巩固；首轮注入的是 manifest 清单而不是检索片段，模型需要时用 `memory_search`/`memory_get` 工具自助取细节。

### 4.2 legacy 混合搜索（`search.rs`）

```
query → FTS5 关键词召回（必有）
      → sqlite-vec KNN（有 embedding 才走）
      → 按 chunk_id 合并 → 分数归一化
      → 跳过空壳/样板 chunk
      → 时间衰减（仅 session 级材料；global/workspace 豁免）
      → source 权重 × 访问频次加成
      → min_score 过滤 → MMR 去重打散 → max_results 截断
```

向量不可用则静默退化为纯 FTS（text weight=1.0）。无 panic、无硬依赖——记忆是增强项不是关键路径。

### 4.3 首轮注入：一次、冻结、可复用

`memory_context.rs` + `turn.rs` 的首轮路径（`context_injected` atomic 保证每会话一次）：

- **v2**：`regenerate_scope_manifest` 生成 global/workspace 两份 manifest（`V2ManifestBudget::default()`/`.compact()`），合进一个 `<memory_context>` 块。
- **legacy**：以最近一条用户消息为 query 搜出 Top-N，格式化成 reminder：每条带 score、source、文件路径与行号、500 字截断、时效性说明。
- **已注入就冻结**：`conversation_has_memory_context` 检测到 leading system 里已有 memory 块时**原样复用**——注释写得很直白："重新打分会变 system 前缀，毁掉整个下游的 KV cache"。
- **显式的非权威声明**：注入文本明确说"memory 是历史上下文，不自动等于当前计划"，并指示模型用 live 工具核实路径/命令/仓库状态/外部事实。`memory_search` 工具描述重申同一要求。

### 4.4 模型自助检索

`xai-grok-tools/.../memory/` 提供 `memory_search`（语义+关键词查记忆片段）和 `memory_get`（按行号取整文）——v2 模式下这是主要的 memory 读取方式；典型场景覆盖"压缩后找回细节"。

## 5. 每轮请求组装：`build_request`

`ChatStateHandle::build_request`（文档原话："Prunes, repairs, injects memory, and returns a ready-to-send request"）在 actor 内串行执行 `build_conversation_request`：

1. **memory reminder upsert**：首轮注入过的块持久进 system 项；本次首启则注入本地请求级副本。
2. **图片预算**（`image_budget.rs`）：请求体超阈值时逐出旧图片直到降到 reclaim 水位——**hysteresis 双阈值**避免在边界反复抖动造成缓存失效。被逐出的图片替换成模型可见的占位说明（"图片已不可用，需要请重新提供"），工具结果图片则在内容尾部追加说明。
3. **完整性修复**（`mutations.rs`）：去重 tool_result、剥离孤儿 tool_result（无对应调用的）、为悬挂 assistant tool_call 合成 tool_result。**写边界幂等修复**；只读查询不修复，因为正在执行的工具调用看起来恰好是"悬挂"的。`BuildConversationRequest` 在构造前跑一遍修复——**assistant tool_call ↔ tool_result 的配对是供应商 API 的硬约束**。
4. **tool_result 裁剪**（`request_builder.rs` + `PruningConfig`）：估计总用量超窗口 50% 才启动——保护近 3 轮不动；>4000 字符的 tool_result 软裁剪（头 1500 + `[…trimmed…]` + 尾 1500）；≥10 轮前的直接硬清为 `[Tool result omitted — too old]`。**只改请求副本，不改历史**。
5. **角色标注与截尾**：`ModelRequestHistory::from_raw` 标注 AgentMessage 角色；截掉尾部未配对的半截 tool_call。

## 6. Token 计量：到处用的 bytes/4

`xai-token-estimation` 提供两个全局常数：`BYTES_PER_TOKEN = 4`、`IMAGE_TOKEN_ESTIMATE = 765`。这个粗糙启发式被全链路共用：上下文占用显示、自动压缩门槛、preflight 溢出检测、`/context`、压缩计划、输入阶梯裁剪。

`ChatStateActor`（`actor/state.rs`）按类别估算 item：system/user/tool_result 按文本字节，assistant 另计工具参数字节，backend tool call 序列化计入，user 图片按张数加 765。actor 同时维护**模型回报的权威 total**与**回报后的估计增量**，estimated_total = reported + delta——工具输出在下一次采样前就推高估算，preflight 才能提前拦截。

## 7. 大输入的处理：prompt offload

单条用户输入超限（`LARGE_PROMPT_THRESHOLD = READ_FILE_MAX_TOKENS × 4` bytes）时 `prompt_offload.rs`：

- 全文写到 `prompts/prompt_{n}.txt`，模型收到有界 head/tail 摘录 + 指向文件的稳定 offload notice；写失败则给失败 notice，绝不指虚空路径。
- 摘录预算分配保尾部问题（tail budget 4000B），query 约占 80%；skill/context 段各有 4000B 下限，防止巨大输入挤掉必要上下文。
- 同样思路体现在压缩输入阶梯（§9）和 segment 截断（§10）——**任何位置都不允许无界文本直连模型**。

## 8. 压缩触发与抑制

### 8.1 配置与门槛

`CompactionPolicy`（`xai-grok-agent/compaction.rs`）：`auto_compact_threshold_percent`（默认 85%）、`compact_model`（默认跟随当前模型）、`memory_flush_enabled`（默认关）、`wall_clock_budget_secs`（默认 300s）、`two_pass_enabled`（默认关）。

门槛解析六级优先级（`util/config/resolve/compaction.rs`）：`GROK_AUTO_COMPACT_THRESHOLD_PERCENT` env → 用户/管理 TOML `[model.<id>]` → 用户 TOML `[session]` → 远端按模型 → 远端全局 → 默认 85%。`0..=100` 之外的非法值直接丢弃。墙钟预算：`GROK_COMPACTION_WALL_CLOCK_SECS` → 远端 → 客户端默认，`0` 表示关闭。模式与细节另有 `GROK_COMPACTION_MODE`（Summary/Transcript/Segments）和 `GROK_COMPACTION_DETAIL` env。

### 8.2 触发源

`session/compaction.rs` 里五种入口汇到 `run_compact_inner`：

- **阈值触发**：轮首 `check_auto_compact_needed` 估算超门槛。
- **preflight**：工具结果把估算推过窗口上限 → `CONTEXT_OVERFLOW_PREFLIGHT`，不等下一次采样失败先压缩。
- **采样报错**：context-overflow 类错误触发 compact-and-resubmit。
- **模型切换**：新窗口更小时主动压。
- **手动**：`/compact`（可带用户补充说明进摘要 prompt）。

检查逻辑带短路：memory flush 进行中跳过、抑制状态命中跳过、debug force_compact 直通；每次检查同步刷新 UI 的 context-usage 信号。

### 8.3 失败分类与抑制

`compaction_config.rs` 的抑制状态机：`SUPPRESS_NONE/TURN/STICKY/UNTIL_SUCCESS/AUTH`。失败按 `SuppressReason` 分类（CreditBlock/Auth/Size/Schema/Other）：额度与认证类立刻粘滞抑制+用户提示，context-size 类进入阶梯降级而非重试，瞬态错误（空响应、429、断流）按重试策略走。自动压缩被抑制时发 `AutoCompactSuppressed` 遥测 + 用户通知；**手动压缩豁免抑制**——用户明确要压，系统不拦。取消走 `CompactCancelGate`（holder 计数），Esc 可以打断。

## 9. 全量替换压缩：采样与降级

引擎在共享 crate `xai-grok-compaction/src/code_compaction/`，宿主（shell）负责传输、持久化、状态替换——`build prompt → sample → retry/classify → clean → assemble`。

### 9.1 摘要采样

`sample_summary_with_retries`（`sample.rs`）：默认 `FullReplaceConfig` 3 次尝试、3s 间隔、120s 超时。三种尝试结果：

- **成功**：非空且过退化检查——`is_degenerate_summary`：清洗后 <500 字符视为退化，按瞬态重试。
- **瞬态**：空响应/断流 → 重试。
- **确定性**：`is_context_length_error` 命中（含多种 provider 措辞、"request too large" 锚定、413、size slug，且特意排除渲染过的 429 以免 Retry-After 误判）或 `is_deterministic` → **不重试同输入**，返回让上层降阶梯。

### 9.2 输入阶梯（`compaction_utils.rs` + shell 侧 ladder）

压缩请求本身也可能超窗口，输入按信息保真度分三级逐级降级：

| 级 | 准备函数 | 保留 | 剥离 |
|----|----------|------|------|
| verbatim | `prepare_conversation_for_verbatim_summarization` | 工具 I/O、图片 | reasoning（可选剥离防 provider 拒收）、尾部悬空 tool_call |
| verbatim fitted | 同上 + `fit_conversation_to_budget` | 预算内最近轮次 | 整轮丢弃、对齐工具边界 |
| lossy | `prepare_conversation_for_summarization` | 对话正文 | 全部 tool_result/backend call（assistant 调用降级成工具名文本标记）、图片、reasoning |

预算：`SUMMARY_BUDGET_RESERVE_TOKENS = 32_768` 留给输出，lossy 预算再压到窗口 70%。`context_overflow` 类失败 → 降一级重进 `sample_full_replace_summary`；其他确定性失败 → 按 §8.3 抑制。

### 9.3 摘要 prompt

两种变体（`code_compaction/prompt.rs`）：结构化长版（9 节：用户请求、已完成工作、文件/代码细节、错误与修复、待办……grok-build 固定用这个）和短版 self-summarization。`/compact <text>` 的用户上下文会拼进结构化 prompt。

## 10. 两段压缩（two-pass）：把 95% 移出关键路径

`two_pass.rs` + `compaction.rs` prefire 机制：

- **触发**：估算用量 ≥ 阈值 − `prefire_lead_percent`（默认 10%，`GROK_PREFIRE_LEAD_PERCENT` 可调）且无压缩在跑 → 后台 pass-1。
- **切分**：`split_conversation_for_two_pass` 按 token 权重切 ~95/5，`snap_split_idx_to_tool_boundaries` 保证不切开 tool_call/tool_result 配对。
- **pass-1**：`prefix + 压缩指令` 采样 → 提取 `<summary>` 块（至少 1000 字符才认，否则用全文）→ 截到 `TWO_PASS_MAX_NOTE1_CHARS = 60_000` → 存 `AsyncCompactionCache`（note1、prefix_len、**prefix 指纹**、模型 slug、延迟）。
- **pass-2**（压缩真正触发时，在关键路径上）：校验缓存——`fingerprint_prefix`（item 数 + 类型 tag + 文本内容的廉价 hash）不匹配说明 rewind/edit/branch 污染了前缀，弃缓存走单段；模型换了也弃。有效则构造 `system 项 + NOTE₁(user_meta) + tail + 压缩指令` 采样 → NOTE₂ 作为最终摘要；pass-1 还在跑就等它（等待时长进遥测）。

收益：压缩耗时的大头（95% 历史的总结）摊在后台空闲时间，触发时只剩尾部小输入要同步处理。

## 11. 历史重建与压缩后状态

### 11.1 组装顺序（`code_compaction/assemble.rs` + `build_compacted_history`）

替换历史按序拼接：

```
[0] 原 system 项
[1] 用户/项目信息前缀（user_meta，原样保留）
[2] AGENTS.md/项目指令 reminder（原文重注入，不依赖摘要复述）
[3] 最后一条真实用户 query（is_real_user_turn 筛掉合成轮次）
[4] 近期 verbatim 消息（保留窗口内原文）
[5] 压缩摘要（user_meta + transcript_hint 指针）
[6] system_reminder（运行时状态重建）
[7] transcript/segment 指针（如适用）
```

三条硬规则：AGENTS.md 逐字重注入（防止摘要稀释项目指令）；最后真实用户 query 逐字保留（防"接续后答非所问"）；摘要与运行态分离——摘要说"发生过什么"，reminder 说"现在是什么状态"。

`format_compact_summary` 清洗模型输出（剥 `<analysis>` 前缀），`format_compact_summary_content` 包成 "This session is being continued…" 的接续文本。

### 11.2 压缩后 reminder（`xai-grok-compaction/reminder.rs` + shell 增补）

`to_system_reminder` 重建操作态：后台任务、TODO、在跑子代理、调度循环、workflow；shell 侧 `CompactionStateContext` 增补：本会话编辑过的文件、发现的 AGENTS.md 列表、可用技能（`format_compaction_skill_listing`）、workflow、已连接 MCP、**memory 检索补注**（以最近真实 query 搜 Top-3 注入，帮模型接回被压缩掉的上下文）。Plan/goal 模式的状态段落也在这里注入。

**goal 续跑特殊处理**：`goal_compaction_user_context` 把活跃目标作为用户上下文喂给摘要器；`reseed_active_goal_after_compaction` 压缩后重植目标延续指令，先 `prune_prior_goal_continuation_directives` 清掉旧指令——防摘要意外重启旧目标。

### 11.3 持久化与可恢复性

- **checkpoint**：`compaction_checkpoints/{id}.json` —— 替换后的历史、压缩时 prompt_index、原始 user_info、需重读路径清单、schema 版本。
- **segments**（`xai-compaction-transcript` + `compaction_segments.rs`）：`Segments(detail)` 模式把原始材料切成 `compaction/segment_*.md` + `INDEX.md`（大小封顶+截断说明），摘要里附指针；`Transcript` 模式指针直接指向 `updates.jsonl` 原始转录。模型被指示用 `read_file`/`grep` 自行找回精确细节——**压缩不再是不可逆的信息销毁**。
- **请求工件**：压缩采样请求落盘供离线诊断。
- **状态替换**：`replace_conversation_for_compaction` → `ReplaceConversation{is_compaction:true}` → 重算 token 估计、delta 清零、`ConversationReset`+token 事件、rebase 进行中的 turn capture 偏移、封存 harness trace。
- **祖先前缀**（fork/resume）：`resolve_forked_compacted_history` 尝试保留继承前缀，投影超阈值则释放并记 `compaction_prefix_released`；释放后仍超限则抑制自动压缩防死循环。

### 11.4 完整性兜底

`sanitize_compacted_history`/`validate_compacted_history`（`compaction_utils.rs`）：重建历史过一遍工具配对校验，孤儿 ToolResult 剥掉，仍不合法则回退最小历史（system + 摘要 + reminder）——**宁可丢细节也不给 provider 送非法配对**。

## 12. 会话初始化时的上下文清点

`context_snapshot.rs` 在会话启动时对各类静态上下文调 `/tokenize-text` 精确分词（特意用 bake 的默认模型保证 tokenizer 兼容，而非当前会话模型），分类上报 telemetry：system prompt、工具定义、skills、MCP、AGENTS/项目指令、workflows。这让"上下文窗口被谁吃掉"可观测——工具 schema 和 MCP 广告这类隐藏大头有据可查。

## 13. 观测面

```
轮次:     turn.build_request span
启动:     session_context_snapshot（分类 token 数）
memory:   MEMORY_INJECT（注入/复用、大小）、MEMORY_FLUSH（触发/窗口/计数）
压缩触发: AutoCompactFired / CONTEXT_OVERFLOW_PREFLIGHT / AutoCompactSuppressed
压缩过程: CompactionBegin → CompactionRetryDegraded（阶梯降级）→
          尝试级 outcome（Success/Degenerate/EmptyResponse/Failure，
          标 deterministic/context_overflow/will_retry）→
          AutoCompactCompleted / AutoCompactFailed / Cancelled
两段:     session.prefire_pass1 span（compaction_prefire_outcome 枚举为稳定遥测键）、pass2 等待延迟
持久化:   checkpoint / segment / INDEX.md / request artifact
特殊:     compaction_prefix_released（继承前缀释放）
```

## 14. 设计权衡与边界

- **估算是启发式**：bytes/4 对代码偏乐观（真实 tokenizer 一般更高）；系统靠"authoritative reported total + delta"和 preflight 双保险兜底，不靠估算单点正确。
- **memory 非权威**：所有注入路径都明确要求 live 工具核实——压缩补注同样遵守。
- **保 cache 优先于保新**：memory 块冻结复用、AGENTS 独立成项、图片 hysteresis、前缀持久化，全部为 KV-cache 稳定服务；代价是注入的 memory 会随会话变旧。
- **tool 配对是不可协商的正确性**：写边界修复、切分对齐、重建校验三处独立守这条不变式。
- **失败是分类的不是均质的**：同输入的确定性错误绝不重试；抑制状态细分（turn/sticky/until-success/auth）区分可恢复与死路。
- **默认较保守**：memory flush、两段压缩、verbatim 输入（远端旗标）默认关——这些增强逐项灰度。

## 15. 端到端时序小结

```
会话建立:  system + skill listing → 后台 prefix(MCP 宽限) → 上下文清点遥测
首轮:      memory 注入/manifest → build_request → 采样
每轮:      drain 注入物 → prefire 判定 → auto-compact 检查 → build_request
           (修复→图片预算→裁剪→截尾) → 采样 → 工具执行回填
接近阈值:  (可选) memory flush 提取要点 → (可选) pass-1 后台总结 95%
到阈值:    run_compact_inner → 输入阶梯 → 采样(重试/降级) →
           sanitize/validate → 组装(system+prefix+AGENTS+query+summary+reminder)
           → checkpoint/segments → replace_history → goal reseed → 继续轮次
后续:      模型按 transcript/segment 指针与 memory 工具自行找回细节
```

## 16. 源码索引

| 主题 | 路径 |
|------|------|
| 轮次与 build_request 调用点 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/turn.rs`、`sampler_turn.rs` |
| 请求组装/裁剪/图片预算 | `crates/codegen/xai-chat-state/src/{handle.rs,actor/request_builder.rs,actor/mutations.rs,actor/state.rs,image_budget.rs,types.rs}` |
| 完整性修复 | `crates/codegen/xai-chat-state/src/actor/mutations.rs` + `compaction_utils.rs` |
| 前缀与规则分桶 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/prompt_build.rs`、`session_setup.rs` |
| 上下文清点 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/context_snapshot.rs` |
| prompt offload | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/prompt_offload.rs` |
| memory 注入 | `crates/codegen/xai-grok-shell/src/session/helpers/memory_context.rs` |
| memory 搜索/flush/v2 | `crates/codegen/xai-grok-memory/src/{search.rs,flush.rs,v2.rs}` |
| memory 工具 | `crates/codegen/xai-grok-tools/src/implementations/memory/{search_tool.rs,get_tool.rs}` |
| memory flush 执行 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/memory_dream.rs`、`session/helpers/memory_flush_window.rs` |
| 压缩主流程 | `crates/codegen/xai-grok-shell/src/session/compaction.rs`、`compaction_segments.rs`、`helpers/full_replace_compaction.rs`、`crates/codegen/xai-compaction-transcript` |
| 压缩配置/抑制 | `crates/codegen/xai-grok-shell/src/session/compaction_config.rs`、`util/config/resolve/compaction.rs`、`crates/codegen/xai-grok-agent/src/compaction.rs` |
| 两段压缩 | `crates/codegen/xai-grok-shell/src/session/two_pass.rs` |
| 共享压缩引擎 | `crates/common/xai-grok-compaction/src/code_compaction/{sample.rs,failure.rs,prompt.rs,assemble.rs}`、`reminder.rs` |
| 摘要输入准备 | `crates/codegen/xai-chat-state/src/compaction_utils.rs`、`compaction_mode.rs` |
| token 估算 | `crates/codegen/xai-token-estimation/src/lib.rs` |
| 规则渲染/AGENTS reminder | `crates/codegen/xai-grok-agent/src/prompt/context.rs` |
| 技能清单格式化 | `crates/codegen/xai-grok-tools/src/types/skill_discovery_tracker/listing.rs` |
