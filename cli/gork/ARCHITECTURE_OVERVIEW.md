# Grok Build 项目架构与亮点速览

> Grok Build（`grok`）是 SpaceXAI 出品的终端 AI 编程 Agent：全屏 TUI + Agent 运行时，可交互使用、headless 用于脚本/CI，也可以通过 ACP（Agent Client Protocol）嵌入 IDE。
> 本仓库是从内部 monorepo 定期同步过来的 Rust 源码（`SOURCE_REV` 记录对应的 monorepo commit）。

## 1. 基本信息

| 项 | 说明 |
|----|------|
| 语言 | Rust（工具链版本由 `rust-toolchain.toml` 固定），tokio 异步 |
| 规模 | 约 100 个 crate，核心 4 个 crate 合计约 126 万行 |
| 产物 | `xai-grok-pager-bin` → 二进制 `xai-grok-pager`（发布时改名为 `grok`） |
| 构建依赖 | DotSlash（用于调用 `bin/protoc` 等 hermetic 工具）、protoc |
| 注意 | 根目录 `Cargo.toml` 是自动生成的，视为只读；改依赖请改各 crate 自己的 `Cargo.toml` |

## 2. 目录结构

```
crates/
  codegen/   CLI/Agent 相关的全部 crate（主体）
  common/    共享底层：Computer Hub 工具协议/运行时、压缩引擎、熔断器、tracing
  build/     proto 代码生成
prod/mc/     cli-chat-proxy 类型定义
third_party/ 内置的 Mermaid 渲染栈（dagre_rust / graphlib_rust / mermaid-to-svg）
bin/         DotSlash 工具描述文件（protoc）
```

## 3. 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  入口层  xai-grok-pager-bin（组合根，clap 子命令分发）              │
│    grok (TUI) | agent/headless | leader | mcp | plugin | worktree … │
└──────────────┬──────────────────────────────────────────────────┘
               │
┌──────────────▼──────────────┐    ┌──────────────────────────────┐
│ 表现层 xai-grok-pager (~55万行)│    │ 其他客户端：IDE 插件（ACP/stdio）│
│  scrollback / 输入框 / 弹窗 /  │    │ headless CLI（websocket）      │
│  slash 命令 / 设置 / 语音 /    │    └──────────────┬───────────────┘
│  headless / acp               │                   │
│  + pager-render / markdown /  │                   │
│    mermaid / ratatui-*        │                   │
└──────────────┬───────────────┘                   │
               │  ACP (JSON-RPC)，经 Unix Socket ~/.grok/leader.sock
┌──────────────▼───────────────────────────────────▼──────────────┐
│ Agent 运行时 xai-grok-shell (~44万行)                              │
│  leader/   单机单 leader 进程，多客户端共享 Agent 状态（IPC 路由）    │
│  session/  会话 actor：对话轮次、上下文压缩、持久化/恢复/fork、        │
│            plan mode、goal 编排、MCP 分发、记忆、worktree 池          │
│  agent/    MvpAgent、配置、子 Agent、模型 provider、relay/proxy       │
│  remote/   云端会话/技能/工作区同步                                   │
└───┬──────────────┬──────────────┬──────────────┬────────────────┘
    │              │              │              │
┌───▼────────┐ ┌───▼──────────┐ ┌─▼────────────┐ ┌▼───────────────────┐
│xai-grok-   │ │xai-grok-tools│ │xai-grok-     │ │横向能力             │
│agent       │ │(~16万行)      │ │workspace     │ │ hooks / mcp / memory│
│Agent 定义、 │ │bash/grep/    │ │(~12万行)      │ │ sandbox / config    │
│system prompt│ │read/edit/    │ │文件系统、VCS、 │ │ plugin-marketplace  │
│组装、提醒与  │ │web/todo/task/│ │权限、信任、    │ │ telemetry / otel    │
│压缩策略     │ │plan/lsp/图像  │ │checkpoint、   │ │ auth / login / update│
│            │ │视频生成…      │ │workspace hub  │ │                     │
└───┬────────┘ └───┬──────────┘ └─┬────────────┘ └─────────────────────┘
    │              │               │
