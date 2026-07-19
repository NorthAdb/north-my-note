---
created: 2026-07-07
updated: 2026-07-07
tags: [AI, Agent, 学习路线, 教程, 入门到精通]
source:
  - "https://github.com/microsoft/ai-agents-for-beginners"
  - "https://huggingface.co/learn/agents-course"
  - "https://www.anthropic.com/academy/building-effective-agents"
  - "https://academy.langchain.com/"
  - "https://roadmap.sh/ai-ml-engineer"
---

# AI Agent 智能体学习路线 2026

> 本文综合整理自 Datawhale Agent-Learning-Hub 及相关资源，结合 2026 年 AI Agent 领域最新动态进行了补充和完善。

---

## 路线总览

```
Stage 0: LLM 基础
    ↓
Stage 1: 认识 Agent（Agent Loop / Tool Calling / ReAct）
    ↓
Stage 2: 构建 Agent 循环（Prompt / Tool Call / Observation）
    ↓
Stage 3: 工具使用 / RAG / 记忆（Tool / RAG / Memory）
    ↓
Stage 4: Agent Harness（框架与工程化）
    ↓
Stage 5: 多 Agent 系统（Multi-Agent）
    ↓
Stage 6: MCP / A2A / Skills（协议与生态）
    ↓
Stage 7: 浏览器 / Computer-Use Agent
    ↓
Stage 8: 评估 / 可观测性 / 安全（Eval / Observability / Safety）
    ↓
Stage 9: 生产部署与运维
```

---

## Stage 0：LLM 基础（入门必备）

在开始 Agent 之前，需要打好 LLM 基础。

### 前置知识

| 技能                      | 说明           | 工具/资源                           |
| ----------------------- | ------------ | ------------------------------- |
| **Python / TypeScript** | Agent 开发主流语言 | Python 优先，TS 用于产品化              |
| **包管理**                 | 环境管理         | uv / pip / conda / npm          |
| **API 使用**              | 调用 LLM API   | OpenAI、Anthropic、Google AI SDK  |
| **Prompt Engineering**  | 提示词工程        | 结构化提示、Few-shot、Chain-of-Thought |
| **JSON / YAML**         | 结构化数据        | JSON Schema、Pydantic            |
| **Git**                 | 版本控制         | GitHub、Git Flow                 |

### 推荐学习资源

- **Microsoft AI Agents for Beginners**（第 0-1 课）— 环境搭建与 Agent 入门
- **Hugging Face Agents Course** — Agent 基础概念
- **吴恩达 ChatGPT Prompt Engineering 课程** — 提示词工程入门

---

## Stage 1：认识 Agent（Agent 初探）

### 理解 Agent 核心概念

```
                    ┌──────────────┐
                    │    LLM 模型   │
                    └──────┬───────┘
                           │
               ┌───────────┴───────────┐
               │     Agent Loop        │
               │  Think → Act → Observe │
               └───────────┬───────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────┴────┐      ┌────┴────┐      ┌────┴────┐
    │  Tools   │      │ Memory  │      │  Plan   │
    └─────────┘      └─────────┘      └─────────┘
```

### Agent ≠ LLM

| 概念 | 说明 |
|------|------|
| **LLM** | 语言模型，只生成文本 |
| **Agent** | LLM + 工具 + 循环 = 能自主行动的系统 |

### 关键公式（LangChain 定义）

> **Agent = Model + Harness**

- **Model**：LLM 本身
- **Harness**：围绕模型的一切（工具、上下文、记忆、护栏、重试逻辑）

### 学习目标

- [x] 理解 Agent Loop（Think → Act → Observe）
- [x] Tool Calling / Function Calling 概念
- [x] ReAct 模式（Reason + Act）
- [x] 能写一个简单的 Calculator Agent

### 推荐阅读

- **Anthropic: Building Effective Agents** — 官方最佳实践指南
- **OpenAI: A Practical Guide to Building Agents** — 实操指南
- **ReAct 论文**（Reason + Act 模式奠基之作）

---

## Stage 2：构建 Agent 循环

### 核心循环

```
System Prompt → User Input → LLM 思考 → Tool Call → Observation → LLM 推理 → Final Answer
```

### 关键技术

| 技术 | 说明 |
|------|------|
| **Tool Calling / Function Calling** | LLM 调用外部工具的机制 |
| **max_steps / timeout / retry** | 循环控制参数 |
| **Error Handling** | 工具调用失败的容错处理 |
| **ReAct** | Reason + Act 交替进行 |

### 实现方式

```python
# 最简单的 Agent 循环伪代码
def agent_loop(task):
    messages = [system_prompt, user_message(task)]
    for step in range(max_steps):
        response = llm(messages)
        if response.has_tool_call():
            result = execute_tool(response.tool_call)
            messages.append(result)
        else:
            return response.text
```

