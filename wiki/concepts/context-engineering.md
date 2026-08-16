---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "上下文工程"
  - "上下文编排"
generation_complete: true
---


# Context Engineering

## 定义
Context Engineering（上下文工程）指在候选资料被召回（Retrieval）之后、进入 Prompt 之前的编排层，负责决定哪些资料真正进入上下文、是否需要引入父块/摘要/邻居/历史记忆、资料按什么顺序放置、如何标注来源与版本、如何压缩过长内容，以及如何隔离资料内容与系统指令。该概念明确区分了 RAG 与 Context Engineering 的边界：RAG 只负责"找到一些候选资料"，是 Context Engineering 的一个输入渠道而非全部；后者才是决定最终 Prompt 上下文组装质量的关键环节。

## 关键特征
- 定位清晰：位于"召回后、Prompt 前"的编排层，是连接检索结果与最终 Prompt 的枢纽。
- RAG 仅作为输入渠道：RAG 负责候选资料的召回，Context Engineering 负责对候选资料做最终筛选、组织与编排。
- 决策维度丰富：涵盖资料取舍、父块/摘要/邻居/历史记忆的引入、顺序排列、来源与版本标注、内容压缩、与系统指令隔离。
- 环节衔接紧密：与 Chunking、Reranking、Provenance（溯源）等环节衔接，构成完整的上下文组装链路。
- 放大生成质量：在相同检索结果下，不同的编排策略会显著影响最终生成效果，是理解 Advanced RAG 与 LLM Wiki 上下文组装的关键概念。

## 应用
- Advanced RAG 系统：在 Reranker 之后对候选资料做最终取舍与排序，构建精炼且信息完整的 Prompt 上下文。
- LLM Wiki 上下文组装：将知识库召回的文本片段整理为带来源、带版本、结构清晰的上下文块，保证可追溯性。
- 长上下文压缩：对超出窗口限制的候选资料进行摘要或关键信息抽取，控制 token 开销。
- 多源检索融合：统一编排向量检索、BM25、知识图谱等不同渠道的召回结果。
- 指令隔离与安全：将系统指令、历史记忆与资料内容分层放置，避免资料内容干扰系统行为。

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/chunking|chunking]]
- [[concepts/reranker|reranker]]
- [[concepts/hybrid-search|Hybrid Search]]
- [[concepts/agentic-rag|agentic-rag]]
- [[concepts/llm-wiki|llm-wiki]]

## 相关实体
该概念属于通用方法层，不直接绑定特定实体；其思想可普遍应用于各类 RAG 与知识库系统。

## 来源提及

- "RAG 只负责“找到一些候选资料”。Context Engineering 负责决定：哪些资料真的进入上下文；是否需要父块、摘要、邻居或历史记忆；资料按什么顺序放置；如何设置来源、版本和可信度；如何压缩过长内容；如何隔离资料中的指令和系统指令。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "所以 RAG 是 Context Engineering 的一个输入渠道，而不是全部。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]