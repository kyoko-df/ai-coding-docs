# Bun 项目架构分析

## 概述

Bun 是一个 all-in-one JavaScript 运行时和工具包，集成了打包器、测试运行器和 Node.js 兼容的包管理器。项目使用 **Rust（主语言）+ C++（JSC 集成）+ TypeScript（构建系统/代码生成）+ Zig（移植参考，不再编译）** 的多语言架构。

| 语言 | 文件数 | 代码行数(约) | 当前是否编译 | 角色 |
|------|--------|-------------|-------------|------|
| Rust | ~1,433 | ~981k | **是** | 核心运行时、打包器、包管理器 |
| Zig | ~1,298 | ~710k | **否** | 原始实现，保留作为逐行移植参考 |
| C++ | ~554 (.cpp) + ~706 (.h) | ~大量 | **是** | JavaScriptCore 引擎绑定 |
| TypeScript | ~数百 | ~数十万 | 构建时 | 构建系统、代码生成器、JS 内置模块 |

---

## Rust 架构（111 个 Crate）

Rust 代码组织为 Cargo workspace，定义在 `Cargo.toml` 中，包含约 111 个成员 crate。

### 第 0 层：基础设施 Crate（被所有上层依赖）

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_core` | `src/bun_core/` | 字符串类型(`BunString`/`ZigString`)、格式化输出(ANSI 颜色)、环境变量缓存、全局状态、压缩状态枚举 |
| `bun_alloc` | `src/bun_alloc/` | 内存分配器(mimalloc)、`MimallocArena`、AST 堆分配、`hashbrown` 桥接 |
| `bun_collections` | `src/collections/` | 自定义数据结构：`MultiArrayList`(SoA)、`ArrayHashMap`、`BitSet`、`HiveArray`、`ObjectPool`、`LinearFifo`、`ComptimeStringMap` |
| `bun_threading` | `src/threading/` | 同步原语(`Mutex`/`RwLock`/`Semaphore`/`WaitGroup`/`Futex`)和 `ThreadPool`(工作窃取) |
| `bun_paths` | `src/paths/` | 跨平台路径操作(连接、规范化、`dirname`/`basename`)，支持 `Posix`/`Windows`/`Nt` |
| `bun_ptr` | `src/ptr/` | 智能指针：`RefCount`/`Owned`/`Shared`/`CowSlice`/`TaggedPointer`/`Interned` |
| `bun_dispatch` | `src/dispatch/` | `link_interface!` 宏——链接时接口分发系统，避免动态分发 |

### 第 1 层：系统接口

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_sys` | `src/sys/` | 跨平台系统调用：`open`/`read`/`write`/`stat`/`mkdir` 等，Linux 通过 `rustix` 直接内联 syscall，Windows 通过 `libuv` 桥接 |
| `bun_io` | `src/io/` | 异步 I/O：`PipeReader`/`PipeWriter`(非阻塞管道 I/O)、kqueue/epoll/IOCP 事件循环包装 |
| `bun_event_loop` | `src/event_loop/` | 事件循环抽象：`AnyEventLoop`(JS/非JS)、`MiniEventLoop`(CLI/安装)、`ConcurrentTask` |
| `bun_dns` | `src/dns/` | DNS 解析，包装 c-ares |
| `bun_watcher` | `src/watcher/` | 文件系统监视器(inotify/FSEvents) |
| `bun_errno` | `src/errno/` | errno 代码枚举 |

