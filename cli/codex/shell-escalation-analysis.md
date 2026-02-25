# Shell-Escalation 架构深度分析

> 本文档分析 Codex 项目中 Shell-Escalation 模块的架构设计、重构历程和技术实现。

---

## 1. 概述

### 1.1 什么是 Shell-Escalation？

Shell-Escalation 是 Codex 项目中用于**安全执行 Shell 命令**的核心模块。它实现了一个精巧的进程间通信协议，允许父进程（Server）监控和控制子进程（Shell）中的每一个 `exec()` 调用。

### 1.2 核心价值

- **安全隔离**：命令在沙箱环境中执行，受策略控制
- **细粒度控制**：可以拦截、审批或拒绝每个命令执行
- **透明代理**：对 Shell 脚本完全透明，无需修改
- **可扩展性**：通过 trait 抽象，支持不同的执行器和策略

---

## 2. 架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Codex Core                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    unix_escalation.rs                         │   │
│  │  ┌─────────────────────┐    ┌─────────────────────────────┐  │   │
│  │  │ CoreShellActionProvider │    │ CoreShellCommandExecutor    │  │   │
│  │  │ (EscalationPolicy)  │    │ (ShellCommandExecutor)      │  │   │
│  │  └─────────────────────┘    └─────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ trait objects
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    codex-shell-escalation                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      EscalateServer                           │   │
│  │                           │                                   │   │
│  │           ┌───────────────┼───────────────┐                  │   │
│  │           ▼               ▼               ▼                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │   │
│  │  │ Escalate    │  │ Escalate    │  │ Socket      │          │   │
│  │  │ Protocol    │  │ Client      │  │ Layer       │          │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    codex-execve-wrapper                       │   │
│  │                    (独立二进制)                                │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Unix Socket (SCM_RIGHTS)
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Patched Shell                                 │
│                    (Bash/Zsh with EXEC_WRAPPER)                      │
│                                                                      │
│    每次 exec() 调用 → 委托给 codex-execve-wrapper                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| `EscalateServer` | `escalate_server.rs` | 主服务器，监听和处理升级请求 |
| `EscalateProtocol` | `escalate_protocol.rs` | 通信协议定义 |
| `EscalateClient` | `escalate_client.rs` | 客户端逻辑（execve wrapper 使用） |
| `EscalationPolicy` | `escalation_policy.rs` | 策略 trait，决定命令命运 |
| `Socket` | `socket.rs` | 异步 Unix Socket 封装 |
| `Stopwatch` | `stopwatch.rs` | 可暂停的计时器 |
| `ExecveWrapper` | `execve_wrapper.rs` | exec 拦截器入口 |

---

## 3. 通信协议详解

### 3.1 协议消息类型

```rust
// 客户端 → 服务器：请求执行
pub struct EscalateRequest {
    pub file: PathBuf,      // 可执行文件路径
    pub argv: Vec<String>,  // 参数列表（含 argv[0]）
    pub workdir: PathBuf,   // 工作目录
    pub env: HashMap<String, String>,  // 环境变量
}

// 服务器 → 客户端：决策响应
pub struct EscalateResponse {
    pub action: EscalateAction,
}

pub enum EscalateAction {
    Run,                    // 在沙箱内直接执行
    Escalate,               // 提升到服务器执行
    Deny { reason: Option<String> },  // 拒绝执行
}

// 客户端 → 服务器：传递文件描述符
pub struct SuperExecMessage {
    pub fds: Vec<RawFd>,    // stdin/stdout/stderr
}

// 服务器 → 客户端：执行结果
pub struct SuperExecResult {
    pub exit_code: i32,
}
```

