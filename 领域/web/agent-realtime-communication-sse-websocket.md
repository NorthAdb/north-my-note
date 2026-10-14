---
title: Agent 实时通信：SSE、WebSocket 与 Agent Harness
tags:
  - Agent
  - Agent-Harness
  - WebSocket
  - SSE
  - Realtime
  - Event-Protocol
  - State-Machine
---

# Agent 实时通信：SSE、WebSocket 与 Agent Harness

> 核心结论：**Agent 不应该先问“要不要 WebSocket”，而应该先拆分通信职责：命令、流式输出、实时控制、状态同步。**
>
> 一个常见且实用的默认架构是：
>
> **HTTP POST 负责命令，SSE 负责服务器到客户端的流式事件，WebSocket 负责需要低延迟双向通信的控制与协作，WebRTC 负责实时音频。**
>
> 真正值得学习的重点不是 WebSocket API，而是：
>
> **Transport → Event Protocol → Event Log → State Machine → Agent Loop → Cancellation / Recovery**

---

## 1. Agent 为什么需要实时通信？

传统 HTTP 更接近：

```text
Client
  │
  │ Request
  ▼
Server
  │
  │ Response
  ▼
Client
```

但 Agent 的执行通常不是瞬时完成的，而是：

```text
User Message
     │
     ▼
Agent Run
     │
     ├── LLM
     ├── Tool Call
     ├── Tool Result
     ├── RAG
     ├── Sub-agent
     ├── More LLM
     └── Final Answer
```

因此前端通常希望在 Agent 执行期间持续看到：

```text
Agent started
↓
Searching...
↓
Tool called
↓
Tool completed
↓
Generating...
↓
Completed
```

这就产生了 **Realtime Agent Communication** 的需求。

---

# 2. 一个完整的 Agent Realtime 架构

```text
                    ┌──────────────┐
                    │   Browser    │
                    │   / Client   │
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
         HTTP POST         SSE        WebSocket
            │              │              │
         Commands      Streaming       Control
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                 ┌──────────────────┐
                 │ Realtime Gateway │
                 └────────┬─────────┘
                          ▼
                 ┌──────────────────┐
                 │   Agent Runtime │
                 │                  │
                 │ Agent Loop       │
                 │ LLM              │
                 │ Tools            │
                 │ Memory / RAG     │
                 │ Sub-agents       │
                 └────────┬─────────┘
                          │
                 ┌────────┼────────┐
                 ▼        ▼        ▼
                LLM      Tools     DB
```

其中：

- **Transport Layer**：HTTP / SSE / WebSocket / WebRTC
- **Event Protocol**：定义事件的类型、字段和顺序
- **Agent State Machine**：定义 Agent 当前处于什么状态
- **Agent Runtime**：真正执行 LLM、Tool、RAG、Sub-agent 等工作
- **Event Store / Message Bus**：为可靠投递、重放、恢复提供基础

---

# 3. WebSocket 在 Agent 中能做什么？

## 3.1 流式输出

可以传递：

```text
message_delta
tool_call
tool_result
progress
done
```

前端据此渲染：

```text
🤖 Agent started

🔍 Searching...

🛠 Calling search()

✅ Search completed

✍ Generating answer...

✅ Done
```

但是：

> **“需要流式输出”并不等于“必须使用 WebSocket”。**

LLM Token Streaming 通常是服务器单向推送：

```text
Client ─────────────→ Server
Client ←───────────── Server
             stream
```

因此 SSE 往往更自然。

---

## 3.2 双向实时控制

WebSocket 更适合：

```text
cancel
pause
resume
interrupt
approval
追加指令
```

例如：

```json
{
  "type": "cancel_run",
  "run_id": "run123"
}
```

典型链路：

```text
Browser
   │ cancel
   ▼
WebSocket
   │
   ▼
Agent Runtime
   │
   ├── abort LLM request
   ├── cancel tool
   ├── cancel sub-agent
   ├── cancel RAG
   └── release resources
```

关键概念是：

> **Cancellation Propagation（取消信号传播）**

不能只关闭前端连接，而必须让取消信号真正传到 Agent 执行层。

---

## 3.3 人工审批 Human-in-the-loop

例如 Agent 想执行：

```text
delete_database()
```

服务端发送：

```text
tool_approval_required
```

UI 显示：

```text
Agent wants to execute:

DELETE DATABASE

[Allow] [Reject]
```

用户操作后，再通过 HTTP 或 WebSocket 发送：