### 第 2 层：解析器、打印机和 AST

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_ast` | `src/ast/` | JS/TS AST 节点定义：`Expr`/`Stmt`/`Symbol`/`Scope`/`Binding`/`ImportRecord`，`KnownGlobal` 数据库，AST 内存分配器 |
| `bun_js_parser` | `src/js_parser/` | JS/TS 解析器(从 esbuild 分支)：词法分析器、递归下降解析器、AST 折叠、TypeScript 降级、装饰器降级、ESM HMR 转换 |
| `bun_js_printer` | `src/js_printer/` | JS/TS 代码生成器：AST → 源文本，符号重命名(压缩)，源映射生成 |
| `bun_sourcemap` | `src/sourcemap/` | 源映射：`InternalSourceMap`、VLQ 编码/解码、行偏移表 |
| `bun_semver` | `src/semver/` | npm 语义版本控制：`Version`(解析/比较/排序)、`SemverQuery`(`^1.2.3`/`~1.2`)、`SemverRange` |

### 第 3 层：打包器管道

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_resolver` | `src/resolver/` | 模块解析：处理 `package.json` exports/imports/main/browser、`tsconfig.json` paths、目录信息缓存、Node.js 回退 |
| `bun_resolve_builtins` | `src/resolve_builtins/` | 内置模块名称到实现的解析 |
| `bun_bundler` | `src/bundler/` | **打包器核心**：`bundle_v2`(编排)、`Transpiler`、`Options`、`ParseTask`(并行解析)、`Chunk`(代码块)、`LinkerGraph`(依赖图)、`LinkerContext`(链接编排，含 20+ 子模块) |
| `bun_transpiler` | `src/transpiler/` | `bun_bundler::Transpiler` 的轻量重新导出层 |
| `bun_options_types` | `src/options_types/` | 配置类型：`BundleEnums`/`CompileTarget`/`JSX`/`Context`/`GlobalCache` |
| `bun_css` | `src/css/` | **完整 CSS 管道**(从 lightningcss 分支)：CSS 解析器/打印机/转换器，支持 vendor 前缀、浏览器兼容性、颜色转换、媒体查询、CSS 模块、Tailwind 集成 |

### 第 4a 层：HTTP 和网络（客户端）

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_http` | `src/http/` | HTTP 客户端实现：`AsyncHTTP`、`HTTPThread`、`ProxyTunnel`、HTTP/2 客户端、HTTP/3 客户端、WebSocket 客户端、HPACK 解压 |
| `bun_http_types` | `src/http_types/` | HTTP 类型(JSC 无关)：`MimeType`(大查找表)、`Method`、`ETag`、Fetch 枚举 |
| `bun_http_jsc` | `src/http_jsc/` | HTTP 类型的 JSC 绑定 |
| `bun_uws` | `src/uws/` | uWebSockets 的安全 Rust 包装器 |
| `bun_uws_sys` | `src/uws_sys/` | uWebSockets C 库的原始 FFI 绑定(HTTP/WebSocket/TLS/TCP/UDP/H3/QUIC) |
| `bun_picohttp` | `src/picohttp/` | HTTP/1.1 请求行解析(picohttpparser) |
| `bun_url` | `src/url/` | WHATWG URL 解析(JSC 无关) |
| `bun_url_jsc` | `src/url_jsc/` | URL 类型的 JSC 绑定 |
| `bun_router` | `src/router/` | HTTP 请求路由器(用于 Bun Bake 的基于文件系统路由) |

### 第 4b 层：包管理器

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_install` | `src/install/` | **包管理器核心**：npm registry 客户端、锁文件管理(`bun.lock`/`bun.lockb`)、依赖图解析、tarball 流式提取、生命周期脚本执行、包补丁、安全扫描、Windows shim 生成 |
| `bun_install_types` | `src/install_types/` | 安装器类型(JSC 无关)：NodeLinker 枚举、解析器钩子 |
| `bun_install_jsc` | `src/install_jsc/` | 安装器类型的 JSC 绑定 |
| `bun_patch` | `src/patch/` | 补丁创建和应用 |
| `bun_patch_jsc` | `src/patch_jsc/` | 补丁系统的 JSC 绑定 |

### 第 4c 层：Shell 解析器

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_shell_parser` | `src/shell_parser/` | Bun Shell 脚本解析器：解析类 Bash 语法、引号字符串、变量、命令替换、大括号扩展、管道和重定向 |

### 第 5a 层：JSC 运行时绑定（核心枢纽）

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_jsc` | `src/jsc/` | **JavaScriptCore Rust 绑定**：`JSValue`/`JSObject`/`JSString`/`JSPromise`/`JSArray`/`JSBigInt`/`JSFunction` 包装器，`VirtualMachine`(主 VM 对象)，`JSGlobalObject`(全局对象)，`ConsoleObject`，`ModuleLoader`(ES 模块)，`GC Strong/Weak`(GC 根保护)，FFI 支持，`Debugger`(WebKit Inspector)，`WebWorker`，`HotReloader`，事件循环集成 |
| `bun_jsc_macros` | `src/jsc_macros/` | Proc-macro crate：`#[JsClass]`、`#[host_fn]`、`#[host_call]` |

