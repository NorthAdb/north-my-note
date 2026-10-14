# A2A 协议与 REST API：多 Agent 协作

> [!summary] 核心结论
> **REST API** 主要解决“客户端如何调用服务”；
> **A2A（Agent2Agent）** 解决“Agent 之间如何发现、调用、跟踪并持续协作”。
>
> A2A 的关键特征可以概括为：**Async-first + Stateful Task**。

---

## 1. A2A 是什么？

A2A（Agent2Agent）是面向 **Agent ↔ Agent** 协作的开放协议。

它的目标不是规定 Agent 内部如何思考，而是规定 Agent 的**边界与协作语义**，例如：

- 如何发现一个 Agent
- 如何知道它能做什么
- 如何向它提交任务
- 如何跟踪任务状态
- 如何进行多轮交互
- 如何接收流式更新
- 如何获取最终产物（Artifact）

因此可以把 A2A 理解成：

```text
Agent A
   │
   │ A2A
   ▼
Agent B
```

A 不需要知道 B 内部使用了什么模型、Prompt、Tool、RAG、Memory 或 Agent Loop。

> **A2A 的核心思想：把另一个 Agent 当成一个“可协作的黑盒工作者”。**

---

## 2. 为什么需要 A2A？

普通的多 Agent 系统也可以自己定义接口，例如：

```text
Agent A → HTTP API → Agent B
```

但如果每个 Agent 都自定义协议，就会出现大量点对点适配：

```text
A ↔ B
A ↔ C
A ↔ D
B ↔ C
...
```

大家还要分别约定：

```text
任务格式
状态格式
认证方式
流式返回
错误处理
多轮上下文
结果/文件如何传递
```

A2A 的意义就是把这些 **Agent 协作语义标准化**，从而促进跨框架、跨厂商的 Agent 互操作。

---

# 3. REST API 是什么？

REST（Representational State Transfer）是一种 Web API 的**架构风格**。

最常见的形式是：

```text
Client
  │
  │ HTTP Request
  ▼
Server
  │
  │ HTTP Response
  ▼
Client
```

例如：

```http
GET /weather?city=Tokyo
```

返回：

```json
{
  "city": "Tokyo",
  "temperature": 22
}
```

或者：

```http
POST /orders
Content-Type: application/json

{
  "product_id": 123,
  "quantity": 2
}
```

REST API 最典型的思路是：

> **请求一个资源或操作，得到一次响应。**

---

# 4. 为什么说 A2A 是 Async-first + Stateful Task？

## 4.1 Stateless 与 Stateful

### 传统 REST 常见的 Stateless 思路

每个请求应尽量独立，服务端不依赖“上一个请求是什么”才能理解当前请求。

例如：

```text
GET /users/123
GET /users/456
```

每个请求本身就是完整的。

### A2A 的 Stateful Task

Agent 任务往往不是一次请求立刻完成，而是一个持续运行的工作单元：

```text
提交任务
   ↓
Task 创建
   ↓
working
   ↓
持续执行
   ↓
产生中间结果
   ↓
completed / failed / canceled
```

因此 A2A 会围绕一个 **Task ID** 保存任务状态。

例如：

```text
Task: task-123
State: working
```

调用方可以之后继续查询：

```text
GetTask(task-123)
```

所以：

> **Stateful Task = 把“一个 Agent 正在执行的任务”作为一个有生命周期的对象。**

---

# 5. Async-first 是什么意思？

Async-first 不是说 A2A 不能使用 HTTP，也不是说 REST 不能做异步。

它真正强调的是：

> **长时间运行、异步完成的 Agent 任务，是协议设计的一等公民。**

例如：

```text
Client
  │
  │ “研究 500 篇论文并生成报告”
  ▼
Agent B
  │
  ├─ 创建 Task 001
  ├─ working
  ├─ 搜索资料
  ├─ 分析
  ├─ 生成报告
  └─ completed
```

调用方不需要一直阻塞等待。

可以：

```text
查询 Task 状态
或
订阅 Task 更新
或
等待异步通知
```

这非常适合 Agent 的长任务、流式执行以及 Human-in-the-loop 场景。

---

# 6. REST API 与 A2A 的本质区别

> 注意：**A2A 并不是 REST 的替代品。**
> A2A 甚至可以建立在 HTTP 等常见网络协议之上。
> 二者真正区别在于**抽象层和语义**。

| 对比项 | REST API | A2A |
|---|---|---|
| 核心目标 | 调用 Web 服务 | Agent 间协作 |
| 核心对象 | Resource / Endpoint | Agent / Task |
| 典型模式 | Request → Response | Message → Task → Update → Artifact |
| 长时间任务 | 可以自行实现 | 核心场景 |
| 状态管理 | 通常由业务自己定义 | Task 状态是核心模型 |
| Agent 能力发现 | 自己设计 | Agent Card |
| 多轮上下文 | 自己设计 | 有 context/task 等标准概念 |
| 流式更新 | 自己设计 | 标准支持 |
| 最终产物 | 自己设计 | Artifact |
| 跨 Agent 互操作 | 不保证 | 核心目标 |

因此更准确的说法是：

> **REST 是通用 Web 服务接口风格；A2A 是针对 Agent 协作场景进行标准化的一套协议语义。**

---

# 7. A2A 的核心对象

可以先记住下面这条主线：

```text
Agent Card
    ↓
Message
    ↓
Task
    ↓
Artifact
```