```json
{
  "type": "tool_approval",
  "call_id": "call_123",
  "decision": "allow"
}
```

这类场景比单纯的 Token Streaming 更能体现 WebSocket 的价值。

---

## 3.4 多 Agent / 多客户端协作

多个用户或 Agent 可以订阅相同的 Agent 状态：

```text
             Agent Workspace
                    │
             ┌──────┼──────┐
             │      │      │
           User A User B User C
```

例如所有客户端都看到：

```text
Researcher started
Coder running
Reviewer waiting approval
```

这时 WebSocket 更接近：

> **实时状态同步 / Collaborative State Synchronization**

---

# 4. SSE 与 WebSocket 的关系

## 4.1 SSE

SSE 基于 HTTP：

```text
Client ───── HTTP Request ─────→ Server
Client ←──── SSE Stream ─────── Server
```

特点：

- 单向：Server → Client
- 基于普通 HTTP
- 非常适合 Token Streaming
- 适合 Agent Progress / Tool Events
- 与 Nginx、CDN、HTTP 基础设施结合自然
- 原生支持 `Last-Event-ID` 这一类断线恢复语义

---

## 4.2 WebSocket

```text
Client ←──────────────→ Server
```

特点：

- 全双工
- 客户端可以随时向服务端发消息
- 服务端也可以随时推送
- 适合实时控制
- 适合多人实时协作
- 适合交互式 Agent

---

## 4.3 对比

| 场景 | SSE | WebSocket |
|---|---:|---:|
| Token Streaming | ✅ | ✅ |
| Agent Progress | ✅ | ✅ |
| Tool Events | ✅ | ✅ |
| 单向实时推送 | ✅ | ✅ |
| Cancel | ✅ HTTP 配合 | ✅ |
| Pause / Resume | ✅ HTTP 配合 | ✅ |
| Human Approval | ✅ HTTP 配合 | ✅ |
| 双向实时控制 | ❌ | ✅ |
| 多人实时协作 | ⚠️ | ✅ |
| 实时音频 | ❌ | ⚠️ |
| 简单 Agent Chat | ✅ | ✅ |

因此：

> **Agent ≠ WebSocket**

更准确地说：

> **WebSocket 是 Agent 的一种 Realtime Transport。**

---

# 5. 一个常见的折中架构：HTTP + SSE

很多 Agent 根本不需要 WebSocket。

可以采用：

```text
                Client
                  │
        ┌─────────┴─────────┐
        │                   │
    HTTP POST              SSE
        │                   │
    Commands           Event Stream
        │                   │
        └─────────┬─────────┘
                  ▼
             Agent Runtime
```

例如：

### 启动 Agent

```http
POST /runs
```

```json
{
  "session_id": "s123",
  "input": "分析这个项目"
}
```

返回：

```json
{
  "run_id": "r456"
}
```

### 订阅流

```http
GET /runs/r456/events
Accept: text/event-stream
```

返回：

```text
event: run_started
data: {...}

event: token
data: {"text":"首先"}

event: tool_called
data: {...}

event: tool_result
data: {...}

event: run_finished
data: {...}
```

### 控制

用户要取消：

```http
POST /runs/r456/cancel
```

这种方案已经可以覆盖大量 Agent 应用。

---

# 6. Agent 中真正重要的是 Event Protocol

WebSocket / SSE 只是传输通道。

真正需要设计的是：

> **事件协议（Event Protocol）**

一个事件通常至少应考虑：

```json
{
  "event_id": "evt102",
  "sequence": 102,
  "session_id": "s123",
  "run_id": "run456",
  "type": "tool_result",
  "timestamp": 1791450000,
  "data": {}
}
```

核心字段：

| 字段 | 作用 |
|---|---|
| `event_id` | 事件唯一 ID |
| `sequence` | 严格的事件顺序 |
| `session_id` | 长期会话标识 |
| `run_id` | 一次 Agent 执行标识 |
| `type` | 事件类型 |
| `timestamp` | 时间信息 |
| `data` | 事件负载 |
| `version` | Schema / 协议版本 |

### 为什么需要 `sequence`？

因为：

```text
event 100
event 101
event 102
```

必须有明确顺序。

**timestamp 不能替代 sequence。**

---

# 7. Session、Run、Event 三层 ID

建议把三个概念区分：

```text
session_id
    │
    ├── run_id #1
    │      ├── event 1
    │      ├── event 2
    │      └── event 3
    │
    └── run_id #2
           ├── event 4
           └── event 5
```

