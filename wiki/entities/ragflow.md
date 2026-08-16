---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "infiniflow/ragflow"
generation_complete: true
---


# RAGFlow

## 描述

RAGFlow 是 InfiniFlow 开源的完整 RAG 引擎，GitHub 仓库为 infiniflow/ragflow，采用 Apache-2.0 许可证，截至资料记录时拥有约 87.2k 星。它的重点不只在于向量搜索，还覆盖文档理解、模板化切块、混合检索、融合重排、引用溯源以及 Agentic RAG 工作流。源文件指出，一个“上传 PDF 后问答”的产品背后包含 Parser、Chunker、Index、Retriever、Reranker、Citation 与 Agent Graph 等多个层次。其关键代码路径包括 rag/flow/pipeline.py、rag/flow/chunker/token_chunker.py 与 agentic_rag_graph.py，是研究完整 RAG 产品架构的重要开源样例。在相关讨论中，RAGFlow 常与 [[entities/graphrag|graphrag]]、[[entities/lightrag|LightRAG]] 等开源方案对照，并被视作 [[concepts/rag|RAG]] 与 [[concepts/agent-memory|Agent Memory]] 落地的参照实现。

## 相关实体

- [[entities/graphrag|graphrag]] — 微软开源的知识图谱驱动 RAG 方案，与 RAGFlow 的检索式架构形成对比
- [[entities/lightrag|LightRAG]] — 轻量级 RAG 开源实现，代表另一种简化的架构取舍
- [[entities/nashsullm_wiki|nashsullm_wiki]] — 相关讨论中与之并列的 LLM Wiki 项目参考

## 相关概念

- [[concepts/rag|RAG]] — RAGFlow 所实现的核心领域概念
- [[concepts/agent-memory|Agent Memory]] — Agentic RAG 工作流中记忆与检索相结合的概念
- [[concepts/llm-wiki|LLM Wiki]] — LLM 驱动的维基式知识库概念

## 来源提及

- "RAGFlow 是完整的 RAG 引擎，重点不只在向量搜索，而在文档理解、模板化切块、混合检索、融合重排、引用和 Agent 工作流。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "阅读它时重点观察：一个“上传 PDF 后问答”的产品，背后其实包含 Parser、Chunker、Index、Retriever、Reranker、Citation 和 Agent Graph 多个层。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| [RAGFlow](https://github.com/infiniflow/ragflow) | 87.2k | Apache-2.0 | 完整 RAG 引擎 | 文档解析、Chunk、混合检索、Agentic RAG |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]