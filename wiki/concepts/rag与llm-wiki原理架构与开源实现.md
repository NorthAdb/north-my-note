---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [theory]
aliases:
  - "RAG 与 LLM Wiki"
  - "RAG & LLM Wiki"
generation_complete: true
---


# RAG与LLM Wiki：原理、架构与开源实现

## 定义

RAG与LLM Wiki：原理、架构与开源实现是一套系统梳理大语言模型知识增强两大范式的理论框架与工程指南。它以 2020 年 Lewis 等人的 RAG 论文为起点，将外部文档索引视为非参数记忆、参数化模型作为生成核心，形成"检索-生成"闭环；并延伸至 Andrej Karpathy 提出的 LLM Wiki 理念——将原始资料增量编译为持久、互链、可维护的 Markdown 知识层。该概念同时覆盖 RAG 变体谱系、Prompt Injection 安全边界、基于 Ragas 的分层评测指标，以及多个开源实现项目的对比与选型建议，为生产级 LLM 知识应用提供从原理到落地的完整方法论。

## 关键特征

- **双范式融合**：将 RAG 的检索增强生成与 LLM Wiki 的知识编译纳入统一框架，前者解决"即用型"知识注入，后者解决"沉淀型"知识长期管理。
- **完整工程链路**：覆盖文档解析、分块、Embedding、向量数据库、查询改写、多路召回、重排、上下文组装与引用溯源的端到端流水线。
- **变体谱系**：系统归纳 Naive、Advanced、Hybrid、Hierarchical、Conversational、Agentic、GraphRAG、LightRAG 等 RAG 变体及其适用场景。
- **安全与评测并重**：关注 Prompt Injection 等安全边界，并采用 Ragas 分层评测指标量化检索质量与生成质量。
- **LLM Wiki 三层架构**：以 Raw Sources、Wiki、Schema 三层结构组织知识层，围绕 Ingest、Query、Lint 三个核心操作实现增量编译与可维护性。
- **开源落地导向**：以 nashsu/llm_wiki、claude-obsidian、TencentDB Agent Memory、DeepWiki-Open、RAGFlow、Qdrant 等项目为参考实现，给出选型建议与渐进式生产路线。

## 应用

- **企业知识库问答**：应用生产级 RAG 工程链路实现大规模文档的检索问答、多路召回与引用溯源。
- **个人知识管理**：基于 LLM Wiki 理念面向 Obsidian Vault 构建持久、互链、可维护的 Markdown 知识层。
- **智能体记忆系统**：结合 Agent Memory 类方案为对话智能体提供长期记忆、上下文管理与状态沉淀。
- **开源技术选型**：在 RAGFlow、Qdrant、DeepWiki-Open、LightRAG、GraphRAG 等项目之间依据检索规模、图谱需求、部署成本等条件进行选型。
- **渐进式生产路线**：从原型验证、评测监控到安全加固的演进路径设计，指导 RAG 系统从实验走向生产。

## 相关概念

- [[concepts/rag|RAG]]
- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/naive-rag|朴素 RAG]]
- [[concepts/agentic-rag|Agent 式 RAG]]
- [[concepts/knowledge-graph|知识图谱]]
- [[concepts/vector-database|向量数据库]]
- [[concepts/embedding|向量嵌入]]
- [[concepts/chunking|切块]]
- [[concepts/hybrid-search|混合检索]]
- [[concepts/semantic-search|语义搜索]]
- [[concepts/reranker|重排器]]
- [[concepts/bm25|BM25]]
- [[concepts/rrf|RRF]]
- [[concepts/hyde|HyDE]]
- [[concepts/provenance|来源溯源]]
- [[concepts/context-engineering|上下文工程]]
- [[concepts/knowledge-compilation|知识编译]]
- [[concepts/rag-sequence|RAG-Sequence]]
- [[concepts/rag-token|RAG-Token]]
- [[concepts/code-wiki|代码 Wiki]]
- [[concepts/agent-memory|Agent 记忆]]

## 相关实体

- [[entities/andrej-karpathy|Andrej Karpathy]]
- [[entities/nashsullm_wiki|llm_wiki]]
- [[entities/claude-obsidian|Claude Obsidian]]
- [[entities/tencentdb-agent-memory|TencentDB Agent Memory]]
- [[entities/deepwiki-open|deepwiki-open]]
- [[entities/ragflow|RAGFlow]]
- [[entities/qdrant|Qdrant]]
- [[entities/lightrag|LightRAG]]
- [[entities/graphrag|GraphRAG]]
- [[entities/ragas|Ragas]]
- [[entities/langchain|LangChain]]