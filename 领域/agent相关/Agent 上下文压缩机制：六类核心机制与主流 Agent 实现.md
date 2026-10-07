# Agent 上下文压缩机制：六类核心机制与主流 Agent 实现

> [!abstract] 核心结论
> Agent 的上下文压缩（Context Compression / Context Compaction）不是单一算法，而是一套由 Runtime、LLM、Provider 和 External Memory 共同参与的 Context Management 体系。
>
> 为避免概念混乱，本文将其统一归纳为 **6 类核心机制**：
>
> 1. **Truncation**：截断过大的单条内容
> 2. **Pruning**：删除已经没有价值的历史内容
> 3. **Boundary Selection**：决定从哪里开始压缩
> 4. **Compaction / Summarization**：使用 LLM 将旧历史压缩成可继续执行的状态
> 5. **Externalization**：把完整历史移出 Active Context
> 6. **Retrieval**：在需要时重新取回外部历史
>
> 此外还有一个重要的**实现层级变化**：
>
> **Native Compaction** 并不是第 7 种独立算法，而是 Compaction 的一种 Provider / Model-side 实现形式。
>
> 因此，现代 Agent 的完整上下文管理可以抽象为：
>
> ```text
> Agent Trajectory
>        │
>        ├── Truncation
>        ├── Pruning
>        │
>        ↓
>   Boundary Selection
>        │
>        ↓
>   Compaction / Summarization
>        │
>        ↓
>   Head + Summary + Tail
>        │
>        ↓
>   Active Context
>        │
>        ├──────────────→ Retrieval
>        │                    ↑
>        ↓                    │
>   Agent Continues      External History
> ```
>
> 最重要的认识是：
>
> > **Context Window 不等于 Agent Memory。**
>
> Context 是 Agent 当前的工作内存；完整历史可以存在 Context 外部，并通过 Retrieval 按需恢复。

---

# 1. 为什么 Agent 需要上下文压缩

Agent 和普通 Chat 的上下文增长方式不同。

普通 Chat：

```text
User
 ↓
Assistant
 ↓
User
 ↓
Assistant
```

而 Coding Agent 往往是：

```text
User
 ↓
LLM
 ↓
Tool Call
 ↓
Tool Result
 ↓
LLM
 ↓
Tool Call
 ↓
Tool Result
 ↓
...
```

例如一次代码修复任务：

```text
用户：
修复登录问题

LLM：
读取 auth.ts

Tool：
返回代码

LLM：
读取配置

Tool：
返回配置

LLM：
运行测试

Tool：
返回 3000 行日志

LLM：
修改代码

Tool：
返回 diff

LLM：
重新测试

Tool：
返回新的错误日志

...
```

随着 Agent 持续运行：

```text
Context Tokens
    ↑
    │
    │             /
    │           /
    │         /
    │       /
    │     /
    │___/________________
          Time
```

最终会遇到：

```text
Context Window 接近上限
```

这时 Runtime 必须决定：

```text
哪些内容可以删？
哪些内容必须保留？
哪里开始压缩？
如何保留语义？
详细历史放在哪里？
之后如何恢复？
```

这就是 Context Management。

---

# 2. 先统一概念：Agent Context 到底是什么

Agent Context 并不只是“聊天记录”。

更完整地看，它可能包含：

```text
Context
├── System Instructions
├── Developer Instructions
├── Tool Definitions
├── Skills / Rules
├── User Request
├── Conversation History
├── Tool Calls
├── Tool Results
├── Previous Compaction Summary
└── Recent Trajectory
```

因此：

> **Context Management 的目标不是简单地缩短聊天记录，而是维护一个能够继续驱动 Agent 执行任务的有效工作状态。**

---

# 3. 六类核心上下文压缩机制

这是本文统一采用的分类。

|中文说法|英文|含义|
|---|---|---|
|截断|**Truncation**|一条内容太长，直接截短|
|剪枝|**Pruning**|把已经没必要继续放在 Context 里的历史删掉|
|边界选择|**Boundary Selection**|决定从哪里开始进入压缩区|
|压缩/总结|**Compaction / Summarization**|用 LLM 把旧历史变成更短的语义状态|
|外置|**Externalization**|把完整历史移到 Context 外|
|检索|**Retrieval**|需要时再把外部历史取回来|

