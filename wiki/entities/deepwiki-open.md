---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "AsyncFuncAI/deepwiki-open"
  - "deepwiki-open"
generation_complete: true
---


# DeepWiki-Open

## 描述

DeepWiki-Open 是一个 MIT 许可的开源 [[concepts/code-wiki|Code Wiki]] 项目，截至 2026-08-11 在 GitHub 上获得约 17.6k 星，目标是把代码库编译成可浏览的结构化 Wiki 文档。其典型流程包括克隆仓库、递归读取文件、切块并计算 [[concepts/embedding|Embedding]]、用 FAISS 检索相关代码、让 LLM 规划 Wiki 结构并生成带文件/行号引用的分页文档。该项目明确说明了 [[concepts/rag|RAG]] 与 Wiki 的分工：RAG 负责找相关文件，Wiki 负责把局部检索结果提升为可浏览的结构文档。源码中的 `api/rag/pipeline.py`、`api/rag/rag.py`、`api/services/wiki/structure.py` 等路径展示了代码感知 [[concepts/chunking|Chunking]]、结构规划、页面生成与引用后处理的具体实现。它是把 Code Wiki 概念落到工程实现的直接样例，与 [[entities/nashsullm_wiki|nashsullm_wiki]]、[[entities/claude-obsidian|claude-obsidian]] 同属 LLM 驱动的知识库文档化工具。

## 相关实体

- [[entities/nashsullm_wiki|nashsullm_wiki]]
- [[entities/claude-obsidian|claude-obsidian]]

## 相关概念

- [[concepts/code-wiki|Code Wiki]]
- [[concepts/rag|RAG]]
- [[concepts/chunking|Chunking]]
- [[concepts/embedding|Embedding]]

## 来源提及

- "它面向代码库，不是个人研究 Wiki。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它的启示是：**RAG 负责找相关文件，Wiki 负责把局部检索结果提升为可浏览的结构文档。**" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "api/rag/rag.py：FAISS 检索；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]