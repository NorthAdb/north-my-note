---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [term]
aliases:
  - "来源溯源"
  - "溯源记录"
generation_complete: true
---


# Provenance

## 定义

Provenance（来源溯源）是 RAG 系统中记录证据来源元数据的机制：它保存每条主张（claim）来自哪个 source_id、chunk_id、URL、page、section、相关性评分以及索引版本，使回答中的每个结论都能逐级追溯回原始证据。Provenance 将「引用正确」与「来源真实」区分为两个独立命题——前者表示结论确实能在该来源中找到支持，后者表示来源本身可信、没有过时、没有被污染；引用机制只能解决第一层问题。它是 RAG 建立可约束、可追溯、可评测、可拒答证据链的关键，也是 LLM Wiki 中「每个重要结论都有来源和状态」的设计基础。

## 关键特征

- 细粒度定位：记录 source_id、chunk_id、URL、page、section，将回答中的每个断言锚定到具体文本片段
- 分层验证：明确区分「引用正确」（结论确由来源支持）与「来源真实」（来源可信、未过时、未污染），避免以「有引用」掩盖来源问题
- 版本可追溯：保存索引版本，使检索结果可复现，防止数据更新或重建索引后被污染的内容继续充当证据
- 可评测性：完整的 provenance 记录是 citation eval 的前提，缺少它就无法自动判断回答是否真正被证据支持
- 支撑拒答：当证据缺失或来源不可信时，系统可依据 provenance 判定「无据可答」，从而拒绝作答而非虚构引用

## 应用

- RAG 回答的证据链构建：生成时附带来源引用，用户可点击追溯原文，实现可约束、可追溯的回答
- 引用评测（citation eval）：借助 Ragas 等评测框架，使用保存的 chunk/page/source_id 检验结论是否确实由检索结果支持
- 向量数据库元数据追踪：在 Qdrant 等数据库中为向量保存 payload（source_id、page、section），实现按来源过滤与追溯
- 失败模式诊断：当答案引用不可信时，通过 provenance 记录定位问题根源——是 Provenance 缺失、chunk 切分不当，还是 reranker 排序错误
- LLM Wiki 与知识库维护：保证每个重要结论都有来源和状态，形成可评测、可拒答的引用体系

## 相关概念

- [[concepts/rag|rag]]
- [[concepts/context-engineering|context-engineering]]
- [[concepts/reranker|reranker]]
- [[concepts/llm-wiki|llm-wiki]]

## 相关实体

- [[entities/ragas|ragas]]
- [[entities/qdrant|qdrant]]

## 来源提及

- "Provenance（来源溯源）至少应记录：" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "要区分两个命题：**引用正确**：这个结论确实能在该来源中找到支持；**来源真实**：来源本身可信、没有过时、没有被污染。引用只能解决第一层，不能自动解决第二层。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "答案引用不可信 | Provenance 缺失 | 保存 chunk/page/source_id，做 citation eval" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]