| # | 机制 | 核心问题 | 主要执行者 | 是否需要 LLM |
|---|---|---|---|---|
| 1 | Truncation | 一条结果太大怎么办 | Runtime | ❌ |
| 2 | Pruning | 哪些历史可以删除 | Runtime | ❌ |
| 3 | Boundary Selection | 从哪里开始压缩 | Runtime | ❌ |
| 4 | Compaction / Summarization | 怎样保留旧历史的语义 | LLM + Runtime | ✅ |
| 5 | Externalization | 完整历史放到哪里 | Runtime / Storage | ❌/✅ |
| 6 | Retrieval | 需要旧信息时如何找回来 | Runtime / Search / LLM | ❌/✅ |

> [!important]
> **Reconstruction 不是第 7 类机制。**
>
> `Head + Summary + Tail` 属于 Compaction 完成后的 **Context Reconstruction / Assembly**。
>
> 同样，**Native Compaction 也不作为第 7 类核心机制**；它描述的是 Compaction 发生在哪一层。

---

# 4. 机制一：Truncation

## 4.1 定义

Truncation 是最简单的长度控制：

> **单条 Tool Output 或其他内容过大时，直接截断。**

例如：

```text
read_file()
```

返回 10000 行代码。

Runtime 可以变成：

```text
前 500 行
...
后 200 行
[content truncated]
```

---

## 4.2 它解决的是什么问题

Truncation 解决的是：

```text
单条内容太大
```

而不是：

```text
整个历史太大
```

因此：

```text
Truncation
=
Local Size Control
```

---

## 4.3 Truncation 的特点

优点：

- 简单
- 确定性强
- 不消耗 LLM
- 可以在 Tool Result 进入 Context 前执行

缺点：

- 不理解语义
- 可能截掉真正重要的信息
- 不能代替真正的 Context Compaction

---

# 5. 机制二：Pruning

## 5.1 定义

Pruning 是：

> **将已经完成使命、价值较低、但占据大量 Token 的历史内容从 Active Context 中删除。**

例如：

```text
npm install
→ 返回 3000 行日志
```

安装已经完成后，3000 行日志通常没有必要一直留在 Context。

Runtime 可以将其裁掉。

---

## 5.2 Truncation 与 Pruning 的区别

### Truncation

```text
一条结果太大
↓
把这一条截短
```

### Pruning

```text
整个历史里这个结果已经没有价值
↓
把这个历史项移除
```

因此：

```text
Truncation = 控制单条内容大小

Pruning = 删除历史内容
```

---

## 5.3 Pruning 的重要意义

Pruning 往往是：

> **比 LLM Summary 更便宜的一层压缩。**

因为有很多 Tool Result 根本不值得被总结。

例如：

```text
旧 npm install 日志
旧 grep 输出
旧目录列表
旧测试成功日志
旧重复文件内容
```

这些内容可能直接删掉。

所以一个成熟的 Agent 不应该：

```text
所有历史
 ↓
LLM 总结
```

而是通常先：

```text
Pruning
 ↓
减少无价值内容
 ↓
再做 Compaction
```

---

# 6. 机制三：Boundary Selection

这是最容易和“总结”混淆的一个概念。

## 6.1 定义

Boundary Selection 是：

> **确定 Context 中“哪一段历史需要进入 Compaction”。**

可以表示为：

```text
HEAD | COMPRESS REGION | TAIL
```

例如：

```text
A
B
C
D
E
F
G
H
I
J
```

Runtime 决定：

```text
A B C | D E F G | H I J
      ↑         ↑
      压缩区域   最近历史
```

于是：

- `A B C`：HEAD
- `D E F G`：COMPRESS REGION
- `H I J`：TAIL

---

## 6.2 Boundary Selection 不负责“理解内容”

它主要负责回答：

> **“从哪里开始压缩？”**

而不是：

> “这里面哪一句最重要？”

后者是 LLM Semantic Summarization 的工作。

因此：

```text
Boundary Selection
=
位置与结构问题
```

而：

```text
Summarization
=
语义问题
```

---

## 6.3 Boundary Selection 通常考虑什么

Runtime 通常会考虑：

```text
Context Window
Token Budget
Recent Tail Budget
Message Boundary
Tool Call / Tool Result Pair
System / Developer Context
Previous Summary
```

特别重要的是：

```text
assistant → tool_call
tool → tool_result
```

不能随便从中间切断。

否则可能出现：

```text
tool_result
```