### 第 5b 层：完整运行时（JS 可见 API）

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_runtime` | `src/runtime/` | **顶层运行时 crate**，依赖 70+ 工作区 crate，实现所有 JS 可见的 API |

子模块：
- **`server/`** — `Bun.serve` HTTP/WebSocket 服务器实现(`RequestContext`、`ServerConfig`、`ServerWebSocket`、`NodeHTTPResponse`、`FileResponseStream`、`StaticRoute`、`RangeRequest`)
- **`api/`** — `BunObject`(全局 Bun API)、`JSBundler`、`html_rewriter`、`Archive`、`HashObject`、`JSTranspiler`、`cron`、`filesystem_router`、`MarkdownObject`、`YAMLObject`、`globs`，以及 `bun/` 子目录(spawn/process/subprocess/socket/dns/udp/tls)
- **`bake/`** — DevServer(HMR 开发服务器 298K)、生产模式、FrameworkRouter
- **`cli/`** — CLI 参数解析和命令分发(`Arguments`、`build_command`、`create_command`、`init_command`、`bunx_command`、`audit_command`)
- **`webcore/`** — Web API 实现：fetch、encoding、streams、blob、Response、Request
- **`node/`** — Node.js API 兼容层(fs、path、process、Buffer)
- **`shell/`** — Bun Shell 运行时
- **`crypto/`** — WebCrypto + `node:crypto`(`EVP`、`HMAC`、`CryptoHasher`)
- **`test_runner/`** — `bun test` 测试运行器
- **`napi/`** — Node-API(N-API) 绑定
- **`socket/`** — TCP/UDP/TLS 套接字 API
- **`timer/`** — 计时器实现
- **`ffi/`** — FFI 实现
- **`dns_jsc/`** — DNS 的 JSC 绑定

### 第 5c 层：SQL 数据库

| Crate | 路径 | 功能 |
|-------|------|------|
| `bun_sql` | `src/sql/` | **本地 SQL 协议实现**：MySQL 线协议(~50 文件)、PostgreSQL 线协议(~74 文件)、共享类型 |
| `bun_sql_jsc` | `src/sql_jsc/` | SQL 的 JSC 绑定 |

### 系统绑定 Crate（`_sys` 后缀，FFI 到 C/C++ 库）

| Crate | 绑定的库 |
|-------|---------|
| `bun_zlib_sys` | zlib 压缩 |
| `bun_brotli_sys` | Brotli 压缩 |
| `bun_picohttp_sys` | picohttpparser |
| `bun_libarchive_sys` | libarchive(tar/zip) |
| `bun_libdeflate_sys` | libdeflate |
| `bun_libuv_sys` | libuv(Windows 事件循环) |
| `bun_mimalloc_sys` | mimalloc(内存分配器) |
| `bun_lolhtml_sys` | lol-html(HTML 重写器) |
| `bun_cares_sys` | c-ares(异步 DNS) |
| `bun_boringssl_sys` | BoringSSL(TLS/加密) |
| `bun_tcc_sys` | TinyCC(FFI JIT 编译器) |
| `bun_simdutf_sys` | simdutf(SIMD UTF 转换) |
| `bun_windows_sys` | Windows API |

### 其他实用 Crate

| Crate | 功能 |
|-------|------|
| `bun_zlib` | zlib 压缩/解压 Rust 包装器 |
| `bun_brotli` | Brotli 压缩/解压 Rust 包装器 |
| `bun_libarchive` | libarchive 归档读写 Rust 包装器 |
| `bun_boringssl` | BoringSSL 加密原语 Rust 包装器 |
| `bun_sha_hmac` | SHA/HMAC 计算 |
| `bun_hash` | 哈希原语(xxhash/wyhash/sha256/sha512) |
| `bun_wyhash` | wyhash 快速哈希 |
| `bun_highway` | HighwayHash 实现 |
| `bun_base64` | Base64 编码/解码 |
| `bun_unicode` | Unicode 支持(大小写折叠、类别) |
| `bun_glob` | Glob 模式匹配 |
| `bun_ini` | INI 文件解析器 |
| `bun_which` | `which` 命令查找(PATH 定位) |
| `bun_dotenv` | `.env` 文件解析器 |
| `bun_csrf` | CSRF 令牌生成和验证 |
| `bun_s3_signing` | AWS S3 请求签名 |
| `bun_md` | Markdown 解析/渲染 |
| `bun_valkey` | Valkey(Redis) 协议解析器 |
| `bun_bunfig` | Bunfig 配置文件加载 |
| `bun_crash_handler` | 崩溃信号处理器(SIGSEGV/SIGABRT) |
| `bun_safety` | 安全工具(线程 ID 包装器) |
| `bun_standalone_graph` | 独立二进制模块图 |
| `bun_opaque` | `Opaque<T>` 类型 |
| `bun_exe_format` | 可执行格式处理(独立二进制) |
| `bun_perf` | 性能分析支持 |
| `bun_clap` | CLI 参数解析(Bun 特化) |
| `bun_meta` | 构建元数据/版本信息 |
| `bun_js` | JS 回退模块(当原生实现不可用时) |
| `bun_bin` | Cargo 入口点，链接为 `libbun_rust.a` |

### Proc-macro Crate

| Crate | 功能 |
|-------|------|
| `bun_core_macros` | `bun_core` 的宏(日志声明等) |
| `bun_clap_macros` | CLI 参数解析宏 |
| `bun_output_tags` | 输出标签宏 |
| `bun_css_derive` | CSS 属性/值类型的派生宏 |

---

## Rust 依赖关系图

```
  bun_runtime (顶层 — JS 可见 API、服务器、Shell、Bake、CLI、测试运行器)
     |
  bun_jsc (JSC 值类型、VM、事件循环、模块加载器、调试器)
     |
  +----------------------------+----------------------------+
  |                            |                            |
  bun_bundler              bun_install                bun_http
  (打包管道：解析、       (包管理器：npm            (HTTP 客户端
   链接、代码生成)          注册表、锁文件、          实现、H2/H3)
                             提取、安装)
  |                            |                            |
  +----------------------------+----------------------------+
  |
  bun_js_parser + bun_js_printer + bun_resolver + bun_css + bun_ast
  |
  bun_paths + bun_collections + bun_threading + bun_semver + bun_sourcemap
  |
  bun_sys + bun_io + bun_event_loop + bun_ptr + bun_dispatch
  |
  bun_core + bun_alloc (字符串、格式化、环境变量、堆、分配器)
