---
title: Skill 加载模式与 KV Cache 影响
tags:
  - AI-Agent
  - Agent-Harness
  - Skills
  - Context-Engineering
  - KV-Cache
  - Prompt-Cache
  - Coding-Agent
---

# Skill 加载模式与 KV Cache 影响

## 1. 核心结论

现代 Agent 中，Skill 的设计主要涉及两个独立问题：

1. **Skill Metadata / Description 在什么时候、什么位置进入 Context**
2. **Skill Body（如 `SKILL.md`）什么时候真正加载**

常见的三种模式：

| 模式 | Skill Description | Skill Body | 新增 Skill 对 KV Cache 的典型影响 |
|---|---|---|---|
| **Static Registry** | 一开始进入 System Prompt / Stable Prefix | 可能一开始全部加载 | **影响最大** |
| **Dynamic Registry** | 运行时动态进入 Tool Description / Context | 调用后进入 Context | **较小，但取决于动态内容的位置** |
| **On-demand Loading** | 初始只暴露 metadata | 需要时才加载 | **正文影响最小，但 Description 仍可能影响 Prefix Cache** |

最重要的结论：

> **Progressive Disclosure 解决的是 Context 膨胀问题；Stable Prefix 设计解决的是 KV Cache 命中问题。**
>
> 二者有关，但不是同一个问题。

---

# 2. Skill 的基本组成

一个 Skill 可以抽象为：

```text
Skill
├── Metadata
│   ├── name
│   └── description
│
└── Body
    └── SKILL.md
```

其中：

- `name + description`：用于 **Skill Discovery**
- `SKILL.md body`：用于 **实际执行 Skill**

现代 Skill 系统越来越倾向：

```text
Discovery
    ↓
Metadata
    ↓
模型判断是否需要
    ↓
Load Skill
    ↓
Skill Body
```

而不是：

```text
启动
 ↓
所有 Skill 全部加载
```

---

# 3. Static Registry

## 3.1 基本结构

Static Registry 的核心特点：

> **Skill Registry 本身就是 System Prompt / Stable Prefix 的一部分。**

例如：

```text
System Prompt
│
├── Basic Instructions
│
├── Skill A description
├── Skill B description
├── Skill C description
│
└── Conversation
```

如果采用全量加载，则可能进一步变成：

```text
System Prompt
│
├── Skill A description
├── Skill A body
├── Skill B description
├── Skill B body
└── Skill C ...
```

---

## 3.2 新增 Skill

原来：

```text
System
├── A
├── B
└── C

Conversation
└── Long Agent Trajectory
```

新增 D：

```text
System
├── A
├── B
├── C
└── D   ← 新增

Conversation
└── Long Agent Trajectory
```

如果 Skill Registry 位于 Conversation 之前：

```text
System
A
B
C
D        ← 变化发生的位置
Conversation
Tool Calls
Tool Results
...
```

---

## 3.3 对 KV Cache 的影响

Prefix Cache 的核心是：

> **只有前缀保持一致，后续对应的 KV 才能直接复用。**

原来：

```text
System A B C Conversation
██████████████████████████
```

新增 D：

```text
System A B C D Conversation
██████████
^^^^^^^^^^
可以复用

D Conversation
xxxxxxxxxxxxxxxx
需要重新计算
```

因此：

- `System A B C`：可以继续复用
- `D` 后面的内容：Prefix 已改变，通常需要重新计算

### 关键结论

> **新增 Skill 虽然可能只有 100 tokens，但如果它插在一个 100K Token Agent Trajectory 的前面，那么后面的 100K Token Prefix Cache 都可能无法直接复用。**

因此 Static Registry 最大的问题不是 Skill description 本身占用了多少 token，而是：

> **它是一个可能发生变化的 Prefix。**

---

# 4. Dynamic Registry

## 4.1 基本思想

Dynamic Registry 不把 Skill Registry 永久固定在 System Prompt 中，而是在运行过程中动态生成。

一种抽象形式：

```text
System
│
├── Stable Instructions
└── Stable Tools

Conversation
│
└── Agent Trajectory
        │
        ↓
Dynamic Skill Registry
```

另一种常见形式：

```text
Tool Definition
└── skill
    └── available_skills
```

例如：

```text
available_skills

- pdf-analysis
  Analyze PDF documents.

- web-research
  Conduct web research.

- git-release
  Create releases.
```

