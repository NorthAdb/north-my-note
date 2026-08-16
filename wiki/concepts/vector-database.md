---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [term]
aliases:
  - "向量数据库"
  - "向量库"
  - "Vector DB"
generation_complete: true
---


# Vector Database

## 定义

Vector Database（向量数据库）是面向大规模向量近邻检索优化设计的数据基础设施。它以高维向量（embedding）为核心数据对象，通过近似最近邻（ANN）索引实现高效检索。来源指出，向量数据库除保存向量本身外，还必须同时保存原始文本或引用、稳定的 document_id/chunk_id、title、section、page、timestamp、tenant、ACL 等 Payload 元数据，并支持 dense/sparse 索引、更新删除、快照、分片、复制和可观测性。来源同时强调，向量数据库只是 RAG 的基础设施，不等于知识库本身；高相似度向量不代表内容真实、不代表内容最新，也不代表用户有权阅读对应的原始内容。

## 关键特征

- 面向高维向量的近似最近邻检索优化，工程上常采用 HNSW、IVF、PQ 与倒排索引等 ANN 结构
- 向量与 Payload 共同存储：除向量外，同时保存原始文本或引用、document_id/chunk_id、title、section、page、timestamp、tenant、ACL 等元数据，支撑溯源、过滤与权限控制
- 同时支持 dense（稠密向量索引）与 sparse（稀疏/倒排索引）两类索引
- 具备完整数据库能力：支持更新、删除、快照、分片（sharding）、复制（replication）与可观测性（observability）
- 支持 metadata 过滤与 ACL 权限过滤，可在检索阶段实现租户隔离与访问控制
- 定位为基础设施而非知识库本身：高相似度不等于真实、最新或可访问，检索结果仍需验证与授权
- 生产系统中通常组合 dense ANN + BM25/sparse + metadata/ACL filter，再进入 hybrid fusion 与 rerank 环节

## 应用

- RAG（检索增强生成）中的知识片段存储与召回，为 LLM 提供上下文依据
- 大规模语义搜索与相似内容匹配，如文档检索、图片/视频特征检索、推荐系统中的相似项查找
- 多租户企业知识库检索，通过 tenant 与 ACL 元数据实现数据隔离与权限控制
- Agent 记忆存储：为智能体提供长期记忆的写入与语义检索（如腾讯云 Agent Memory 的存储底座场景）
- 混合检索管线：与 BM25 稀疏检索、metadata filter、hybrid fusion 及 reranker 配合，构成生产级检索系统

## 相关概念

- [[concepts/rag|rag]] — 向量数据库是 RAG 的检索基础设施，负责知识片段的存储与召回
- [[concepts/embedding|embedding]] — 向量数据库索引与检索的核心对象，即文本等数据的向量表示
- [[concepts/bm25|bm25]] — 稀疏检索方法，与 dense ANN 检索互补，常构成混合检索
- [[concepts/hybrid-search|hybrid-search]] — 融合稠密向量与稀疏检索的检索范式，向量数据库是其核心组件
- [[concepts/reranker|reranker]] — 混合检索融合后的精排环节，对向量数据库召回结果重新排序
- [[concepts/provenance|provenance]] — 依赖 Payload 中的原始文本、引用与 document_id/chunk_id 实现检索结果的出处溯源

## 相关实体

- [[entities/qdrant|qdrant]] — 来源中介绍的具体向量数据库实现
- [[entities/tencentdb-agent-memory|tencentdb-agent-memory]] — 依托向量数据库能力构建的 Agent 记忆服务，与向量存储和检索密切相关

## 来源提及

- "但 RAG 本身不等于 Wiki，向量数据库也不等于知识库。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "| **Vector Database** | 向量、Payload、ANN 索引 | 高效近邻搜索 | 只是基础设施 |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "向量数据库通常需要同时保存：
- 向量；
- 原始文本或原文引用；
- 稳定的 document_id / chunk_id；
- title、section、page、timestamp、tenant、ACL 等 Payload；
- dense/sparse 索引；
- 更新、删除、快照、分片、复制和观测信息。" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]