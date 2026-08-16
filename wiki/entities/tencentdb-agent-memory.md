---
type: entity
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [project]
aliases:
  - "TencentDB Agent Memory"
  - "TencentCloud/TencentDB-Agent-Memory"
  - "腾讯云 Agent Memory"
generation_complete: true
---


# TencentDB-Agent-Memory

## 描述

TencentDB-Agent-Memory 是腾讯云开源的团队级 Agent Memory Hub（TencentCloud/TencentDB-Agent-Memory），2026-08-11 时 GitHub 星标约 19.5k，采用 MIT 许可证。与 [[entities/andrej-karpathy|andrej-karpathy]] 式个人 Wiki 不同，它将记忆明确划分为 Chat Memory、Skill、LLM-Wiki、Code-Graph 四类资产，强调团队协作与多 Agent 共享。其短期任务流程从原始 tool logs 生成 JSONL 步骤摘要，最终可视化为 Mermaid Canvas；长期记忆分为 L0 Conversation、L1 Atom、L2 Scenario、L3 Persona 四层，底层事实/日志/轨迹以 SQLite、向量与 BM25 存储，上层 Persona/Scenario/Canvas 输出为可读 Markdown。项目强调"可下钻、可恢复"，上层摘要必须能通过 node_id、result_ref 回溯到原始文本。关键实现包括 l0-recorder.ts、l1-extractor.ts、scene-extractor.ts、persona-generator.ts、bm25-local.ts 与 embedding.ts，并可通过 OpenClaw 插件接入记忆捕获与召回。

## 相关实体

- [[entities/mem0|mem0]] — 同为开源 Agent 记忆层（Memory Layer）方向的项目，可对照其记忆存储与召回设计。
- [[entities/claude-obsidian|claude-obsidian]] — 面向个人知识库的记忆方案，与团队级 Memory Hub 形成定位对比。
- [[entities/andrej-karpathy|andrej-karpathy]] — LLM Wiki 概念的代表人物，本项目并非其个人 Wiki 路线的直接复刻。
- [[entities/nashsullm_wiki|nashsullm_wiki]] — llm_wiki 相关的个人 Wiki 实践，可对比理解 LLM Wiki 在个人与团队场景的差异。

## 相关概念

- [[concepts/agent-memory|Agent Memory]] — 本项目的核心主题，将记忆资产化、分层化并支持多 Agent 共享。
- [[concepts/llm-wiki|LLM Wiki]] — 四类记忆资产之一，对应面向 LLM 的知识库组织方式。
- [[concepts/code-wiki|Code Wiki]] — 四类记忆资产之一，对应 Code-Graph 代码图形态的记忆组织。

## 来源提及

- "它不是纯粹的 Karpathy 式个人 Wiki，而是团队级 Agent Memory Hub，明确把记忆分成 Chat Memory、Skill、LLM-Wiki、Code-Graph 四类资产。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它强调“可下钻、可恢复”，上层摘要必须能够通过 `node_id`、`result_ref` 回到原始文本。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它把“LLM Wiki”放进了更大的 Memory 分层体系：页面不是唯一的知识对象，原始事件、原子事实、场景、Persona、技能和代码图可以协同存在。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]