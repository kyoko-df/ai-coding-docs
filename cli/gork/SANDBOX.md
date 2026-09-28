# Grok Build 沙箱与安全体系解析

> 本文说明 grok 的沙箱与安全设计：**不是单一 sandbox，而是一条从进程入口到工具调用的多层防线**。
> 姊妹篇：`INNER_LOOP.md`（单轮采样-工具循环）、`ORCHESTRATION_AND_TOOLS.md`（编排与工具能力）。
> 本文关注「模型生成的动作在落地前被哪些层检查、以什么机制强制执行」。

## 1. 总览：六层防御

```
用户 / 配置层                        策略层（用户态判断）              内核层（不可绕过）
─────────────────────────────────────────────────────────────────────────────────
GROK_CONFIG env overlay ──┐
managed-settings.json     │        ┌─ permission manager ─┐        ┌─ Seatbelt (macOS)
folder trust 决策 ────────┼───────▶│  AccessKind 化简       │        │  Landlock (Linux)
permission.toml grants    │        │  rules/grants/policy   │        │  bwrap 再执行 (Linux)
钩子文件写保护             │        │  bash tree-sitter 分解  │───────▶│  seccomp 子进程网络锁
                          │        │  protected-target 底线  │        │  mount 命名空间封锁
                          │        │  auto-mode LLM 分类器   │        │  runtime socket 掩码
                          │        └────────────────────────┘        └─────────────────
                          ▼
                  ┌─── 工具调用门禁（每次 tool call）───┐
                  │  PreToolUse hook → permission.request │
                  │  → 执行 → PostToolUse hook            │
                  └──────────────────────────────────────┘
```

关键设计取向（贯穿全库）：

- **Fail-closed**：解析不了的 shell、扩展不了的 deny glob、验不了的挂载、判不了的路径——一律拒绝或升级为人审，而不是放行。
- **权限 ≠ 沙箱**：权限层（permission）决定「要不要问用户」，沙箱层（kernel enforcement）决定「就算骗了权限问不出来也做不了」。两者独立叠加。
- **不可重入**：`Sandbox::apply` 不可逆；bwrap 通过环境标记 + 挂载验证防止二次进入或伪造。
- **自我防护**：agent 自己的配置文件（hooks、grok config、sandbox.toml、.claude settings）被列为受保护编辑目标，防止 agent 改写自己的权限规则。

## 2. 进程级沙箱 `xai-grok-sandbox`

代码：`crates/codegen/xai-grok-sandbox/src/`（约 6100 行 + 测试）。底层依赖 `nono = "=0.53.0"`（锁版本——注释说明 nono 的规则发出顺序被依赖，升级可能静默改变语义）。

### 2.1 Profile 体系

`profiles.rs` 定义五个内置 profile，`resolve()` 在 `ProfileName` 上把名字展开为 `SandboxProfile{read_only, read_write, deny, write_deny, default_read, restrict_network}`：

| Profile | 读 | 写 | 网络（Linux 子进程） | 备注 |
|---------|----|----|---------------------|------|
| `off` | 不限 | 不限 | 不限 | `resolve` 时 bail——`off` 不能作为 extends 的 base |
| `workspace` | 全盘 | CWD + `~/.grok` + temp | 不限 | 开发默认推荐 |
| `devbox` | 全盘 | 根目录下除 `/data` 外所有一级目录 | 不限 | `/data` 是 read_write 排除而非 kernel deny；hook write-deny 也跳过 |
| `read-only` | 全盘 | `~/.grok` + temp | **阻断** | `essential_writable_paths_minimal` |
| `strict` | 显式白名单（`/usr /lib /bin /etc /dev /proc /sys /tmp /run /var` + `~/Library` + `grok_home` + CWD） | CWD + `~/.grok/sessions` + temp | **阻断** | `default_read=false`；grok_home 本身只读，只有 `sessions/` 可写（事件 JSONL 落这里） |

自定义 profile：`~/.grok/sandbox.toml` 和 `<workspace>/.grok/sandbox.toml` 的 `[profiles.<name>]`，`extends = "workspace"` 等（不允许继承 `off` 或其他自定义 profile）。字段：`restrict_network`、`read_only`、`read_write`、`deny`。

