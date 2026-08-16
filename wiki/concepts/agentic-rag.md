---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [concept  # MUST be exactly "concept" - do not change this value]
aliases:
  - "Agent 式 RAG"
  - "智能体 RAG"
generation_complete: true
---


# Agentic RAG

## 定义
Agentic RAG（Agent 式 RAG）是检索增强生成（RAG）的一种进阶变体。它将一次性的"检索 → 生成"函数调用扩展为一个决策循环：Agent 先识别目标与约束，选择检索工具并执行搜索，随后判断证据是否充分；若不足，则改写问题、拆分题目或更换索引后继续搜索，直到满足证据条件或达到上限。

## 关键特征
- 决策循环替代单次调用：从"检索 → 生成"的线性流程升级为"规划 → 检索 → 评估 → 重新规划"的迭代循环。
- 显式的资源与成本控制：必须显式控制最大搜索轮数、单轮与总 token 预算、允许访问的工具与域名。
- 证据充分性判定：定义明确的证据充分条件，作为是否继续检索的决策依据。
- 查询漂移与非终止防护：通过轮数上限、失败降级策略防止查询漂移、死循环与成本失控。
- 全程可追踪：要求保留完整 Trace，以记录每一步的决策、检索与评估结果。
- 以更多模型调用换取更强能力：与 Naive RAG、GraphRAG 相比，具备更强的规划与证据核查能力，但推理开销更高。

## 应用
- 研究型与多跳问答：需要跨多个文档、多个索引整合证据链的复杂问题。
- 开放性问题求解：答案无法由单次检索直接获得，需要多轮探索与验证的场景。
- Agent 工作流核心：作为 RAGFlow 等产品的工作流引擎，如 agentic_rag_graph.py 中实现的问题理解、路由、计划、检索充分性检查与重新规划。
- 受控检索场景：在预算约束下平衡答案质量与推理成本，防止检索过程失控。

## 相关概念
- [[concepts/rag|rag]]
- [[concepts/llm-wiki|llm-wiki]]
- [[concepts/agent-memory|agent-memory]]
- [[concepts/reranker|reranker]]
- [[concepts/knowledge-graph|knowledge-graph]]

## 相关实体
- [[entities/ragflow|ragflow]]
- [[entities/graphrag|graphrag]]
- [[entities/lightrag|lightrag]]

## 来源提及

- "Agentic RAG | Agent 规划、检索、检查证据并循环 | 研究、多跳、开放问题 | 延迟、非终止、查询漂移 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它把 RAG 从一次函数调用变成了一个决策循环。必须显式控制：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "问题
  → 识别目标与约束
  → 选择检索工具
  → 搜索一轮
  → 判断证据是否充分
       ├─ 足够：组织答案
       └─ 不足：改写、拆题、换索引、继续搜索" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]