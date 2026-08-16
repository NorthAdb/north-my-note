---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "切块"
  - "分块"
  - "Chunk 切分"
generation_complete: true
---


# Chunking

## 定义
Chunking（切块/分块）是 RAG（检索增强生成）系统中将解析后的文档切分为可独立检索的最小知识单元的过程。该过程位于文档解析与索引之间，直接决定检索的粒度与上下文完整性。切分过小会丢失上下文；切分过大则会引入噪声、降低检索精度并增加 token 成本。Chunk 同时携带标题、来源、页码、版本等 metadata，是检索、重排、引用与溯源的基础单位。

## 关键特征
- 粒度与上下文的权衡：chunk 大小直接决定检索粒度，过小丢失上下文，过大引入噪声、降低精度并增加 token 成本。
- 方法多样：涵盖固定窗口、递归切割、结构切割、语义切割、代码感知切割、父子块、Contextual Chunk、Agent/LLM 切割等多种切分策略。
- 参数需实验验证：不应迷信"256 token""512 token"等固定答案，而应固定模型、语料与评测集，比较不同 chunk size 与 overlap 对 Recall@K、Faithfulness 等指标的影响。
- 携带丰富 metadata：每个 chunk 附带标题、来源、页码、版本等元数据，支持检索结果的引用与溯源。
- 架构基础组件：贯穿 Hybrid RAG、Hierarchical RAG、Context Engineering 等方案，在 RAG 开源项目中均有对应实现。

## 应用
- 文档问答与检索：将长文档切分为可独立检索的知识单元，供向量检索与混合检索使用。
- 父子块策略：父块提供完整上下文，子块提供精确匹配，兼顾检索精度与上下文完整性。
- 层级检索：在 Hierarchical RAG 中先检索摘要层、再定位块层，降低噪声干扰。
- 代码库检索：代码感知切割按函数、类等语法边界切分，避免破坏代码结构。
- 开源实现：[[entities/ragflow|ragflow]]、[[entities/llamaindex|LlamaIndex]]、[[entities/haystack|haystack]] 等项目中均提供各自的切分器或切分模板。

## 相关概念
- [[concepts/rag|rag]]：Chunking 是 RAG 流程中连接文档解析与索引的关键环节。
- [[concepts/embedding|embedding]]：切分后的 chunk 是文本向量化的基本单位。
- [[concepts/reranker|Reranker]]：chunk 粒度直接影响重排阶段的候选质量与最终精度。
- [[concepts/agentic-rag|Agentic RAG]]：Agent/LLM 切割可结合模型决策动态确定切分边界。
- [[concepts/llm-wiki|llm-wiki]]：LLM Wiki 以 chunk 为知识写入与链接的最小单元，chunk 质量决定知识库的检索效果。

## 相关实体
- [[entities/ragflow|ragflow]]：提供深度文档理解能力与多种 chunk 切分模板。
- [[entities/llamaindex|LlamaIndex]]：提供 NodeParser 体系与多种文本切分器（TextSplitter），支持递归、语义等切分方式。

## 来源提及

- "Chunk 是系统可以独立召回的最小知识单元。切得太小会失去上下文，切得太大会引入噪声、降低精度并增加 token 成本。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "实践中不要迷信“256 token”或“512 token”这样的固定答案。应该固定模型、语料和评测集，比较：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| 父子块 | 小子块负责召回，大父块负责补充上下文 | 精度与完整性兼顾 | 需要维护 parent-child 关系 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]