```

每个层面都有对应的 `*_jsc` crate 提供 JSC 绑定，以及 `*_sys` crate 提供系统级 C FFI 绑定。

---

## Zig 代码：移植参考（不编译）

原始 Bun 完全用 Zig 编写（~710k 行），现已移植到 Rust。Zig 文件保留在源代码树中作为逐行移植参考。

### Zig 文件分布

| 模块 | .zig 文件数 | 覆盖范围 |
|------|-----------|---------|
| `src/jsc/` | ~113 | JavaScriptCore 绑定：虚拟机、JSValue、JSObject、字符串、Promise、堆、FFI 类型、调试器、异常处理、WebWorker |
| `src/install/` | ~69 | 包管理器：lockfile、解析器、tarball 提取、隔离安装 |
| `src/bundler/` | ~50 | 打包器/链接器：链接图、块生成、AST、HTML/JS/CSS 处理 |
| `src/bun_core/` | ~30 | 基础工具：字符串、格式化、env、输出、tty、结果类型 |
| `src/js_parser/` | ~30 | JavaScript/TypeScript 解析器：词法分析、解析、扫描、降级、AST 遍历 |
| `src/runtime/` | ~25 | 运行时 API：node 兼容层、Bun API、shell、fetch、bake、CLI |
| `src/css/` | ~20 | CSS 解析器和属性定义 |
| `src/sys/` | ~15 | 跨平台系统调用封装 |
| `src/js_printer/` | ~10 | JavaScript 代码生成器 |
| `src/resolver/` | ~10 | 模块解析：package.json、tsconfig、文件系统 |
| `src/http/` | ~15 | HTTP 客户端(h2、h3)和 HTTP 类型 |

### 最大的 Zig 文件（Top 10）

```
35,621  src/bun_core/string/immutable/grapheme_tables.zig  (生成的 Grapheme 数据表)
19,306  src/boringssl_sys/boringssl.zig                     (BoringSSL FFI 绑定)
10,362  src/css/properties/properties_generated.zig          (生成的 CSS 属性数据)
 7,344  src/runtime/node/node_fs.zig                        (Node.js fs 适配器)
 7,329  src/css/css_parser.zig                              (CSS 解析器)
 6,966  src/js_parser/p.zig                                  (JS 解析器主体)
 6,422  src/js_printer/js_printer.zig                        (JS 代码生成器)
 5,743  src/parsers/yaml.zig                                 (YAML 解析器)
 5,155  src/runtime/webcore/Blob.zig                         (Blob 实现)
 4,879  src/runtime/api/bun/h2_frame_parser.zig             (HTTP/2 帧解析器)
