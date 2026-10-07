# Agent Skills 与 Tool 的渐进式披露、上下文注入与 KV Cache

> 本笔记整理本次对话中关于 **Skills 加载方式、Tool Schema 延迟加载、Claude Code / Codex 上下文注入、System Prompt 与 API role 的区别，以及它们对上下文与 KV Cache 的影响** 的讨论。

---

## 1. 核心结论

现代 Agent 的一个重要演进方向是：

> **不要在 Agent 启动时把所有能力的完整定义都塞进上下文，而是先提供能力目录（Catalog / Index），模型真正需要时再加载完整内容。**

这一思想最早可以直观看作 **Skills 的渐进式披露（Progressive Disclosure）**，现在也逐渐扩展到 **Tool Schema**：

```text
传统模式
System Prompt
+ 全部 Skill 正文
+ 全部 Tool Schema
+ Conversation

        ↓

渐进式披露
System / Developer Instructions
+ Skill Catalog
+ Tool Catalog
+ Conversation

        ↓ 按需检索 / 选择

Full Skill Body
Full Tool Schema

        ↓

Execution
```

因此可以把现代 Agent 的上下文理解为一个：

> **按需动态组装的工作空间（Dynamically Assembled Context）**

而不是一个启动时一次性准备完毕的巨大 System Prompt。

---

# 2. Tool Definition 与静态前缀

## 2.1 Tool Definition 是什么

一个 Tool Definition / Tool Schema 一般描述：

- 工具名称
- 工具功能描述
- 参数定义
- 参数类型
- 必填参数
- 其他调用约束

例如：

```json
{
  "name": "search_web",
  "description": "Search the web",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string"
      }
    },
    "required": ["query"]
  }
}
```

模型需要依赖这些定义才能正确生成 Tool Call。

---

## 2.2 为什么说 Tool Definition 是静态前缀的一部分

传统 API 调用可以抽象成：

```text
System Prompt
↓
Tools Definition
↓
Conversation
↓
User
```

System Prompt 和 Tools 通常位于请求前部，因此它们构成一个相对稳定的 **Prefix**。

如果连续请求之间前缀一致，模型服务商可能对这部分进行：

- Prefix Caching
- Prompt Caching
- KV Cache 复用

因此：

```text
System Prompt
+
Tool Definitions
====================
Static Prefix
```

---

# 3. 为什么全部 Tool Schema 会成为问题

当 Agent 的工具越来越多，例如：

```text
20 个 MCP Server
↓
每个 20 个工具
↓
约 400 个工具
```

如果每个 Tool Schema 都提前加载，可能得到：

```text
Tool Definitions
≈ 数万 Token
```

极端情况下甚至可能达到更高规模。

问题是：

> 用户当前任务可能只需要其中一个工具，但模型却必须携带全部工具定义。

例如只需要：

```text
github_search
```

却需要提前看到：

```text
github_search
github_create_issue
github_create_pr
github_merge
slack_send
slack_search
jira_create
jira_update
...
```

于是出现：

```text
Tool 越多
↓
静态前缀越大
↓
上下文占用越大
↓
初始计算 / 缓存空间压力越大
```

---

# 4. Skills 的渐进式披露

传统 Skill 加载方式可以是：

```text
启动时：

Skill A 正文
Skill B 正文
Skill C 正文
...
```

而 Progressive Disclosure 更倾向于：

```text
启动时：

Skill Catalog

pdf
  "处理 PDF"

slides
  "创建和编辑 PPT"

research
  "进行研究任务"
```

模型真正需要某个 Skill 时：

```text
用户任务
↓
模型判断需要 pdf
↓
load_skill("pdf")
↓
加载完整 SKILL.md
```

因此：

```text
Skill Catalog
=
能力索引

SKILL.md
=
能力详情
```

这与 RAG 的思想高度相似：

```text
Knowledge Base
↓
Retrieve
↓
Relevant Documents
↓
Context
```

Skill 则是：

```text
Skill Library
↓
Retrieve / Select
↓
Relevant Skill
↓
Context
```

---

# 5. Tool 也开始采用 Progressive Disclosure

Tool 层可以采用完全类似的结构：

```text
Tool Registry
↓
Tool Search / Retrieval
↓
Relevant Tool
↓
Full Tool Schema
↓
Tool Call
```