**解析优先级**（`agent/config.rs::SandboxSettingsConfig::resolve_profile`）：`requirement > CLI --sandbox > GROK_SANDBOX env > config.toml > "off"`。`auto_allow_bash` 同法解析（`GROK_SANDBOX_AUTO_ALLOW_BASH`）。

### 2.2 Allow path 规范化（`allow_path.rs`）

`read_only`/`read_write` 是**字面目录授权**，安全敏感的解析独立成小模块便于审计：

- 只剥一种尾随 glob（`/**`、`/**/`、`/**/*`、`/*`）到父目录——用户明显意图是"这个目录"。
- 剥完仍是 glob 形状的条目被**丢弃并告警**，绝不放宽。
- 前后空白不 trim——空白在字面路径里有意义；含空白的条目丢弃并告警。
- `deny` 条目不走这条路（glob 由 deny/glob 模块处理）。

### 2.3 Linux：Landlock + bwrap 再执行 + seccomp 三段式

Landlock 本身是**纯 allow-list**（不能 deny 已授权树的子路径），所以 Linux 上要 deny 需要 mount 技巧：

```
main() (xai-grok-pager-bin/main.rs:2200)
  → apply_sandbox(profile, cwd)                          config/mod.rs:1458
    │  Linux 判定 requires_read_deny / requires_hook_write_deny / requires_data_write_deny
    ├─ 需要 deny 且不在 bwrap 里 → bwrap_reexec_command_ex()
    │     cmd.exec() 用 bwrap 重启本进程：
    │       --cap-drop ALL / --bind / / / --ro-bind deny_write /
    │     hook plan binds / deny_read 的 mode-000 占位符 bind-over /
    │     sentinel 目录 ro-bind / --dev-bind /dev / --proc /proc
    │     env __GROK_INSIDE_BWRAP=1 + __GROK_BWRAP_RUNTIME_SOCKET_DENY=<encoded>
    ├─ 已在 bwrap 里 → BwrapStartup::Verify：
    │     verify_hook_write_deny_enforced() / read_deny_verify 逐项核对 mountinfo
    │     发现哨兵缺失或可写 → "possible __GROK_INSIDE_BWRAP spoof"，拒绝启动
    └─ Sandbox::apply(workspace)                           lib.rs:181
          ensure_grok_hook_slots（需要时先建 hook 槽位）
          maybe_install_namespace_lockdown_inside_bwrap()   ← 进 namespace 后立刻装 seccomp
          resolve_profile → to_capability_set → nono::Sandbox::apply(caps)
          失败 → apply_failed 事件 + warn，继续运行（降级但记审计）
```

几个值得注意的细节：

- **deny_read 的 bind-over 占位符**（`bwrap_blocked_placeholder`）：在 `~/.grok` 下创建 `chmod 000` 的文件/目录（文件名带 PID 后缀防并发竞争），`--ro-bind` 到目标路径上，内核层面让目标读返回 EPERM。占位符创建失败 → 拒绝启动，不做半套沙箱。
- **哨兵目录**（`read_deny_verify.rs`）：`sandbox-bwrap-sentinel` 总是被 ro-bind 进 namespace；验证要求它是精确的只读挂载点。`__GROK_INSIDE_BWRAP` 环境变量本身可被伪造，所以验证不看它，看 `/proc/self/mountinfo` 的挂载属性和 statx 的 mount_id/文件类型。
- **namespace 封锁**（`child_net.rs` 的 `ns_lockdown` 模块）：经典 BPF seccomp 过滤器，对 `unshare`/`setns`/`clone3`/`clone` 带 namespace flag（NEWNS/NEWCGROUP/NEWUTS/NEWIPC/NEWUSER/NEWPID/NEWNET/NEWTIME）返回 EPERM，x32 ABI 位也封。这样进程进了 namespace 之后无法再逃逸或改动自己的 mount namespace——前面验证过的只读挂载无法被 `umount` 回滚。
- **运行时 socket 掩码**（`runtime_sockets.rs`）：`/run`、`/var`、home 在受限 profile 里仍可读（DNS/NSS 需要），导致容器 runtime API socket（docker/podman/containerd，含 per-uid 和 `~/.docker` 端点）和 D-Bus/systemd 私有 socket 路径可达。启动时对这些已知 socket 做 deny mount（防御纵深，盖不住启动后新建的）。**容器 runtime 的真正兜底是每条子进程都装的网络 seccomp**——拦住 socket 连接比拦文件路径更根本。D-Bus/systemd 掩码挡的是"通过总线让树外 systemd 起 unit"。
- **`/data` 写拒绝**（devbox）：Linux 上 devbox 用 bwrap `--ro-bind /data /data` 实现（Landlock 无法 carve out），所以自定义 profile `extends = "devbox"` 也会走 bwrap 路径（`is_devbox_based`）。