```

---

## C++ 架构：JavaScriptCore 集成

C++ 代码集中在 `src/jsc/bindings/`（~402 文件），是 Bun 与 WebKit JavaScriptCore 引擎的桥接层。

### 手写 C++ 文件（关键部分）

#### 平台胶水层
- **`root.h`** — 主包含头，必须在所有 WebCore/JSC 头之前包含。定义 `BUN_EXPORT`，转发 `BunString`
- **`headers-handwritten.h`** — 跨 C/Zig 边界的基本类型：`ZigString`、`BunString`、`BunStringTag`、`VirtualMachine` 前向声明
- **`headers.h`** / **`headers-cpp.h`** — 聚合头文件

#### 全局对象和虚拟机
- **`ZigGlobalObject.cpp/.h`** — JS 全局对象，注册所有内置类、函数和属性
- **`BunObject.cpp/.h`** — `Bun` 全局对象（`Bun.version`、`Bun.nanoseconds()`）
- **`BunProcess.cpp/.h`** — Node.js 兼容的 `process` 对象
- **`BunClientData.cpp/.h`** — 每个 `JSGlobalObject` 的客户端数据

#### 各 JS 类实现

| 类 | 文件 |
|----|------|
| Blob | `blob.cpp/.h` |
| Buffer | `JSBuffer.cpp/.h`、`JSBufferList.cpp/.h` |
| 压缩/解压流 | `JSCompressionStream.cpp/.h`、`JSDecompressionStream.cpp/.h` |
| Crypto (X509Certificate) | `JSX509Certificate.cpp` + `*Constructor.cpp` + `*Prototype.cpp` |
| Fetch | `NodeFetch.cpp/.h` |
| 文件系统 (process.binding('fs')) | `ProcessBindingFs.cpp/.h` |
| HTTP (Node.js 兼容) | `NodeHTTP.cpp/.h` |
| TLS | `NodeTLS.cpp/.h` |
| URL | `NodeURL.cpp/.h`、`DOMURL.cpp/.h` |
| NodeVM | `NodeVM.cpp`、`NodeVMModule.cpp`、`NodeVMScript.cpp` 等 |
| WebSocket/Cookie | `Cookie.cpp`、`CookieMap.cpp` |
| 流(Streams) | `Sink.h`、`ResumableSink`(通过 `.classes.ts`) |
| 文本编解码 | `TextCodec.cpp` + CJK/SingleByte/UserDefined/Replacement 变体 |
| Crypto/非对称密钥 | `AsymmetricKeyValue.cpp/.h` |
| Undici(HTTP 客户端) | `Undici.cpp/.h` |

#### Node.js Polyfill 层 (`process.binding(...)` 模块)
- `ProcessBindingBuffer.cpp`、`ProcessBindingConstants.cpp`、`ProcessBindingFs.cpp`
- `ProcessBindingHTTPParser.cpp`、`ProcessBindingNatives.cpp`
- `ProcessBindingTTYWrap.cpp`、`ProcessBindingUV.cpp`

#### 其他子系统
- **N-API**：`napi.cpp`、`napi_external.cpp`、`napi_finalizer.cpp`、`napi_handle_scope.cpp`
- **Crypto**：`ncrypto.cpp`、`ncrpyto_engine.cpp`
- **错误基础设施**：`ErrorCode.cpp`、`ErrorStackTrace.cpp`、`FormatStackTraceForJS.cpp`
- **IPC**：`IPC.cpp`
- **异步上下文**：`AsyncContextFrame.cpp`
- **Fuzzing**：`fuzzilliREPRL.cpp`

### C++/Zig/Rust 桥接层

```
JavaScript (src/js/out/... 内置模块)
    |
    |  (via JSClassCreate/JSObjectSet/函数调用)
    v