因此：

```text
Skills Progressive Disclosure
            │
            │ 思想扩展
            ↓
Tools Progressive Disclosure
```

它们都遵循：

> **先告诉模型“有哪些能力”，再告诉模型“这个能力完整怎么用”。**

---

# 6. Tool Progressive Disclosure 的三个层次

理解这套机制时，可以分为三个层次。

## 6.1 Tool Registry

回答：

> 我有哪些工具？

例如：

```text
github_search
github_issue
slack_send
db_query
```

---

## 6.2 Tool Retrieval

回答：

> 当前任务需要哪些工具？

例如：

```text
User:
帮我在 GitHub 找一下 XXX

↓

Tool Retrieval

↓

github_search
```

搜索实现可以是：

- BM25
- Embedding Search
- Hybrid Search
- LLM-based Retrieval

---

## 6.3 Tool Schema Loading

回答：

> 把这个工具完整的参数定义交给模型。

即：

```json
{
  "name": "github_search",
  "parameters": {
    "query": "...",
    "repo": "...",
    "type": "..."
  }
}
```

这一步就是：

- Deferred Loading
- Lazy Loading
- On-demand Schema Loading

---

# 7. OpenAI / Anthropic / Claude Code / Codex 的共同思想

不同系统实现细节不同，但核心可以统一成：

```text
Tool Catalog
↓
Tool Search
↓
Full Tool Schema
↓
Tool Call
```

例如：

### OpenAI

可以把：

- `tool_search`
- `defer_loading: true`

理解为：

> 工具可以先不加载完整 schema，模型需要时再搜索并加载。

### Anthropic

Tool Search / `tool_reference` 体现的是：

> 先提供工具引用或目录，再解析为实际工具定义。

### Claude Code + MCP

MCP 工具规模较大时，可以：

```text
MCP Server
↓
Tool Catalogue
↓
Lazy Loading
↓
Actual Tool Schema
```

### Codex

Tool 搜索可以使用类似 **BM25** 的信息检索方法：

```text
用户需求
"create github issue"
↓
BM25
↓
Tool Description Search
↓
github_create_issue
```

这里说明：

> Tool Search 本质上是“从工具库中检索能力”，不要求一定使用 LLM 检索。

---

# 8. “System Prompt”不要和 `role: "system"` 混为一谈

这是理解 Claude Code / Codex 上下文结构的关键。

## 8.1 逻辑层

架构上可以说：

```text
Stable Instructions
```

或者：

```text
System / Instruction Layer
```

表示 Agent 的稳定规则与行为约束。

---

## 8.2 API 层

真正 API 请求里的消息可能是：

```json
[
  {"role": "system", ...},
  {"role": "developer", ...},
  {"role": "user", ...},
  {"role": "assistant", ...},
  {"role": "tool", ...}
]
```

因此：

> **逻辑上的“系统级指令层”不等于 API 中一定存在 `role: "system"`。**

一个系统可以把稳定指令放到：

- System
- Developer
- Runtime Instruction

中的某一层。

所以：

```text
逻辑层：
Stable Instruction

≠

API 层：
role = "system"
```

---

# 9. Claude Code 的 Skill 上下文模型

Claude Code 可以抽象成：

```text
Skill Catalog
↓
运行时上下文
↓
模型选择 Skill
↓
完整 SKILL.md
↓
作为运行时消息追加
```

---

## 9.1 启动时

不会把所有 Skill 全文直接放进去，而是先让模型知道：

```text
Available Skills

pdf
  Work with PDF files.

slides
  Create and edit presentations.

research
  Conduct deep research.
```

也就是：

> **Skill Catalog = Capability Index**

---

## 9.2 调用 Skill 后

模型判断：

```text
需要 pdf
```

之后：

```text
load pdf
↓
完整 SKILL.md
↓
追加到当前上下文
```

这里的重要点是：

> **完整 Skill 正文可以作为 user message 注入。**

因此不能简单认为：

```json
{
  "role": "system",
  "content": "完整 SKILL.md"
}
```

才叫“Skill”。

Skill 的语义属于 Agent 能力层，而具体 API role 属于消息协议层。

---

# 10. Claude Code：新增 Skill 时怎么理解

假设已有：

```text
Skill A
Skill B
```

现在增加：

