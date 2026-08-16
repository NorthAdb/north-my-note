---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "AgriciDaniel/claude-obsidian"
  - "Claude Obsidian"
generation_complete: true
---


# claude-obsidian

## 描述

claude-obsidian 是 AgriciDaniel 维护的开源项目（MIT 许可，截至 2026-08-11 星标约 10.7k），将 [[entities/andrej-karpathy|andrej-karpathy]] 提出的 [[concepts/llm-wiki|LLM Wiki]] 想法落地为本地优先、Agent Skills 兼容的 Obsidian 知识系统。其核心设计是：原始资料保留为不可变、带内容 hash 的副本；重要主张记录来源、支持、冲突、可信度和 review 状态；多个平行 Agent 只返回草稿，由一个编排器统一应用，保证一次写入是可恢复事务而不是互相覆盖；在没有远程模型或插件时自动降级为确定性 [[concepts/bm25|BM25]] 检索。实现入口包括 skills/wiki-ingest、wiki-query、wiki-lint、wiki-retrieve 四个 Skill 与 scripts/claude-obsidian.py。该项目最重要的启示是：LLM Wiki 的工程难点不只是让模型写页面，还在于避免写错、如何审阅、如何回滚以及并行 Agent 的写冲突处理。

## 相关实体

- [[entities/andrej-karpathy|andrej-karpathy]] — LLM Wiki 想法的提出者
- [[entities/nashsullm_wiki|nashsullm_wiki]] — LLM Wiki 的开源实现项目之一，与 claude-obsidian 直接相关

## 相关概念

- [[concepts/llm-wiki|LLM Wiki]] — 由 LLM 编写 wiki 页面的核心理念
- [[concepts/bm25|BM25]] — 无远程模型或插件时使用的确定性检索算法

## 来源提及

- "它把 Karpathy 的模式扩展为一个本地优先、Agent Skills 兼容的 Obsidian 知识系统，强调：原始资料保留为不可变、有内容 hash 的副本；重要主张有来源、支持、冲突、可信度和 review 状态；平行 Agent 只返回草稿，由一个编排器统一应用；一次写入是可恢复事务，而不是多个 Agent 同时修改 Vault；没有远程模型或插件时，自动降级到确定性的 BM25。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它展示了 LLM Wiki 最容易被忽略的工程层：**不要只研究“怎么让模型写页面”，还要研究如何避免写错、如何审阅、如何回滚以及如何让并行 Agent 不互相覆盖。**" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]