C++    (src/jsc/bindings/ — JSC NativeCallFrame, JSC::JSValue)
    |
    |  [[ZIG_EXPORT]] 注解被 cppbind.ts 解析
    |  ZigGeneratedClasses.cpp 从 .classes.ts 生成
    |  GeneratedBindings.h 从 .bind.ts 生成
    v
Zig    (src/jsc/bindings/ZigGeneratedClasses.zig 等)
    |
    |  src/jsc/*.zig 中的手写 Zig 包装器
    v
Rust   (src/jsc/*.rs — bun_jsc crate)
```

---

## 代码生成系统

位于 `src/codegen/`，是 TypeScript 驱动的代码生成系统，在构建时自动运行。

### 生成器清单

| 生成器 | 行数 | 功能 |
|--------|------|------|
| `generate-classes.ts` | ~3,975 | 从 `.classes.ts` 生成 C++/Zig/Rust 类绑定（原型、构造函数、属性表、DOMJIT 快速路径） |
| `bindgen.ts` | ~1,676 | 解析 `.bind.ts` 文件，生成 C++ 头 + Zig 代码，使用复杂类型系统和 C-ABI 降级策略 |
| `cppbind.ts` | ~1,180 | 扫描 C++ 源文件中 `[[ZIG_EXPORT]]` 注解的函数，生成 Zig 绑定 |
| `generate-jssink.ts` | ~1,234 | 生成 JavaScript Sink(WritableStream) 的 C++/Zig 代码 |
| `bundle-functions.ts` | ~854 | 将 JS 函数打包为紧凑内置模块 |
| `bundle-modules.ts` | ~588 | 将 JS 模块打包为运行时优化格式 |
| `generate-js2native.ts` | ~380 | 管理 `$zig`/`$cpp` 预处理器指令，使 JS 内置模块可调用原生函数 |
| `bake-codegen.ts` | ~210 | Bake 运行时(SSR 支持)代码生成 |

### `.classes.ts` 文件（29 个，定义 JS 类规范）

| 领域 | 文件 |
|------|------|
| 运行时 API | `Archive.classes.ts`、`BunObject.classes.ts`、`Glob.classes.ts`、`JSBundler.classes.ts`、`ParsedShellScript.classes.ts`、`ResumableSink.classes.ts`、`S3Client.classes.ts`、`SecureContext.classes.ts`、`Shell.classes.ts`、`Terminal.classes.ts` |
| 网络 | `h2.classes.ts`、`cron.classes.ts`、`filesystem_router.classes.ts`、`html_rewriter.classes.ts`、`sourcemap.classes.ts`、`sql.classes.ts`、`streams.classes.ts`、`zlib.classes.ts` |
| Node.js 兼容 | `node.classes.ts`、`crypto.classes.ts`、`jest.classes.ts` |
| Web 标准 | `encoding.classes.ts`、`response.classes.ts` |
| 其他 | `ffi.classes.ts`、`Image.classes.ts`、`server.classes.ts`、`sockets.classes.ts`、`valkey.classes.ts` |

---

## 构建系统

构建系统由 TypeScript 脚本协调，生成 Ninja 构建文件。

### 构建流程

1. **代码生成** — 运行所有生成器，产生 `.cpp`/`.h`/`.zig`/`.rs` 工件到 `build/<profile>/codegen/`
2. **Cargo 构建** — 编译所有 Rust crate → `libbun_rust.a`
3. **C++ 编译** — 使用 Clang 编译所有 C/C++ 源(含预编译头)
4. **链接** — 将所有内容链接为最终可执行文件

### 关键构建脚本 (`scripts/build/`)

| 文件 | 功能 |
|------|------|
| `bun.ts` | 主 Bun 构建步骤 |
| `rust.ts` | Rust/Cargo 编译集成 |
| `compile.ts` | C++ 通过 clang 编译 |
| `codegen.ts` | **核心编排器**：为所有代码生成步骤定义 ninja 规则 |
| `config.ts` | 构建配置(目标三元组、路径、标志) |
| `ninja.ts` | Ninja 构建文件写入器 |
| `source.ts` | 源文件发现/globbing(最大的构建脚本) |
| `tools.ts` | 工具链发现(编译器、链接器) |
| `flags.ts` | 编译器标志管理 |
| `download.ts` | 依赖下载(WebKit 等) |

构建命令：
- `bun bd` — 构建调试版本
- `bun run build` — 构建调试版本(同 `bun bd`)
- `bun run build:release` — 构建发布版本
- `bun run rust:check-all` — 跨所有目标(linux/macos/windows × x64/aarch64)检查编译

---

## 测试组织

```
test/
├── js/bun/        — Bun 特定 API 测试(http、crypto、ffi、shell 等)
├── js/node/       — Node.js 兼容性测试
├── js/web/        — Web API 测试(fetch、WebSocket、streams 等)
├── cli/           — CLI 命令测试(install、run、test)
├── bundler/       — 打包器和转译器测试(使用 itBundled 辅助)
├── integration/   — 端到端集成测试
├── napi/          — N-API 兼容性测试
├── v8/            — V8 C++ API 兼容性测试
└── regression/issue/${issueNumber}.test.ts — 回归测试
```

---

## Vendor 目录（第三方依赖）

Bun 将大量 C/C++ 库直接 vendoring 到源码树中：

| 目录 | 库 | 用途 |
|------|-----|------|
| `vendor/boringssl/` | BoringSSL | TLS/加密 |
| `vendor/brotli/` | Brotli | 压缩 |
| `vendor/cares/` | c-ares | 异步 DNS |
| `vendor/hdrhistogram/` | HdrHistogram | 延迟跟踪 |
| `vendor/highway/` | Google Highway | SIMD |
| `vendor/libarchive/` | libarchive | tar/zip |
| `vendor/libdeflate/` | libdeflate | 快速 deflate |
| `vendor/libuv/` | libuv | Windows 事件循环 |
| `vendor/lolhtml/` | lol-html | HTML 重写器 |
| `vendor/lshpack/` | ls-hpack | HTTP/2 HPACK |
| `vendor/lsqpack/` | ls-qpack | HTTP/3 QPACK |
| `vendor/lsquic/` | lsquic | QUIC/HTTP/3 |
| `vendor/mimalloc/` | mimalloc | 内存分配器 |
| `vendor/nodejs/` | Node.js headers | 兼容性 |
| `vendor/picohttpparser/` | PicoHTTPParser | HTTP 解析 |
| `vendor/tinycc/` | TinyCC(oven-sh fork) | FFI JIT 编译器 |
| `vendor/WebKit/` | WebKit/JavaScriptCore | JS 引擎 |
| `vendor/zig/` | Zig 工具链 | 遗留，Rust 构建不使用 |
| `vendor/zlib/` | zlib-ng | 压缩(zlib 兼容模式) |
| `vendor/zstd/` | Zstandard | 压缩 |

---

## 关键架构决策

1. **Rust 为主语言，Zig 为参考**：原始 ~710k 行 Zig 代码已逐行移植到 ~981k 行 Rust，Zig 文件保留在源码树中但不参与编译
2. **Cargo workspace 分 111 个 crate**：高内聚低耦合，每个 crate 职责明确
3. **自定义数据结构**：`bun_collections` 提供针对 Bun 分配模式定制的 SoA/哈希表/BitSet 等数据结构
4. **链接时接口分发**：`bun_dispatch::link_interface!` 避免动态分发，通过链接时多态实现跨 crate 接口
5. **代码生成驱动**：JS 类的定义通过 `.classes.ts` 和 `.bind.ts` 文件声明，构建时自动生成 C++/Zig/Rust 胶水代码
6. **多层 JSC 绑定**：C++ ↔ Zig ↔ Rust 三层桥接，最终在 `bun_jsc` crate 中提供类型安全的 Rust API