存在，但对应的：

```text
tool_call
```

已经被压掉。

因此：

> **Boundary Selection 本质上是一个 Context Slicing Algorithm。**

---

# 7. 机制四：Compaction / Summarization

这是通常意义上真正的“语义压缩”。

## 7.1 基本流程

```text
Old Agent Trajectory
        ↓
Compaction Prompt
        ↓
LLM
        ↓
Structured Summary
```

例如原始轨迹：

```text
读取 auth.ts
发现 JWT 过期判断错误
修改 verifyToken()
运行测试
发现 timezone bug
重新修改
测试通过
```

压缩后：

```text
Goal:
修复登录认证问题

Completed:
- 修复 JWT 过期判断
- 修复 timezone bug
- 测试通过

Relevant Files:
- src/auth.ts

Next Steps:
- 无
```

---

# 8. Compaction 的本质不是“总结聊天”

这是最重要的认知之一。

普通 Summary：

```text
“我们讨论了一个登录问题，然后修改了代码。”
```

对于 Agent 没什么用。

Agent 需要的是：

```text
Goal
Constraints
Current State
Completed Work
Pending Work
Key Decisions
Relevant Files
Errors
Next Steps
```

所以 Agent Compaction 的本质是：

> **将过去的 Agent Trajectory 转换为当前可继续执行的 Agent State。**

可以表示为：

```text
Trajectory
    ↓
State Extraction
    ↓
Compressed State
```

因此：

> **Compaction 更接近 State Serialization，而不是普通摘要。**

---

# 9. 为什么 Compaction Prompt 往往很长

你看到的 ZCode 等 Agent 的长压缩 Prompt，本质上是在指导模型：

> “生成一个足够完整的 Checkpoint，使新的 Agent 可以从这里继续工作。”

因此它往往要求：

```text
Goal
Constraints
Progress
Key Decisions
Files
Errors
Current State
Next Steps
Critical Context
```

典型形式：

```text
# Goal

# Constraints & Preferences

# Progress

## Completed

## In Progress

## Blocked

# Key Decisions

# Relevant Files

# Next Steps

# Critical Context
```

这不是简单的“摘要格式”。

它实际上是一种：

```text
Agent State Schema
```

---

# 10. Compaction 后怎么重建 Context

Compaction 完成后，并不是只留下 Summary。

典型结构是：

```text
HEAD
+
SUMMARY
+
TAIL
```

也就是：

```text
Stable Context
+
Compressed State
+
Recent Trajectory
```

例如：

```text
System Prompt
Developer Rules
Project Rules
Tool Definitions
        │
        ↓
Compaction Summary
        │
        ↓
Recent Messages
Recent Tool Calls
Recent Tool Results
```

然后：

```text
LLM
 ↓
继续 Agent Loop
```

> [!important]
> `Head + Summary + Tail` 是 **Reconstruction**。
>
> 它不是第七种压缩机制，而是 Compaction Pipeline 的重组步骤。

---

# 11. 为什么一定要保留 Tail

因为最近的行为往往和当前状态最相关。

例如：

```text
20 分钟以前：
已经读取项目结构

5 秒以前：
刚刚运行测试，测试失败

2 秒以前：
刚刚修改 auth.ts
```

对于下一次 LLM 推理：

```text
5 秒以前、2 秒以前
```

通常比：

```text
20 分钟以前
```

更加重要。

所以很多 Agent 采用：

```text
Long-term State
+
Short-term Recent Trajectory
```

即：

```text
Summary = 长期状态
Tail = 短期工作记忆
```

---

# 12. 机制五：Externalization

Compaction 是 Lossy 的。

如果：

```text
100000 Token History
 ↓
5000 Token Summary
```

那么很多细节必然消失。

因此另一种方法是：

> **不要把完整历史都放在 Model Context，而是把它存在外部。**

例如：

```text
Active Context
    ↓
只保留：
Goal
State
Summary
Recent Context
```

而完整数据：

```text
Session Store
├── transcript
├── tool_results
├── files
├── branch history
└── metadata
```

---

## 12.1 Externalization 的意义

它把：

```text
Context
```

和：

```text
Memory
```

分开。

以前很多系统默认：

```text
Agent Memory
=
Context Window
```

现在更准确的模型是：

```text
Agent Memory
=
Active Context
+
External History
```

---

# 13. 机制六：Retrieval

Externalization 后会出现一个新问题：