### 2.4 macOS：Seatbelt

- deny 路径通过 `nono::CapabilitySet::add_platform_rule` 发出 Seatbelt `(deny file-read* ...)` 规则。
- **路径别名全覆盖**（`deny/mod.rs::macos_deny_aliases`）：macOS 的 `/tmp` ↔ `/private/tmp` firmlink 别名能绕过单路径 deny，所以每个 deny 路径展开为：原始路径 + canonical 化路径 + 各自的 `/private` 前缀变体，全部写进规则。
- Seatbelt 路径字面量转义拒绝控制字符——转义不放行的话会指向错误路径（静默放行比报错更糟）。
- **glob deny 在 macOS 是运行时正则**：`deny/glob.rs` 把 glob 翻译成锚定 Seatbelt regex，启动后新建的文件也覆盖（对比 Linux 是启动时 expansion，只管已存在的）。两侧用同一个 `validate_deny_glob` 保证接受/拒绝集一致，并有跨平台 property test 断言同一份 glob 翻译结果相同。

### 2.5 Linux glob deny：启动时展开（`deny/glob.rs`）

- `DENY_GLOB_CAPS`：`depth=64`、`matches=4096`、`entries=2,000,000`——遍历预算封顶，超限报错而不是截断静默放行。
- 展开走 `ignore` crate 的 walker（含 gitignore/hidden）；`PermissionDenied` 的遍历错误可跳过（OS 反正也拒绝），其他错误 fail-closed。
- 每个命中项变成一个 `--ro-bind`；上限 4096 受 bwrap argv 预算约束。

### 2.6 Hook 文件写保护（`hook_write_deny.rs`）

Hooks 是会话间持久存在的可执行配置——如果 agent 能写自己的 hook 文件，等于写自己未来的权限规则。所以非 devbox 的强制 profile 要求全局 hook 源（`~/.grok/hooks/`、`~/.claude/settings.json` 等 `GlobalHookSource`）在沙箱内**只读**：

- `resolve_write_deny` 把 hook 源编成 `write_deny` 列表（Landlock 的写拒绝在能力集层做；Linux bwrap 用 `--ro-bind`）。
- 进入 namespace 前的 **identity revalidation**：每个受保护路径检查 `IdentityChanged`（rename race）、`Symlink`（可重定向）、`HardLink`（st_nlink>1 → 有别名可写入口，宁可拒绝启动）。
- `hooks-paths` 配置指向的绝对路径若不存在，报 `MissingConfigured`——要求用户在沙箱外创建或移除配置，而不是静默跳过保护。

### 2.7 子进程网络限制（`child_net.rs`）

`should_restrict_child_network()` 只看解析后的 `restrict_network` 标志，**不看 Landlock 是否生效**——设计上这是故意的：

> 在降级态（Landlock 不支持、或 bwrap 里 apply 失败）这个过滤器是仅剩的执行层。按 Landlock 成败做开关会静默关掉会话级的子网络控制；按配置做是 fail-closed 方向。

`restrict_child_network(&mut cmd)` 在 fork 后 `pre_exec` 挂预构建的 seccomp BPF（只跑 `prctl`，async-signal-safe）。所有已知子进程启动点都挂了它：

- `xai-grok-tools/.../terminal.rs`（4 处 spawn 点）
- `static_shell.rs`、`shell_state.rs`（持久 shell 后端）
- `lsp/client.rs`（LSP 服务器子进程）
- `xai-grok-workspace/envrc.rs`（envrc 求值子进程）

macOS 上是 no-op（Seatbelt 不支持网络域，文档明示）。

### 2.8 事件审计（`logging.rs` + `types.rs`）

`SandboxEvent`（ProfileApplied/ApplyFailed/FsViolation/NetViolation/BypassGranted/BypassDenied）带 profile、workspace、platform、restrict_network、allow/deny 路径快照，内存里攒，退出时 `flush()` 写 `~/.grok/sessions/sandbox-events.jsonl`。strict 下靠 `sessions/` 的写授权落盘（不在 grok_home 父目录上授权，防 symlink 重定向）。

