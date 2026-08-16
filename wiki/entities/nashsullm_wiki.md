---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "llm_wiki"
generation_complete: true
---


# nashsu/llm_wiki

## 描述

nashsu/llm_wiki 是与 [[entities/andrej-karpathy|Andrej Karpathy]] 的 [[concepts/llm-wiki|LLM Wiki]] gist 理念最贴近的产品化开源实现，仓库以 GPLv3 许可证发布，截至 2026-08-11 在 GitHub 上约有 16.1k 星标。它不是简单的"文档问答"应用，而是将原始资料编译为互链的 Markdown 知识页面，并提供查询、图分析和体检（lint）能力。核心实现包括 src/lib/ingest.ts（ingest 写入流程与 Frontmatter）、src/lib/search.ts（词法搜索与可选向量搜索）、src/lib/wiki-graph.ts（从 Markdown 链接构建 Wiki 图）、src/lib/graph-relevance.ts（链接图相关性计算）、src/lib/lint.ts（断链、孤立页、矛盾、过期检查）以及 src/lib/chat-save-to-wiki.ts（把有价值的聊天回答写回 Wiki）。该项目的学习价值在于"[[concepts/rag|RAG]] 之外的部分"：页面类型、写回、链接图、来源溯源、维护和 lint，并证明了小规模 Wiki 可以先用 index.md + 词法搜索 + 图遍历工作，不一定一开始就需要向量数据库。

## 相关实体

- [[entities/andrej-karpathy|Andrej Karpathy]] — LLM Wiki 理念的提出者，本项目是其 gist 构思最贴近的产品化实现
- [[entities/claude-obsidian|claude-obsidian]] — 同类将 LLM 输出与本地知识库整合的开源项目

## 相关概念

- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/rag|RAG]]
- [[concepts/knowledge-graph|Knowledge Graph]]

## 来源提及

- "这是与 Karpathy gist 最贴近的产品化实现。它不是简单的“文档问答”，而是把资料编译为互链页面，并提供查询、图分析和体检能力。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它最适合学习“RAG 之外的部分”：页面类型、写回、链接图、来源溯源、维护和 lint。它也证明了：小规模 Wiki 不一定一开始就需要向量数据库，`index.md + 词法搜索 + 图遍历` 可以先工作。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "src/lib/ingest.ts：startIngest、写入流程、来源身份和 Frontmatter；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]