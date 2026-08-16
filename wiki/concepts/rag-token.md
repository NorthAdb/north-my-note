---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "Token-level RAG"
  - "Token 级 RAG"
generation_complete: true
---


# RAG-Token

## 定义
RAG-Token 是原始 RAG 论文（Lewis et al., 2020）定义的两种标准生成模式之一。在该模式下，模型生成回答中的每一个 token 时，都会独立地从检索文档分布 p(z|x) 中采样一个潜在文档 z，并以该文档为条件生成当前 token yᵢ。整个回答的生成概率形式为 p(y|x) ≈ Πᵢ Σ p(z|x) p(yᵢ|x,z,y₁:ᵢ₋₁)，即在每个 token 位置上对潜在文档求和（边缘化）。

## 关键特征
- 逐 token 决策：每个 token 的生成都可能由不同的潜在文档支撑，而非整个回答共享同一篇文档
- 概率形式为逐 token 边缘化：p(y|x) ≈ Πᵢ Σ p(z|x) p(yᵢ|x,z,y₁:ᵢ₋₁)
- 表达能力更灵活：回答中不同位置的信息可分别来源于不同文档，适合多文档综合生成
- 推理与训练解释更复杂：需要为每个 token 分别维护"文档-生成"的对齐关系，计算与调试成本更高
- 与 RAG-Sequence 并列为 2020 年原始 RAG 论文的两种标准生成变体

## 应用
- 开放域问答中需要从多篇文档各取部分信息、拼合为完整答案的场景
- 作为理解 RAG 生成模式演进的基准：后续 Agentic RAG 等方法在此基础之上引入循环检索与动态规划
- 用于对比研究：评估"逐 token 采样文档"与"整段序列使用同一文档"在答案质量上的差异
- 注意边界：现代工程中的"RAG"已扩展出查询改写、混合检索、重排、引用与 Agent 循环等能力，不应把该 2020 年论文变体直接等同于当前产品效果

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/rag-sequence|RAG-Sequence]]
- [[concepts/agentic-rag|agentic-rag]]
- [[concepts/naive-rag|naive-rag]]
- [[concepts/hybrid-search|hybrid-search]]
- [[concepts/reranker|reranker]]

## 相关实体
- 无相关实体。

## 来源提及

- "生成不同 token 时，允许不同文档发挥作用：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "p(y | x) ≈ Πᵢ Σ p(z | x) p(yᵢ | x, z, y₁:ᵢ₋₁)" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "表达能力更灵活，但推理和训练解释更复杂。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]