### 2.9 启动期整合与坑位

- **Leader 不可用**（`leader/mod.rs:1376`、`main.rs:1358`）：请求了非 `off` profile 时拒绝复用或启动 leader——leader 是 socket 发现+长驻进程，复用未受限的旧 leader 等于整个沙箱白做。
- **`grok workspace` 子命令受限**（`main.rs:585`）：sandbox 下 `start/restart/resume` 拒绝执行。
- **会话恢复校验**（`main.rs:2153`）：`startup_sandbox_profile(saved)`——resume 的 session 若存了不同 profile，报错要求不带 `--sandbox` 用原值、或显式用匹配值。防止"创建时 strict、恢复时 off"的静默降级。
- **托管策略先行**（`main.rs:91` `command_needs_pre_sandbox_policy_heal`）：`apply_sandbox` 之前先把 managed policy 落盘——进了沙箱就写不动了。
- **workspace_server**（`xai-grok-workspace/src/bin/workspace_server.rs:365`）：`trust_bwrap_marker_for_devbox()` 允许 devbox profile 下信任 bwrap marker（devbox 是全盘写，marker 伪造无收益）。

## 3. 权限层 `xai-grok-workspace/src/permission/`

权限层决定「这个工具调用要不要问用户」——是策略判断，不是执行机制（执行靠 §2 的内核层兜底）。

### 3.1 AccessKind：工具输入 → 访问类型

`types.rs:254` `From<&ToolInput> for AccessKind` 把每个工具输入化简成权限视图：

```rust
enum AccessKind {
    Read(Option<String>),             // read_file/list_dir/memory_get/lsp 等
    Grep { path, glob },              // grep 族
    Edit(String),                     // search_replace/hashline_edit/write
    Bash(String),                     // bash 与 monitor（monitor 的 command 走 bash 评估）
    MCPTool { name, input },          // mcp_tool 和 use_tool(inline)
    WebFetch(String), WebSearch(String),
    AgentMessage { subagent_id },
    Tool(String),                     // 无 grant scope 覆盖的杂项变异类：task/scheduler/workflow/
                                      // image_gen/apply_patch/send_feedback —— 每次都问
}
```

要点：**`AccessKind::Tool` 是 fail-closed 桶**——任何没被显式分类的变异工具都落进去，没有 grant scope 能覆盖它，每次都提示。只读白名单只有 `Read/Grep/WebSearch` 三个免问（`manager/mod.rs:1164`）。

### 3.2 权限模式（`rules.rs::DefaultPermissionMode`）

| `permissions.defaultMode` | 效果 |
|---|---|
| `default` / `plan` | 正常询问 |
| `acceptEdits` | 编辑自动允许（受保护目标仍问，见 §3.5） |
| `auto` | LLM 分类器仲裁（`PromptPolicy::Auto`） |
| `dontAsk` | 提示即拒绝（`PromptPolicy::Deny`）——"别烦我=别做" |
| `bypassPermissions` | yolo：全放行 |

未识别的字符串 `FromStr` 失败、落回 `default`，但**仍认领该 settings 层级**——打错的 mode 会挡住外层更宽松的 mode（错配不会静默放宽）。

### 3.3 决策顺序（`manager/mod.rs`）

`PermissionManager` 是一个 actor（`PermissionCommand` 通道），一次 `request()` 的判定链大致是：

```
GatePreflight::evaluate (gate_preflight.rs)
  │  managed policy 直判 + bash_command gate + shell_file gate 一次算完
  │  Reject > AskRuleMatch > AskFailClosed（combine_gate_decisions）
  ├─ PolicyDeny / Reject → 立即否决（PolicyDeny 不可被用户 override）
  ├─ Read/WebSearch/Grep 且非 plan_mode → fast-path Allow
  ├─ session_grant_pre_decision（grants.rs）
  │    MCP → mcp_pre_decision（session denylist 先拒、persisted grants）
  │    WebFetch → DomainMatcher 静态域名表 + persisted grant
  │    其余 → permission.toml 持久授权
  ├─ hook_forced_prompt → 降级为 Ask（hook 强制过的不能 auto-allow）
  ├─ protected_target → 受保护编辑地板（§3.5）
  ├─ bash 静态评估 evaluate_bash（§3.4）：段级 grant、deny-wins、环境风险
  ├─ sandbox_may_auto_allow_bash → 沙箱 active 且无 floor 触发 → Allow
  │    （auto_allow_bash=true 时；写目标/危险命令仍要 floor）
  ├─ auto 模式 → classifier_assessment → SharedClassifier
  │    LLM side-query（带 transcript + AGENTS.md）判定 allow/block/unavailable
  │    AUTO_DENY_CONSECUTIVE_LIMIT=3 / TOTAL=20 预算耗尽后回到人工
  └─ Ask → prompter.rs 发 ACP PermissionRequest 给用户，挂 PendingInteractionGuard
        结果 → record_prompt_outcome（部分授权写回 session grants）
```