## 7.1 Agent Card

Agent Card 是 Agent 的“数字名片”或“能力说明书”。

它用于描述：

```text
我是谁
我能做什么
从哪里访问
支持什么协议接口
是否支持 Streaming
需要什么认证
有哪些 Skills
```

本质上解决的是：

> **Agent Discovery + Capability Discovery**

---

## 7.2 Message

Message 表示 Agent 之间的一次通信。

例如：

```text
“请研究 A2A 协议并总结核心概念”
```

Message 可以包含一个或多个 Part。

---

## 7.3 Part

Part 是消息中的内容单元，可以承载不同类型的数据，例如：

```text
Text
File
URL
Structured Data
```

因此 Agent 间传递的不一定只是纯文本。

---

## 7.4 Task

Task 是 A2A 最重要的对象之一。

它表示：

> **一个正在执行、可能持续较长时间、具有生命周期的工作单元。**

典型过程：

```text
submitted
   ↓
working
   ↓
completed
```

也可能：

```text
working
 ├─ failed
 ├─ canceled
 └─ input-required / auth-required
```

---

## 7.5 Artifact

Artifact 表示 Agent 在 Task 中产出的实际结果。

例如：

```text
Task
 ├─ research-report.md
 ├─ citation.json
 └─ summary.pdf
```

可以简单记：

```text
Message  = “我在和你交流什么”
Task     = “我正在为你做什么工作”
Artifact = “我最终给了你什么成果”
```

---

# 8. 一次典型的 A2A 协作流程

假设：

```text
Agent A = Orchestrator
Agent B = Research Agent
```

用户要求：

> “研究 Agent Memory，并生成一份报告。”

流程可以理解成：

```text
① Agent A 发现 Agent B
        │
        ▼
   获取 Agent Card
        │
        ▼
② 判断 B 是否具备所需能力
        │
        ▼
③ A 发送 Message
        │
        ▼
④ B 创建 Task
        │
        ▼
⑤ B 内部运行自己的 Agent Loop
        │
        ├─ LLM
        ├─ Tools
        ├─ MCP
        ├─ RAG
        └─ Memory
        │
        ▼
⑥ Task 持续更新
        │
        ├─ working
        ├─ artifact updates
        └─ ...
        │
        ▼
⑦ Task completed
        │
        ▼
⑧ A 获取 Artifact
        │
        ▼
⑨ A 继续自己的 Agent Loop
```

注意：**A2A 只规定 A 与 B 的协作边界，不规定 B 内部怎么实现。**

---

# 9. A2A、MCP、Function Calling、Harness 的关系

这是理解整个 Agent 技术栈最重要的一层：

```text
                     User
                       │
                       ▼
                ┌──────────────┐
                │ Agent / LLM  │
                └──────┬───────┘
                       │
                 Agent Harness
                       │
          ┌────────────┴────────────┐
          │                         │
      Tool Calling                A2A
          │                         │
          ▼                         ▼
        MCP                    Other Agent
          │
          ▼
      Tools / APIs
```

可以用一句话记：

```text
Function Calling：LLM 调用工具
MCP：Agent → Tool / Resource
A2A：Agent → Agent
Harness：负责把 LLM、Tools、Memory、Planning、Policy 等组织成一个可运行的 Agent
```

---

# 10. A2A 与多 Agent 架构的关系

> **多 Agent 是架构模式，A2A 是通信协议。**

不使用 A2A，也可以做多 Agent：

```text
Orchestrator
 ├─ Agent B
 ├─ Agent C
 └─ Agent D
```

只是这些 Agent 之间的接口需要自己定义。

使用 A2A 后，可以把 Agent 间协作标准化：

```text
           Multi-Agent System
                    │
             Orchestrator
                    │
          ┌─────────┼─────────┐
          │         │         │
         A2A       A2A       A2A
          │         │         │
          ▼         ▼         ▼
      Research   Coding    Reviewer
       Agent      Agent      Agent
```

因此：

> **A2A 不负责“决定该调用哪个 Agent”；Orchestrator / Harness 负责协调与决策，A2A 负责标准化通信。**

---

# 11. A2A 与 REST 最值得记住的心智模型

### REST

```text
“调用一个服务，拿到一个结果。”

Request
   ↓
Service
   ↓
Response
```

### A2A

```text
“委托另一个 Agent 完成一个工作，并持续跟踪这项工作。”

Message
   ↓
Task
   ↓
Task Updates
   ↓
Artifact
   ↓
Completed
```

因此最简洁的理解是：

> **REST 更像“调用服务”；A2A 更像“委托 Agent 做事”。**

---

# 12. 最终总结

```text
REST API
= 通用 Web 服务调用风格

A2A
= 面向 Agent 协作的标准协议

Async-first
= 长任务、异步执行是核心场景

Stateful Task
= 把 Agent 的工作表示成有生命周期的 Task

Agent Card
= Agent 的能力说明书

Message
= Agent 之间的通信

Task
= 一项具体工作

Artifact
= 工作产物

MCP
= Agent ↔ Tool

A2A
= Agent ↔ Agent

Harness / Orchestrator
= 决定怎么思考、怎么规划、什么时候调用谁
```

## 一句话记忆

> **MCP 让 Agent “会用工具”，A2A 让 Agent “找其他 Agent 协作”；REST 是通用的 Web 服务接口风格，而 A2A 在此之上标准化了 Agent 的发现、任务、状态、上下文和结果交换。**
