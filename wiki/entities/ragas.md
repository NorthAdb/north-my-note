---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [product]
aliases:
  - "vibrantlabsai/ragas"
  - "RAG Assessment"
generation_complete: true
---


# Ragas

## 描述
Ragas 是面向 [[concepts/rag|RAG]] 系统的开源评测工具，托管于 vibrantlabsai/ragas 仓库，采用 Apache-2.0 许可，截至 2026-08-11 星标约 15.3k。其设计目标是把"感觉好用"这类主观体验拆解为可重复测量的指标，评测体系覆盖检索层（Context Precision、Context Recall）与生成层（Faithfulness、Answer Correctness、Answer Relevance、Citation Correctness 等），并支持基于图结构的测试集生成。核心机制采用 LLM-as-a-Judge：先将回答拆分为独立主张，再逐一判断每个主张能否从检索上下文中推导出来，从而度量生成的忠实度与正确性。关键源码位于 src/ragas/evaluation.py、metrics/_faithfulness.py、metrics/_context_precision.py 与 testset/graph.py。源文档特别强调，Ragas 是评测工具而非"评测结果真理"——Chunk 切分、模型、Prompt 与评测数据的变化都会改变分数，因此不同项目之间的 Ragas 分数不能直接比较。在 [[concepts/context-engineering|Context Engineering]] 视角下，其检索层指标也为上下文构建策略提供了可量化的验证手段。

## 相关实体
源文档未显式建立 Ragas 与其他实体的直接关联；作为 RAG 评测工具，Ragas 可用于评估 Wiki 中已有的以下 RAG 生态项目：

- [[entities/graphrag|graphrag]] — 微软的图结构 RAG 系统
- [[entities/lightrag|lightrag]] — 轻量级 RAG 项目
- [[entities/ragflow|ragflow]] — 一站式 RAG 引擎
- [[entities/qdrant|qdrant]] — 向量数据库，常用于 RAG 检索层

## 相关概念
- [[concepts/rag|RAG]] — 检索增强生成，Ragas 的评测对象
- [[concepts/context-engineering|Context Engineering]] — 上下文工程；Ragas 的检索层指标可用于衡量检索上下文的质量

## 来源提及

- "Ragas 是一个评测工具，不是“评测结果真理”。它可以对回答拆分主张，再判断主张是否能从检索上下文推导出来，也提供 Context Precision 等检索相关指标。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "不同 Chunk、模型、Prompt 和评测数据会改变分数，不能把不同项目的 Ragas 分数直接比较。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "学习 Ragas 时要先理解指标假设，再运行脚本。LLM-as-a-Judge 也会错，必须搭配人工抽样、固定评测集和版本记录。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]