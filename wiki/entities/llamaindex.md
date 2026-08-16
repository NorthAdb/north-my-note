---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "run-llama/llama_index"
  - "LlamaIndex 框架"
generation_complete: true
---


# LlamaIndex

## 描述
LlamaIndex（run-llama/llama_index）是一个以数据为中心的 RAG 与 Agent 开源框架，GitHub 星标约 51.5k（2026-08-11 快照），采用 MIT 许可证。它将 Documents、Nodes、Ingestion Pipeline、Index、Retriever、Postprocessor、Response Synthesizer 与 Storage 组织为完整的数据框架，覆盖文档分块、Embedding、索引构建、多查询融合（fusion retriever）与 LLM 重排等环节。其核心代码路径包括 ingestion/pipeline.py（转换、缓存与去重）、token.py（Token 切块）、fusion_retriever.py（多查询生成与结果融合）和 llm_rerank.py（LLM 重排）。它常与 [[entities/qdrant|qdrant]] 等向量数据库配合作为存储后端，并与 [[entities/ragflow|ragflow]]、[[entities/graphrag|graphrag]] 等并列为 RAG 工程链路的重要参考实现，是研究"文档如何逐步变成 Node、Node 如何携带 metadata、多个 Retriever 如何融合以及回答如何合成"的典型样本。

## 相关实体
- [[entities/qdrant|qdrant]] — LlamaIndex 常用的向量存储后端，负责索引向量的持久化与相似度检索。
- [[entities/ragflow|ragflow]] — 同为 RAG 开源参考实现，可对照其文档解析、分块与检索管线的设计取舍。
- [[entities/graphrag|graphrag]] — 图增强检索路线，与 LlamaIndex 的可组合检索器架构形成互补对比。

## 相关概念
- [[concepts/rag|RAG]] — LlamaIndex 的核心问题域，框架围绕检索增强生成的全链路设计。
- [[concepts/chunking|Chunking]] — 对应 token.py 与文档切块逻辑，决定 Node 的粒度与质量。
- [[concepts/embedding|Embedding]] — 索引构建与检索匹配的向量化基础。
- [[concepts/reranker|Reranker]] — 对应 llm_rerank.py 的 LLM 重排能力，用于精排检索结果。

## 来源提及

- "LlamaIndex 把 Documents、Nodes、Ingestion Pipeline、Index、Retriever、Postprocessor、Response Synthesizer 和 Storage 组织成一套数据框架。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它适合研究“文档如何逐步变成 Node、Node 如何带 metadata、多个 Retriever 如何融合，以及回答如何合成”。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "llama-index-core/llama_index/core/retrievers/fusion_retriever.py：多查询生成和结果融合；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]