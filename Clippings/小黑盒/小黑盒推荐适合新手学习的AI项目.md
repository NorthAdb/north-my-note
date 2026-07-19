---
created: 2026-07-07
updated: 2026-07-07
tags: [github, 项目推荐, AI, RAG, agent, 新手, 学习]
source:
  - "https://www.xiaoheihe.cn/bbs/post_share?link_id=e042722adb85"
  - "https://github.com/YuiHlk/Enterprise-AI-Knowledge-Base-Platform"
  - "https://github.com/YuiHlk/Music-AI-Agent"
---

# 小黑盒推荐：适合新手学习/练手的 AI 项目

> 来源：小黑盒用户"你蒸芝士"（Lv.16）· 昨天发布
> lz 是大二学生，主学 Java 后端，用 Codex 做了两个 AI 方向的项目

---

## 概览

| 项目 | 方向 | ⭐ | 技术栈 |
|------|:----:|:--:|--------|
| [Enterprise-AI-Knowledge-Base-Platform](https://github.com/YuiHlk/Enterprise-AI-Knowledge-Base-Platform) | **RAG 知识库平台** | 2 | Java + Spring Boot + Vue3 + Spring AI + ChromaDB |
| [Music-AI-Agent](https://github.com/YuiHlk/Music-AI-Agent) | **AI Agent 音乐创作** | 2 | Java + LangChain4j + DeepSeek + MCP + Vue3 |

---

## 项目一：Enterprise-AI-Knowledge-Base-Platform

> 代号 QA — 企业级 AI 知识库平台

### 简介

基于 Spring Boot + Vue3 的全栈 AI 应用平台，覆盖工业级 AI 应用开发全套工程能力：**提示词工程、RAG 检索增强生成、LLM-as-Judge 自动化评测、消融实验、模型微调联动**。

### 技术栈

| 层级 | 技术 |
|------|------|
| 后端框架 | Java 17, Spring Boot 3.4.5 |
| AI 框架 | Spring AI 1.0.0-M6（OpenAI 兼容协议） |
| ORM | MyBatis-Plus 3.5.9 |
| 数据库 | MySQL 8.0 |
| 向量数据库 | ChromaDB |
| 文档解析 | Apache PDFBox 3.0.3 |
| API 文档 | SpringDoc OpenAPI 2.7.0 |
| 前端框架 | Vue 3, Vite, Element Plus |
| 容器化 | Docker Compose（一键部署） |
| Python 微调 | Python 3.10, FastAPI, transformers/peft |

### 功能模块

- **提示词工程**：管理和测试 Prompt
- **RAG 知识库**：文档上传 → 解析 → 向量化 → 检索增强生成
- **LLM-as-Judge 评测**：自动化评估回答质量
- **消融实验**：对比不同配置的效果
- **模型微调联动**：通过 Python 子服务调用 QLoRA 微调

### 项目结构

```
QA/
├── ai-java-main/           # SpringBoot 核心后端（7 个 Controller，40+ 端点）
├── ai-frontend/            # Vue3 + Element Plus 前端（8 个页面组件）
├── python-train-side/      # Python QLoRA 微调子服务
├── docker-compose.yml      # 5 服务容器编排
└── start.sh / start.bat    # 一键部署脚本
```

### 适合学习什么

- ✅ **RAG 完整链路**：从文档解析到向量检索到生成回答
- ✅ **Java + Spring Boot AI 集成**（Spring AI 框架）
- ✅ **全栈项目**：从后端 API 到前端界面
- ✅ **Docker Compose 部署**：多服务编排
- ✅ **Python + Java 混合架构**：微调服务用 Python

---

## 项目二：Music-AI-Agent

> 面向吉他手和音乐创作者的音乐创作 Agent

### 简介

用户使用自然语言描述调性、速度、拍号、风格和情绪，系统将其转换为结构化创作约束，由纯 Java 音乐内核生成并校验可演奏的吉他 Riff，保存项目版本，最终导出 Guitar Pro 8 可编辑的 MIDI 与 MusicXML。

**核心哲学**："AI 理解与编排 + 确定性规则生成"的模块化单体 MVP。

### 技术栈

| 模块 | 技术 |
|------|------|
| 后端 | Java 21, Spring Boot 3.5.16, Maven, MyBatis-Plus |
| AI / Agent | LangChain4j 1.17.2, DeepSeek, Chat Memory, Tool Calling |
| 协议 | REST, SSE, MCP SDK 2.0 |
| 数据 | H2（默认本地）/ MySQL 8.4 |
| 前端 | Vue 3, Vite, Element Plus, Web Audio |
| 音频服务 | Python 3.11, FastAPI |
| 部署 | Docker Compose |

### Agent 架构亮点

- **中英文提示词解析** + 无模型离线规则解析器
- **LangChain4j Tool Calling**：Agent 通过工具调用生成音乐
- **Chat Memory**：项目级对话记忆，支持多轮交互
- **MCP Server**：提供 Model Context Protocol 接口
- **SSE 进度事件**：异步生成任务实时推送进度
- **确定性规则生成**：DeepSeek 负责理解意图，Java 领域层负责音乐事件生成

### 音乐生成链路

```
自然语言 → AI 意图解析 → 结构化约束 (Score) → Java 内核生成 Riff
  → 校验（拍号/音高/调弦/品位置）→ MIDI + MusicXML 导出
```

其中 **Score** 是生成、修改、校验和导出的**唯一事实来源**，大模型不直接写数据库或文件路径。

### 项目结构

```
demo/
├── music-backend/           # Spring Boot、领域模型、Agent、持久化与导出
├── music-frontend/          # Vue 3 创作工作台与 MIDI 播放
├── music-ai-python/         # 可选 WAV 元数据/BPM 分析服务
├── Music-AI-Agent-Docs/     # Obsidian 项目知识库
├── .claude/                 # Claude Code 配置文件
├── AGENTS.md                # 架构边界与开发规范
└── docker-compose.yml
```

### 适合学习什么

- ✅ **LangChain4j Agent 开发**（Java 生态的 AI Agent 框架）
- ✅ **Tool Calling 模式**：Agent 调用工具完成复杂任务
- ✅ **MCP 协议集成**：提供 MCP Server
- ✅ **领域驱动设计**：纯 Java 领域模型 + AI 编排
- ✅ **Chat Memory 对话记忆**：跨轮次上下文管理
- ✅ **全栈 + 多服务架构**：Java + Python + Vue 三端协作

---

## 📊 项目对比 & 学习建议

| 维度 | RAG 知识库平台 | Music AI Agent |
|------|:-------------:|:--------------:|
| 难度 | ⭐⭐ 中等 | ⭐⭐⭐ 中等偏难 |
| 代码量 | 较大（5 服务） | 中等（3 服务） |
| 偏重方向 | RAG / 检索 / 评测 | Agent / Tool Calling / 领域模型 |
| AI 框架 | Spring AI | LangChain4j + DeepSeek |
| 新手友好度 | ⭐⭐⭐ 注释/文档齐全 | ⭐⭐⭐ 注释/文档齐全 |
| 可展示性 | ⭐⭐⭐ 完整管理后台 | ⭐⭐⭐⭐ 音乐创作有视觉反馈 |

### 学习路线建议

```
先做 RAG 知识库平台 → 理解 RAG 链路、Spring AI 集成
        ↓
再做 Music AI Agent → 理解 Agent + Tool Calling + MCP
        ↓
两者互补 → RAG 是 AI 应用的"数据基础"，Agent 是"行动能力"
```

### 作者说

> "代码方面都做了很多注释以及写了一些技术文档，所以不用担心看不懂，当然还是能有一点项目经验再来学习最好啦，最后贴上项目地址希望大家点点star。"

两个项目都包含：
- 📝 **详细的代码注释**
- 📚 **技术文档**
- 🐳 **Docker Compose 一键部署**
- Obsidian 项目知识库（Music-AI-Agent 含 `.claude/` 配置）

---

## 🔗 关联笔记

- [[领域/AI Agent 智能体学习路线 2026.md]] — 完整 AI Agent 学习路线
- [[领域/Python vs TypeScript AI时代编程语言之争.md]] — AI 开发语言选择
- [[领域/Claude Code记忆系统与Agent记忆架构.md]] — Agent 记忆系统
- [[Clippings/小红书/2026-07-07-小红书热议AI Agent开发用什么框架.md]] — 框架选型讨论