自动拒绝防爆：auto 模式分类器连续拒绝 3 次或累计 20 次后关闭自动通道回人工——分类器不能无限期把用户锁在循环外。

### 3.4 Bash 静态评估（`grants.rs::evaluate_bash` 家族）

bash 是权限面最厚的部分，因为一条命令能伪装任何东西。栈：

- **`bash_command_splitting.rs`**：tree-sitter 解析 shell AST；`unwrap_wrappers`/`peel_transparent_prefixes` 剥 `env`/`command`/`nice`/`sudo -n`/`timeout` 等 wrapper 到不动点（`MAX_WRAPPER_DEPTH`/`MAX_TRANSPARENT_PREFIX_DEPTH`/`MAX_NORMALIZE_ROUNDS` 预算内剥不动 → `FailClosed`）。
- **`bash_permission_script.rs`**：对不可完全分解的脚本做恢复性扫描，保 argv 位置；`WITNESS_DENIED_KEYS` 列出能被 export 属性继承来劫持加载器的变量（`RIPGREP_CONFIG_PATH`、`LD_*`、`GCONV_PATH`、`MALLOC_TRACE`……）——写这些变量视同写代码。
- **`exec_risk.rs`**：argv 里能 spawn 程序的 flag（`find -exec`、`xargs` 变体、`git -c`……）+ **ambient git2 风险**：仓库本地 `.git/config` 里 `filter.<driver>.clean/smudge/process` 之类的 content filter 会被 `git checkout` 等操作自动执行（`git_content_filters.rs` 枚举可 pin 的 driver 名），所以 `git status` 在脏仓库里也不安全——`spawn_blocking` 里跑 git2 扫 ambient 配置。
- **`shell_access.rs`**：命令里的文件读写检测——`cat`/`sed -i`/redirect/`tee`/`dd`/创建型命令（`mkdir`/`touch`/`install -D`）操作数提取，让"读 deny 的文件不能靠 `cat` 绕过、写 deny 的文件不能靠 `>` 绕过"。无法 pin 的操作数 → Ask。
- **deny-wins**：一段脚本里任何子命令命中 deny 规则，整条拒绝；grant 只覆盖同一条命令的逐段匹配。

### 3.5 受保护编辑地板（`shell_access.rs::edit_target_protection` + `grants.rs::protected_target`）

即使 `acceptEdits`/`yolo` 开着，这些目标永远要人工确认（`ProtectedEditReason`）：

| 目标 | 原因（kind） |
|------|-------------|
| `~/.grok/hooks*`、`.claude` hooks、`cursor` hooks | hook_root / cursor_hooks：写入=未来任意会话无审批执行 |
| `.git/hooks/` | git_hooks：commit/push 自动触发 |
| `~/.ssh/` | ssh：凭据与认证 |
| shell 启动文件（`.bashrc`/`.zshrc`/`profile`…） | startup_file：未来终端自动执行 |
| `/etc` | etc：系统级 |
| `~/.grok` 配置（config.toml/permission.toml） | grok_config：改权限规则本身 |
| `sandbox.toml` | grok_sandbox：放宽沙箱本身 |
| `.claude/settings.json` 等 | claude_settings：可装 hooks/改 mode |
| 无法解析的路径、非绝对路径、`..` 残留 | Sensitive（fail-closed 兜底） |

判定同时看 **lexical normalize** 和 **resolve_following_symlinks** 两遍——symlink 指向受保护区也算命中。bash 侧的 `protected_paths`（redirect/创建型命令的写操作数）走同一地板。

### 3.6 Plan mode 的叠加门禁