模型需要 Skill 时：

```text
skill("pdf-analysis")
```

然后才加载：

```text
SKILL.md
```

---

## 4.2 新增 Skill

例如当前：

```text
System
Conversation
Trajectory
Dynamic Registry
├── A
├── B
└── C
```

新增 D：

```text
System
Conversation
Trajectory
Dynamic Registry
├── A
├── B
├── C
└── D
```

如果 Dynamic Registry 位于已经缓存的长上下文之后：

```text
System
████████████████

Conversation
████████████████

Trajectory
████████████████████
                ← 已缓存

Dynamic Registry
A
B
C
D
```

那么前面的长上下文仍然可以继续复用。

---

## 4.3 关键限制

不能简单理解成：

> **Dynamic Registry = 一定不会影响 Cache**

真正取决于：

> **Dynamic Registry 在最终请求中的位置。**

如果最终请求仍然是：

```text
System
Skill Registry
Conversation
Long Trajectory
```

那么 Skill Registry 发生变化后，后面的 Long Trajectory 依然可能失去 Prefix Cache。

因此：

```text
Dynamic
≠
Cache Safe
```

正确理解：

```text
Dynamic Registry
        ↓
把变化从 Stable Prefix 中隔离
        ↓
如果放置位置合理
        ↓
减少 Cache Invalidation
```

---

# 5. On-demand Loading

## 5.1 基本结构

On-demand Loading 是现代 Skill 系统非常重要的设计。

初始只加载：

```text
Skill metadata

A
description...

B
description...

C
description...
```

而完整正文：

```text
A SKILL.md
B SKILL.md
C SKILL.md
```

不会全部进入初始 Context。

模型需要某个 Skill 时：

```text
skill("B")
      ↓
load B
      ↓
SKILL.md
      ↓
进入当前 Context
```

---

## 5.2 最大价值：减少初始 Context

假设：

```text
Skill description = 100 tokens
Skill body = 5000 tokens
```

新增一个 Skill：

### Static 全量加载

```text
新增 ≈ 100 + 5000
```

### On-demand Loading

初始只增加：

```text
新增 ≈ 100
```

只有真正调用：

```text
skill("D")
```

时：

```text
+5000 tokens
```

因此 On-demand Loading 的主要价值是：

> **把大量低概率使用的 Skill 内容从初始 Context 中移出去。**

---

# 6. 一个非常重要的区别

不要把：

```text
Progressive Disclosure
```

和：

```text
KV Cache Optimization
```

混为一谈。

## 6.1 Progressive Disclosure 解决什么？

解决：

```text
Skill 太多
      ↓
Context 太大
      ↓
Token 消耗高
      ↓
模型注意力负担增加
```

通过：

```text
Metadata
   ↓
需要时再加载 Body
```

解决。

---

## 6.2 KV Cache Optimization 解决什么？

解决：

```text
Prompt Prefix 经常变化
      ↓
Cache Miss
      ↓
重新 Prefill
      ↓
计算成本增加
```

通过：

```text
Stable Prefix
      ↓
Dynamic Context 后置
```

解决。

---

# 7. 为什么“新增 Skill”特别值得关注

假设一个 Coding Agent 已经运行了很长时间：

```text
System                 5K
Skill Registry          2K
Conversation           20K
Tool Results           30K
Agent Trajectory       50K
---------------------------
Total                 107K
```

如果新增 Skill：

```text
Description = 100 tokens
```

而 Skill Registry 在最前面：

```text
System
Skills ← 变化
Conversation
Tool Results
Trajectory
```

虽然只增加：

```text
100 tokens
```

但由于 Prefix 发生变化：

```text
后面约 100K tokens
```

可能都无法继续直接使用原来的 Prefix Cache。

因此：

> **新增 token 数量与 Cache 损失不是线性关系。**

更准确地说：

```text
Cache Loss
≈
变化点之后的 Prefix 长度
```

---

# 8. Prefix Cache 的正确理解

不要把 KV Cache 理解成：

> “某个 Token 之前算过，所以以后可以直接拿来。”

更准确的是：

> **Prompt 的某个完整前缀之前已经计算过，因此这个 Prefix 对应的 KV 可以复用。**

例如：

```text
A B C D E F G
█████████████
```