> 之前的细节被移出了 Context，什么时候需要它？

答案：

> **需要时再 Retrieval。**

例如：

```text
当前任务：
检查之前失败的测试

当前 Context：
只有 Summary

Agent：
“我要看第一次失败的完整日志”

      ↓

Retrieval

      ↓

历史 Session / Tool Result

      ↓

相关日志

      ↓

重新进入 Context
```

因此：

```text
Externalization
+
Retrieval
```

一起形成：

> **On-Demand Context**

---

# 14. 六类机制之间是什么关系

可以这样理解：

```text
             Agent Trajectory
                    │
          ┌─────────┴─────────┐
          │                   │
     Truncation            Pruning
          │                   │
          └─────────┬─────────┘
                    ↓
           Boundary Selection
                    ↓
            Compaction / LLM
                    ↓
          Head + Summary + Tail
                    ↓
             Active Context
                    │
          ┌─────────┴─────────┐
          │                   │
       Agent继续            Retrieval
                              ↑
                              │
                     External History
```

这六个机制解决的是不同问题。

---

# 15. Claude Code 的实现路线

Claude Code 可以概括为：

```text
Context 接近限制
        ↓
旧 Tool Output 清理
        ↓
Conversation Compaction
        ↓
生成压缩后的状态
        ↓
继续 Agent Loop
```

因此它的典型思路是：

```text
Tool Output Management
+
LLM Compaction
+
Persistent Project Context
+
Continued Execution
```

---

## 15.1 Claude Code 的核心特点

Claude Code 中一个重要现象是：

> **旧 Tool Output 会优先被清理，然后再进入更重的 Compaction。**

这说明它实际上存在多层 Context Management：

```text
低成本：
Tool Result Cleanup

↓

高成本：
LLM Compaction
```

而项目级规则、相关 Memory 等持久性上下文并不简单等同于普通对话历史。

---

# 16. Codex 的实现路线

Codex 的变化尤其值得关注。

## 16.1 早期思路

早期可以理解为：

```text
Codex Runtime
      ↓
调用模型
      ↓
生成 Summary
      ↓
Summary 重新进入 Context
```

也就是典型的：

```text
Client-side Compaction
```

---

## 16.2 现代 Codex

现代 Codex 更进一步：

```text
Codex Runtime
      ↓
Responses Compaction API
      ↓
Provider / Model-side Compaction
      ↓
Compaction Item
      ↓
继续执行
```

因此 Codex 的核心特点是：

> **Compaction 不再完全由 Agent Runtime 自己实现，而是越来越多地由 Provider / Responses API 承担。**

---

# 17. Native Compaction 到底是什么

这里需要把概念单独说清楚。

Native Compaction：

```text
不是一种新的“语义压缩算法”
```

而是：

```text
Compaction
发生在 Provider / Model Runtime 层
```

例如：

```text
普通做法：

Agent Runtime
 ↓
请求模型总结
 ↓
拿 Summary
 ↓
自己重建 Context
```

Native Compaction：

```text
Agent Runtime
 ↓
告诉 Provider：
“请压缩这个 Context”
 ↓
Provider
 ↓
完成 Context Compaction
 ↓
返回特殊 Compaction State
```

---

# 18. pi 的实现路线

pi 的设计非常清晰。

它主要有：

```text
Compaction
+
Branch Summarization
```

---

## 18.1 pi 的 Compaction

基本过程：

```text
Context 超过阈值
        ↓
从尾部向前扫描
        ↓
保留最近的一段
        ↓
确定旧历史
        ↓
LLM 总结
        ↓
CompactionEntry
        ↓
Summary + Recent Context
```

典型结构：

```text
HEAD | OLD HISTORY | RECENT TAIL
             ↓
           SUMMARY

最终：

HEAD | SUMMARY | RECENT TAIL
```

---

## 18.2 pi 的 Token Budget 思路

pi 会根据类似：

```text
contextWindow
reserveTokens
keepRecentTokens
```

来决定：

```text
什么时候压缩
+
最近保留多少内容
```

这实际上就是非常典型的：

> **Token-budget-driven Compaction**

---

## 18.3 pi 的重要设计

pi 会尽量避免在：

```text
Tool Call
Tool Result
```

之间切开。

此外它还支持：

```text
Branch Summarization
```

也就是当 Agent 在历史树上切换分支时，对另一条分支进行总结。

