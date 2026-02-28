# JS REPL 多模态工具实现分析

## 概述

JS REPL 是 Codex 项目中的 JavaScript 交互式执行环境，让 AI 模型可以编写和执行 JavaScript 代码。最近的重构增加了多模态输出支持——工具现在可以返回文本和图像混合内容。

## 架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AI Model (Claude)                          │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │ Tool Call
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     js_repl Handler (Rust)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ FunctionCallOutputBody::ContentItems(items)                 │   │
│  │   ├── InputText { text: "执行结果..." }                      │   │
│  │   └── InputImage { image_url: "data:image/png;base64,..." } │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │ JSON-lines Protocol
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Node.js Kernel (kernel.js)                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ codex.tool("view_image", { path: "/abs/path.png" })         │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │ Tool Request
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   ToolRouter (Rust)                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ dispatch_tool_call() → Tool Execution                       │   │
│  │ response_content_items() → 提取多模态内容                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 数据结构

### FunctionCallOutputContentItem

```rust
// codex-rs/protocol/src/models.rs
pub enum FunctionCallOutputContentItem {
    InputText { text: String },
    InputImage { image_url: String },
}
```

文本类型用于普通输出，图像类型通过 base64 编码的 data URL 传递图像数据。

### JsExecResult

```rust
// codex-rs/core/src/tools/js_repl/mod.rs
pub struct JsExecResult {
    pub output: String,                              // 文本输出
    pub content_items: Vec<FunctionCallOutputContentItem>,  // 多模态内容
}
```

`output` 存控制台输出，`content_items` 存工具调用产生的多模态内容。

### ExecToolCalls

```rust
struct ExecToolCalls {
    in_flight: usize,           // 正在执行的工具数量
    content_items: Vec<FunctionCallOutputContentItem>,  // 累积的内容项
    notify: Arc<Notify>,        // 通知机制
    cancel: CancellationToken,  // 取消令牌
}
```

追踪单次执行中所有工具调用的状态，累积它们产生的多模态内容。

## 通信协议

Rust 宿主与 Node.js 内核通过 stdin/stdout 进行 JSON-lines 通信。

请求消息 (Rust → Node.js):
```json
{"id": "exec-123", "type": "exec", "code": "const result = await codex.tool('view_image', {path: '/tmp/img.png'});"}
```

工具调用请求 (Node.js → Rust):
```json
{
  "type": "run_tool",
  "id": "exec-123-tool-0",
  "exec_id": "exec-123",
  "tool_name": "view_image",
  "arguments": "{\"path\":\"/tmp/img.png\"}"
}
```

工具调用响应 (Rust → Node.js):
```json
{
  "type": "tool_response",
  "id": "exec-123-tool-0",
  "ok": true,
  "response": {"description": "Image displayed: img.png"}
}
```

执行结果 (Node.js → Rust):
```json
{
  "type": "result",
  "id": "exec-123",
  "output": "[object Object]",
  "content_items": [
    {"type": "input_text", "text": "图像已显示"},
    {"type": "input_image", "image_url": "data:image/png;base64,iVBORw0KGgo..."}
  ]
}
```

## 实现流程

### JavaScript 端的 tool() API

```javascript
// kernel.js (lines 485-514)
const tool = (toolName, args) => {
  if (typeof toolName !== "string" || !toolName) {
    return Promise.reject(new Error("codex.tool expects a tool name string"));
  }
  const id = `${message.id}-tool-${toolCounter++}`;
  let argumentsJson = "{}";
  if (typeof args === "string") {
    argumentsJson = args;
  } else if (typeof args !== "undefined") {
    argumentsJson = JSON.stringify(args);
  }

  return new Promise((resolve, reject) => {
    const payload = {
      type: "run_tool",
      id,
      exec_id: message.id,
      tool_name: toolName,
      arguments: argumentsJson,
    };
    send(payload);
    pendingTool.set(id, (res) => {
      if (!res.ok) {
        reject(new Error(res.error || "tool failed"));
        return;
      }
      resolve(res.response);
    });
  });
};
```

创建唯一 ID、序列化参数、发送请求、等待响应。