`plan_mode.rs::PlanModeTracker` + `tool_calls.rs::plan_mode_edit_gate`：plan 激活时，`AccessKind::Edit` 且目标不是 plan 文件 → `RejectNonPlanFile` 直接拒；`apply_patch` 恒拒（补丁文本里的文件名不可控）；`task` 放行（子代理有自己的门禁）。退出 plan 用 `exit_plan_mode` 审批，未识别 outcome 映射为 `Cancelled`——**审批失败默认留在 plan 模式**，不放行。

### 3.7 Hub 侧门禁（`hub_gate.rs`）

`grok workspace`（远程/hub 形态）有独立的 approval gate：工具集解不出的调用不执行；`ToolApprovalPolicy`（workspace bind metadata 下发）`GrantsAllowed` vs `AlwaysPrompt` 决定走 grants 还是全问；无 transport/无应答都 deny；grants 与 TUI 共用同一份 `permission.toml`——一面给的授权两面通用。

### 3.8 MCP 的权限位

- `mcp.rs`：`MAX_ADVERTISED_MCP_TOOLS = 256` 会话上限。
- `mcp_claim.rs`：tool id 认领计划——native id 永不被 MCP 认领；first-party server（bind config 显式声明）优先于 third-party；同 tier 内同名歧义 → 谁都拿不到（不允许启动顺序决定归属）；cap 先花给 first-party，第三方 server 的工具数量挤不掉应用自带工具。
- 每个 MCP 调用映射 `AccessKind::MCPTool{name,input}`，过正常权限链；session denylist 优先于 persisted grant。

## 4. 环境变量与进程隔离

### 4.1 `shell_environment_policy`（`xai-grok-tools/src/util/shell_env_policy.rs`）

`[shell_environment_policy]` 配置控制 agent 子进程（bash 工具、终端、静态 shell）继承的 env：

- `inherit: all|core|none`（默认 all；core 只留 PATH/HOME/SHELL 等平台基础变量）。
- `ignore_default_excludes=false` 时默认丢 `*KEY*`/`*SECRET*`/`*TOKEN*`（大小写不敏感 glob）。
- `exclude` / `set` / `include_only` 依序叠加；`allows(name)` 判定同时用于 base env 和 login-shell 捕获的追加 env——两条路径共用同一组 matcher，防漂移。
- 持久 shell 后端（`shell_state.rs:301`）：policy 只过滤持久 shell 的**基础** env——运行中 `export` 的变量不再被重过滤（代码里明确 warn 这个边界）。

### 4.2 进程组与终端隔离（`spawn.rs`/`env.rs`/`xai-tty-utils`）

- `ProcessGroup`/`new_process_group`/`detach_command`/`detach_from_tty`/`pager_env`：agent 子进程进独立 process group、脱离 controlling tty——kill 整组不会误杀用户 shell，Ctrl-C 语义清晰。
- `GROK_AGENT=1`（`apply_grok_agent_marker`）：给 agent spawn 的终端打标，宿主工具（如 `x ban`）能区分 agent 调用和真人交互 shell；request/login env 不能清掉这个标记。
- `reap_killed_search_child`：kill 后有界等待 reap，超时交给孤儿 reaper 而不是泄漏 zombie。

### 4.3 配置注入：`GROK_CONFIG`/`GROK_CONFIG_PATH` overlay

`xai-grok-config/env_overlay.rs`：env 注入的 JSON/TOML overlay 在受控时点读（`MAX_OVERLAY_BYTES=4MB` 上限防 `/dev/zero` 型特殊文件把 agent 读死），`expand_env_vars_in_toml` 对配置值做 `$VAR` 展开。这是 env 层的另一条输入面：因为能被外部设置，所以它只能收紧不能放宽地叠在解析链上。

## 5. 信任与注入防护

### 5.1 Folder trust（`agent/folder_trust.rs`）

「信不信任这个文件夹」一次性门禁，**在任何 repo-local 进程 spawn 之前**判定。被门控的配置源：`.mcp.json`、`.grok/config.toml` 的 `[permission]`/`[mcp_servers]`/`[plugins].paths`、`.grok/lsp.json`、`AGENTS.md`/`CLAUDE.md`、`.grok/skills`、`~/.claude.json` 的 `projects.<cwd>`——都是"仓库里夹带的可执行配置"，克隆即 1-click RCE 的入口。