### 练习项目

- **Level 1**：Calculator Agent（简单工具调用）
- **Level 2**：Web Research Agent（搜索工具）
- **Level 3**：PDF QA Agent（RAG 基础）

### 推荐框架

- **LangChain**（Simple ReAct Agent from Scratch）
- 理解 Agent 框架的设计原理

---

## Stage 3：工具使用 / RAG / 记忆

### 工具（Tool Use）

Agent 通过工具与外部世界交互：

```
工具类型：
├── 搜索工具：search / brave / serp
├── 计算工具：calculator / code interpreter
├── 文件工具：read_file / write_file / database
├── API 工具：Slack / GitHub / 飞书
└── 浏览器工具：Playwright / browser-use
```

**最佳实践**：Anthropic 建议设计"工具契约"——清晰的 tool 名称、参数描述、返回值格式。

### RAG（检索增强生成）

| 组件 | 说明 |
|------|------|
| **Chunking** | chunk size / overlap / metadata |
| **Embedding** | 文本向量化 |
| **向量存储** | Chroma / Milvus / Pinecone |
| **检索** | Hybrid Search / Rerank |
| **生成** | answer with citations |

### 记忆（Memory）

| 记忆类型 | Claude Code 实现 | 说明 |
|----------|:----------------:|------|
| 短期记忆 | Context Window | 当前会话上下文 |
| 长期记忆 | Auto Memory / MEMORY.md | 跨会话持久化 |
| 程序记忆 | Skills 系统 | 可复用的能力模块 |
| 情景记忆 | 事件记录 | 过去的操作和结果 |

> 更多记忆系统详解 → [[领域/Claude Code记忆系统与Agent记忆架构.md]]

### 练习项目

- **Level 4**：Coding Review Agent
- **Level 5**：Browser Agent（含 DOM 解析）
- **PDF QA Agent**：RAG + Agent 结合
- **Agent + RAG** 检索增强问答

---

## Stage 4：Agent Harness（框架与工程化）

### Harness 工程概念

> **Harness Engineering** — Harrison Chase（LangChain CEO）提出：随着模型能力增强，工程重心从提示词转向围绕模型的"Harness"（工具、上下文、记忆、护栏、重试逻辑等）。

```
Harness 组成：
├── Tool Abstraction（工具抽象层）
├── Context Management（上下文管理）
├── Memory System（记忆系统）
├── Guardrails（护栏/安全）
├── Retry & Error Handling（重试与容错）
├── Observability（可观测性）
└── State Management（状态管理）
```

### 主流框架对比

| 框架 | 语言 | ⭐ | 定位 |
|------|:----:|:--:|------|
| **LangChain** | Python/TS | ~100k | Agent 应用框架 |
| **LangGraph** | Python/TS | ~15k | AGent 工作流图引擎 |
| **LlamaIndex** | Python | ~40k | 数据 + Agent 框架 |
| **Semantic Kernel** | C#/Py/Java | ~25k | 微软 AI 编排 |
| **AutoGen** | Python | ~60k | 微软多 Agent 框架 |
| **CrewAI** | Python | ~30k | 多 Agent 角色编排 |
| **Google ADK** | Python | ~5k | Google Agent 开发套件 |

### LangGraph 核心概念

```
StateGraph
├── State（状态定义）
├── Node（节点 = Agent 或功能模块）
├── Edge（边 = 条件路由）
├── Checkpoint（检查点保存）
├── Human-in-the-loop（人工介入）
├── Streaming（流式输出）
└── Persistence / Time Travel（持久化/时间旅行）
```

### 推荐资源

- **LangChain 官方 Agents 文档** — Agent = Model + Harness
- **LangGraph 实战** — 15 个以上实战案例
- **LlamaIndex Agent 文档** — 数据驱动的 Agent

---

## Stage 5：多 Agent 系统（Multi-Agent）

### 为什么需要多 Agent

> 一个通用 Agent 做所有事 → 多个专业 Agent 分工协作

### 架构模式

```
模式 1：Supervisor（管理者模式）
  Supervisor Agent
    ├── Researcher Agent（研究）
    ├── Coder Agent（编码）
    └── Writer Agent（写作）

模式 2：Swarm（群体协作）
  Agent A ↔ Agent B ↔ Agent C（消息传递）

模式 3：Debate（辩论模式）
  Agent A（正方） ↔ Agent B（反方） → Judge Agent（裁决）

模式 4：Pipeline（流水线模式）
  Planner → Executor → Reviewer → Reviser
```

### Supervisor 模式（LangGraph 实现）

LLM 作为路由器，根据任务需求将工作分配给不同的专业 Agent：

```
STAR → Supervisor → Researcher → Supervisor → Coder → Supervisor → END
```