### Rust 端的工具路由

```rust
// mod.rs (lines 1277-1399)
async fn run_tool_request(
    exec: ExecContext,
    req: RunToolRequest,
    exec_tool_calls: Arc<Mutex<HashMap<String, ExecToolCalls>>>,
) -> RunToolResult {
    let router = ToolRouter::from_config(
        &exec.turn.tools_config,
        Some(mcp_tools),
        None,
        exec.turn.dynamic_tools.as_slice(),
    );

    let call = crate::tools::router::ToolCall {
        tool_name: tool_name.clone(),
        call_id: req.id.clone(),
        payload,
    };

    match router.dispatch_tool_call(...).await {
        Ok(response) => {
            if let Some(items) = response_content_items(&response) {
                Self::record_exec_tool_call_content_items(
                    &exec_tool_calls,
                    &req.exec_id,
                    items,
                ).await;
            }
        }
    }
}
```

### 内容项累积

```rust
async fn record_exec_tool_call_content_items(
    exec_tool_calls: &Arc<Mutex<HashMap<String, ExecToolCalls>>>,
    exec_id: &str,
    items: Vec<FunctionCallOutputContentItem>,
) {
    let mut map = exec_tool_calls.lock().await;
    if let Some(exec) = map.get_mut(exec_id) {
        exec.content_items.extend(items);
        exec.in_flight = exec.in_flight.saturating_sub(1);
        if exec.in_flight == 0 {
            exec.notify.notify_one();
        }
    }
}
```

工具调用完成后，多模态内容被累积到 `content_items` 列表。

### 输出组装

```rust
// handlers/js_repl.rs (lines 157-164)
let content = result.output;
let mut items = Vec::with_capacity(result.content_items.len() + 1);
if !content.is_empty() {
    items.push(FunctionCallOutputContentItem::InputText {
        text: content.clone()
    });
}
items.extend(result.content_items);

Ok(FunctionCallOutputBody::ContentItems(items))
```

文本内容和多模态内容项合并，形成完整响应。

## 自定义工具的多模态输出

### 动态工具

动态工具（Dynamic Tools）可以返回多模态内容：

```rust
#[test]
fn dynamic_tool_can_return_multimodal_content() {
    let dynamic_tool = DynamicToolSpec {
        name: "generate_chart".to_string(),
        description: "生成图表".to_string(),
        input_schema: json!({"type": "object"}),
    };

    let response = json!({
        "content_items": [
            {"type": "input_text", "text": "图表生成完成"},
            {"type": "input_image", "image_url": "data:image/png;base64,..."}
        ]
    });
}
```

### view_image 工具

```rust
// handlers/view_image.rs
pub(crate) fn local_image_content_items_with_label_number(
    path: &Path,
    label_number: Option<usize>,
) -> Result<Vec<FunctionCallOutputContentItem>> {
    let image_data = std::fs::read(path)?;
    let base64 = base64::encode(&image_data);
    let mime_type = guess_mime_type(path);

    let label = label_number
        .map(|n| format!("[Image {}] ", n))
        .unwrap_or_default();

    Ok(vec![
        FunctionCallOutputContentItem::InputText {
            text: format!("{}{}", label, path.display()),
        },
        FunctionCallOutputContentItem::InputImage {
            image_url: format!("data:{};base64,{}", mime_type, base64),
        },
    ])
}
```

读取本地图像文件，转换为 base64 编码的 data URL。

## 使用示例

在 JS REPL 中查看图像：

```javascript
const imagePath = '/Users/user/screenshot.png';
await codex.tool('view_image', { path: imagePath });
```

AI 模型收到图像内容后可以进行视觉分析。

调用自定义多模态工具：

```javascript
const chart = await codex.tool('generate_chart', {
    type: 'bar',
    data: [10, 20, 30, 40]
});
console.log(chart);
```

## 设计要点

- 异步工具调用：Promise 机制，支持 `await`
- 内容项分离：`output` 存控制台输出，`content_items` 存结构化内容
- 执行隔离：每次执行有独立的 `exec_id`
- 防止递归：`is_js_repl_internal_tool()` 检查避免 JS REPL 调用自身
