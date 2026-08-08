---
created: 2026-08-06
updated: 2026-08-06
tags: [Agent, LLM, LangChain, LangGraph, LlamaIndex, CrewAI, OpenAI Agents SDK, 框架选型, 面试, AI面试, 小林面试笔记]
source:
  - "https://mp.weixin.qq.com/s/-LRxcCjYTN5exg-egxvL5A"
  - "公众号：小林面试笔记"
---

# AI Agent 开发框架全解析：LangChain / LangGraph / LlamaIndex 怎么选

> **作者**：小林面试笔记（小林哥） | **来源**：微信公众号 | **发布**：2026-08-06

---

## 🎬 面试场景引入

> 👔 面试官：你了解过哪些 AI Agent 开发框架？
> 🙋 我：了解过 LangChain、LangGraph、LlamaIndex、CrewAI、AutoGen……反正都是封装大模型和工具调用的，用哪个都差不多。
> 👔 面试官：只会报名字不算了解。它们解决的问题、抽象层次和适用场景都不同，怎么会差不多？
> 👔 面试官（追问）：复杂的有状态流程还只想到 Chain？LangGraph 为什么会出现？LlamaIndex 又为什么长期强调数据和上下文增强？
> 👔 面试官（点破）：框架名称说得多，不等于理解得深。你应该围绕项目难点说明为什么选，而不是拿热度和名词数量代替技术判断。

**考点**：这道题看似在考你知道多少框架，其实面试官想听的是——**是否真正理解主流框架的定位，能不能根据业务问题完成技术选型。**

---

## 💡 简要回答（可直接背的版本）

我重点了解过 **LangChain、LangGraph 和 LlamaIndex**：

- **LangChain**：提供模型、Prompt、工具、Agent 和中间件等通用抽象，集成范围广，适合快速构建**工具调用、RAG、SQL 查询**等 AI 应用
- **LangGraph**：更偏底层的**状态与流程编排**。面对循环、分支、并行、断点恢复和人工审批等复杂流程，用图结构显式控制 Agent 执行路径。**现在 LangChain 的 Agent 也运行在 LangGraph 之上**——二者是上下层关系，不是互相替代
- **LlamaIndex**：优势集中在**数据接入、文档解析、索引和检索**，适合企业知识库、文档问答和复杂 RAG 等数据密集型应用

另外了解 **OpenAI Agents SDK**（适合以 OpenAI 模型为主的轻量 Agent）和 **CrewAI**（用角色和任务表达多 Agent 协作）。通用 Python Agent 岗位优先掌握前三者，再按公司技术栈补充。

---

## 📝 详细解析

### Agent 框架解决了什么？

不用框架自己实现一个 Agent，需要：接入模型、定义工具协议、实现 Agent 循环、把工具结果交回模型；还要处理**状态、重试、超时、流式输出、人工确认、运行追踪**。

- 完成一次演示不难，难的是：**系统执行十几步后能否恢复、出错时能否快速定位**
- 框架把这些重复工程抽象成可复用组件，但侧重不同：
  - LangChain → 通用组件与快速集成
  - LangGraph → 状态与流程控制
  - LlamaIndex → 数据与检索