因此 pi 不仅解决：

```text
Context Too Large
```

也解决：

```text
Branch Too Large
```

---

# 19. OpenCode 的实现路线

OpenCode 也明确区分：

```text
Pruning
Compaction
```

典型流程：

```text
Old Tool Outputs
      ↓
Prune
      ↓
Serialize Conversation
      ↓
Boundary Selection
      ↓
LLM Summary
      ↓
Summary + Recent Context
```

其 Summary 更偏向：

```text
Objective

Important Details

Work State
  Completed
  Active
  Blocked

Next Move

Relevant Files
```

这其实反映出一个共同趋势：

> **Agent Summary 应该描述工作状态，而不是复述聊天。**

---

# 20. Hermes 的实现路线

Hermes 的特点是把 Context Management 显式抽象为：

```text
ContextEngine
```

也就是说：

```text
Agent
 ↓
ContextEngine
 ↓
should_compress()
compress()
token tracking
history management
```

---

## 20.1 Hermes 的典型 Pipeline

可以直接对应本文的六类机制：

```text
Phase 1
Old Tool Result Pruning

        ↓

Phase 2
Boundary Selection

        ↓

Phase 3
LLM Structured Summary

        ↓

Phase 4
Head + Summary + Tail
```

这四个 Phase 对应：

```text
Pruning
Boundary Selection
Compaction
Reconstruction
```

---

# 21. Hermes 的 Lossy / Lossless 思路

Hermes 一个比较有意思的地方，是它不仅考虑：

```text
Lossy Compaction
```

还探索：

```text
Lossless Context Management
```

---

## 21.1 Lossy

```text
完整历史
   ↓
LLM Summary
   ↓
只保留压缩状态
```

问题：

```text
细节可能永远丢失
```

---

## 21.2 Lossless

```text
完整历史
   ↓
Immutable / External Message Store
   ↓
Active Context 只保存必要部分
   ↓
需要时 Retrieval
```

于是：

```text
Context 小
+
History 仍然完整
```

这是一条和传统 Summary 完全不同的路线。

---

# 22. 各家实现的横向比较

| Agent | Truncation | Pruning | Boundary | LLM Compaction | Native Compaction | External / Retrieval | 主要特点 |
|---|---:|---:|---:|---:|---:|---:|---|
| Claude Code | ✅ | ✅ | ✅ | ✅ | 间接依赖底层能力，但 CLI 不应简单等同 API Native Compaction | ✅ | Tool Output 清理 + 自动 Compaction |
| Codex | ✅ | ✅ | ✅ | 历史上 ✅ | **✅ 强** | ✅ / Provider 管理 | Compaction 向 Provider 下沉 |
| pi | ✅ | ✅/有限 | **✅** | **✅** | ❌/非核心 | ✅ | 极简、Token Budget 明确 |
| OpenCode | ✅ | **✅** | ✅ | **✅** | ❌/非核心 | ✅ | Pruning 与 Compaction 分层 |
| Hermes | ✅ | **✅** | **✅** | **✅** | ✅ 可选 | **✅ 强** | ContextEngine + Lossless 路线 |

这里最值得观察的不是：

```text
谁“有没有 Summary”
```

而是：

```text
谁负责 Context Management？
```

---

# 23. 各家最大的架构差异

可以压缩成下面几句话。

## Claude Code

```text
Runtime 负责清理
+
LLM 负责 Compaction
```

核心目标：

```text
让长期 Coding Session 能持续运行
```

---

## Codex

```text
Runtime 负责触发
+
Provider / Responses API 负责 Compaction
```

核心变化：

```text
Context Management 下沉
```

---

## pi

```text
Runtime 明确管理 Token Budget
+
LLM 做摘要
```

核心特点：

```text
极简
+
可控
+
显式
```

---

## OpenCode

```text
Pruning
+
Boundary
+
Compaction
```

核心特点：

```text
把“删”和“总结”明确分开
```

---

## Hermes

```text
ContextEngine
=
Pruning
+
Boundary
+
Compaction
+
Reconstruction
```

并进一步探索：

```text
Lossy
+
Lossless
```

核心特点：

```text
把 Context Management 设计成独立子系统
```

---

# 24. Runtime 和 LLM 的职责分工

这是整个问题里非常值得抽象的一层。

## Runtime 擅长的事情

