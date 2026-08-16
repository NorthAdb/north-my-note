---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "朴素 RAG"
  - "Basic RAG"
generation_complete: true
---


# Naive RAG

## 定义
Naive RAG（朴素 RAG）是检索增强生成（RAG）最基础的实现形态。其处理流程为：用户问题 → 问题 Embedding → 向量库 Top-K 检索 → 将检索到的文本块拼接进 Prompt → LLM 生成回答。整个流程不包含重排、混合检索或多轮迭代等增强环节，来源将其定位为适合做基线的参考实现，也是理解更复杂 RAG 工程改进的起点。

## 关键特征
- 流程简单直接：仅依靠向量相似度完成检索，无查询改写、重排、混合检索等额外优化环节
- 基线定位：作为参考实现，用于对比和衡量更复杂 RAG 方案的效果增益
- 在复杂场景中存在典型缺陷：
  - 问题措辞与文档措辞不一致时，向量检索难以召回相关内容
  - 产品编号等精确词容易被向量相似度忽略，导致精确匹配失效
  - Top-K 检索结果可能重复且包含噪声，影响生成质量
  - 单次检索无法完成多跳连接，难以回答需要跨文档推理的问题
  - 文档块缺少来源信息，难以支持引用与溯源
  - 检索到的文本可能携带 Prompt Injection，带来安全风险

## 应用
- 作为渐进式生产路线的第一步，先以 Naive RAG 基线跑通 RAG 基础流程
- 作为评测基线，对比 Advanced RAG（重排、查询改写）、Hybrid RAG（混合检索）、GraphRAG（知识图谱）等改进方案的实际增益
- 作为学习入口，帮助理解 RAG 的基本工作原理与后续工程改进的动机

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/chunking|chunking]]
- [[concepts/embedding|embedding]]
- [[concepts/hybrid-search|hybrid-search]]
- [[concepts/agentic-rag|agentic-rag]]
- [[concepts/reranker|reranker]]

## 相关实体
- [[entities/qdrant|qdrant]]
- [[entities/graphrag|graphrag]]
- [[entities/llamaindex|LlamaIndex]]

## 来源提及

- "这就是 **Naive RAG（朴素 RAG）**。它适合做基线，但在复杂场景中通常会遇到：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **Naive RAG** | 向量 Top-K → Prompt → LLM | 做基线、简单问答 | 召回噪声、多跳弱 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "1. Naive RAG 基线
   文档 → Chunk → Embedding → Vector DB → 回答" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]