![Agent框架解决什么：框架把重复工程抽象为可复用组件](https://gitee.com/cheng-jiaqing/images/raw/master/agent-fw-01.png)

> 回答时不要只比谁更强，要说清楚**在什么业务约束下，哪个框架更合适**。

### 为什么重点看 LangChain？

- 定位早已不只是「把多个 Prompt 串成 Chain」：提供模型、消息、Prompt、工具、结构化输出、中间件、Agent 等**通用抽象**，集成大量模型供应商、向量数据库、外部工具
- 最大价值：**集成范围广、开发速度快**（切换模型、接搜索/数据库/MCP 工具、快速实现 RAG Agent / SQL Agent / 客服助手）
- 局限：高层抽象更适合**常见 Agent 模式**；复杂循环、精细分支、长时间暂停、断点恢复时，需下沉 LangGraph

### LangGraph 和 LangChain 是什么关系？

- LangGraph 用 **State + Node + Edge** 表达工作流：State 保存共享状态，Node 执行模型/工具，Edge 决定下一步
- 重点解决：**循环、条件分支、并行执行、持久化、暂停恢复、人工介入**
- 例子：报销 Agent 必须先读单据 → 合规检查 → 金额超限暂停等主管审批 → 通过后调用付款工具——图结构比塞进一个 Agent 循环更清晰
- **关系**：LangChain 的 Agent 高层接口运行在 LangGraph 之上。可把 LangChain 理解为"常用组件和预制路线"，LangGraph 是"支撑路线的道路系统"
  - 简单 Agent 优先 LangChain；要精细控制时下沉 LangGraph

![LangGraph与LangChain是上下层关系](https://gitee.com/cheng-jiaqing/images/raw/master/agent-fw-02.png)

  - **不是两个互相替代的框架，而是上下层关系，经常组合使用**

### LlamaIndex 强在哪里？

- 别把它当成"另一个 LangChain"——它最有辨识度的是**在私有数据之上构建 AI 应用**
- 长期积累：数据连接、文档解析、切分、索引、检索、重排、Query Engine、结构化数据访问；也能把 RAG Pipeline 封装为 Agent 工具
- 企业知识库 Agent 真正难的不是工具调用循环，而是**整条数据链路**：PDF 表格解析、多源接入、文档切分建索引、召回过滤重排、权限隔离
- 尤其适合：企业知识库、文档 Agent、研究助手、复杂 RAG

![LlamaIndex数据链路：问题沿整条数据链路出现](https://gitee.com/cheng-jiaqing/images/raw/master/agent-fw-03.png)

- 它也有 Agent / Memory / 多 Agent Pattern / Workflow——只是选型入口不同：LangChain 偏通用 Agent 组装，LlamaIndex 偏数据密集型

### 三个框架怎么配合？（组合不是三选一）

```
企业知识库 Agent 示例：
LlamaIndex 处理文档、建索引、提供检索
      ↓ 把检索能力包装成 Tool
LangChain Agent 决定何时调用
      ↓ 外围有查询改写/答案校验/人工审核/失败恢复
LangGraph 负责这些步骤如何衔接
```

- 三者分别解决：**数据**（LlamaIndex）、**Agent 组装**（LangChain）、**流程控制**（LangGraph），不在同一层重复造轮子

![三框架配合：数据→Agent组装→流程控制](https://gitee.com/cheng-jiaqing/images/raw/master/agent-fw-04.png)

- 但**是否同时引入取决于复杂度**：简单工具调用不必引入 LlamaIndex；普通知识库问答不必上 LangGraph 工作流

### 其他框架要了解吗？

| 框架 | 定位 | 适合场景 |
|------|------|---------|
| **OpenAI Agents SDK** | Agent / Runner / Tools / Handoffs / Guardrails / Sessions / Tracing 的轻量 SDK | 以 OpenAI 模型为主：客服分流、语音助手、工具 Agent |
| **CrewAI** | 用角色、目标、任务、团队表达多 Agent 协作 + Flow 管理状态/条件/事件 | 研究、内容生产、多角色审核；角色越多调用成本和协作不确定性越高 |

![其他框架一览：OpenAI Agents SDK / CrewAI / AutoGen / Dify](https://gitee.com/cheng-jiaqing/images/raw/master/agent-fw-05.png)

| **AutoGen / Semantic Kernel / Microsoft Agent Framework** | 偏微软生态或存量项目 | 知道定位即可，通用 Python Agent 题不用展开 |
| **Dify** | 低代码 AI 应用开发平台 | 不和 Python Agent 框架放同一层面比较 |

### 到底该怎么选？（选型方法论）

1. **最外层先判断是否真的需要 Agent**：步骤固定、规则明确 → 普通函数/工作流更便宜更稳定。把本可写成 `if/else` 的流程交给模型 = 平白增加不确定性
2. **确认需要 Agent 后，找项目真正困难的那一层**：
   - 模型和工具接入最费力 → LangChain
   - 难点在私有数据、文档解析、检索质量 → LlamaIndex
   - 业务路径含复杂分支、循环、状态恢复 → LangGraph
3. **不能停在上线前**：Demo 能跑 ≠ 系统能上线。流程越长越要追问生产约束：
   - 中断后能否恢复？
   - 敏感动作是否需要审批？
   - 重复执行会不会产生副作用？
   - 不同用户的数据能否隔离？
   - 出错后是否留有完整轨迹？
   > 真正决定框架是否合适的，往往是**原型阶段看不见的生产约束**

### 框架对比总表

| 框架 | 核心定位 | 更适合的场景 | 掌握程度 |
|------|---------|-------------|---------|
| **LangChain** | 通用模型、工具和 Agent 抽象 | 工具型 Agent、RAG Agent、SQL Agent | 重点掌握 |
| **LangGraph** | 有状态的图式流程编排 | 循环分支、暂停恢复、人工审批 | 重点掌握 |
| **LlamaIndex** | 数据接入、索引和检索 | 企业知识库、文档 Agent、复杂 RAG | 重点掌握 |
| **OpenAI Agents SDK** | OpenAI 技术栈下的轻量 Agent SDK | 客服分流、语音助手、工具 Agent | 了解并按需深入 |
| **CrewAI** | 角色化多 Agent 协作 | 研究、内容生产、多角色审核 | 了解并按需深入 |

---

## 🎯 面试总结

1. **不要一口气罗列十几个框架**——说得越多越可能被追问，没有实际使用经验的框架容易露馅
2. **围绕 LangChain / LangGraph / LlamaIndex 三个展开**：
   - LangChain 偏通用组件与快速集成
   - LangGraph 偏有状态流程编排
   - LlamaIndex 偏数据接入与检索
   - 再补充对 OpenAI Agents SDK、CrewAI 等有所了解
3. **把框架特点落到真实场景**：简单 Agent 为什么选 LangChain？复杂审批流程为什么用 LangGraph？企业知识库为什么考虑 LlamaIndex？
   > **能讲清楚「什么场景为什么选」，比记住框架名称更有说服力。**

---

## 关联笔记
- [[2026-07-30-Claude聊天记录汇总]] — Java 生态选型实况

- [[2026-07-30-Agent高频面试题全解析-小林面试笔记]] — 同作者：Agent 基础八连问（LLM/MCP/Skills/A2A）
- [[2026-07-29-RAG核心知识全解析-小林面试笔记]] — 同作者：RAG 全链路
- [[2026-07-31-为什么顶级大模型都在卷MoE-新物种日记]] — 大模型架构（MoE）
- [[2026-07-07-小红书热议AI Agent开发用什么框架]] — Agent 框架选型讨论
- [[领域/AI Agent 智能体学习路线 2026]] — Agent 学习路线

## 来源、覆盖与局限

- **URL**：https://mp.weixin.qq.com/s/-LRxcCjYTN5exg-egxvL5A
- **作者**：小林面试笔记（小林哥）| **发布**：2026-08-06
- **获取方式**：Playwright 浏览器抓取公众号文章全文
- ⚠️ 文章含 3 张示意图（框架对比/三者配合等），文字部分已完整收录；图片以原文为准