修改为：

```text
A B C X D E F G
█████
xxxxx
```

可以复用：

```text
A B C
```

不能简单复用：

```text
D E F G
```

因为：

- Token position 发生变化
- Prefix 发生变化
- 后续 attention 计算依赖已经变化

所以：

> **Prefix Cache 是“前缀级复用”，而不是任意 Token 的独立缓存。**

---

# 9. 三种模式放在一起

假设：

```text
System = 5K
Skill Description = 100
Skill Body = 5K
Agent Trajectory = 100K
```

新增一个 Skill：

| 模式 | Description | Body | 初始新增 Token | 对后面 100K Cache |
|---|---|---|---:|---|
| **Static Registry** | System Prefix | 可能提前加载 | 100~5100 | **可能大量失效** |
| **Dynamic Registry** | Dynamic Context / Tool Description | 调用时加载 | ~100 | **取决于位置** |
| **On-demand** | Skill metadata | 真正使用时加载 | ~100 | **Body 不影响初始 Cache；Metadata 仍可能影响 Prefix** |

因此：

```text
Static
    ↓
变化容易污染 Stable Prefix

Dynamic
    ↓
可以隔离变化
    ↓
但取决于 Context Layout

On-demand
    ↓
减少 Body 对 Context 的污染
    ↓
但不能自动解决 Metadata 的 Cache 问题
```

---

# 10. 现代 Agent 更合理的组合方式

真正值得关注的不是“三选一”，而是：

> **Dynamic Registry + On-demand Loading + Stable Prefix**

典型结构：

```text
                    Agent Context
                         │
          ┌──────────────┴──────────────┐
          ↓                             ↓
     Stable Prefix                Dynamic Context
          │                             │
          ├── System                    ├── Skill Registry
          ├── Stable Tools              ├── Skill Body
          └── Project Rules             ├── Tool Results
                                        └── References
```

进一步：

```text
Stable Prefix
      │
      ├── System Prompt
      ├── Stable Tool Definitions
      └── Project Instructions
      │
      │  ← 尽量保持不变
      ↓
Conversation / Agent Loop
      │
      ↓
Skill Discovery
      │
      ↓
skill(...)
      │
      ↓
SKILL.md
      │
      ↓
Dynamic Context
```

目标是：

```text
Stable
████████████████████████████
          ↑
       尽量长期 Cache

Dynamic
xxxxxxxxxxxxxxxxxxxxxxxxxxx
          ↑
   可以频繁变化
```

---

# 11. Skill 与 MCP Tool 的相似性

从 Context Architecture 的角度看，Skill 和 MCP Tool 很相似。

## Skill

```text
Skill Metadata
      ↓
模型发现
      ↓
skill(...)
      ↓
SKILL.md
      ↓
Dynamic Context
```

## MCP Tool

```text
Tool Metadata
      ↓
模型发现
      ↓
tool_call(...)
      ↓
tool_result
      ↓
Dynamic Context
```

两者都可以抽象成：

```text
Capability Advertisement
          ↓
       Discovery
          ↓
      Invocation
          ↓
 Context Injection
```

主要区别：

```text
Skill
=
把一套“工作方法 / 知识 / 流程”按需加载进 Context

MCP Tool
=
把外部“可执行能力”暴露给模型
```

---

# 12. 主流 Agent 的典型趋势

目前主流 Coding Agent 的 Skill 设计都越来越接近：

```text
Metadata
    ↓
Discovery
    ↓
On-demand Loading / Execution
```

典型思路：

### Claude Code

```text
Skill description
      ↓
模型判断是否需要
      ↓
Skill body 按需加载
```

### OpenCode

```text
available_skills
      ↓
skill tool
      ↓
加载具体 Skill
```

### Codex

可以区分：

```text
AGENTS.md
    ↓
Persistent Project Instructions
```

和：

```text
Skills
    ↓
Progressive / On-demand Capability
```

因此现代 Agent 的趋势不是：

```text
所有 Skill 正文
      ↓
启动时全部注入
```

而是：

```text
“知道有哪些能力”
      ↓
“需要时才加载能力”
```

---

# 13. 最重要的概念

## 13.1 Stable Prefix

应该尽量保持稳定：

```text
System
+
Stable Tool Definitions
+
Project Instructions
```

原因：