### 3.2 执行流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Escalation Flow                                  │
│                                                                           │
│  Command    Server     Shell     Execve Wrapper                           │
│     │         │         │            │                                    │
│     │         │──┐      │            │                                    │
│     │         │  │spawn │            │                                    │
│     │         │<─┘      │            │                                    │
│     │         │         │            │                                    │
│     │         │         │──exec()──→│                                    │
│     │         │         │            │                                    │
│     │         │<──EscalateRequest───│                                    │
│     │         │         │            │                                    │
│     │         │──EscalateResponse──→│                                    │
│     │         │         │            │                                    │
│     │         │<──SuperExecMessage──│  (传递 stdin/stdout/stderr)        │
│     │         │         │            │                                    │
│     │<────────│─────────│────────────│  (服务器执行命令)                   │
│     │         │         │            │                                    │
│     │────────>│         │            │                                    │
│     │         │──exit code──────────→│                                    │
│     │         │         │            │                                    │
│     │         │         │<──exit────│                                    │
│     │         │         │            │                                    │
│     │         │<────────│            │                                    │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                        Non-Escalation Flow                                │
│                                                                           │
│  Server     Shell     Execve Wrapper    Command                           │
│    │         │            │               │                               │
│    │──┐      │            │               │                               │
│    │  │spawn │            │               │                               │
│    │<─┘      │            │               │                               │
│    │         │            │               │                               │
│    │         │──exec()──→│               │                               │
│    │         │            │               │                               │
│    │<──EscalateRequest───│               │                               │
│    │         │            │               │                               │
│    │──Run───────────────→│               │                               │
│    │         │            │               │                               │
│    │         │            │──exec()─────→│                               │
│    │         │            │               │                               │
│    │         │<───────────│───exit──────│                               │
│    │         │            │               │                               │
│    │<────────│            │               │                               │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 文件描述符传递机制

使用 Unix Domain Socket 的 `SCM_RIGHTS` 辅助数据传递文件描述符：

```rust
// socket.rs 中的核心实现
fn make_control_message(fds: &[OwnedFd]) -> std::io::Result<Vec<u8>> {
    let mut control = vec![0u8; control_space_for_fds(fds.len())];
    unsafe {
        let cmsg = control.as_mut_ptr().cast::<libc::cmsghdr>();
        (*cmsg).cmsg_len = libc::CMSG_LEN(size_of::<RawFd>() as c_uint * fds.len() as c_uint);
        (*cmsg).cmsg_level = libc::SOL_SOCKET;
        (*cmsg).cmsg_type = libc::SCM_RIGHTS;
        // ... 写入 FD 数据
    }
    Ok(control)
}
```

---

## 4. Trait 设计模式

### 4.1 EscalationPolicy - 决策策略

```rust
#[async_trait::async_trait]
pub trait EscalationPolicy: Send + Sync {
    async fn determine_action(
        &self,
        file: &Path,
        argv: &[String],
        workdir: &Path,
    ) -> anyhow::Result<EscalateAction>;
}
```

**实现示例** (`CoreShellActionProvider`):

```rust
impl EscalationPolicy for CoreShellActionProvider {
    async fn determine_action(
        &self,
        file: &Path,
        argv: &[String],
        workdir: &Path,
    ) -> anyhow::Result<EscalateAction> {
        // 1. 解析命令
        let command = build_command(file, argv);

        // 2. 检查策略规则
        let evaluation = self.policy.check(&command);

        // 3. 根据决策返回动作
        match evaluation.decision {
            Decision::Forbidden => EscalateAction::Deny { ... },
            Decision::Prompt => {
                // 请求用户审批
                match self.prompt(&command, workdir).await? {
                    ReviewDecision::Approved => EscalateAction::Escalate,
                    ReviewDecision::Denied => EscalateAction::Deny { ... },
                    // ...
                }
            }
            Decision::Allow => EscalateAction::Escalate,
        }
    }
}
```

### 4.2 ShellCommandExecutor - 执行器抽象

```rust
#[async_trait::async_trait]
pub trait ShellCommandExecutor: Send + Sync {
    async fn run(
        &self,
        command: Vec<String>,
        cwd: PathBuf,
        env: HashMap<String, String>,
        cancel_rx: CancellationToken,
    ) -> anyhow::Result<ExecResult>;
}
```

**设计意义**：
- 将进程执行、沙箱集成与协议逻辑解耦
- 允许调用者控制输出捕获和取消逻辑
- 支持不同的沙箱实现（macOS sandbox-exec、Linux bwrap 等）

---

## 5. Stopwatch - 可暂停计时器

### 5.1 设计动机

在等待用户审批时，命令执行时间不应计入超时。Stopwatch 实现了可暂停的计时逻辑。

### 5.2 实现原理

