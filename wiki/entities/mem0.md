---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "mem0ai/mem0"
  - "mem0"
generation_complete: true
---


# Mem0

## 描述

Mem0（mem0ai/mem0）是一个面向 AI Agent 的开源长期记忆项目，GitHub 星标约 63.0k（2026-08-11 快照），采用 Apache-2.0 许可证。与 [[entities/nashsullm_wiki|nashsullm_wiki]] 生成可浏览的 Markdown 概念页不同，Mem0 从对话与经历中抽取原子记忆项（事实、偏好、事件），并记录身份范围、历史和元数据。其核心操作包括 Add/Search Memory 的记忆新增、检索、更新与删除，支持向量持久化、SQLite 历史存储与 Graph Memory。源码强调 Mem0「选取并更新长期记忆，而不是保存整段对话」，与 LLM Wiki 的共同点是「写回」（write-back），差别在于 Mem0 以原子记忆、偏好和事件为中心，而非人可浏览的概念页。关键实现路径包括 mem0/memory/main.py 的记忆抽取、向量持久化、实体与检索逻辑，是研究 Agent Memory 写回、记忆分层和混合检索的重要开源实现。

## 相关实体

- [[entities/nashsullm_wiki|nashsullm_wiki]]

## 相关概念

- [[concepts/agent-memory|Agent Memory]]
- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/rag|RAG]]
- [[concepts/knowledge-graph|Knowledge Graph]]

## 来源提及

- "Mem0 的持久对象是经过提取的记忆项，带有身份范围、历史和元数据，而不是完整对话 Transcript。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它与 LLM Wiki 的共同点是“写回”，差别是 Mem0 以原子记忆、偏好和事件为中心，不以人可浏览的 Markdown 概念页为中心。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "选取并更新长期记忆，而不是保存整段对话" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]