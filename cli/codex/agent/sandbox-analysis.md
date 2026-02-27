# Sandbox 系统分析

这个项目实现了一套跨平台沙箱系统，支持 macOS、Linux 和 Windows。代码分布在以下位置：

| 文件路径 | 功能 |
|---------|------|
| `codex-rs/core/src/sandboxing/mod.rs` | 沙箱管理器，处理命令转换 |
| `codex-rs/core/src/safety.rs` | 安全检查和策略评估 |
| `codex-rs/core/src/seatbelt.rs` | macOS Seatbelt 配置 |
| `codex-rs/core/src/landlock.rs` | Linux Landlock 规则 |
| `codex-rs/core/src/exec.rs` | 执行引擎 |
| `codex-rs/protocol/src/protocol.rs` | 策略数据结构 |
| `codex-rs/linux-sandbox/` | Linux 独立可执行文件 |
| `codex-rs/windows-sandbox-rs/` | Windows 沙箱实现 |

---

## 数据流

```
CommandSpec (便携式命令)
    ↓ SandboxManager.transform()
ExecRequest (平台特定请求)
    ↓
macOS: /usr/bin/sandbox-exec
Linux: codex-linux-sandbox
Windows: 受限令牌进程
    ↓
Process Execution
```

`CommandSpec` 是用户无关的命令定义，包含程序、参数和权限要求。`SandboxManager` 将其转换为平台特定的 `ExecRequest`，后者包含沙箱前缀（如 `codex-linux-sandbox`），可以直接传递给操作系统执行。

---

## 沙箱策略

策略定义在 `codex-rs/protocol/src/protocol.rs` 的 `SandboxPolicy` 枚举中：

```rust
pub enum SandboxPolicy {
    DangerFullAccess,                    // 无限制

    ReadOnly { access: ReadOnlyAccess }, // 只读

    ExternalSandbox { network_access },  // 已在容器内

    WorkspaceWrite {                     // 推荐配置
        writable_roots: Vec<AbsolutePathBuf>,
        read_only_access: ReadOnlyAccess,
        network_access: bool,
        ...
    },
}
```

`WorkspaceWrite` 是默认选项。它允许写入指定目录，其他位置只读，网络访问可配置。

---

## SandboxManager

`SandboxManager` 有两个方法：`select_initial()` 选择沙箱类型，`transform()` 转换命令。

选择逻辑比较直接：

- `Forbid` → 不使用沙箱
- `Require` → 使用平台沙箱
- `Auto` → 根据策略和是否有网络需求决定

`transform()` 会根据沙箱类型包装命令：

- macOS: `/usr/bin/sandbox-exec -p "(policy)" command`
- Linux: `codex-linux-sandbox --sandbox-policy '{"type":"workspace-write",...}' -- command`
- Windows: 命令不变，在进程创建时应用受限令牌

---

## 平台实现

### macOS Seatbelt

使用 `/usr/bin/sandbox-exec`（硬编码路径避免 PATH 注入）。策略用 Seatbelt Policy Language 编写，支持三个基础策略文件。网络控制通过提取代理 URL 的回环端口，创建 Unix 域套接字白名单。

### Linux (Bubblewrap + Seccomp)

```
codex-linux-sandbox (外部)
    ↓
Bubblewrap (--ro-bind / / --bind writable writable --unshare-pid --unshare-net)
    ↓
codex-linux-sandbox (内部，--apply-seccomp-then-exec)
    ↓
Seccomp 网络过滤 + PR_SET_NO_NEW_PRIVS
    ↓
execve(command)
```

Bubblewrap 做文件系统隔离，Seccomp 过滤网络系统调用。`PR_SET_NO_NEW_PRIVS` 防止进程获取新权限。

### Windows (受限令牌)

需要 UAC 提升来创建沙箱用户账户（`CodexSandboxOffline` 和 `CodexSandboxOnline`）。进程创建时使用受限令牌，移除管理员权限，应用 ACL。

排除的敏感目录包括 `.ssh`、`.gnupg`、`.aws`、`.azure`、`.kube` 等。

---

## 安全检查

补丁应用前会检查修改范围。如果补丁只修改 `writable_roots` 内的文件，且有沙箱可用，就自动批准。超出范围的修改需要用户确认。

执行命令也有类似的审批流程。`AskForApproval` 策略决定是否需要用户确认：

- `Never`/`OnFailure`: 跳过审批
- `OnRequest`: 受限沙箱需要审批
- `UnlessTrusted`: 始终需要审批
- `Reject`: 禁止（除非有沙箱）

---

## 配置示例

Linux 命令行：

```bash
codex-linux-sandbox \
    --sandbox-policy-cwd /path/to/cwd \
    --sandbox-policy '{"type":"workspace-write","writable_roots":["/tmp"],"network_access":false}' \
    --use-bwrap-sandbox \
    -- \
    command arg1 arg2
```

JSON 策略：

```json
{
  "type": "workspace-write",
  "writable_roots": ["/home/user/project"],
  "read_only_access": {
    "include_platform_defaults": true
  },
  "network_access": false
}
```

---

## 安全特性对比

| 特性 | macOS | Linux | Windows |
|------|-------|-------|---------|
| 文件隔离 | Seatbelt | Bubblewrap | ACL |
| PID 隔离 | 否 | 是 | 否 |
| 网络隔离 | Seatbelt | Seccomp | 防火墙 |
| 权限提升防护 | 系统级 | no_new_privs | 受限令牌 |
| .git 保护 | 是 | 是 | 是 |

---

## 设计思路

这个沙箱系统的设计遵循几条原则：fail-closed（宁可拒绝也不放行）、最小权限、防止权限提升。审批层会缓存用户的批准决策，避免重复询问。

`★ Insight ─────────────────────────────────────`
1. Linux 用 Bubblewrap 做 namespace 隔离，比简单的 chroot 更彻底
2. Windows 需要预先创建沙箱用户账户，这是 Windows 权限模型的限制
3. `.git` 目录在所有平台都被保护，防止恶意代码通过 hook 扩散
`─────────────────────────────────────────────────`
