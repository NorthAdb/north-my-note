---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "open-webui/open-webui"
  - "Open WebUI 平台"
generation_complete: true
---


# Open WebUI

## 描述
Open WebUI 是一个自托管的 AI 与 [[concepts/rag|RAG]] 平台，截至 2026-08-11 GitHub 星标约 148.4k，是笔记项目总览中星标最高的项目。它不只做向量检索，还同时覆盖文件、网页、YouTube 接入、知识库、权限、[[concepts/hybrid-search|Hybrid Search]]、[[concepts/reranker|重排]]、会话以及后台 [[concepts/agent-memory|Agent Memory]] 写入。其 retrieval.py 负责接入与集合管理，retrieval/utils.py 实现向量检索、BM25、ensemble fusion、重排和阈值，models/knowledge.py 管理知识库与文件访问关系。Open WebUI 特别能体现多用户 RAG 平台中知识库、权限边界和混合检索如何协同工作。值得注意的是，它采用自定义 Open WebUI License，不能简单地当作 MIT 项目使用，这是选型时必须考虑的因素。

## 相关实体
- [[entities/ragflow|ragflow]]
- [[entities/onyx|Onyx]]
- [[entities/anythingllm|AnythingLLM]]

## 相关概念
- [[concepts/rag|RAG]]
- [[concepts/hybrid-search|Hybrid Search]]
- [[concepts/reranker|Reranker]]
- [[concepts/agent-memory|Agent Memory]]

## 来源提及

- "Open WebUI 的学习价值在于：真正的 RAG 平台要同时处理文件、网页、YouTube、知识库、权限、混合检索、重排、会话和 Memory。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "注意：它使用的是自定义 Open WebUI License，不应简单地当成 MIT 项目使用。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "backend/open_webui/routers/retrieval.py：文件、文本、网页、YouTube 接入、分块、Embedding 和集合管理；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]