---
title: JSON-RPC 是什么
aliases:
  - JSON-RPC
  - RPC 通信协议
tags:
  - Agent
  - Protocol
  - API
  - JSON
---

# JSON-RPC 是什么

## 一句话理解

**JSON-RPC 是一种远程过程调用（RPC）协议，规定通信双方如何用 JSON 表达方法调用、参数、响应结果和错误。**

它与通信有关，但要注意：**JSON-RPC 主要规定消息格式与交互规则，不负责规定底层必须如何传输数据。**

## 1. 基本概念

### RPC（Remote Procedure Call）

远程过程调用。客户端可以像调用一个方法一样，请求另一台机器或另一个进程执行操作，并取得返回结果。

### JSON

一种结构化数据交换格式。JSON-RPC 使用 JSON 对请求和响应进行编码。

### JSON-RPC

把 RPC 调用约定与 JSON 消息格式结合起来，明确请求应该包含什么、响应如何对应请求、错误如何表达等。

## 2. 一次 JSON-RPC 调用是什么样的？

客户端请求：

```json
{
  "jsonrpc": "2.0",
  "method": "search",
  "params": {
    "query": "JSON-RPC"
  },
  "id": 1
}
```

服务端成功响应：

```json
{
  "jsonrpc": "2.0",
  "result": ["文档 A", "文档 B"],
  "id": 1
}
```

关键字段：

| 字段 | 含义 |
|---|---|
| `jsonrpc` | 协议版本，JSON-RPC 2.0 中为 `"2.0"` |
| `method` | 要调用的方法名 |
| `params` | 传递给方法的参数，可使用对象或数组形式 |
| `id` | 请求标识；响应会使用对应的 ID，帮助客户端匹配请求与响应 |
| `result` | 成功响应中的结果 |
| `error` | 失败响应中的错误对象，通常包含 `code`、`message`，也可包含 `data` |

### 错误响应示例

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found"
  },
  "id": 1
}
```

### 请求、响应与通知

- **Request（请求）**：包含 `method`，通常带有 `id`，用来等待对应响应。
- **Response（响应）**：包含 `result` 或 `error`，并携带对应的 `id`。
- **Notification（通知）**：不带 `id` 的请求；发送方不等待 JSON-RPC 响应。

因此，并不是每条 JSON-RPC 消息都一定有返回结果。

## 3. JSON-RPC 与传输方式的关系

JSON-RPC 不强制绑定某一种传输方式。消息可以通过不同通道发送，例如：

- **HTTP**：常见的 Web 通信方式。
- **WebSocket**：支持双向、持续的连接。
- **stdio**：本地进程通过标准输入和标准输出交换消息。

可以把一次调用拆成三个层次：

```text
应用层：要执行什么操作？
        例如 search(query="JSON-RPC")
             ↓
消息协议层：如何表示请求与响应？
        JSON-RPC：method、params、id、result / error
             ↓
传输层：消息如何到达对端？
        HTTP / WebSocket / stdio 等
```

**JSON-RPC ≠ HTTP，也不等于 WebSocket。** 它们解决的是不同层面的问题。

## 4. 经常一起出现的概念

| 名词 | 是什么 | 与 JSON-RPC 的关系 |
|---|---|---|
| **RPC** | 远程过程调用的思想与机制 | JSON-RPC 是 RPC 的一种协议实现形式 |
| **JSON** | 数据交换格式 | JSON-RPC 用 JSON 编码消息 |
| **HTTP / WebSocket / stdio** | 消息传输通道 | 可用于承载 JSON-RPC 消息 |
| **MCP** | 面向 AI 应用与工具、资源等交互的协议 | MCP 使用 JSON-RPC 风格的消息结构，并定义更具体的交互方法与语义 |
| **LSP** | Language Server Protocol，为编辑器提供代码补全、诊断等能力 | LSP 使用 JSON-RPC 进行请求、响应与通知 |
| **REST API** | 常见的 Web API 设计风格 | 常与 RPC 风格对比，但两者并非同一层面的严格互斥选项 |
| **gRPC** | RPC 框架与生态 | 也用于远程服务调用，但与 JSON-RPC 在接口定义、序列化和工具链等方面有区别 |

## 5. JSON-RPC 与 REST API 的区别

| 对比维度 | JSON-RPC 风格 | REST 风格 |
|---|---|---|
| 主要表达方式 | 调用某个方法 | 访问或操作某个资源 |
| 示例 | `user.get`，参数为 `id=1` | `GET /users/1` |
| 常见关注点 | 方法名、参数、调用结果 | 资源路径、HTTP 方法、状态码等 |
| 是否必须使用 HTTP | 否 | 通常以 HTTP 为基础 |

以上是典型风格的对比，不代表所有 API 都严格符合某一种模式。实际系统也可以在 HTTP 上承载 RPC 风格的 API。

## 6. 在 Agent 与 MCP 中的作用

以 MCP 为例，可以把各层职责理解为：

```text
Agent / MCP Client
      │ 决定请求哪个 MCP 方法、传入什么参数
      ▼
JSON-RPC 消息结构
      │ 表达 method、params、id，以及 result / error
      ▼
传输通道（例如 stdio 或 HTTP）
      ▼
MCP Server
      │ 按 MCP 规范处理请求、执行操作
      ▼
返回 JSON-RPC 响应
```

例如，MCP 客户端可以通过 `tools/list` 请求工具列表，通过 `tools/call` 请求调用工具。

这里有三个不同层次：

- **MCP**：定义客户端与服务端可以进行哪些交互，以及交互的语义。
- **JSON-RPC**：定义这些交互消息如何表达，以及请求与响应如何关联。
- **stdio / HTTP 等传输方式**：负责实际传递消息。

JSON-RPC 本身并不负责决定模型应该使用哪个工具，也不负责执行工具；这些逻辑属于 Agent、客户端实现或服务端程序。

## 7. 常见误区

1. **JSON-RPC 不是 JSON 的别名。** JSON 是数据格式，JSON-RPC 则是在 JSON 之上约定了一套远程调用消息结构。
2. **JSON-RPC 不等于 HTTP。** HTTP 可以传输 JSON-RPC 消息，但 JSON-RPC 也可通过其他通道传递。
3. **JSON-RPC 不等于 MCP。** JSON-RPC 是消息协议；MCP 在其基础上定义面向 AI 客户端和服务端的交互规范。
4. **`method` 不是工具执行代码本身。** 它只是请求中标明要调用的方法；服务端需要根据该名称找到对应的处理逻辑。
5. **有请求不一定有响应。** JSON-RPC Notification 不带 `id`，发送方不等待 JSON-RPC 响应。

## 总结

> **JSON-RPC = 用 JSON 表达远程方法调用的消息协议。**
>
> 记忆顺序：RPC 说明“远程调用”这个模式，JSON 提供数据格式，JSON-RPC 规定请求与响应怎么写，HTTP / WebSocket / stdio 负责传输，而 MCP、LSP 等协议会利用 JSON-RPC 来完成更具体的交互。

## 参考资料

- [JSON-RPC 2.0 官方规范](https://www.jsonrpc.org/specification)
- [Model Context Protocol 规范](https://modelcontextprotocol.io/specification)
- [Language Server Protocol 规范](https://microsoft.github.io/language-server-protocol/)