```text
Skill C
```

更适合把 Claude Code 理解为：

```text
已有 Runtime Context
↓
新的 Skill 被发现 / 加入
↓
Skill Catalog 更新
↓
后续 Context 构造时可见
```

也就是说：

> Claude Code 更接近“运行时维护 / 追加能力目录”。

但这并不意味着 Skill C 永远成为一个独立、固定的 API `system` 前缀。

更合理的抽象是：

```text
Skill Catalog = Runtime Capability State
Skill Body    = On-demand Runtime Content
```

---

# 11. Codex 的 Skill 上下文模型

Codex 的核心表述是：

> **每轮上下文构造阶段重新渲染 Skills Catalog。**

可以理解为：

```text
Agent Loop 第 1 轮
↓
重新生成 Skills Catalog

Agent Loop 第 2 轮
↓
重新生成 Skills Catalog

Agent Loop 第 3 轮
↓
重新生成 Skills Catalog
```

这个 Catalog 被放入：

```text
Developer Context
```

而显式选中的 Skill 正文则作为：

```text
带标记的 User Fragment
```

加入上下文。

因此 Codex 可以抽象为：

```text
Developer
├── Agent Instructions
└── Skills Catalog

User
├── Original User Request
└── Selected Skill Content
```

---

# 12. Claude Code 与 Codex 的核心区别

两者的目标其实高度一致：

> **Catalog 常驻、正文按需加载。**

真正不同的是：

| 维度 | Claude Code | Codex |
|---|---|---|
| Skill Catalog | 运行时提供 | 每轮重新渲染 |
| Catalog 所在逻辑位置 | Runtime Context | Developer Context |
| Skill 正文 | 调用时追加 | 显式选中后注入 |
| Skill 正文物理 role | 可表现为 User Message | User Fragment |
| 是否 Progressive Disclosure | 是 | 是 |

因此不要简单记成：

```text
Claude Code = 静态
Codex = 动态
```

更准确的是：

```text
Claude Code
= Runtime Capability Context

Codex
= Per-turn Context Reconstruction
```

---

# 13. “静态”与“动态”究竟应该怎么理解

这里至少有三个概念不能混淆。

## 13.1 静态内容

指：

> 多轮请求之间基本保持不变的内容。

例如：

```text
基础 Agent Instructions
```

---

## 13.2 动态内容

指：

> 随当前任务或 Agent Loop 变化的内容。

例如：

```text
Conversation
Tool Result
Loaded Skill
Tool Schema
```

---

## 13.3 每轮重新构造

指：

> Runtime 每轮都重新拼装上下文。

但：

```text
重新构造
≠
重新计算
```

即使 Codex 每轮重新生成 Skills Catalog，只要最终请求前缀一致，底层模型服务仍可能复用 Prefix/KV Cache。

这是一个非常重要的概念区分。

---

# 14. 新增 Skill 与 KV Cache

考虑：

```text
原始：

System / Developer
+ Skill A
+ Skill B
+ Conversation
```

新增：

```text
Skill C
```

如果新增内容改变了缓存前缀：

```text
Old Prefix
████████████████

New Prefix
██████████████████
```

那么改变点之后的缓存复用可能受到影响。

但如果新增 Skill 只是在后面的 Runtime Context / Conversation 中追加：

```text
Old Stable Prefix
████████████████████

New Runtime Content
+ Skill C
```

那么稳定前缀本身仍可能保持命中。

因此：

> **Skill 是否破坏 KV Cache，关键不是“Skill 是不是动态的”，而是“新增内容是否改变了可缓存的前缀”。**

---

# 15. “每轮重新渲染”不等于“每轮重新计算”

这是 Codex 场景中尤其需要注意的一点。

例如：

```text
第 N 轮
Developer Context
Skill A
Skill B
```

下一轮：

```text
第 N+1 轮
Developer Context
Skill A
Skill B
Skill C
```

虽然 Codex 重新构建了 Context，但：

```text
Skill A
Skill B
```

以及更前面的稳定指令部分如果保持一致，底层仍可能共享缓存。

因此：

```text
Context Construction
        ≠
Model Prefill Computation
```

更准确地说：

```text
Runtime 重新拼接请求
↓
服务商检查 Prefix
↓
相同前缀可复用
↓
不同部分继续计算
```

