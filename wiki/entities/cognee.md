---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "topoteretes/cognee"
  - "Cognee 知识记忆引擎"
generation_complete: true
---


# Cognee

## 描述
Cognee 是一个图与向量混合的知识记忆引擎，采用 Apache-2.0 许可，截至 2026-08-11 在 GitHub 上拥有约 29.9k 星。它通过 cognify 流程将数据加工为实体、关系、摘要与 Embedding，并提供多通道搜索能力，可同时检索 Chunk、Entity、Edge、Summary 与 Graph Context。其代码中 extract_graph_from_data.py 负责 LLM 图抽取、Ontology 校验、关系整合与溯源，hybrid_retriever.py 实现图与向量结合的混合检索，cognify_session.py 支持会话记忆持久化。相比简单向量库，Cognee 更接近知识图谱与 [[concepts/agent-memory|Agent Memory]] 的整合层，与 [[entities/mem0|mem0]]、[[entities/graphrag|graphrag]] 同属记忆与检索增强方向，但更强调 [[concepts/knowledge-graph|Knowledge Graph]] 的可追溯、可下钻能力。检索时同时查询图、块、摘要和全局上下文，是其最值得学习的特性。

## 相关实体
- [[entities/mem0|mem0]] — 同为面向 Agent 的记忆层项目，Cognee 更强调图结构记忆
- [[entities/graphrag|graphrag]] — 同样利用图索引增强检索，但侧重文档级问答场景
- [[entities/tencentdb-agent-memory|tencentdb-agent-memory]] — 腾讯云 Agent 记忆服务，同属记忆管理方向

## 相关概念
- [[concepts/agent-memory|Agent Memory]]
- [[concepts/knowledge-graph|Knowledge Graph]]
- [[concepts/hybrid-search|Hybrid Search]]
- [[concepts/chunking|Chunking]]

## 来源提及

- "Cognee 将数据经过 `cognify` 转化为实体、关系、摘要和 Embedding，并提供多通道搜索。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "cognee/tasks/graph/extract_graph_from_data.py：LLM 图抽取、Ontology 校验、关系整合和溯源；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它展示了一个比简单向量库更完整的知识层：检索时可以同时查 Chunk、Entity、Edge、Summary 和 Graph Context。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]