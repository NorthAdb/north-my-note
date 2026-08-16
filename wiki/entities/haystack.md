---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "deepset-ai/haystack"
  - "Haystack 框架"
  - "deepset Haystack"
generation_complete: true
---


# Haystack

## 描述

Haystack 是 deepset 团队维护的 Apache-2.0 开源 [[concepts/rag|RAG]] 管线框架，截至 2026-08-11 在 GitHub 上约有 26.2k 星。与 [[entities/langchain|langchain]]、[[entities/llamaindex|LlamaIndex]] 等同属 RAG 框架阵营，但 Haystack 更强调 Pipeline 内部的可观察性与组件化组合。其核心特点是组件与 Pipeline 之间显式连接，开发者可以直接观察真实的数据流、条件分支和循环，而非把所有逻辑隐藏在一次 Agent 调用中。源码提供了层级文档切分器（[[concepts/chunking|Chunking]]）、multi-query retriever（[[concepts/query-rewriting|Query Rewriting]]）、LLM ranker（[[concepts/reranker|Reranker]]）等组件，适合学习父子块切分、多查询检索与重排的具体实现。对于想理解 RAG 从切块到检索再到重排完整数据流的开发者而言，Haystack 是很好的参考实现。

## 相关实体

- [[entities/langchain|langchain]] — 同为 RAG 框架，但更偏 Agent 式编排，逻辑常隐藏在高层抽象中
- [[entities/llamaindex|LlamaIndex]] — 同为 RAG 框架，侧重文档索引与数据连接器生态
- [[entities/ragflow|ragflow]] — 同为 RAG 框架，强调端到端可落地的 RAG 工作流与可视化

## 相关概念

- [[concepts/chunking|Chunking]] — Haystack 提供层级文档切分器，可学习父子块切分策略
- [[concepts/query-rewriting|Query Rewriting]] — multi-query retriever 组件背后的查询改写思路
- [[concepts/rag|RAG]] — 检索增强生成范式，Haystack 所属的框架类别
- [[concepts/reranker|Reranker]] — 重排阶段组件，Haystack 提供 LLM ranker 实现

## 来源提及

- "Haystack 的特点是组件和 Pipeline 显式连接，适合观察真实的数据流、条件分支和循环，而不是把所有逻辑隐藏在一个 Agent 调用中。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "haystack/components/preprocessors/hierarchical_document_splitter.py：父子层级切块；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "haystack/components/retrievers/multi_query_text_retriever.py：多查询检索；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]