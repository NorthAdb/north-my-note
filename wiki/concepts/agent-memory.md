---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [method]
aliases:
  - "Agent 记忆"
  - "智能体记忆"
generation_complete: true
---


# Agent Memory

## 定义
Agent Memory（智能体记忆）是与 RAG、LLM Wiki 并列的第三类知识架构，以事实、偏好、事件、技能和状态为持久化对象，核心动作是跨轮次召回与写回。它与 LLM Wiki 的关键区别在于：尽管两者都涉及写回，Memory 关注的是 Agent 自身状态，而非人可浏览的 Markdown 知识页——文中明确指出 Memory "通常不是" LLM Wiki。

## 关键特征
- 持久化对象为 Agent 自身状态：事实、偏好、事件、技能与状态
- 核心动作是跨轮次召回（recall）与写回（write-back）
- 与 LLM Wiki 明确区分：Memory 通常不是 LLM Wiki，尽管两者都涉及写回
- 落地形态多样：Mem0 提取原子记忆项而非保存整段对话；TencentDB-Agent-Memory 将记忆分为 Chat Memory、Skill、LLM-Wiki、Code-Graph 四类资产；Cognee 采用图+向量的记忆引擎
- 具备分层抽象：TencentDB-Agent-Memory 定义 L0 Conversation → L1 Atom → L2 Scenario → L3 Persona 的分层结构
- 强调"可下钻、可恢复"：上层 Persona/Scenario 摘要必须能通过 node_id、result_ref 回溯到原始文本

## 应用
- Agent 跨轮次对话中的状态记忆与上下文召回，使 Agent 记住用户偏好、历史事实与事件
- 多 Agent 系统中的技能沉淀（Skill），将可复用的能力持久化为记忆资产
- 与 RAG、Semantic Search、Knowledge Graph、Code Wiki 和 LLM Wiki 共同构成知识系统全景——该概念位于第 6.7 节的关系图中

## 相关概念
- [[concepts/rag|RAG]]
- [[concepts/llm-wiki|LLM Wiki]]
- [[concepts/knowledge-graph|Knowledge Graph]]

## 相关实体
- [[entities/mem0|mem0]]
- [[entities/tencentdb-agent-memory|tencentdb-agent-memory]]
- [[entities/cognee|cognee]]
- [[entities/nashsullm_wiki|nashsullm_wiki]]

## 来源提及

- "| **Agent Memory** | 事实、偏好、事件、技能、状态 | 跨轮次召回与写回 | 通常不是 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "- **Agent Memory**：解决“Agent 应该记住哪些事实、经历、偏好和技能”；" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它不是纯粹的 Karpathy 式个人 Wiki，而是团队级 Agent Memory Hub，明确把记忆分成 Chat Memory、Skill、LLM-Wiki、Code-Graph 四类资产。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]