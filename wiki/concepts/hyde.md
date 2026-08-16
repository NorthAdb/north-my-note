---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "Hypothetical Document Embeddings"
  - "假设文档嵌入"
generation_complete: true
---


# HyDE

## 定义
HyDE（Hypothetical Document Embeddings，假设文档嵌入）是一种查询扩展技术。其核心做法是：在检索前先让 LLM 根据用户问题生成一个假设性回答文档，再利用该文档的向量表示去检索知识库，从而使查询向量更接近文档空间（而非问题空间）的表达方式。在 RAG 链路中，HyDE 属于查询转换层，与 Query Rewrite、Multi-Query、Query Decomposition 等查询预处理手段并列。

## 关键特征
- **查询扩展**：通过 LLM 生成假设文档来扩展原始查询，而非直接改写查询词。
- **文档空间对齐**：使用假设文档的向量替代原始查询向量进行检索，缩小查询与文档之间的表示鸿沟。
- **属于查询转换层**：与 Query Rewrite、Multi-Query、Query Decomposition 同属查询预处理手段。
- **典型代价——偏差风险**：模型生成的假设文档可能与真实文档空间存在偏差，从而影响检索质量。
- **典型代价——额外调用成本**：每次查询都需要额外的 LLM 调用以生成假设文档。
- **归属于 RAG 在线查询平面**：作为查询预处理环节，与多路召回、融合和重排链路配合使用。

## 应用
- 作为 RAG 在线查询平面的查询预处理环节，与多路召回、融合（fusion）和重排（reranking）链路配合使用。
- 适用于用户问句与知识库文档表达方式差异较大的场景，例如短提问对应长文档、口语化查询对应书面化文档。
- 可与其他查询转换方法（如 Query Rewrite、Multi-Query、Query Decomposition）组合使用，以进一步提升召回质量。

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/agentic-rag|agentic-rag]]
- [[concepts/embedding|embedding]]
- [[concepts/hybrid-search|hybrid-search]]
- [[concepts/reranker|reranker]]

## 相关实体
暂无关联实体。

## 来源提及

- "4. **HyDE**：先生成假设性回答文档，再用该文档向量检索；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **HyDE** | 如何用假设答案接近文档空间 | 查询扩展 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]