### Session

一次长期对话：

```text
session_id = s123
```

### Run

用户发起一次任务后产生的一次 Agent execution：

```text
run_id = r456
```

### Event

Run 执行过程中产生的单个状态 / 内容 / 工具事件：

```text
event_id = evt102
```

这个层次划分对于断线恢复、多次执行、日志追踪都很重要。

---

# 8. Agent State Machine

Agent 不应该理解成：

```text
Request → Response
```

而应该理解成一个状态机。

例如：

```text
                    ┌───────────┐
                    │   IDLE    │
                    └─────┬─────┘
                          │
                    user_message
                          ▼
                    ┌───────────┐
                    │  RUNNING  │
                    └─────┬─────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        LLM_STREAM    TOOL_CALL    WAITING
             │            │            │
             │            ▼            │
             │       TOOL_RESULT       │
             │            │            │
             └──────┬─────┴────────────┘
                    ▼
               COMPLETED
```

此外还可能存在：

```text
RUNNING
├── WAITING_USER
├── WAITING_APPROVAL
├── PAUSING
├── PAUSED
├── CANCELLING
├── CANCELLED
├── FAILED
└── COMPLETED
```

所以 WebSocket / SSE 发出去的并不只是“文字”，而是：

> **Agent 状态机变化产生的事件。**

---

# 9. 事件应该分层

不要把 Agent 所有输出都当成 Token。

推荐至少分为：

```text
Realtime Events
│
├── Content Events
│   ├── message_start
│   ├── message_delta
│   └── message_end
│
├── Tool Events
│   ├── tool_called
│   ├── tool_result
│   └── tool_error
│
├── Lifecycle Events
│   ├── run_started
│   ├── run_finished
│   ├── run_failed
│   └── run_cancelled
│
├── Control Events
│   ├── approval_required
│   ├── pause
│   ├── resume
│   └── interrupt
│
└── Progress Events
    ├── retrieval_started
    ├── subagent_started
    └── step_completed
```

这样前端就可以按事件类型进行不同渲染。

---

# 10. Agent 流式输出不应简单等于“暴露思考过程”

Agent UI 可以展示：

```text
Searching...
Calling tool...
Retrieving documents...
Running sub-agent...
Generating answer...
```

但并不意味着应该把模型的隐藏推理 / Chain-of-Thought 原样发送给前端。

更合理的是暴露：

```text
status
progress
tool activity
retrieval activity
assistant content
```

也就是**可观察的执行状态**，而不是内部隐藏推理。

---

# 11. 断线恢复：Connection ≠ Run

一个很重要的设计原则：

> **WebSocket / SSE 连接断开，不应该天然意味着 Agent Run 停止。**

例如：

```text
Browser
   │
   │ WebSocket / SSE
   X 断线
   │
   │
Agent Runtime
   ├── still running
   ├── tool executing
   └── eventually completed
```

因此应该分开：

```text
Connection
Session
Run
```

连接只是临时通信关系。

Run 才是真正的执行任务。

---

# 12. Event ID + Replay

假设客户端最后收到：

```text
event 105
```

然后断线。

重新连接时：

```json
{
  "type": "resume",
  "run_id": "run123",
  "last_event_id": 105
}
```

服务端补发：

```text
106
107
108
109
110
```

因此完整的断线恢复需要：

```text
Event ID
+
Sequence
+
Event Store
+
Replay
```

### SSE 的优势

SSE 天生有类似：

```http
Last-Event-ID: 105
```

的机制。

但：

> **浏览器自动重连 ≠ Agent 状态恢复**

自动重连只能解决“再次建立连接”，不能保证中间事件不丢。

---

# 13. 为什么需要 Event Store？

不要把 WebSocket 当成唯一的数据源：

```text
Agent
  ↓
WebSocket
  ↓
Browser
```

更可靠的架构是：

```text
             Agent Runtime
                    │
                    ▼
              Event Store
                    │
            ┌───────┼────────┐
            ▼       ▼        ▼
        WebSocket   SSE     Other Consumers
            │       │
            ▼       ▼
         Browser   Browser
```

Event Store / Event Stream 可以采用：

```text
PostgreSQL
Redis Streams
Kafka
NATS
```

等方案，具体选择取决于可靠性、吞吐量、持久化和消费模型。

核心思想：

> **WebSocket / SSE 是实时投递渠道，Event Store 才是恢复和重放的依据。**

---

# 14. Idempotency：防止断线重试导致重复执行

