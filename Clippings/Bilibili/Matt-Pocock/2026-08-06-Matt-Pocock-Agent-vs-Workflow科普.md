---
created: 2026-08-06
updated: 2026-08-06
tags: [Matt Pocock, Agent, Workflow, Anthropic, Building Effective Agents, AgentKit, 工具调用循环, 科普, B站笔记, OpenCLI字幕]
source:
  - "https://www.bilibili.com/video/BV1N9M26wEso/"
  - "原作者：Matt Pocock | 搬运翻译：ChHsich"
---

# Matt Pocock 科普：大多数人根本没搞懂 Agent 到底是什么

> **原作者**：Matt Pocock（Total TypeScript / AI Hero） | **时长** 16:19 | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI，206 段逐句）
> **证据等级**：`bilibili-ai-subtitles`
> **主题**：Agent 和 Workflow 的本质区别——谁来决定程序何时停止。

---

## 一句话总论

> **Agent 是一个由 LLM 决定何时停止的循环；Workflow 是一组预设代码路径的确定性流程。** 区分它们的唯一标准：**是谁决定程序停止——LLM 自己，还是你写死的代码？**

---

## 起因：OpenAI AgentKit 引发的争议

- OpenAI 推出 **AgentKit**：宣称是"构建 agent 的工具"，但示例全是**预设步骤的确定性流程**（反滥用防护 → 过滤恶意输入 → 路由到三个"独立 agent"）
- Matt 观点：**这在我看来不是 agent 构建工具，而是 workflow**
- 争议根源：AI 工程社区**并没有真正达成过 agent vs workflow 的统一定义**，OpenAI 采用的定义和社区不同

> 最佳参考：**Anthropic《Building Effective Agents》**（2024年12月发布，已成行业标准），它首次清晰定义了 agent 和 workflow。

---

## Anthropic 定义（核心）

### Agent = 一个循环，由 LLM 决定停止

```
LLM ──调用工具──→ 执行代码 ──结果返回──→ LLM（获得新信息）
  ↑                                                  │
  └──────────────── 继续循环 / 发停止标记 ◄───────────┘
```

- 本质是**连续多次调用大语言模型**；每次都带着**新信息**（否则用相同信息调用毫无意义）
- 新信息通过**工具调用**获得：LLM 说"执行这段代码"→ 工具执行 → 结果返回 LLM
- **例子**（writeFile 工具）：用户让 agent 创建 `.gitignore` → 助手返回"调用此工具 + 内容 + 路径" → 执行 → 结果回传 → 循环
- **停止**：agent 可以选择继续调用或停止；停止时发出特殊标记，前端捕获后不再调用模型
- 这就是编程 agent（Claude Code / Cursor 等）的核心：**每次模型获得更多信息，优雅地驱动代码生成**

### Workflow = 预设代码路径

```
模型调用1 → 结果 → 模型调用2 → 结果 → 模型调用3 → ……
              ↑
       确定性逻辑分支（若返回A走这、返回B走那）——但所有路径预先确定
```

- 没有循环，只是**按序执行的预定步骤**；可能包含 `if/else` 式确定性分支
- **可优化性**：因为路径早已明确，可以用各种方式优化——例如**并行工作流**：把文本拆两半，让模型并行分别总结，再合并
- 这使 workflow 极其强大

> **单次 LLM 调用**既不算 agent 也不算 workflow——那只是**普通 API 调用**，不需要额外定义。

---

## 各擅长什么

|        | Agent                                           | Workflow                        |
| ------ | ----------------------------------------------- | ------------------------------- |
| **适用** | 解决方案路径**不明确**、需广泛适应任务                           | 路径**明确**、需重复执行/优化               |
| **例子** | Claude Code / Cursor 代码助手（不知道会遇到什么代码库/错误，需实时调整） | 重复一千次的固定流程                      |
| **能力** | **即兴发挥**                                        | **被优化**                         |
| **比喻** | 🎷 **爵士乐**（一切即兴、充满感觉）                           | 🎼 **古典音乐**（花大量时间优化前期设置，确保输出最佳） |

> 💡 Matt 的荒岛二选一：**如果只能留一个，他选 Workflow**（"因为我天生就是个反潮流的人"——个人偏好，非技术结论）

---

## 现实：一个连续体（不是非此即彼）

大多数真实系统处于 agent 与 workflow 之间的**渐变区间**：

| 形态 | 说明 |
|---|---|
| **Agent + max steps 计数器** | 纯 LLM 控制的 agent 会无限运行，所以加确定性停止机制——SDK 里常见的 max steps 参数 |
| **Agent 里嵌 Workflow** | 许多 agent 能从工具中调用 workflow——获得 agent 的通用性 + 可优化的工具 |
| **Workflow 里嵌 Agent** | 生成文本后多次评估、持续优化输出（如反思循环）——关键看 **LLM 是否具备提前终止的能力**：有 = 偏 agent |

> **定义的价值在于用模式思考**：帮助沟通"你在构建什么"以及权衡取舍。术语本身有歧义，但 Anthropic 的两步循环定义值得成为行业标准。

---

## 关键洞察

1. **判定标准一句话**：谁决定停止？LLM（agent）还是代码（workflow）？
2. **工具调用 = agent 循环的心脏**：没有新信息注入的循环没有意义，工具是获得新信息的通道
3. **agent 不是"更高级"，workflow 不是"更低级"**——它们适合不同的场景（即兴 vs 优化）
4. **现实中大多是混合体**：带 max steps 的 agent、嵌 workflow 的 agent、嵌 agent 的 workflow

---

## 证据与原文位置

- 字幕来源：agent-reach / OpenCLI `opencli bilibili subtitle BV1N9M26wEso`（206 段逐句带时间轴）

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1N9M26wEso/
- **BVID**：BV1N9M26wEso | **UP 主**：ChHsich（搬运翻译） | **原作者**：Matt Pocock
- **互动**：1340 播放 / 138 收藏 / 33 赞
- **字幕**：agent-reach OpenCLI 提取 B站 AI 字幕（206 段，全程覆盖）
- ⚠️ B站 AI 字幕为翻译字幕，个别术语可能因音译有偏差（AgentKit/Anthropic/max steps 等已结合语境校正）
- ⚠️ 文中提及的 Anthropic 文章为《[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)》（2024-12），可配合原文阅读

## 关联笔记
- [[2026-07-30-Agent高频面试题全解析-小林面试笔记]] — 同主题面试题（第 2 题 Agent vs Workflow）
- [[Matt-Pocock方法论精华-完整工作流与技能体系]] — Matt 方法论汇总

- [[领域/AI Agent 智能体学习路线 2026]] — Agent 学习路线
- [[2026-07-30-Agent架构Skill和Tool本质区别-动画详解]] — Agent 架构基础概念
- [[2026-08-06-AI-Agent开发框架选型-LangChain-LangGraph-LlamaIndex-小林面试笔记]] — 框架选型（小林面试笔记）
- [[2026-07-31-为什么顶级大模型都在卷MoE-新物种日记]] — 大模型架构
