---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "onyx-dot-app/onyx"
  - "Onyx 企业搜索"
generation_complete: true
---


# Onyx

## 描述

Onyx 是一个面向企业和团队知识系统的开源搜索与 RAG 项目（onyx-dot-app/onyx），截至 2026-08-11 在 GitHub 上拥有约 31.5k 星标，采用 MIT + Enterprise 混合许可证。其核心能力围绕连接器、权限（ACL）、增量索引、混合搜索、项目和 Persona 设计，关键实现包括 indexing_pipeline.py（过滤、Hook、切块、可选上下文摘要、Embedding 与写入）、search_runner.py（语义、关键词、混合和联邦检索）以及 context/search/pipeline.py（权限、Document Set、Persona、Project 与 Chunk 合并），文档索引与 ACL 过滤基于 OpenSearch。来源笔记强调，企业级 RAG 的难点往往不是"换哪个 Embedding"，而是数据同步、权限、删除、增量更新、连接器失败和可审计性。在多用户企业知识库的选型建议中，Onyx 与 [[entities/ragflow|ragflow]]、Open WebUI 被列为首选方案。

## 相关实体

- [[entities/ragflow|ragflow]] — 同为开源企业级 RAG 知识库项目，在多用户企业知识库选型建议中与 Onyx 并列首选。

## 相关概念

- [[concepts/rag|RAG]] — Onyx 的核心应用形态，面向企业知识库的检索增强生成。
- [[concepts/hybrid-search|Hybrid Search]] — Onyx 的语义、关键词与联邦混合检索能力。
- [[concepts/provenance|Provenance]] — 与可审计性相关的引用溯源设计。
- [[concepts/context-engineering|Context Engineering]] — 项目（Project）与 Persona 所体现的上下文工程思想。

## 来源提及

- "Onyx 面向企业和团队知识系统，重点是连接器、权限、增量索引、混合搜索、项目和 Persona。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "Onyx 的教训是：企业级 RAG 的难点往往不是“换哪个 Embedding”，而是数据同步、权限、删除、增量更新、连接器失败和可审计性。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| [Onyx](https://github.com/onyx-dot-app/onyx) | 31.5k | MIT + Enterprise 混合 | 企业搜索/RAG | 连接器、ACL、增量索引、联邦检索 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]