---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "RAG-Sequence Model"
generation_complete: true
---


# RAG-Sequence

## 定义

RAG-Sequence 是 Lewis et al.（2020）在论文 *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* 中定义的 RAG 两种生成变体之一。它假设一条完整回答 y 使用同一组潜在文档 z 作为证据，生成概率近似为：

p(y|x) ≈ Σ_z p(z|x) p(y|x,z)

即：先按问题 x 检索文档（计算 p(z|x)），再基于这一组相同的文档生成整条回答（p(y|x,z)）。该机制把每个候选文档视为整条回答的主要证据，建模与解释相对直接。

## 关键特征

- **序列级证据共享**：一条完整回答绑定同一组检索文档，证据在生成过程中不随 token 切换。
- **整段边际化求和**：生成概率通过对候选文档 z 求和来近似，而非对每个生成位置单独求期望。
- **假设明确**：适合"单个候选文档即可支撑整条回答"的任务，如单文档问答与事实型内容生成。
- **与 RAG-Token 互补**：[[concepts/rag-token|RAG-Token]] 允许每个 token 使用不同文档，而 RAG-Sequence 在整个回答上统一证据，两者代表检索证据参与生成的不同粒度选择。
- **可解释性较好**：整条回答与证据文档一一对应，结果与检索证据之间的归属关系更易追溯。

## 应用

- **开放域问答（Open-domain QA）**：当答案通常出自某一篇检索文档时，用该变体生成与证据对齐的完整回答。
- **知识密集型内容生成**：摘要、报告等任务中，需要以检索到的整篇文档作为统一依据进行生成。
- **RAG 系统对比研究**：在与 RAG-Token 等变体的对照实验中，用于分析"检索证据如何参与生成"这一机制选择对生成质量与可解释性的影响。

## 相关概念

- [[concepts/rag|rag]]
- [[concepts/rag-token|RAG-Token]]
- [[concepts/naive-rag|naive-rag]]

## 相关实体

暂无相关实体。

## 来源提及

- "对一条完整回答使用同一组潜在文档：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "p(y | x) ≈ Σ p(z | x) p(y | x, z)" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "适合把一个候选文档看作整条回答的主要证据。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]