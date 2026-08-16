---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "LangChain 框架"
  - "langchain-ai/langchain"
generation_complete: true
---


# LangChain

## 描述
LangChain 是一个以 MIT 许可证开源的 LLM/Agent 基础框架，GitHub 仓库为 langchain-ai/langchain，截至 2026-08-11 星标约 143.9k。它并非一个单一的 RAG 产品，而是把模型、文档、Retriever、VectorStore、Runnable 与 Callback 抽象为可组合的接口契约，让开发者用标准范式组装 RAG 与 Agent 应用。其典型数据流为 Document → Splitter → Embeddings → VectorStore → Query → Retriever → Reranker/Postprocessor → Model，覆盖从文档摄取到生成回答的完整链路。与 [[entities/llamaindex|LlamaIndex]]、Haystack 并列为最常见的 RAG 框架，LangChain 属于框架层而非产品层，适合研究 [[concepts/rag|RAG]] 各组件如何通过接口解耦。学习重点应放在抽象边界上，包括 Retriever 契约、VectorStore 基础接口、文档索引 API 以及 Document/metadata 对象。

## 相关实体
- [[entities/llamaindex|LlamaIndex]]
- [[entities/ragflow|ragflow]]

## 相关概念
- [[concepts/rag|RAG]]
- [[concepts/embedding|Embedding]]
- [[concepts/chunking|Chunking]]
- [[concepts/reranker|Reranker]]

## 来源提及

- "LangChain 不是一个单独的 RAG 产品，而是把 LLM 应用中的模型、文档、Retriever、VectorStore、Runnable 和 Callback 抽象成可组合接口。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "学习重点不是记住多少个集成类，而是理解：

Document → Splitter → Embeddings → VectorStore
Query → Retriever → Reranker/Postprocessor → Runnable/Prompt → Model" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]