多 Agent 系统的**关键设计点**：
- **共享状态**：消息历史在不同 Agent 间传递
- **条件路由**：Supervisor 决定下一个执行者
- **终止条件**：任务完成后结束循环
- **子图嵌套**：每个 Agent 可以是独立的 LangGraph 子图

### 练习项目

- **Level 6**：LangGraph Multi-Agent Writer
- **Research → Write → Review → Revise** 论文写作管线
- **Diff Reader + Risk Classifier + Test** 代码评审管线

---

## Stage 6：MCP / A2A / Skills（协议与生态）

### MCP（Model Context Protocol）

由 Anthropic 提出的**模型上下文协议**，标准化 AI 应用与外部工具/数据源的连接方式：

```
AI 应用（Client） ←→ MCP 协议 ←→ 数据源 / 工具（Server）
```

| 组件 | 说明 |
|------|------|
| **MCP Server** | 提供工具和资源 |
| **MCP Client** | 调用 Server 的工具 |
| **Tools** | 可执行的操作 |
| **Resources** | 静态数据源 |

**MCP + Agent 的意义**：Agent 通过 MCP 协议发现并调用任意工具，无需硬编码。

### A2A（Agent-to-Agent）

Google 提出的**Agent 间通信协议**，让不同框架构建的 Agent 可以相互协作：

- **handoff**：将任务转交给另一个 Agent
- **Agent 发现**：自动发现可用的 Agent 服务
- **跨平台**：不同技术栈的 Agent 互联

### Skills（Agent 技能包）

> Skills = 可复用的 Agent 能力模块

```
Skills 生态
├── 官方 Skills（Claude Code / Codex 内置）
├── 社区 Skills（GitHub 开源）
├── 自定义 Skills（项目特定）
└── Skill 管理（/plugin install / npx skills add）
```

Skills 的关键要素：
- **SKILL.md**：Skill 的指令文件
- **smoke test**：简单验证机制
- **工具绑定**：Skill 需要的 API 工具

### 学习资源

- **Model Context Protocol 官方文档**
- **A2A 协议文档**（Google）
- **Claude Code Skills 系统**

---

## Stage 7：浏览器 / Computer-Use Agent

### 浏览器 Agent

Agent 通过浏览器与网页交互：

```
Browser Agent
├── Playwright / browser-use（浏览器控制）
├── DOM 解析（提取页面结构）
├── URL 导航（页面跳转）
├── 元素交互（点击/输入/选择）
└── AI 决策（判断下一步操作）
```

### Computer-Use Agent

Agent 直接操作系统界面（Anthropic 的 Computer Use 能力）：

- 屏幕截图理解
- 鼠标键盘操作
- 跨应用操作
- 适合 UI 测试和桌面自动化

### 练习项目

- **Level 7**：MCP Tool Agent
- **Browser Agent**：自动填写表单
- **Computer-Use Agent**：桌面自动化

---

## Stage 8：评估 / 可观测性 / 安全

### 评估（Evaluation）

| 评估维度 | 说明 |
|---------|------|
| **AgentBench** | LLM-as-Agent 性能评测 |
| **任务成功率** | 是否完成目标 |
| **幻觉率** | 生成虚假信息的频率 |
| **工具调用准确率** | 是否正确选择/调用工具 |

### 可观测性（Observability）

```
Agent 可观测性
├── Trace（链路追踪）：prompt → tool call → observation → final answer
├── Log（日志）：记录每次交互
├── Replay（回放）：重现 Agent 决策过程
├── Cost（成本追踪）：token 消耗统计
└── LangSmith / LangFuse / Helicone（监控平台）
```

### 安全（Safety）

| 风险 | 说明 |
|------|------|
| **Prompt Injection** | 提示注入攻击 |
| **Tool Abuse** | 工具滥用 |
| **Data Exfiltration** | 数据泄露 |
| **权限控制** | Agent 操作范围限制 |

### 推荐资源

- **OpenAI: Guardrails and deployment guidance**
- **Anthropic: Simple, composable agent patterns**
- **AgentBench 论文**

---

## Stage 9：生产部署与运维

### 部署方案

| 方式 | 说明 |
|------|------|
| **CLI Agent** | 命令行工具 |
| **Web App** | Web 应用 |
| **Slack / 飞书 Bot** | IM 集成 |
| **GitHub Action / CI Agent** | CI/CD 集成 |
| **API 服务** | RESTful API 部署 |

### 运维要点

```yaml
生产环境关键配置：
  Docker:       容器化部署
  CI/CD:        自动化测试与部署
  Secrets:      环境变量 / .env 管理
  Smoke Test:   基本功能验证
  日志系统:     追踪、监控、告警
  Token 预算:   成本控制
```

### 成本优化

- **Prompt Caching**：减少重复计算
- **Token 压缩**：RTK / Caveman 等工具
- **模型选择**：Flash vs Pro 按任务分配
- **缓存机制**：降低 API 调用频率