- 判定/持久化在 `xai-grok-workspace::folder_trust`（`~/.grok/trusted_folders.toml`）；shell 侧只消费 `DECISIONS` 缓存。
- 两种 **provisional（不缓存的）allow**：没有 repo 配置的文件夹（之后 `git pull` 进来的配置下次重新判定）、以及不可持久化 key 的 cwd（$HOME/文件系统根）。
- 与 plugin trust（`~/.grok/trusted-plugins`）**互相独立**，任一授权不蕴含另一面。

### 5.2 Managed policy（`permission/managed_policy/`）

企业管控面：`managed-settings.json` + 各 `managed_config.toml`/`requirements.toml` 层，**strictest-wins** 合成（任何 deny 都赢、受限源必须全部允许、pin 只收紧）。覆盖 MCP server 允许列表、marketplace 源、`enableAllProjectMcpServers`、`permissions.defaultMode` 钉死、hooks 源。

### 5.3 间接注入面的硬化细节

| 面 | 机制 | 位置 |
|----|------|------|
| git content filter | `filter.<driver>.clean/smudge/process` 是 git 自动执行的代码，进 exec-risk 评估 | `git_content_filters.rs` |
| git ambient config | repo-local `.git/config` 在评估命令时 `spawn_blocking` 扫描 | `exec_risk.rs` |
| hook 写保护 | hook 文件沙箱内只读 + 硬链接/符号链接/rename 竞争检查 | `hook_write_deny.rs` |
| Unicode 混淆字符 | 窄表（smart quotes/dash/nbsp）只用于匹配对比不回写，防止文件名"看着像"绕过 | `util/unicode_confusables.rs` |
| 读路径解析 | canonicalize → unicode 文件名兜底 → gitignore 过滤 | `util/read_policy.rs` |
| MCP 工具名 | 冲突仲裁 + 256 上限 + first-party 优先 | `mcp_claim.rs` |
| web_fetch 域名 | `DEFAULT_ALLOWED_DOMAINS` 静态白名单 + `DomainMatcher`（x.ai、docs.rs、MDN 等文档域） | `web_fetch/config.rs` |
| 计划文件守卫 | `PlanGuard` RAII 快照/恢复 `plan.md`，symlink 篡改走 telemetry | `goal_strategist.rs` |

## 6. 工具调用路径上的门禁链（端到端）

把一次 `bash("rm -rf build")` 串起来：

```
prepare_tool_call (tool_calls.rs)
  1. access_kind_for_resolved_tool → AccessKind::Bash(cmd)
  2. plan_mode_edit_gate          → plan 激活时非 plan 文件编辑直拒
  3. mcp_preparation.approval     → MCP 专属审批/命名空间
  4. PreToolUse hook              → 可 block / 强制 ask（hook_ask 标记降级 auto-allow）
  5. plan_file_auto_approve       → plan 文件的编辑是唯一模型可自批的写
  6. permissions.request          → §3.3 的整条链
       Decision::Allow            → 执行
       Ask                        → ACP permission prompt（等用户，计 permission.wait 遥测）
       Reject/PolicyDeny          → ToolLoop::Continue + permission_denied hook 事件
       FollowupMessage(msg)       → 把用户改写后的指令注入会话（不执行原调用）
       Cancelled                  → 级联取消本轮剩余 tool calls
  7. 执行时                       → 终端 spawn 点上 child_net seccomp +
                                   shell_environment_policy env 过滤 +
                                   ProcessGroup 隔离 +
                                   进程整体已在 §2 的 Landlock/Seatbelt/bwrap 内
```

旁路设计同样显眼：模型没有「这次跳过沙箱」的参数——`BashToolInput` 只有 `command/timeout/description/is_background`，没有任何 bypass 字段；能打开缺口的只有用户改配置、退出沙箱 profile、或开 yolo。

## 7. 失效模式与降级语义

| 场景 | 行为 |
|------|------|
| Landlock/Seatbelt 不支持 | `apply` warn + `apply_failed` 事件，继续运行（审计可知）；但 child_net seccomp 仍按 profile 装 |
| Linux deny 需要 bwrap 而 exec 失败 | **拒绝启动**（deny 列表必须受保护） |
| 已在 bwrap 但挂载验证不过 | 拒绝启动（怀疑 `__GROK_INSIDE_BWRAP` 伪造） |
| hook write-deny 预备不了 | 拒绝启动 |
| glob deny 展开超限/遍历错误 | 报错拒绝启动 |
| 权限 manager 不可用 | `Decision::Reject("permission manager unavailable")` |
| 审批 transport 断 / 无应答 | deny（`ext_method_no_client` 只把"送达后客户端消失"判为非取消） |
| exit_plan_mode 返回未知 outcome | `Cancelled`，留在 plan 模式 |
| shell 解析不动（含 heredoc/`$()` 过深/无法 pin 操作数） | AskFailClosed → 人审（auto 模式下可交分类器） |
| 自定义 sandbox profile 不存在/extends 非法 | `resolve` 报错，不进半配状态 |

