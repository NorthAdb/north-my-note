# HelloAgent 项目学习指南

## 项目全景

"HelloAgent" 是一个项目家族，核心是 **Datawhale 社区的《Hello-Agents：从零开始构建 AI Agent》教程**（GitHub 60,400+ stars），以及由此衍生出的多语言框架实现。

---

## 一、项目组成

### 1. 核心教程：datawhalechina/hello-agents
- **定位**：免费开源的中文 AI Agent 教程，"从零手写 AI Agent"
- **Star 数**：60,400+，7,400+ forks，76+ contributors
- **许可证**：CC BY-NC-SA 4.0（非商业用途）
- **章节结构**：16 章，分 5 个部分 + 社区扩展

### 2. Python 框架：jjyaoao/HelloAgents
- **定位**：教程配套的生产级多 Agent 框架，边学边构建
- **Star 数**：~2,100
- **安装**：`pip install hello-agents`（Python 3.10+）
- **架构**：
  - `core/` — LLM 适配器（OpenAI / Anthropic / Gemini）、Agent 基类、会话管理、生命周期、流式输出
  - `agents/` — 多种 Agent 实现（SimpleAgent, ReActAgent, ReflectionAgent, PlanAndSolveAgent）
  - `tools/` — 工具注册、熔断器、工具过滤器、内置工具集
  - `context/` — 历史管理、Token 计数、上下文截断
  - `observability/` — 链路追踪日志
  - `skills/` — 技能加载系统

### 3. JavaScript 工作流层：hellowind777/helloagents
- **定位**：AI 编程 CLI 的工作流执行层（Claude Code / Gemini CLI / Codex CLI）
- **Star 数**：~610
- **已内置 14 个工作流技能**：`hello-ui`、`hello-security`、`hello-test`、`hello-arch`、`hello-debug` 等
- **npm**：`helloagents`

### 4. Go 移植版：chaojixinren/HelloAgents-go
- 完整复刻 Python 版的 16 项核心能力

### 5. 实战项目：helloagents-trip-planner
- 智能旅行助手，Vue 3 + FastAPI，含 SFT/DPO 训练数据

---

## 二、教程覆盖的知识体系（16 章）

### 第一部分：基础理论（第 1-3 章）
| 章节 | 内容 | 学完后掌握 |
|------|------|-----------|
| 1 | Agent 理论、LLM 基础 | 理解 Agent 的定义、分类、与 workflow 的本质区别 |
| 2 | Transformer 架构 | 了解 Attention 机制、GPT 系列演化 |
| 3 | Prompt Engineering | 掌握 CoT、Few-shot、System Prompt 等核心技巧 |

### 第二部分：核心范式（第 4-7 章）
| 章节 | 内容 | 学完后掌握 |
|------|------|-----------|
| 4 | ReAct、Plan-and-Solve、Reflection | 三种主流 Agent 范式的原理与实现 |
| 5 | 低代码平台（Coze、Dify、n8n） | 了解行业现状，能快速搭建原型 |
| 6 | 主流框架（AutoGen、LangGraph） | 理解生产级框架的设计取舍 |
| 7 | **手写自己的 Agent 框架** | 从零实现一套可用的多 Agent 框架 |

### 第三部分：进阶能力（第 8-12 章）
| 章节 | 内容 | 学完后掌握 |
|------|------|-----------|
| 8 | 记忆系统 / RAG | 理解 Context Engineering，能设计长期记忆 |
| 9 | MCP / A2A 协议 | 掌握 Agent 互操作标准（Anthropic MCP、Google A2A） |
| 10 | Agentic RL（GRPO） | 从 SFT 到强化学习的完整训练链路 |
| 11 | 评估与基准 | Benchmark 设计、自动化评估 pipeline |
| 12 | Token 成本优化 | 上下文窗口管理、截断策略、成本控制 |

### 第四部分：实战项目（第 13-15 章）
| 章节 | 内容 | 学完后掌握 |
|------|------|-----------|
| 13 | 智能旅行助手 | 完整的多工具协作 Agent 开发 |
| 14 | DeepResearch Agent | 类 GPT-research 的深度调研 Agent |
| 15 | Cyber Town 社会模拟 | 多 Agent 社会仿真系统 |

### 第五部分：结业项目（第 16 章）
- 完整的多 Agent 应用，综合运用全部所学

---

## 三、学完后能达到的水平

### 按能力层级划分

| 层级 | 对应章节 | 能做什么 |
|------|---------|---------|
| **L1 入门** | 1-3 | 理解 Agent 是什么，能用 Prompt 调教 LLM，能阅读 Agent 论文 |
| **L2 会用** | 4-5 | 能用 ReAct 模式写简单 Agent，能用 Dify/Coze 搭建原型 |
| **L3 能造** | 6-7 | 理解 LangGraph/AutoGen 的设计思想，**能从零手写 Agent 框架** |
| **L4 精通** | 8-10 | 能设计 RAG 和记忆系统，能配置 MCP 服务器，能用 RL 微调模型 |
| **L5 工程化** | 11-12 | 能搭建评估体系，能做成本优化，能上线生产级 Agent |
| **L6 全栈** | 13-16 | 能独立完成复杂多 Agent 项目，具备架构设计和团队指导能力 |

### 具体可交付能力

1. **框架层面**：手写一套支持 ReAct / Reflection / Plan-and-Solve 的多 Agent 框架
2. **模型层面**：理解 Agentic RL（GRPO），能对模型做 SFT 和强化学习微调
3. **协议层面**：理解 MCP 和 A2A，能开发和接入 MCP Server
4. **工程层面**：掌握 Context Engineering、成本优化、评估体系、可观测性
5. **项目层面**：能独立完成从需求分析到上线的完整 Agent 项目

---

## 四、推荐学习路径

```
Python 基础
    │
    ▼
第 1-3 章（理论 + Prompt）──── 1-2 周
    │
    ▼
第 4 章（三种 Agent 范式）──── 1 周
    │
    ▼
第 5-6 章（框架对比）──────── 1 周
    │
    ▼
第 7 章（手写框架）◄─────── 核心，建议反复实践
    │
    ├──► 第 8-9 章（RAG + MCP）── 1-2 周
    │
    ├──► 第 10 章（Agentic RL）── 2-3 周（需 GPU）
    │
    ├──► 第 11-12 章（工程化）─── 1 周
    │
    └──► 第 13-16 章（实战）──── 2-4 周
```

### 前置要求
- **必需**：Python 基础语法、基本的 API 调用经验
- **推荐**：了解基本的机器学习概念
- **硬件**：前 9 章仅需普通电脑；第 10 章 RL 训练需要 GPU

---

## 五、与我们当前项目的关系

当前项目 [cryptomator_lab.py](cryptomator_lab.py) 是一个密码学实验脚本。如果后续希望在此基础上引入 AI Agent 能力，可以：

| 方向 | 路径 |
|------|------|
| 用 Agent 自动化密码学实验 | 将实验步骤封装成 Tool，用 ReAct Agent 编排 |
| 构建安全知识问答 Agent | 结合 RAG + MCP，对接安全文档库 |
| 自动生成测试用例 | 用 Reflection Agent 自动验证加密实现正确性 |

---

## 六、参考资源

- **教程仓库**：https://github.com/datawhalechina/hello-agents
- **Python 框架**：https://github.com/jjyaoao/HelloAgents
- **PyPI 包**：`pip install hello-agents`
- **JS 工作流层**：https://github.com/hellowind777/helloagents
- **在线阅读**：教程配套的在线文档（中英文双语）
