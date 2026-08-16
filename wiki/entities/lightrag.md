---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "HKUDS/LightRAG"
generation_complete: true
---


# LightRAG

## 描述
LightRAG 是香港大学数据科学实验室（HKUDS）开源的轻量级图增强检索增强生成（RAG）项目，代码托管于 GitHub 仓库 HKUDS/LightRAG，采用 MIT 许可证，目前约 38.7k 星。它在索引阶段保留实体、关系和向量索引，但通常不预先构建完整的社区摘要，而是在查询时通过高层与低层关键词进行双层检索，并提供 Local、Global、Hybrid、Naive 等查询模式。相比 [[entities/graphrag|graphrag]]，LightRAG 的增量更新更简单且索引成本更低；相比传统 [[concepts/rag|RAG]]，它保留了关系检索能力。源文件将其定位为在传统 RAG 与完整 GraphRAG 之间取工程折中的代表项目，并与 [[entities/nashsullm_wiki|nashsullm_wiki]] 等 [[concepts/llm-wiki|LLM Wiki]] 型项目直接相关。

## 相关实体
- [[entities/graphrag|graphrag]]：完整图增强 RAG 方案，LightRAG 通过简化社区摘要与双层关键词检索在索引成本与查询能力之间折中
- [[entities/nashsullm_wiki|nashsullm_wiki]]：与其关联的 Wiki 型 RAG 项目，共享 RAG 与图检索的应用场景

## 相关概念
- [[concepts/rag|RAG]]：检索增强生成，LightRAG 所基于的核心范式
- [[concepts/llm-wiki|LLM Wiki]]：以大语言模型构建和维护 Wiki 的方法，LightRAG 为该类应用提供关系检索支撑

## 来源提及

- "LightRAG 是一种更轻量的图增强 RAG 思路：保留实体、关系和向量索引，通常不预先做完整社区摘要，而是在查询时使用高层/低层关键词进行双层检索。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "相对 GraphRAG，它更容易增量更新、索引成本更低；相对传统 RAG，它保留了关系检索能力。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "LightRAG 的关键思想是保留实体/关系结构，但减少重量级社区预计算，使用高层与低层关键词结合图和向量检索。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]