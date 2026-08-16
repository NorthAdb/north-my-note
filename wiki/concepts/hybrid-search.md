---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "混合检索"
  - "Hybrid RAG"
generation_complete: true
---


# Hybrid Search

## 定义

Hybrid Search（混合检索）是一种将多种异构检索方式融合的召回策略，通常并行执行 dense 向量检索（[[concepts/embedding|embedding]]）、BM25/sparse 词法检索（[[concepts/bm25|bm25]]）以及 metadata/ACL filter、图检索等多路检索，再对结果做融合。文中给出的生产级 RAG 典型链路为：`dense ANN + BM25/sparse + metadata/ACL filter → hybrid fusion → rerank`。

该策略的出发点在于两类检索器的能力互补：纯向量检索擅长语义相近的匹配，但对精确编号不敏感；BM25 擅长精确词、型号、错误码的匹配，却不理解语义。二者融合可同时兼顾语义理解与精确命中，因此被视为生产级 RAG 的常见形态。

## 关键特征

- 多路召回并行：dense ANN、BM25/sparse、metadata/ACL filter 等检索路线同时执行，互不依赖。
- 融合只依赖排名：常用 [[concepts/rrf|RRF]]（Reciprocal Rank Fusion，倒数排名融合），只使用各路结果的排名而非原始分数，规避不同检索器分数尺度不可比的问题。
- 检索器互补：向量检索提供语义召回，词法检索提供精确词/型号/错误码匹配，metadata/ACL 过滤提供属性约束与权限控制。
- 融合后接重排：hybrid fusion 的输出通常送入 [[concepts/reranker|reranker]] 精排，构成完整生产链路。
- "Hybrid" 语义需澄清：该词存在历史语义（参数化记忆 + 非参数化记忆的混合）与现代语义（多路检索融合）两种用法，撰写设计文档时必须标明采用的是哪一种。

## 应用

- 企业级 RAG 问答：同时应对语义性问题与精确性查询（如产品型号、错误码、单据编号），是生产级 RAG 的常见选型。
- 企业搜索与知识库检索：融合向量召回与关键词召回，提升长尾精确查询的命中率。
- 权限敏感场景：配合 metadata/ACL filter，在召回阶段即执行数据权限隔离。
- 混合图检索场景：将图检索与向量检索融合，兼顾结构化关系与语义相似度，例如结合 [[entities/graphrag|graphrag]] 的 RAG 方案。
- 工程落地：[[entities/qdrant|qdrant]] 提供 dense + sparse 混合索引能力，[[entities/ragflow|ragflow]] 在生产链路中采用混合检索与重排。

## 相关概念

- [[concepts/rag|rag]]
- [[concepts/bm25|bm25]]
- [[concepts/embedding|embedding]]
- [[concepts/reranker|reranker]]
- [[concepts/knowledge-graph|knowledge-graph]]
- [[concepts/rrf|RRF]]

## 相关实体

- [[entities/qdrant|qdrant]]
- [[entities/ragflow|ragflow]]
- [[concepts/graphrag|graphrag]]

## 来源提及

- "生产系统经常采用：dense ANN + BM25/sparse + metadata/ACL filter → hybrid fusion → rerank" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "现代应用中的 hybrid：dense、BM25、metadata、graph、symbolic 等多种检索方式融合。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **Hybrid RAG** | dense + sparse/BM25，或再加 metadata/graph | 既有语义问题又有精确词 | 需要融合与调参 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]