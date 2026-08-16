---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [concept  # MUST be exactly "concept" - do not change this value]
generation_complete: true
---


# RRF

## 定义
RRF（Reciprocal Rank Fusion，倒数排名融合）是一种只依赖排名、不依赖分数尺度的多路检索结果融合方法。其计算公式为 RRF(d) = Σᵢ 1/(k + rankᵢ(d))，其中 rankᵢ(d) 是文档 d 在第 i 条检索路由中的排名，k 为平滑常数。由于不同检索路由（dense 检索、BM25、图检索等）的分数尺度通常不同，直接把分数相加并不稳定；RRF 将各路排名映射为可叠加的倒数分值，使在多个路由中都排名靠前的文档获得更高的综合排名。

## 关键特征
- 仅依赖排名：不要求各路由的分数在同一尺度上，避免了跨路由分数直接相加的不稳定问题。
- 倒数分值叠加：每个路由对文档的贡献为 1/(k + rank)，排名越靠前贡献越大，且多路贡献可累积。
- 多路由共识加权：同一文档在多条路由中均排名靠前时综合排名更高；仅单路高分而其他路由未召回的文档会被降权。
- 超参数需调优：实际使用中的 k 值、路由权重和候选数量仍需在验证集上调参。
- 常作为生产 RAG 的 hybrid fusion 层：与 MMR 去重、Reranker 精排配合，用于构建最终进入 Prompt 的上下文。

## 应用
- 生产 RAG 中的 hybrid fusion：将 dense 向量检索、BM25 稀疏检索、图检索（知识图谱）等多路召回结果融合为统一排序列表。
- 两阶段排序管线：先以 RRF 融合各路召回得到 Top-K 候选，再交由 Reranker 精排，最终进入 Prompt 上下文。
- 多路证据检索：用于问答、知识库检索等需要综合多来源排序结果的系统，常见于 Qdrant、Haystack、LlamaIndex 等框架的混合检索实现。

## 相关概念
- [[concepts/hybrid-search|hybrid-search]] — RRF 是混合检索中最常用的融合策略之一
- [[concepts/reranker|reranker]] — RRF 之后的精排阶段，二者常配合使用
- [[concepts/bm25|bm25]] — 典型的稀疏检索路由，与 dense 检索共同构成 RRF 的输入
- [[concepts/embedding|embedding]] — 构成 dense 检索路由的向量表示基础
- [[concepts/rag|rag]] — RRF 融合常用于 RAG 的召回管线
- [[concepts/context-engineering|context-engineering]] — 融合与精排后的候选用于构建 Prompt 上下文

## 相关实体
- [[entities/qdrant|qdrant]] — 向量数据库，其混合检索实现中支持 RRF 融合
- [[entities/haystack|haystack]] — 提供多路检索与融合组件的开源框架
- [[entities/llamaindex|LlamaIndex]] — 包含 RRF 等多路检索融合实现的数据框架

## 来源提及

- "不同检索路由的分数尺度通常不同，直接把分数相加并不稳。RRF 只使用排名：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "RRF(d) = Σᵢ 1 / (k + rankᵢ(d))" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "同一个文档在多条路由都排得靠前，综合排名通常更高。实际的 k、权重和候选数量仍需要在验证集上调参。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **RRF / MMR** | 如何融合排名、减少重复 | 结果后处理 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]