### 练习项目

- **Level 8**：Personal Agent / Coding Agent
- **完整项目**：从开发到部署的全流程
- **CI/CD Agent**：自动化代码评审/测试

---

## 项目实战阶梯

从易到难，逐步构建：

```
Project Ladder
├── Level 1: Calculator Agent（入门）
├── Level 2: Web Research Agent（工具调用）
├── Level 3: PDF QA Agent（RAG + Agent）
├── Level 4: Coding Review Agent（代码审查）
├── Level 5: Browser Agent（浏览器自动化）
├── Level 6: LangGraph Multi-Agent Writer（多Agent协作）
├── Level 7: MCP Tool Agent（协议集成）
└── Level 8: Personal Agent / Coding Agent（完整产品）

建议数量：20+ 个小练习，逐步积累经验
```

---

## 推荐资源汇总

### 免费课程

| 课程                           | 提供方          | 链接                                                                                           |
| ---------------------------- | ------------ | -------------------------------------------------------------------------------------------- |
| AI Agents for Beginners      | Microsoft    | [GitHub](https://github.com/microsoft/ai-agents-for-beginners)                               |
| Agents Course                | Hugging Face | [huggingface.co/learn/agents-course](https://huggingface.co/learn/agents-course)             |
| Building Effective Agents    | Anthropic    | [anthropic.com/academy](https://www.anthropic.com/academy/building-effective-agents)         |
| AI Agents in LangGraph       | LangChain    | [academy.langchain.com](https://academy.langchain.com/)                                      |
| Agentic AI Roadmap 2026      | DigitalOcean | [digitalocean.com](https://www.digitalocean.com/resources/articles/ai-agent-mastery-roadmap) |
| Datawhale Agent-Learning-Hub | Datawhale    | [GitHub](https://github.com/datawhalechina/Agent-Learning-Hub)                               |

### 必读论文与文档

- **ReAct Paper** — Agent 循环奠基之作
- **AgentBench Paper** — Agent 评测标准
- **Anthropic Building Effective Agents** — 设计模式最佳实践
- **OpenAI A Practical Guide to Building Agents** — 实操指南
- **MCP 协议文档** — Model Context Protocol
- **A2A 协议文档** — Agent-to-Agent 通信

### 框架与工具

| 类型 | 工具 |
|------|------|
| Agent 框架 | LangChain / LangGraph / LlamaIndex / AutoGen / CrewAI |
| 开发工具 | Claude Code / Codex / Cursor / OpenClaw |
| MCP 生态 | codebase-memory-mcp / codewiki-mcp / 各类 MCP Server |
| Agent Skills | last30days-skill / agent-skills 合集 |
| 可观测性 | LangSmith / LangFuse / Helicone / PeekAI |
| 向量数据库 | Chroma / Milvus / Qdrant / Pinecone |

---

## 2026 年趋势与建议

### 学习重点

1. **打好基础**：先理解 Agent Loop 和 ReAct 模式，不要急着上框架
2. **框架从 LangChain/LangGraph 入手**：生态最成熟，学习资源最丰富
3. **重视 Harness Engineering**：模型会越来越强，但 Harness 永远是工程核心
4. **跟上 MCP 和 Skills 生态**：这是 Agent 能力扩展的关键标准
5. **实践为主**：20+ 个练习项目，从 Calculator Agent 开始逐步升级
6. **关注多 Agent 协作**：单一 Agent 能力有限，多 Agent 是生产级方案

### 推荐学习路径

```
阶段     学习内容                        预计时间
────────────────────────────────────────────────
Stage 0  LLM + Python + API 基础          1-2 周
Stage 1-2 认识 Agent + 构建 Agent 循环    2-3 周
Stage 3   Tool / RAG / Memory             2-3 周
Stage 4   LangChain / LangGraph 框架      3-4 周
Stage 5   多 Agent 系统                    3-4 周
Stage 6   MCP / A2A / Skills              2-3 周
Stage 7-9 高级应用 + 部署                  4-6 周
────────────────────────────────────────────────
总计：约 4-6 个月
```

> 💡 **持续学习**：AI Agent 领域迭代极快，建议关注 GitHub Trending、Hacker News、LangChain 博客、Anthropic 工程博客，保持知识更新。

---

## 🔗 关联笔记

- [[领域/Claude Code记忆系统与Agent记忆架构.md]] — Agent 记忆系统详解
- [[领域/Python vs TypeScript AI时代编程语言之争.md]] — 开发语言选择参考
- [[领域/CodeWiki Google的AI驱动代码文档平台.md]] — AI 文档工具
- [[Clippings/GitHub/2026-07-04-我的GitHub加星项目.md]] — Agent 相关项目
- [[Clippings/Bilibili/2026-07-04-我的B站AI编程收藏清单.md]] — 视频学习资源
