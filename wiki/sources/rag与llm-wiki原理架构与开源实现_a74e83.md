---
type: source
created: 2026-08-11
updated: 2026-08-11
source_file: "[[领域/RAG与LLM Wiki：原理、架构与开源实现.md]]"
tags: [note]
aliases: ["RAG与LLM Wiki：原理、架构与开源实现", "RAG 与 LLM Wiki：从检索增强到知识编译"]
contentHash: 9ce6-6130ad2b
generation_complete: true
---

# RAG 与 LLM Wiki：从检索增强到知识编译 - Summary

## 来源
- 原始文件：[[领域/RAG与LLM Wiki：原理、架构与开源实现.md|RAG 与 LLM Wiki：原理、架构与开源实现]]
- 摄取日期：2026-08-11

## 核心内容

本篇笔记系统梳理了 [[concepts/rag|RAG（检索增强生成）]] 与 [[concepts/llm-wiki|LLM Wiki]] 两大主题。前半部分从 2020 年 Lewis 等人的原始 RAG 论文出发，介绍了将外部文档索引作为非参数记忆与参数化生成器结合的原理，并拆解了生产级 RAG 工程链路：文档解析、[[concepts/chunking|分块]]、[[concepts/embedding|Embedding]]、[[concepts/vector-database|向量数据库]]、查询改写、多路召回、[[concepts/reranker|重排]]、上下文组装与引用溯源，进而归纳了 [[concepts/naive-rag|Naive RAG]]、[[concepts/hybrid-search|Hybrid RAG]]、[[concepts/agentic-rag|Agentic RAG]]、[[entities/graphrag|GraphRAG]]、[[entities/lightrag|LightRAG]] 等变体、失败模式与基于 [[entities/ragas|Ragas]] 的评测方法。后半部分聚焦 [[entities/andrej-karpathy|Andrej Karpathy]] 提出的 LLM Wiki 理念：将原始资料增量编译为持久、互链、可维护的 Markdown 知识层，并介绍 Raw Sources、Wiki、Schema 三层架构与 Ingest、Query、Lint 三操作。最后分析了 [[entities/nashsullm_wiki|nashsu/llm_wiki]]、[[entities/claude-obsidian|claude-obsidian]]、[[entities/