```text
Token Counting
Context Window Management
Pruning
Boundary Selection
Tool Pair Validation
Recent Tail Retention
Persistence
Externalization
Retrieval
Branch Management
Session Management
Cache-aware Layout
```

共同特征：

> **确定性、规则明确、适合程序执行。**

---

## LLM 擅长的事情

```text
理解历史
提取语义
判断重要性
提取当前状态
总结决策
识别约束
理解错误
生成压缩后的 State
```

共同特征：

> **需要语义理解。**

---

# 25. 因此最理想的职责划分

```text
               Agent Runtime
                     │
        ┌────────────┼────────────┐
        │            │            │
     Pruning      Boundary     Token Budget
        │        Selection         │
        └────────────┬────────────┘
                     ↓
                    LLM
                     │
        ┌────────────┼────────────┐
        │            │            │
     Summarize    Extract      Compress
      State       Decisions      History
                     │
                     ↓
               Runtime Again
                     │
        ┌────────────┼────────────┐
        │            │            │
   Reconstruct   Persist       Retrieve
                     │
                     ↓
                  Agent
```

可以把核心原则概括成：

> **Runtime 管结构，LLM 管语义。**

---

# 26. Context Compression 与 Prompt Cache

Context Compression 还会影响 Prompt / KV Cache。

因为很多模型的缓存依赖：

```text
Exact Prefix
```

如果 Context 发生修改：

```text
Old Context
A B C D E F G
```

压缩后：

```text
A B C SUMMARY G
```

那么从发生变化的位置开始：

```text
Prefix Cache
```

可能无法继续复用。

因此：

```text
Compaction
=
节省 Context Token
```

但同时：

```text
可能改变 Prefix
=
影响 Cache Hit
```

这意味着现代 Agent 进行 Compaction 时，实际上还需要考虑：

```text
Token Saving
+
Semantic Preservation
+
Cache Preservation
```

---

# 27. 一个成熟 Context System 不应该只有 Summary

如果一个 Agent 只有：

```text
History
 ↓
LLM Summary
 ↓
New Context
```

它实际上比较脆弱。

更完整的架构应该接近：

```text
                 Full History
                      │
          ┌───────────┴───────────┐
          │                       │
       Pruning               External Store
          │                       │
          ↓                       │
   Boundary Selection             │
          │                       │
          ↓                       │
       LLM Summary               │
          │                       │
          └──────────┬────────────┘
                     ↓
             Head + Summary + Tail
                     │
                     ↓
              Active Context
                     │
                     ↓
                  Agent
                     │
                     └──── Retrieval ───→ External Store
```

---

# 28. 从 Lossy 到 Lossless：Context Management 的演进

可以把技术路线粗略理解为：

```text
阶段 1
直接把所有历史塞进 Context

        ↓

阶段 2
Truncation

        ↓

阶段 3
Pruning

        ↓

阶段 4
LLM Compaction

        ↓

阶段 5
Summary + Tail

        ↓

阶段 6
External History + Retrieval

        ↓

阶段 7
Native / Provider-side Compaction

        ↓

阶段 8
Lossless Context Management
```

这并不意味着后面的方式会完全取代前面的方式。

恰恰相反，成熟系统通常会组合：

```text
Truncation
+
Pruning
+
Compaction
+
Externalization
+
Retrieval
```

---

# 29. 一个统一的数学视角

假设：

```text
H = Agent Full History
C = Active Context
S = Summary
T = Recent Tail
E = External History
```

传统方式：

```text
C = H
```

Context 足够大时没问题。

---

当历史太大：

```text
H = Old + Recent
```

进行 Compaction：

```text
Old → S
```

于是：

```text
C = S + Recent
```

如果还有稳定上下文：

```text
C = Stable + S + Recent
```

如果采用 Externalization：

```text
H = E + C
```

于是：

```text
C = Stable + S + Recent
E = Full Historical Detail
```

如果引入 Retrieval：

```text
C' = C + Retrieve(E, Query)
```

这就是：

```text
Active Context
+
On-Demand History
```

---

# 30. 最关键的区别：Compressed State 与 Full History

这是理解现代 Agent 的一个核心思想。

```text
Full History
=
发生过什么

Compressed State
=
现在是什么状态
```

例如：

```text
Full History：

读取 auth.ts
运行测试
发现错误
修改代码
重新运行
又发现错误
修复
再次测试
成功
```

Compressed State：

