---
type: concept
created: 2026-08-11
updated: 2026-08-11
sources: ["[[sources/rag与llm-wiki原理架构与开源实现_a74e83]]"]
tags: [term]
aliases:
  - "代码 Wiki"
  - "CodeWiki"
generation_complete: true
---


# Code Wiki

## 定义
Code Wiki 是把代码库编译成可追溯的架构与模块文档的一种特殊 Wiki 形态，属于 LLM Wiki 在代码领域的特化。其核心动作可概括为：代码库 → 结构理解 → 可追溯的架构与模块文档。它明确"是一种特殊 Wiki"，区别于个人研究 Wiki；在现有 Vault 中与 Google 的 AI 驱动代码文档平台 CodeWiki 的相关笔记直接关联。

## 关键特征
- 特殊 Wiki 形态：作为 LLM Wiki 的特化，Code Wiki 面向代码库而非个人研究笔记，产出的是工程可用的结构文档。
- 可追溯性：生成文档带有文件与行号引用，每个结论都能回溯到具体源码位置。
- 流水线自动化：以 DeepWiki-Open 等开源项目为典型实现，流程为：克隆仓库 → 递归读取文件 → 文件切块并计算 Embedding → FAISS 检索相关代码 → LLM 规划 Wiki 结构 → 分页生成文档 → 添加文件/行号引用。
- 检索与组织的分工：RAG 负责找到相关文件片段，Wiki 负责把局部检索结果提升为可浏览的结构化文档。
- 依赖向量检索基础设施：管道中的文件切块与 Embedding 计算是支撑 FAISS 检索的关键步骤。

## 应用
- 代码库文档化：为开源或内部仓库自动生成架构总览、模块说明与代码导读，降低新成员理解代码库的门槛。
- 可追溯的工程文档：通过文件/行号引用让文档与源码一一对应，便于审查与维护。
- 与 RAG 生态结合：在检索阶段叠加 Hybrid Search 等策略，将局部相关代码片段重组为全局可浏览的 Wiki 页面。

## 相关概念
- [[concepts/llm-wiki|llm-wiki]] —— 父概念，Code Wiki 是 LLM Wiki 在代码领域的特化
- [[concepts/rag|rag]] —— 负责在代码库中检索相关文件片段
- [[concepts/hybrid-search|hybrid-search]] —— 可结合向量检索与关键词检索提升代码召回质量
- [[concepts/chunking|chunking]] —— 文件切块步骤的支撑技术
- [[concepts/embedding|embedding]] —— 计算向量表示以支持 FAISS 检索

## 相关实体
- [[entities/deepwiki-open|deepwiki-open]] —— Code Wiki 的典型开源实现，完整呈现了从仓库克隆到可追溯文档生成的处理流程

## 来源提及

- "| **Code Wiki** | 代码结构、模块说明、引用链接 | 解释代码库 | 是一种特殊 Wiki |" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它面向代码库，不是个人研究 Wiki。典型流程：克隆仓库 → 递归读取文件 → 文件切块并计算 Embedding → FAISS 检索相关代码 → LLM 规划 Wiki 结构 → 分页生成文档 → 添加文件/行号引用" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]
- "它的启示是：**RAG 负责找相关文件，Wiki 负责把局部检索结果提升为可浏览的结构文档。**" — [[领域/RAG与LLM Wiki：原理、架构与开源实现|RAG与LLM Wiki：原理、架构与开源实现]]