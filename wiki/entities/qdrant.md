---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "qdrant/qdrant"
generation_complete: true
---


# Qdrant

## 描述
Qdrant 是一个用 Rust 编写的开源向量数据库，采用 Apache-2.0 许可证，其 GitHub 仓库 qdrant/qdrant 在 2026-08-11 快照中约有 33.9k 星标。它支持 dense、sparse 与 multi-vector 向量类型，并提供 Payload 过滤、Hybrid 查询、量化、分片与复制能力，是 RAG 基础设施层的代表产品。关键源码包括 lib/segment/src/index/hnsw_index/hnsw.rs（HNSW 近邻索引实现）、lib/segment/src/index/sparse_index（稀疏检索）与 lib/collection/src/shards（分片存储）。官方文档涵盖 Hybrid Queries、RRF/DBSF、Multi-Stage Query、Quantization 与安全配置等主题。在源码阅读路线中，Qdrant 被推荐作为第五站，目标是补齐"框架调用背后，ANN、过滤和分布式存储究竟做了什么"。在 [[entities/nashsullm_wiki|nashsullm_wiki]] 的落地方案中，它被列为页面超出人工导航范围后的本地搜索增强选项。

## 相关实体
- [[entities/ragflow|RAGFlow]]
- [[entities/llamaindex|LlamaIndex]]
- [[entities/open-webui|Open WebUI]]
- [[entities/graphrag|graphrag]]

## 相关概念
- [[concepts/embedding|Embedding]]
- [[concepts/bm25|BM25]]
- [[concepts/reranker|Reranker]]
- [[concepts/hybrid-search|Hybrid Search]]

## 来源提及

- "| [Qdrant](https://github.com/qdrant/qdrant) | 33.9k | Apache-2.0 | 向量数据库 | HNSW、Sparse、Payload、Hybrid、分布式 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "Qdrant 是 Rust 编写的向量数据库，支持 dense、sparse、multi-vector、Payload 过滤、Hybrid、量化、分片和复制。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "官方文档还展示了 Hybrid Queries、RRF/DBSF、Multi-Stage Query、Quantization 与安全配置。学习它能补齐“框架调用背后，ANN、过滤和分布式存储究竟做了什么”。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]