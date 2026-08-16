---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "Microsoft GraphRAG"
  - "microsoft/graphrag"
  - "Graph RAG"
generation_complete: true
---


# GraphRAG

## 描述
GraphRAG 是微软开源的图增强检索生成框架，仓库为 `microsoft/graphrag`，采用 MIT 许可证，目前约 35.4k 星。其索引流程先将文档拆分为 Text Units，再抽取实体、关系与声明，经过实体合并、社区检测后生成社区报告，最终支撑 [[concepts/rag|RAG]] 的 Local Search、Global Search 与 DRIFT Search 等查询模式。来源文件强调 GraphRAG 并不自动优于传统 RAG：简单事实查找通常不值得支付建图成本，它的价值集中在跨文档关系与整个语料的全局归纳。作为图谱检索部分的重要参考，GraphRAG 与 [[entities/lightrag|LightRAG]] 的轻量图增强方案形成对比，也被纳入 [[concepts/llm-wiki|LLM Wiki]] 模式的参考体系。

## 相关实体
- [[entities/lightrag|LightRAG]]：轻量级图增强 RAG 方案，与 GraphRAG 在索引成本与全局检索能力上形成对比。
- [[entities/nashsu-llm-wiki|nashsu/llm_wiki]]：LLM Wiki 模式的参考实现，GraphRAG 是其图谱检索设计的重要参考。
- [[entities/ragflow|RAGFlow]]：同为开源 RAG 工程化项目，可与 GraphRAG 相互参照。

## 相关概念
- [[concepts/rag|RAG]]：GraphRAG 所属的检索增强生成基础范式。
- [[concepts/llm-wiki|LLM Wiki]]：以知识库组织 LLM 信息的模式，GraphRAG 为其图谱检索部分提供参考。
- [[concepts/agent-memory|Agent Memory]]：将长期记忆外部化到知识图谱的 agent 记忆方向，GraphRAG 可作为该方向的实现参照。

## 来源提及

- "GraphRAG 不会自动优于传统 RAG。简单事实查找通常不值得支付建图成本；它主要解决“跨文档关系”和“整个语料的全局归纳”。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "适合重点研究“为什么全局问题不能只靠 Top-K Chunk，以及预计算社区摘要如何换取查询时的全局视角”。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "**代价**：实体抽取、关系抽取、摘要和社区报告都可能调用 LLM；新增文档还可能影响实体、社区和摘要。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]