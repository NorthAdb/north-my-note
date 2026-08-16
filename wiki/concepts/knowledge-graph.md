---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [term]
aliases:
  - "知识图谱"
  - "KG"
generation_complete: true
---


# Knowledge Graph

## 定义

Knowledge Graph（知识图谱，KG）是一种以图结构组织和存储知识的方式，将实体表示为节点、关系表示为边，并以属性与类型标记节点和边的语义。它的持久化对象是节点、边、属性与类型，主要动作是关系查询与图遍历，解决的核心问题是"哪些实体通过什么关系连接"。它是 GraphRAG、LightRAG 与图增强检索的共同基础，在总体地图中与 LLM Wiki 的关系被标为"不一定"——两者并非必然绑定。

## 关键特征

- 以节点（实体）、边（关系）、属性、类型为基本持久化对象，构成显式结构化知识表示
- 主要查询动作是关系查询与图遍历，而非向量相似度检索或全文匹配
- 解决的独特问题可概括为"哪些实体通过什么关系连接"，天然适合多跳关系推理
- 作为 Graph Retrieval 的证据来源：通过实体、关系、邻居和路径寻找证据，适合多跳问题
- 是 RAG 多路召回中的一路，与向量检索、关键词检索互补
- 落地成本较高：需要图构建与实体治理（Entity Resolution），否则图谱质量难以保证
- 派生实体消歧（Entity Resolution）与本体约束（Ontology）等子概念
- 与 LLM Wiki 关系"不一定"：可用于跨页实体链接与知识维护，但 LLM Wiki 不必然依赖知识图谱

## 应用

- **图增强检索**：微软 [[entities/graphrag|graphrag]] 从语料中抽取实体、关系与社区结构，构建分层图谱索引以支持全局与局部查询
- **关键词图检索**：[[entities/lightrag|lightrag]] 构建高低层关键词图，兼顾具体与抽象层面的检索需求
- **图 + 向量记忆**：[[entities/cognee|cognee]] 将知识图谱与向量记忆结合，为智能体提供结构化长期记忆
- **Agent 记忆基础设施**：[[entities/tencentdb-agent-memory|tencentdb-agent-memory]] 等托管服务将知识图谱作为记忆存储形态之一
- **多跳问答**：在 [[concepts/rag|rag]] 中作为一路召回，通过实体、关系、邻居与路径提供可解释证据
- **Wiki 知识维护**：在 [[concepts/llm-wiki|llm-wiki]] 中用于跨页实体链接与知识维护

## 相关概念

- [[concepts/rag|rag]]
- [[concepts/llm-wiki|llm-wiki]]
- [[concepts/agent-memory|agent-memory]]
- [[concepts/entity-resolution|Entity Resolution]]
- [[concepts/ontology|Ontology]]

## 相关实体

- [[entities/graphrag|graphrag]]
- [[entities/lightrag|lightrag]]
- [[entities/cognee|cognee]]
- [[entities/tencentdb-agent-memory|tencentdb-agent-memory]]

## 来源提及

- "| **Knowledge Graph** | 节点、边、属性、类型 | 关系查询与图遍历 | 不一定 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "- **Knowledge Graph**：解决“哪些实体通过什么关系连接”；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "#### Graph Retrieval

通过实体、关系、邻居和路径找证据，适合多跳问题，但需要图构建和实体治理。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]