```rust
pub struct Stopwatch {
    limit: Duration,
    inner: Arc<Mutex<StopwatchState>>,
    notify: Arc<Notify>,
}

struct StopwatchState {
    elapsed: Duration,          // 已累计时间
    running_since: Option<Instant>,  // 当前运行起始点
    active_pauses: u32,         // 活跃暂停计数
}
```

### 5.3 使用方式

```rust
// 用户审批时暂停计时
match self.prompt(&command, workdir, &self.stopwatch).await? {
    // ...
}

// pause_for 内部实现
pub async fn pause_for<F, T>(&self, fut: F) -> T {
    self.pause().await;      // 暂停计时
    let result = fut.await;  // 执行异步操作
    self.resume().await;     // 恢复计时
    result
}
```

---

## 6. 重构历程

### 6.1 重构时间线

```
2026-02-23 ─────────────────────────────────────────────────────────────▶

┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 1: 功能拆分                                                        │
│ feat: split core business logic from exec-server to shell-escalation    │
│ (#12555, #12556)                                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 2: 移除 MCP 支持                                                   │
│ refactor: remove exec-server MCP shell tool support                     │
│ (多个 commits)                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 3: 删除 exec-server                                                │
│ refactor: delete exec-server and move execve wrapper into               │
│ shell-escalation (#12632)                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 4: 解耦依赖                                                        │
│ refactor: decouple shell-escalation from codex-core (#12638)            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 5: 集成新 Zsh Fork Shell Tool                                      │
│ feat: run zsh fork shell tool via shell-escalation (#12649)             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 重构动机

**原始架构问题**：
```
codex-core ──────► exec-server (MCP Server)
     │                  │
     │                  ├── MCP Protocol
     │                  │
     └──────────────────┘ (循环依赖风险)
```

**重构后架构**：
```
codex-core ──────► shell-escalation
     │                  │
     │                  └── 独立协议层
     │
     └── trait 注入 ──────► (解耦)
```

### 6.3 关键变更

#### 变更 1: exec-server 删除 (#12632)

**删除内容**：
- `codex-rs/exec-server` 整个 crate
- MCP 服务器二进制
- MCP 特定模块

**迁移内容**：
- `codex-execve-wrapper` → `shell-escalation/src/bin/`
- 测试 fixtures → `app-server/tests/suite/`

#### 变更 2: 依赖解耦 (#12638)

**问题**：`shell-escalation` 依赖 `codex-core`，导致无法在 `codex-core` 中使用

**解决方案**：
```rust
// 引入 ShellCommandExecutor trait
pub trait ShellCommandExecutor: Send + Sync {
    async fn run(...) -> anyhow::Result<ExecResult>;
}

// EscalateServer 通过 trait 调用
impl EscalateServer {
    pub async fn exec(
        &self,
        params: ExecParams,
        cancel_rx: CancellationToken,
        command_executor: &dyn ShellCommandExecutor,  // 注入执行器
    ) -> anyhow::Result<ExecResult> { ... }
}
```

**依赖关系变化**：
```toml
# 之前
[dependencies]
codex-core = { path = "../core" }  # 直接依赖

# 之后
[dependencies]
# 不再依赖 codex-core，完全独立
```

---

## 7. 环境变量

| 变量名 | 用途 |
|--------|------|
| `CODEX_ESCALATE_SOCKET` | 传递 escalation socket FD |
| `EXEC_WRAPPER` | execve wrapper 路径 |
| `BASH_EXEC_WRAPPER` | 兼容旧版 patched bash |

---

## 8. 文件结构

```
codex-rs/shell-escalation/
├── Cargo.toml
├── src/
│   ├── lib.rs                    # 模块导出
│   ├── bin/
│   │   └── main_execve_wrapper.rs  # execve wrapper 入口
│   └── unix/
│       ├── mod.rs                # Unix 模块导出
│       ├── escalate_protocol.rs  # 协议定义
│       ├── escalate_server.rs    # 服务器实现
│       ├── escalate_client.rs    # 客户端逻辑
│       ├── escalation_policy.rs  # Policy trait
│       ├── execve_wrapper.rs     # exec 拦截
│       ├── socket.rs             # 异步 Socket 封装
│       └── stopwatch.rs          # 可暂停计时器
```

---

## 9. 技术亮点

### 9.1 异步 Socket 设计

使用 `tokio::io::AsyncFd` 包装 `socket2::Socket`：

```rust
pub struct AsyncSocket {
    inner: AsyncFd<Socket>,
}

