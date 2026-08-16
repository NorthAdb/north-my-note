---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "重排器"
  - "重排模型"
  - "Reranking"
generation_complete: true
---


# Reranker

## 定义
Reranker（重排器/重排模型）是 RAG 检索后处理阶段的关键组件，负责在粗召回结果上进一步判断候选文档与查询的真实相关性，将候选文档从 Top-N 精排到 Top-K。典型实现使用 Cross-Encoder 或 LLM Reranker，将"查询 + 文档"放入同一模型输入中进行充分交互，区别于第一阶段双编码器按向量相似度的快速召回。Reranker 是 Naive RAG 升级为 Advanced RAG 的标志性组件之一。

## 关键特征
- **两阶段精排**：在粗召回（如向量检索、BM25）之后，对 Top-N 候选重新打分排序，输出 Top-K 作为最终上下文。
- **交互式编码**：Cross-Encoder 将查询与文档拼接后同时编码，能捕捉细粒度语义匹配信号，精度高于双编码器的向量点积/余弦相似度。
- **精度收益与代价**：Rerank 通常会显著提升检索精度，但会引入额外延迟和计算成本；候选集越大成本越高。
- **能力边界明确**：Rerank 只能对已召回文档重排序，无法修复"文档根本没有被召回"的召回率缺陷。
- **与上下文编排机制协同**：常与 RRF、MMR、去重、父块扩展等机制共同构成 RAG 的上下文编排流程。

## 应用
- **Advanced RAG 检索后处理**：在向量检索或混合检索后的上下文构建阶段执行精排，确保送入 LLM 的上下文与查询高度相关。
- **企业级 RAG 平台**：LlamaIndex、Haystack、RAGFlow、Onyx、Open WebUI 等开源项目均提供重排实现，用于提升问答、企业搜索等场景的检索质量。
- **混合检索融合**：与 BM25、Embedding 检索结果合并后，通过 Reranker 统一重排，消除不同检索器分数不可比的问题。
- **多路召回精排**：对多个召回来源（向量检索、关键词检索、知识图谱检索）的候选集合进行全局重排序。
- **成本敏感场景调控**：通过调整 Top-N 与 Top-K 参数，在延迟、成本和精度之间取舍。

## 相关概念
- [[concepts/rag|rag]] — Reranker 是 RAG 检索后处理阶段的核心组件
- [[concepts/bm25|bm25]] — 常用的一阶段粗召回方法，其结果可作为 Reranker 的输入候选集
- [[concepts/embedding|embedding]] — 双编码器向量检索是一阶段召回的代表，与 Reranker 的两阶段精排形成对比
- [[concepts/knowledge-graph|knowledge-graph]] — 知识图谱检索可与 Reranker 结合进行多路召回统一精排

## 相关实体
- [[entities/llamaindex|llamaindex]] — 提供多种重排器集成（如 Cross-Encoder、Cohere Rerank）
- [[entities/ragflow|ragflow]] — RAGFlow 在文档检索后处理流程中内置重排组件
- [[entities/qdrant|qdrant]] — 向量数据库，常作为粗召回阶段存储与检索候选文档，配合 Reranker 使用

## 来源提及

- "第二阶段使用 Cross-Encoder 或 LLM Reranker：把“查询 + 文档”放在同一个模型输入中，让两者发生更充分的交互，负责从 Top-N 精排到 Top-K。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "Rerank 通常提升精度，但会增加延迟和成本；它不能修复“文档根本没有被召回”的问题。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **Reranking** | 如何在候选结果中判断真实相关性 | 精排 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]