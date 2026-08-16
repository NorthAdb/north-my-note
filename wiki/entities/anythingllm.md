---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "Mintplex-Labs/anything-llm"
  - "AnythingLLM 平台"
generation_complete: true
---


# AnythingLLM

## 描述
AnythingLLM 是 Mintplex-Labs 维护的开源 AI 聊天与 RAG 平台，其 GitHub 仓库为 Mintplex-Labs/anything-llm，许可证为 MIT，截至 2026-08-11 星标约 64.6k。该平台以 Workspace 和向量库 namespace 作为主要持久对象，而不是 [[entities/nashsullm_wiki|nashsullm_wiki]] 式的实体页与概念页。其文档重点介绍四项关键实现：embedding-worker.js 顺序处理文档与嵌入、vectorDbProviders/base.js 提供 addDocumentToNamespace 与相似度搜索抽象、DocumentManager 负责固定文档和上下文管理、agents/aibitat/plugins/memory.js 将明确要求保存的内容写入 Workspace 向量空间，其 agent 记忆方向与 [[entities/mem0|mem0]] 同属 [[concepts/agent-memory|Agent Memory]] 范畴。与 [[entities/open-webui|open-webui]]、[[entities/ragflow|ragflow]] 等平台相比，AnythingLLM 提供“可检索”的向量记忆，但不会自动长出可浏览的概念层——文档以此说明“可检索”与“可理解、可维护”是两件事。

## 相关实体
- [[entities/open-webui|open-webui]]
- [[entities/ragflow|ragflow]]
- [[entities/mem0|mem0]]

## 相关概念
- [[concepts/rag|RAG]]
- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/agent-memory|Agent Memory]]

## 来源提及

- "| [AnythingLLM](https://github.com/Mintplex-Labs/anything-llm) | 64.6k | MIT | Workspace RAG + Agent | Embedding Worker、命名空间、向量 Memory |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "AnythingLLM 的持久对象主要是 Workspace 和向量库 namespace，而不是 LLM Wiki 的实体页和概念页。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它说明：向量 Memory 足够支撑很多文档问答，但不会自动长出可浏览的概念层；“可检索”与“可理解、可维护”是两件事。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]