一致的取向：**凡无法证明安全的地方，要么 fail closed，要么升级为带遥测的人工决定；几乎没有"猜一个宽松解释继续"的路径**。

## 8. 与其他子系统的交界

- **Subagent**：`isolation: "worktree"` 是 **git worktree 级隔离**（编辑落在独立工作区，完成后返回路径），不是内核沙箱——子代理进程继承父进程的整个沙箱（同一进程的 Landlock/Seatbelt 作用于所有 spawn）。
- **Hooks**：`permission_denied` 是可订阅事件；`pre_tool_use` 能强制把 auto-allow 降级为人工；hook 文件本身被 §2.6 写保护 + §3.5 编辑地板双重看护。
- **权限与 INNER_LOOP 的关系**：`Decision::Allow` 后执行、`push_tool_result` 回填；`Reject`/`Cancelled` 同样作为工具结果文本回填，模型看到的是"被拒"而不是进程死亡。
- **沙箱与 yolo**：`bypassPermissions` 只关掉**权限询问**那层；沙箱（内核强制）照跑——yolo 不是 `--no-sandbox`。

## 9. 源文件导航

| 主题 | 路径 |
|------|------|
| Profile 定义与解析 | `xai-grok-sandbox/src/profiles.rs` |
| bwrap 再执行 / deny 占位符 / 哨兵 | `xai-grok-sandbox/src/lib.rs`（`bwrap_reexec_command_ex`）、`read_deny_verify.rs` |
| 子进程网络 seccomp + namespace 封锁 | `xai-grok-sandbox/src/child_net.rs` |
| runtime socket 掩码 | `xai-grok-sandbox/src/runtime_sockets.rs` |
| hook 写保护 | `xai-grok-sandbox/src/hook_write_deny.rs` |
| deny glob（macOS regex / Linux 展开） | `xai-grok-sandbox/src/deny/{mod.rs,glob.rs}` |
| allow path 规范化 | `xai-grok-sandbox/src/allow_path.rs` |
| 路径表（grok_home、devices、temp） | `xai-grok-sandbox/src/paths.rs` |
| 事件与审计 | `xai-grok-sandbox/src/{types.rs,logging.rs}` |
| 启动接线 | `xai-grok-pager-bin/src/main.rs`（~585/1358/2153/2200）、`xai-grok-shell/src/config/mod.rs:1458` |
| AccessKind / Decision | `xai-grok-workspace/src/permission/types.rs` |
| 权限判定链 | `xai-grok-workspace/src/permission/manager/mod.rs` |
| bash 评估 | `permission/{grants.rs,bash_permission_script.rs,bash_command_splitting.rs,exec_risk.rs,shell_access.rs}` |
| managed policy preflight | `permission/gate_preflight.rs`、`managed_policy/` |
| auto 模式分类器 | `permission/auto_mode/` |
| 提示器与持久化 | `permission/{prompter.rs,state.rs}`（`permission.toml`） |
| Claude 兼容 settings | `permission/claude_settings.rs` |
| Hub 门禁 | `permission/hub_gate.rs` |
| MCP 认领计划 | `xai-grok-workspace/src/mcp_claim.rs`（cap=256 at `mcp.rs:360`） |
| folder trust（消费侧） | `xai-grok-shell/src/agent/folder_trust.rs` |
| env 过滤 | `xai-grok-tools/src/util/shell_env_policy.rs` |
| 进程组/GROK_AGENT | `xai-grok-tools/src/util/{spawn.rs,env.rs}` |
| 子进程网络挂载点 | `terminal.rs:769/885/3149/3310`、`static_shell.rs:80`、`shell_state.rs:298`、`lsp/client.rs:431`、`envrc.rs:364` |
| 计划模式门禁 | `xai-grok-shell/src/session/plan_mode.rs`、`tool_calls.rs::plan_mode_edit_gate` |