一个典型问题：

```text
Client
  │
  │ create_order()
  ▼
Server
  │
  │ 已经执行成功
  X 网络断开
```

客户端不知道是否成功，于是再次发送：

```text
create_order()
```

结果可能创建两个订单。

因此 Tool Calling / Command 层需要：

```text
call_id
request_id
idempotency_key
```

例如：

```json
{
  "call_id": "call_123",
  "tool": "create_order",
  "idempotency_key": "order-request-789"
}
```

服务端保证相同的幂等键不会重复执行危险操作。

---

# 15. Backpressure：生产速度和消费速度不一致

例如：

```text
Agent
1000 events/s
      │
      ▼
Network
      │
      ▼
Browser
100 events/s
```

就会出现：

```text
Buffer
████████████████████
```

因此必须考虑：

```text
buffer
queue
batching
flow control
backpressure
```

例如 Token Streaming 可以合并多个 delta：

```text
token
token
token
token
```

变成：

```text
message_delta:
"Hello, this is..."
```

没必要每个 token 都单独产生一个网络消息。

---

# 16. 代理缓冲是 SSE / Streaming 的生产环境常见坑

典型链路：

```text
Agent
  ↓
Nginx / CDN / Proxy
  ↓
Browser
```

Agent 端明明：

```text
token
token
token
token
```

如果反向代理进行了 buffering：

```text
token ┐
token ├── buffer
token ┤
token ┘
  ↓
一次性发送
```

最终用户看到的就不是实时输出。

因此生产环境必须检查：

```text
proxy buffering
response streaming
timeouts
idle timeouts
CDN behavior
load balancer behavior
```

---

# 17. Heartbeat / Keepalive

长时间 Agent Run 可能几十分钟甚至更长。

中间设备：

```text
Nginx
Cloudflare
Load Balancer
NAT
Firewall
```

可能因为长时间没有数据而主动断开连接。

因此需要：

```text
heartbeat
keepalive
ping/pong
SSE comment event
```

注意：

> 心跳解决的是“连接存活”，不解决“事件可靠性”。

可靠性仍然需要 Event ID + Replay。

---

# 18. Tool Call 的流式 JSON 处理

这与 Agent 中 JSON / JSON Schema 的问题直接相关。

LLM 可能逐步生成：

```text
{
  "tool":
  "search",
  "query":
  "..."
}
```

在生成过程中，数据可能只是：

```text
{"tool":"sea
```

因此不能在任意 delta 到达时立即执行：

```python
json.loads(...)
```

更合理的是：

```text
LLM Delta
    ↓
Accumulator
    ↓
Content Block Complete
    ↓
检查 stop_reason
    ↓
Schema / JSON Validation
    ↓
JSON Parse
    ↓
Tool Execution
```

特别要检查：

```text
stop_reason
```

例如如果因为 `max_tokens` 截断：

```text
{
  "tool": "search",
  "query": "xxx
```

即使看起来“快结束了”，也不能直接当成完整 Tool Call。

因此：

> **流式 JSON 的完整性边界与 Agent Tool Calling 强相关。**

---

# 19. HTTP / SSE / WebSocket / WebRTC / Message Bus 的职责划分

可以形成一个更完整的通信地图：

```text
┌──────────────────────────────────────────────┐
│                 Agent System                 │
├──────────────────────────────────────────────┤
│ User Commands                                │
│ → HTTP POST                                  │
├──────────────────────────────────────────────┤
│ Server → Client Streaming                    │
│ → SSE / WebSocket                            │
├──────────────────────────────────────────────┤
│ Interactive Bidirectional Control             │
│ → WebSocket / HTTP                           │
├──────────────────────────────────────────────┤
│ Realtime Audio                               │
│ → WebRTC                                     │
├──────────────────────────────────────────────┤
│ Agent ↔ Agent / Internal Async Work           │
│ → Redis / Kafka / NATS 等                     │
├──────────────────────────────────────────────┤
│ Reliable Replay / Durable State               │
│ → Event Store / Database / Stream             │
└──────────────────────────────────────────────┘
```

核心原则：

> **不要因为一个数据“实时”，就默认选择 WebSocket。**

应该先判断：

1. 数据流向：单向还是双向？
2. 延迟要求：普通实时还是极低延迟？
3. 是否需要持续订阅？
4. 是否需要客户端随时控制 Agent？
5. 是否需要多人共享状态？
6. 是否需要断线重放？
7. 是否需要可靠投递和幂等执行？

---