---

# 16. Skills、Tools、RAG 可以统一理解

三者其实可以用同一种架构抽象：

```text
              External Capability / Knowledge Store
                              │
               ┌──────────────┼──────────────┐
               ↓              ↓              ↓
           Knowledge        Skills         Tools
               │              │              │
           Retrieval       Retrieval      Retrieval
               │              │              │
               ↓              ↓              ↓
          Documents       Full Skill     Tool Schema
               │              │              │
               └──────────────┴──────────────┘
                              ↓
                           Context
                              ↓
                           Agent
```

因此：

### RAG

```text
Knowledge Base
↓
Retrieve
↓
Relevant Documents
↓
Context
```

### Skill

```text
Skill Library
↓
Select
↓
Relevant Skill
↓
Context
```

### Tool

```text
Tool Registry
↓
Search
↓
Relevant Tool
↓
Full Schema
↓
Context
```

共同思想就是：

> **能力仓库与模型当前上下文解耦。**

---

# 17. Capability Progressive Disclosure

因此可以把现代 Agent 的一种重要架构趋势概括成：

```text
Capability Registry
        ↓
Catalog / Index
        ↓
Retrieval / Selection
        ↓
Full Capability Definition
        ↓
Runtime Context
        ↓
Execution
```

这里：

- Skill 是一种能力
- Tool 是一种能力
- Knowledge 也是一种可检索资源

它们都可以采用：

> **Progressive Disclosure / On-demand Loading**

---

# 18. Agent Context 的新理解

传统 Agent：

```text
启动 Agent
↓
把所有东西准备好
↓
发送给模型
```

现代 Agent 越来越接近：

```text
Stable Instructions
        +
Capability Catalog
        +
Current Conversation
        ↓
       Retrieval
        ↓
Relevant Skill / Tool / Knowledge
        ↓
   Assemble Context
        ↓
        Model
        ↓
     Tool / Skill
        ↓
      Observe
        ↓
   Next Context Build
```

因此：

> **Context 不再只是“聊天记录”，而是 Agent Runtime 动态组装出来的工作空间。**

---

# 19. 最终统一模型

可以用下面这张图记忆整个体系：

```text
                         Agent Context
                              │
              ┌───────────────┴───────────────┐
              ↓                               ↓
       Stable Instruction               Runtime Context
              │                               │
        System / Developer                User / Tool
              │                               │
              ↓                               ↓
      Skill Catalog                    Loaded Skill Body
      Tool Catalog                     Tool Schema
                                         Tool Result
              │                               │
              └───────────────┬───────────────┘
                              ↓
                           Context
                              ↓
                            Agent
                              ↓
                        Tool / Skill Call
```

---

# 20. 最值得记住的几个结论

## 结论 1

> **Tool Definition 过去通常是静态前缀的一部分，但随着工具数量增长，开始向按需加载演进。**

## 结论 2

> **Skill Progressive Disclosure 与 Tool Progressive Disclosure 本质上是同一种思想：先索引，后加载。**

## 结论 3

> **`system / developer / user` 是 API 消息层概念，而 Stable Instruction / Skill / Tool 是 Agent 架构层概念，两者不能一一对应。**

## 结论 4

> **Claude Code 与 Codex 的核心思想一致，区别主要在 Catalog 和 Skill 正文在上下文中的具体落位，以及 Context 是如何被构造。**

## 结论 5

> **“每轮重新构造 Context”不等于“每轮重新计算全部 Token”。Prefix Cache / KV Cache 仍可能复用稳定前缀。**

## 结论 6

> **判断新增 Skill 对 KV Cache 的影响，关键看新增内容是否改变了可缓存的 prefix，而不是简单看它属于 Skill、User、Developer 还是 Runtime。**

## 结论 7

> **现代 Agent 越来越像“动态上下文组装系统”，而不是“巨大的静态 System Prompt”。**

---

# 21. 一句话心智模型

```text
不要把所有能力都提前塞给模型。

先告诉模型：
“我有哪些能力。”

模型需要时再告诉它：
“这个能力具体怎么用。”
```

对应到实现：

```text
Catalog
   ↓
Retrieve
   ↓
Load
   ↓
Context
   ↓
Execute
```

这就是：

> **Skills / Tools 的 Progressive Disclosure。**