```text
Goal：
修复认证问题

Current State：
代码已经修复，测试通过

Files：
src/auth.ts

Remaining：
无
```

未来 Agent 真正需要的是：

```text
Current State
```

而不是：

```text
全部过程
```

---

# 31. Boundary Selection 为什么非常重要

Boundary Selection 位于：

```text
Pruning
```

和：

```text
Compaction
```

之间。

它实际上是整个系统的桥梁：

```text
Pruning
=
先减少垃圾

Boundary
=
决定压缩哪一段

Compaction
=
把这一段转化成状态
```

可以记成：

```text
Pruning
“哪些可以不要？”

Boundary
“从哪里开始压？”

Compaction
“剩下的历史怎么压？”

Reconstruction
“压完之后怎么重新拼？”
```

这四句话基本可以解释整个核心 Pipeline。

---

# 32. 推荐记忆方式：六种机制 + 两种实现维度

为了避免概念越来越乱，建议最终记成两层。

## 第一层：六种核心机制

```text
1. Truncation
2. Pruning
3. Boundary Selection
4. Compaction / Summarization
5. Externalization
6. Retrieval
```

---

## 第二层：Compaction 的实现位置

```text
Client / Agent Runtime
        │
        ├── 本地 LLM Summary
        │
        ↓
Provider / Model Runtime
        │
        └── Native Compaction
```

因此：

```text
Native Compaction
```

不是和：

```text
Pruning
```

平行的一种压缩算法。

它更准确地属于：

```text
“Compaction 在哪一层执行”
```

---

# 33. 最终统一架构

> [!important] Modern Agent Context Architecture

```text
                         Full Agent History
                                 │
                  ┌──────────────┴──────────────┐
                  │                             │
             Truncation                     Externalize
                  │                             │
                  ↓                             ↓
              Pruning                    External History
                  │                             │
                  └──────────────┬──────────────┘
                                 ↓
                         Boundary Selection
                                 ↓
                         Compress Region
                                 │
                                 ↓
                        Compaction / LLM
                                 │
                                 ↓
                          Structured State
                                 │
                                 ↓
                      Head + Summary + Tail
                                 │
                                 ↓
                          Active Context
                                 │
                                 ↓
                              Agent
                                 │
                                 │ needs old detail
                                 ↓
                             Retrieval
                                 │
                                 └──────→ External History
```

---

# 34. 最终心智模型

可以把整个系统类比成操作系统：

```text
Full History
    ↓
Disk / External Storage

Active Context
    ↓
RAM

LLM
    ↓
Semantic Processor
```

那么：

```text
Truncation
=
限制单条数据大小

Pruning
=
清理 RAM 中已经不需要的数据

Boundary Selection
=
决定哪些历史进入压缩区

Compaction
=
把大量历史压缩成“当前状态”

Externalization
=
把完整数据放到磁盘

Retrieval
=
需要时从磁盘取回

Native Compaction
=
让底层平台/Provider 直接承担 Context Management
```

这个模型非常适合用来理解现代 Agent。

---

# 35. 最终结论

> [!success] 一句话总结

**Agent 上下文压缩不是“让 LLM 总结聊天记录”，而是一个由 Runtime + LLM + External Memory + Provider 共同完成的 Context Management 系统。**

六种核心机制分别解决：

```text
Truncation
→ 单条内容太大

Pruning
→ 旧内容已经没价值

Boundary Selection
→ 决定从哪里开始压缩

Compaction
→ 把旧历史转换成可继续执行的状态

Externalization
→ 把完整历史移出 Active Context

Retrieval
→ 需要时重新取回历史
```

而：

```text
Native Compaction
```

描述的是：

```text
Compaction 从 Agent Runtime 下沉到 Provider / Model Runtime
```

最终，现代 Agent 更接近：

```text
             Stable Context
                    +
             Compressed State
                    +
               Recent Tail
                    +
           On-Demand History
```

而不是：

```text
Context = Entire Conversation
```

因此可以把现代 Agent 的 Context Management 总结成一句最重要的话：

> **Runtime 管理 Context 的结构和生命周期，LLM 负责理解和压缩语义，External Store 保存不适合长期驻留 Context 的完整历史，Retrieval 在需要时把历史重新带回来。**

这也是理解 **Claude Code、Codex、pi、OpenCode、Hermes** 等现代 Agent 上下文管理架构时最通用的统一视角。