impl AsyncSocket {
    pub async fn send_with_fds<T: Serialize>(
        &self,
        msg: T,
        fds: &[OwnedFd],
    ) -> std::io::Result<()> {
        // 序列化消息
        let payload = serde_json::to_vec(&msg)?;
        // 添加长度前缀
        let mut frame = Vec::with_capacity(LENGTH_PREFIX_SIZE + payload.len());
        frame.extend_from_slice(&encode_length(payload.len())?);
        frame.extend_from_slice(&payload);
        // 异步发送
        send_stream_frame(&self.inner, &frame, fds).await
    }
}
```

### 9.2 CLOEXEC 处理

正确处理 `exec()` 时的文件描述符继承：

```rust
// 使用 pair_raw 避免 SO_NOSIGPIPE 失败
let (server, client) = Socket::pair_raw(Domain::UNIX, Type::STREAM, None)?;
// 显式设置 CLOEXEC
server.set_cloexec(true)?;
client.set_cloexec(true)?;

// 客户端 socket 需要跨 exec 传递
client_socket.set_cloexec(false)?;  // 允许继承
```

### 9.3 并发请求处理

服务器使用 spawn 处理并发请求：

```rust
async fn escalate_task(
    socket: AsyncDatagramSocket,
    policy: Arc<dyn EscalationPolicy>,
) -> anyhow::Result<()> {
    loop {
        let (_, mut fds) = socket.receive_with_fds().await?;
        let stream_socket = AsyncSocket::from_fd(fds.remove(0))?;
        let policy = policy.clone();
        // 每个请求独立处理
        tokio::spawn(async move {
            handle_escalate_session_with_policy(stream_socket, policy).await
        });
    }
}
```

---

## 10. 测试策略

### 10.1 单元测试

```rust
#[tokio::test]
async fn handle_escalate_session_respects_run_in_sandbox_decision() {
    // 使用确定性策略测试
    let policy = DeterministicEscalationPolicy {
        action: EscalateAction::Run,
    };
    // 验证协议交互
}

#[tokio::test]
async fn handle_escalate_session_executes_escalated_command() {
    // 测试升级执行路径
    let policy = DeterministicEscalationPolicy {
        action: EscalateAction::Escalate,
    };
    // 验证命令执行和结果返回
}
```

### 10.2 Socket 测试

```rust
#[tokio::test]
async fn async_socket_round_trips_payload_and_fds() {
    let (server, client) = AsyncSocket::pair()?;
    // 测试消息和 FD 的往返传输
}

#[tokio::test]
async fn async_socket_handles_large_payload() {
    // 测试大消息处理
    let payload = vec![b'A'; 10_000];
}
```

### 10.3 Stopwatch 测试

```rust
#[tokio::test]
async fn pause_prevents_timeout_until_resumed() {
    let stopwatch = Stopwatch::new(Duration::from_millis(50));
    // 测试暂停期间不触发超时
    stopwatch.pause_for(async {
        sleep(Duration::from_millis(100)).await;
    }).await;
}

#[tokio::test]
async fn overlapping_pauses_only_resume_once() {
    // 测试嵌套暂停的引用计数
}
```

---

## 11. 总结

### 11.1 架构优势

1. **解耦设计**：通过 trait 抽象，协议层与业务逻辑完全分离
2. **可测试性**：策略和执行器可替换，便于单元测试
3. **安全性**：每个命令都经过策略检查，支持细粒度审批
4. **透明性**：对 Shell 脚本完全透明，无需修改现有脚本

### 11.2 重构成果

| 方面 | 重构前 | 重构后 |
|------|--------|--------|
| 模块数量 | exec-server + shell-escalation | 仅 shell-escalation |
| 依赖关系 | 循环依赖风险 | 单向依赖，完全解耦 |
| MCP 支持 | 内置 | 移除，简化架构 |
| 可复用性 | 仅限 MCP 场景 | 通用 Shell Tool 场景 |

### 11.3 适用场景

- 安全沙箱命令执行
- 用户交互式命令审批
- 策略驱动的权限控制
- 跨平台 Shell 工具实现

---

*文档生成时间：2026-02-25*
*基于 Git commit: af215eb39 (refactor: decouple shell-escalation from codex-core)*
