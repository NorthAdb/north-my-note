---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "向量嵌入"
  - "文本嵌入"
generation_complete: true
---


# Embedding

## 定义
Embedding（向量嵌入）是将文本映射为稠密向量的核心技术，形式化表示为：文本 x → f(x) ∈ Rᵈ，即把任意文本编码为 d 维实数向量。语义相近的文本在向量空间中彼此靠近，因此它构成 RAG 体系中 dense 检索（语义检索）的基石。查询向量与文档向量之间可以使用余弦相似度（cosine）、点积（dot product）或欧氏距离（Euclidean distance）进行比较，以衡量语义相似程度。

## 关键特征
- **稠密向量映射**：将离散文本转换为固定维度的稠密实值向量，维度 d 由模型架构决定。
- **语义连续空间**：语义越相近的文本，其向量在空间中的距离越近或夹角越小。
- **编码器兼容性约束**：查询向量与文档向量必须由兼容的编码器生成，混用不兼容模型会导致检索结果失效。
- **领域匹配要求**：中英文、代码、法律、医学等不同领域的语料需要匹配的 embedding 模型，领域不匹配会显著降低检索效果。
- **工程一致性约束**：向量维度与距离度量必须保持一致，否则无法进行有效比较。
- **版本可追溯性**：embedding 模型升级后，旧向量通常需要重建；工程上应保留 embedding model/version 信息，以便追溯与重建。
- **语义边界**：Embedding 表示"相似"，不表示"真实""最新"或"用户有权阅读"——高相似度文本仍可能过时、错误、被污染或无权限访问。

## 应用
- **语义检索（dense retrieval）**：在 RAG 流水线中，文档被切块后经 embedding 编码存入向量数据库；查询时对查询文本编码并与文档向量比较，返回语义最接近的片段。
- **混合检索（Hybrid Search）**：与 BM25 等关键词检索互补，兼顾精确关键词匹配与语义泛化匹配，提升召回质量与多样性。
- **向量数据库检索**：作为 Qdrant 等向量数据库的核心检索输入，支撑大规模向量的近似最近邻搜索。
- **RAG 框架基础设施**：LlamaIndex、RAGFlow、Open WebUI 等 RAG 工具链均以 embedding 流水线作为从 Naive RAG 到 Advanced RAG 各变体共同依赖的基础设施。

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/bm25|BM25]]
- [[concepts/reranker|Reranker]]
- [[concepts/vector-database|Vector Database]]

## 相关实体
- [[entities/qdrant|qdrant]]
- [[entities/llamaindex|LlamaIndex]]
- [[entities/ragflow|ragflow]]
- [[entities/open-webui|open-webui]]

## 来源提及

- "Embedding 模型把文本映射为向量：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "查询向量与文档向量之间可以用 cosine、dot product 或 Euclidean distance 比较。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "> Embedding 表示“相似”，不表示“真实”、不表示“最新”、不表示“用户有权阅读”。高相似度文本也可能过时、错误、被污染或无权访问。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]