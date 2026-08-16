---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [theory]
aliases:
  - "知识编译"
  - "知识编译模式"
generation_complete: true
---


# Knowledge Compilation

## 定义

Knowledge Compilation（知识编译）是一种描述 [[concepts/llm-wiki|llm-wiki]] 运作方式的理论框架：把 LLM Wiki 视为一个持续运行的编译器。其中，Raw Sources（原始资料）是源代码，Schema 是编译规则，LLM 是编译器与分析器，Wiki Pages 是中间表示或可读产物，Indexes 是构建产物，Lint 是静态与语义检查，Git 是版本控制与回滚。该框架支撑的核心论点是：LLM Wiki 是一种持续维护的知识编译模式，与传统 [[concepts/rag|rag]] 的"查询时检索"形成本质区别。

## 关键特征

- 原始资料不可覆盖：原始资料是"源代码"，派生页面不能覆盖它；Indexes 可重建，但原始资料必须保留
- 可差分、可回滚：页面更新应产生可审查的 diff，任何变更可借助 Git 回滚到历史版本
- 显式暴露编译错误：Lint 检查发现的冲突与错误必须显式报告，不能静默吞掉；"页面看起来完整"不等于"每个结论都有证据"
- 派生与源头分离：Wiki Pages 是派生的可读中间表示，Raw Sources 是唯一事实来源，两者严格分层
- 构建时持续加工：知识在构建阶段被编译为结构化页面，而非查询时临时检索，这是它与 RAG 模式的根本差异

## 应用

- 知识库构建：将散乱原始资料按 Schema 规则编译为结构化、带溯源的可读页面
- 质量校验与维护：通过 Lint 式检查发现断链、无证据结论和过期页面，把"编译错误"显式暴露给维护者
- 版本管理与审计：对每次页面更新做 diff 审查，支持回滚，保障知识变更的可追溯性
- 架构选型：用于区分"持续维护的知识编译模式"与"查询时检索模式"（如 [[concepts/naive-rag|naive-rag]]），在需要证据溯源和长期可维护性的场景下更占优势

## 相关概念

- [[concepts/llm-wiki|llm-wiki]] —— 该理论描述的实践对象，LLM Wiki 本身即知识编译模式的载体
- [[concepts/rag|rag]] —— 传统查询时检索范式，与知识编译模式形成本质区别
- [[concepts/provenance|provenance]] —— 知识编译要求每个结论都有证据，与溯源要求直接对应
- [[concepts/code-wiki|code-wiki]] —— 同样把 Wiki 内容当作可审查、可版本化的工程产物

## 相关实体

- [[entities/nashsullm_wiki|nashsullm_wiki]] —— LLM Wiki 的开源实现（别名 llm_wiki），是知识编译模式的实际载体
- [[entities/claude-obsidian|claude-obsidian]] —— 将 LLM 与 Obsidian 结合的项目，与 LLM Wiki 知识编译实践相关

## 来源提及

- "**LLM Wiki 是一种持续维护的知识编译模式**：新资料进入后，LLM 不只建立索引，还会更新实体页、概念页、综述页、链接和维护日志。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "把 LLM Wiki 想成编译器：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "这个比喻提醒我们：原始资料不能被派生页面覆盖；页面更新应该有 diff；索引可重建，原始资料必须保留；编译错误应显式暴露，不能静默吞掉冲突；“页面看起来完整”不等于“每个结论都有证据”。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]