```text
Stable Prefix
      ↓
高 Cache Hit Rate
      ↓
减少 Prefill 计算
```

---

## 13.2 Dynamic Context

允许频繁变化：

```text
Tool Result
Skill Body
References
Temporary Instructions
```

这些内容不应该轻易污染 Stable Prefix。

---

## 13.3 Progressive Disclosure

```text
Metadata
   ↓
发现
   ↓
按需加载 Body
```

作用：

```text
减少初始 Context
降低 Token 消耗
避免无关 Skill 干扰模型
```

---

## 13.4 Cache Invalidation Point

分析 Agent Cache 时非常重要：

```text
                  Cache Invalidation Point
                           ↓
System A B C D | E F G H I J K L
██████████████ | xxxxxxxxxxxxxxxxx
```

一般可以记：

```text
Cache Loss
≈
变化点之后的 Prefix 长度
```

因此：

> **不是“新增了多少 Token”决定 Cache 损失，而是“变化发生在哪里”。**

---

# 14. 一个统一的 Agent Context 模型

最终可以把一个现代 Agent 的 Context 理解为：

```text
┌───────────────────────────────────┐
│          Stable Prefix            │
│                                   │
│ System Prompt                     │
│ Stable Tool Definitions           │
│ Project Instructions              │
│                                   │
│ ████████████████████████████████  │
└───────────────────────────────────┘
                  │
                  ↓
┌───────────────────────────────────┐
│        Capability Discovery       │
│                                   │
│ Skill Metadata                    │
│ Tool Metadata                     │
│ MCP Metadata                      │
└───────────────────────────────────┘
                  │
                  ↓
           Agent Decision
                  │
          ┌───────┴────────┐
          ↓                ↓
      load Skill        tool call
          ↓                ↓
      SKILL.md         tool result
          └───────┬────────┘
                  ↓
┌───────────────────────────────────┐
│          Dynamic Context          │
│                                   │
│ Skill Body                        │
│ Tool Results                      │
│ References                        │
│ Temporary Information             │
└───────────────────────────────────┘
                  │
                  ↓
           Agent Trajectory
```

---

# 15. 最终记忆模型

整个问题可以浓缩成：

```text
Skill Metadata
      ↓
Capability Discovery
      ↓
On-demand Loading
      ↓
Dynamic Context
```

同时：

```text
Stable Prefix
      ↓
KV Cache
      ↓
尽量避免动态 Skill 信息插入 Stable Prefix
```

最终的理想状态：

> **让模型知道自己有什么能力，但不要把所有能力的完整内容提前塞进 Context；同时把稳定信息和动态信息分层，使 Skill 的动态变化尽量不会破坏长上下文的 Prefix Cache。**

---

# 16. 最关键的四个问题

以后分析任何 Agent 的 Skill 系统，可以固定问四个问题：

```text
① Skill description 在哪里？

② Skill body 在哪里？

③ 新增 Skill 时，哪个位置发生变化？

④ 这个变化点后面有多少 token？
```

其中第④个问题最直接决定 Cache 损失：

```text
Cache Loss
≈
变化点之后的 Prefix 长度
```

例如：

```text
变化点
 ↓
Skill description
 ↓
100K trajectory
```

即使新增 Skill：

```text
只有 100 tokens
```

也可能导致：

```text
约 100K tokens 重新 prefill
```

而如果变化发生在：

```text
100K trajectory
 ↓
Dynamic Skill Registry
```

那么：

```text
100K trajectory
████████████████████  ← 继续复用
```

---

# 17. 最终结论

Skill Architecture 与 KV Cache Architecture 实际上是同一个 **Agent Context Engineering** 问题的两个侧面：

```text
Skill Architecture
        │
        ├── Metadata
        ├── Discovery
        └── Body Loading
                 │
                 ↓
          Context Layout
                 │
        ┌────────┴────────┐
        ↓                 ↓
 Stable Prefix      Dynamic Context
        │                 │
        ↓                 ↓
    KV Cache       Skill / Tool Result
```

最优的现代 Agent 通常倾向：

```text
Stable Instructions
        +
轻量 Capability Metadata
        +
On-demand Skill Loading
        +
Dynamic Context
```

核心原则：

> **稳定的东西尽量稳定，动态的东西尽量后置；能力可以被发现，但能力正文应该按需进入 Context。**
