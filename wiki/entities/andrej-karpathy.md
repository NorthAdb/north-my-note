---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [entity  # MUST be exactly "entity" - do not change this value]
generation_complete: true
---


# Andrej Karpathy

## 描述

Andrej Karpathy 是 LLM Wiki 概念的最初提出者，其 gist《LLM Wiki idea file》被视为这一理念的源头。他在原文中明确指出该 gist 只是一个 idea file，不是完整应用，也不等同于官方的 karpathy/llm-wiki 仓库。他提出的核心差异是：传统 [[concepts/rag|RAG]] 每次提问都从原始文档中重新检索拼装，而 LLM Wiki 让 LLM 把原始资料增量编译为持久、互链、可维护的 Markdown Wiki。Karpathy 原文提到的约 100 个来源、数百页面的规模，被解读为人工可舒适维护的实践尺度而非系统硬限制。该想法直接启发了 [[entities/nashsu-llm-wiki|nashsu/llm_wiki]]、[[entities/claude-obsidian|claude-obsidian]] 等开源项目的实现。

## 相关实体

- [[entities/nashsu-llm-wiki|nashsu/llm_wiki]]
- [[entities/claude-obsidian|claude-obsidian]]

## 相关概念

- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/rag|RAG]]

## 来源提及

- "Andrej Karpathy 的 [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 是一个"idea file"，不是完整应用，也不是官方的 `karpathy/llm-wiki` 仓库。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "Karpathy 原文提到的约 100 个来源、数百页面，更像一个"人工可舒适维护"的实践尺度，不是系统硬限制。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]