┌───▼──────────────▼───────────────▼──────────────────────────────┐
│ 基础设施                                                          │
│  xai-grok-sampler      actor 模型的 LLM 流式采样、重试、请求压缩、     │
│                        预热、doom-loop 检测与恢复                     │
│  xai-tool-runtime /    统一的 Tool trait 与分发协议（本地工具与远端工具  │
│  computer-hub-*        共用一个注册表，本地同名工具优先）               │
│  xai-grok-compaction   与传输层无关的上下文压缩引擎                     │
│  xai-hunk-tracker      区分 agent 修改和外部修改的 hunk 追踪           │
│  xai-fast-worktree     基于 CoW 的高速 git worktree                   │
│  xai-codebase-graph    基于 tree-sitter 的定义/引用索引               │
└─────────────────────────────────────────────────────────────────┘
```

### 核心数据流（一次对话轮次）

1. 用户在 TUI/IDE/headless 客户端输入 → 以 ACP 消息发给 leader。
2. leader 把消息路由到对应会话（`session/`），会话 actor 拼好上下文：system prompt 由 `xai-grok-agent` 组装，再加上 system reminder、记忆、项目规则。
3. `xai-grok-sampler` 以流式方式请求模型，负责重试、请求压缩和 doom-loop 信号处理。
4. 模型返回的 tool call 经 `xai-tool-runtime` 分发到 `xai-grok-tools` 或 MCP 工具；执行时受 `workspace` 权限/信任、`sandbox` 和 `hooks` 策略约束。
5. 工具结果回填到对话，循环直到这一轮结束；会话持久化到 `~/.grok/`，上下文接近上限时自动压缩。

## 4. 亮点

### 亮点一：Leader-Follower 多客户端架构 + 统一的 ACP 协议

- **每台机器只有一个 leader 进程**持有 Agent 状态。TUI、IDE 插件、headless CLI 都是轻量客户端，通过 Unix Socket（`~/.grok/leader.sock`）连接，并用 `connect_or_spawn` 自动接入已有 leader 或拉起新的（见 `xai-grok-shell/src/leader/mod.rs`）。
- leader 会给请求 ID 加命名空间避免冲突，并跟踪每个会话归属哪个客户端，所以同一会话可以在终端和 IDE 之间无缝切换。
- 所有客户端对外只说 **ACP（JSON-RPC）** 这一种协议，表现层和运行时彻底解耦：TUI 自身也只是一个 ACP 客户端。
- 工具层也按同样思路统一：`xai-tool-runtime` + `computer-hub-*` 用同一个 `Tool` trait 和注册表管理本地工具、MCP 工具和远端工具，同名时本地优先（local-shadows-remote）。

### 亮点二：面向长任务与稳定性的深度工程化

- **两阶段上下文压缩**（`session/two_pass.rs`）：第一遍把约 95% 的历史总结成 NOTE₁，第二遍结合最近约 5% 的尾部重写成 NOTE₂，兼顾长期记忆和近期细节。压缩内核（`xai-grok-compaction`）与传输层解耦，可以复用。
- **Doom-loop 检测与恢复**（`xai-grok-sampler/doom_loop*.rs`）：识别模型陷入重复循环，并自动纠偏。
- **Goal 编排**（`session/goal_*`）：由 planner、strategist、evaluator、stop detector 等角色组成，推进自主长任务。
- **精准编辑工具**：同时提供多套编辑工具，包括 Hashline 锚点编辑（`grok_build_hashline`，基于行哈希锚点，锚点偏移或过期时能在有限范围内恢复）以及移植自 codex/opencode 的工具实现，可按模型选择。
- **安全与隔离**：`xai-grok-sandbox` 通过 nono 在内核层做沙箱（macOS 用 Seatbelt，Linux 用 Landlock），子进程网络由 seccomp 按进程拦截；另有 folder trust、权限模式和 hooks 策略执行。
- **性能**：jemalloc 分配器；`xai-fast-worktree` 用 CoW 并行克隆（Linux 上支持 BTRFS snapshot，O(1) 创建），让子 Agent/并行任务能快速拿到隔离的工作区；`xai-grok-pager-pty-harness` 用 PTY 对 TUI 做回归测试和逐帧性能基准。

## 5. 常用命令

```sh
cargo run -p xai-grok-pager-bin              # 构建并启动 TUI
cargo build -p xai-grok-pager-bin --release  # 构建 release 二进制
cargo check -p <crate>                       # 按 crate 检查（全量构建很慢）
cargo test  -p <crate>
cargo clippy -p <crate>
cargo fmt --all
```

用户文档：`crates/codegen/xai-grok-pager/docs/user-guide/`（27 章，覆盖 MCP、Skills、Plugins、Hooks、Subagents、Sandbox、Plan Mode 等）。