# 20. 从 Agent Harness 角度重新理解整套系统

这是理解这套技术最重要的视角。

```text
┌───────────────────────────────────────┐
│ 6. UI / Client                        │
│ Chat / Progress / Tool UI             │
├───────────────────────────────────────┤
│ 5. Transport Layer                    │
│ HTTP / SSE / WebSocket / WebRTC       │
├───────────────────────────────────────┤
│ 4. Event Protocol                     │
│ event_id / sequence / schema          │
├───────────────────────────────────────┤
│ 3. Agent State Machine                │
│ running / waiting / paused / done     │
├───────────────────────────────────────┤
│ 2. Agent Loop                         │
│ LLM → Tool → Result → LLM             │
├───────────────────────────────────────┤
│ 1. Execution Layer                    │
│ subprocess / sandbox / network        │
└───────────────────────────────────────┘
```

这里最容易产生的错误理解是：

```text
WebSocket
    ↓
直接连接 LLM
```

实际更接近：

```text
WebSocket / SSE
        ↓
Realtime Gateway
        ↓
Event Protocol
        ↓
Agent Runtime
        ↓
Agent Loop
        ↓
LLM / Tools / Memory / RAG / Sub-agent
```

---

# 21. Agent Realtime System 的完整生命周期

```text
① Connect
      │
      ▼
② Authenticate
      │
      ▼
③ Bind Session
      │
      ▼
④ User Message
      │
      ▼
⑤ Create Run
      │
      ▼
⑥ Agent Loop
      │
      ├── LLM streaming
      ├── Tool call
      ├── Tool result
      ├── RAG
      ├── Sub-agent
      └── State update
      │
      ▼
⑦ Event Store
      │
      ▼
⑧ SSE / WebSocket Push
      │
      ▼
⑨ Frontend Render
      │
      ├── cancel
      ├── pause
      ├── resume
      └── approval
      │
      ▼
⑩ Completed / Failed / Cancelled
```

---

# 22. 生产环境需要重点考虑什么？

可以归纳成 8 个关键问题：

| 问题 | 核心机制 |
|---|---|
| 实时输出 | SSE / WebSocket |
| 双向控制 | WebSocket / HTTP |
| 事件顺序 | `sequence` |
| 断线恢复 | Event ID + Replay |
| 数据可靠性 | Event Store |
| 重复执行 | Idempotency Key |
| 长连接稳定性 | Heartbeat / Keepalive |
| 高速输出 | Backpressure / Buffering |

再加上 Agent 特有的：

```text
Cancellation Propagation
Human-in-the-loop
Tool Call Validation
State Machine
Sub-agent lifecycle
```

---

# 23. 最终心智模型

把整套知识压缩成下面这张图：

```text
                         Agent Realtime System
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
         Commands             Streaming             Control
             │                    │                    │
         HTTP POST              SSE               WebSocket
             │                    │                    │
       start / input       token / progress     cancel / pause
       action / cancel     tool / lifecycle     resume / approval
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ Event Protocol  │
                         └────────┬────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
         ordering              replay               idempotency
        sequence / ID       Last-Event-ID          call_id / key
            │                     │                     │
            └─────────────────────┼─────────────────────┘
                                  │
                         ┌────────▼────────┐
                         │ Agent State     │
                         │ Machine         │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │ Agent Runtime   │
                         └────────┬────────┘
                                  │
                    ┌─────────────┼──────────────┐
                    │             │              │
                   LLM          Tools       Memory / RAG
                    │
                    ▼
                Agent Loop
```

---

# 24. 一句话总结

> **WebSocket 负责“实时双向连通”；SSE 负责“简单可靠的服务端流式输出”；HTTP 负责“离散命令”；Event Protocol 负责“消息怎么表达”；State Machine 负责“Agent 现在处于什么状态”；Event Store 负责“断线后能否恢复”；Agent Runtime 才负责“任务真正怎么执行”。**

因此研究 Agent 的 realtime architecture 时，真正应该关注的不是：

```text
WebSocket API 怎么调用？
```

而是：

```text
Transport
   ↓
Event Protocol
   ↓
State Machine
   ↓
Agent Loop
   ↓
Execution
   ↓
Persistence / Replay
   ↓
Recovery
```

这套模型也可以直接迁移到 **Pi、Claude Code、Codex、LangGraph、FastAPI Agent Backend** 等不同实现中；具体使用 SSE、WebSocket 还是其他传输方式，只是最上层 Transport Layer 的不同。
