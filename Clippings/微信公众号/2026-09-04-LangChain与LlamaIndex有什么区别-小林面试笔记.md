---
created: 2026-09-04
updated: 2026-09-04
tags: [Agent, LLM, RAG, LangChain, LangGraph, LlamaIndex, 框架选型, 面试, AI面试, 小林面试笔记]
source:
  - "https://mp.weixin.qq.com/s/-DkZ_pzXK-_PYnIhmE2loA"
  - "公众号：小林面试笔记"
evidence: article-summary
---

# LangChain 和 LlamaIndex 有什么区别？

> **作者**：小林面试笔记 | **来源**：微信公众号 | **发布时间**：2026-09-04
>
> 本笔记基于原文重新整理；框架能力会随版本变化，具体 API 和特性应以官方文档为准。

## 一句话结论

LangChain 和 LlamaIndex 现在都能构建 Agent、调用工具、实现 RAG 和 Workflow。真正的区别不是“谁能做什么”，而是**默认入口和设计重心不同**：

- **LangChain**：偏通用 Agent 组装、模型适配和工具集成；复杂状态与执行控制下沉到 **LangGraph**。
- **LlamaIndex**：偏数据接入、文档解析、索引、检索、重排和上下文增强，适合企业知识库与复杂 RAG。
- **两者可以组合**：LlamaIndex 管数据与检索，LangChain 管 Agent 与工具选择，LangGraph 管复杂流程。

![LangChain 与 LlamaIndex 的设计重心](https://gitee.com/cheng-jiaqing/images/raw/master/wx_img_001.png)

## 面试版回答

> LangChain 和 LlamaIndex 都能做 Agent、Tool、RAG 和 Workflow。LangChain 的优势重心是统一模型、消息、工具、结构化输出和中间件接口，适合快速组装工具型 Agent；复杂分支、循环、暂停恢复和人工审批可以使用 LangGraph。LlamaIndex 的优势重心是数据与上下文增强，在文档接入、解析、切分、索引、检索、重排和 Query Engine 上更完整，适合企业知识库和复杂 RAG。选型应看项目的主要风险，而不是只看框架标签；两边都复杂时，可以把 LlamaIndex 的检索能力封装成 Tool，再交给 LangChain 或 LangGraph 调度。

## 为什么容易混淆？

两者功能清单高度重叠：都支持模型调用、Tools、RAG、Agent 和 Workflow。只按功能比较，容易得出“差不多”的结论。

更有用的比较方式是看**项目最难的部分**：

| 设计重心 | LangChain | LlamaIndex |
| --- | --- | --- |
| 默认入口 | 通用 Agent 组装 | 数据与上下文增强 |
| 主要优势 | 模型、消息、Tools、中间件、第三方集成 | 数据接入、解析、索引、检索、重排、Query Engine |
| 常见场景 | 工具型 Agent、SQL Agent、业务助手 | 企业知识库、文档 Agent、复杂 RAG |
| 复杂流程 | LangGraph 管理状态、恢复和人工介入 | Workflows，或与 LangGraph 组合 |

这张表描述的是**优势重心**，不是能力边界。LangChain 也有 RAG 组件，LlamaIndex 也能创建 Agent。

## LangChain 强在哪里？

当项目要同时接入多个模型、搜索、数据库、浏览器、MCP Server 和内部 API 时，工程成本通常来自接口适配。LangChain 用相对统一的抽象屏蔽差异：

- Model、Message、Tool、Structured Output 等统一接口；
- `create_agent` 快速组装模型和工具；
- Middleware 统一加入权限、重试、摘要、动态模型选择和人工审批；
- 集成范围广，便于快速切换模型供应商和外部服务。

因此，**工具选择、参数填充、权限和重试**是主要难点时，LangChain 更自然。

### LangChain 与 LangGraph 的关系

LangChain Agent 底层运行在 LangGraph 之上。简单的模型-工具循环使用 `create_agent` 即可；出现复杂分支、并行、循环、暂停恢复或人工审批时，再显式编写状态图，不必推翻已有的模型与工具定义。

![LangChain Agent 与 LangGraph 的关系](https://gitee.com/cheng-jiaqing/images/raw/master/wx_img_002.png)

## LlamaIndex 强在哪里？

真实 RAG 的难点不只是“把文档放进向量数据库”，而是整条数据链路：

```text
数据接入 → 解析与切分 → 索引 → 检索与重排 → Query Engine → Agent
```

需要处理的问题包括：

- PDF 表格、跨页内容、OCR 和代码结构是否被正确解析；
- Chunk 如何保留标题、页码、版本和元数据；
- 多版本制度哪一份仍有效；
- 查询应走向量检索、关键词检索还是结构化数据库；
- 多路结果如何过滤、融合、重排并处理冲突。

LlamaIndex 把“如何得到高质量上下文”作为核心工程问题，因此**数据处理和检索质量**是主要风险时，应优先评估它。它也提供 Agent、Memory 和事件驱动 Workflow，不能简单说成“只能做 RAG”。

![LlamaIndex 的数据与检索链路](https://gitee.com/cheng-jiaqing/images/raw/master/wx_img_003.png)

## 如何选型？

先问一句：**这个项目最怕哪件事做不好？**

| 项目主要风险         | 优先评估                             | 原因               |
| -------------- | -------------------------------- | ---------------- |
| 模型和业务工具太多，集成复杂 | LangChain                        | 通用组件和 Tool 接口更自然 |
| 文档解析、切分和检索质量差  | LlamaIndex                       | 数据与上下文链路抽象更细     |
| 流程需要暂停、恢复和人工审批 | LangGraph，可搭配 LangChain          | 状态与执行控制是核心能力     |
| 同时需要复杂检索和复杂流程  | LlamaIndex + LangChain/LangGraph | 数据层与编排层分别选合适组件   |
|                |                                  |                  |

![按项目主要风险选择框架](https://gitee.com/cheng-jiaqing/images/raw/master/wx_img_004.png)

### 组合时的边界

最常见的组合边界是 **Tool**：

```text
LlamaIndex：加载数据、构建索引、实现 Query Engine
      ↓ 封装为“查询企业知识库” Tool
LangChain：判断何时调用知识库，或调用其他业务工具
      ↓ 复杂分支、审批、重试、恢复
LangGraph：编排状态和执行路径
```

示例：

```python
from langchain.tools import tool

@tool
def search_company_knowledge(question: str) -> str:
    """查询企业知识库。"""
    return str(query_engine.query(question))

agent = create_agent(
    model=chat_model,
    tools=[search_company_knowledge, lookup_order],
)
```

边界清楚时，组合能让每个框架解决自己擅长的问题；但如果只是简单知识库或单工具 Agent，同时引入两套框架会增加依赖、追踪和调试成本。

![LlamaIndex、LangChain 与 LangGraph 的组合边界](https://gitee.com/cheng-jiaqing/images/raw/master/wx_img_005.png)

## 常见误区

1. **“LangChain 只做 Chain”**：这是早期印象。当前主线已覆盖 Agent、Tools、Middleware 和 RAG，固定流程仍可用 Runnable/LCEL。
2. **“LlamaIndex 只能做 RAG”**：它也提供 Agent 和 Workflow，辨识度在于数据处理、索引、检索和上下文组织。
3. **“框架必须二选一”**：框架可以通过 Tool 或服务接口组合，关键是职责边界是否清楚。
4. **“热度最高的就是最佳选择”**：应围绕业务的主要风险、数据形态、流程复杂度和生产约束做评估。

## 最终记忆卡

```text
工具/模型集成难 → LangChain
数据/检索质量难 → LlamaIndex
状态/流程控制难 → LangGraph
两边都复杂     → LlamaIndex 做数据层 + LangChain/LangGraph 做运行层
```

> 不要再回答“LangChain 写 Chain，LlamaIndex 做 RAG”。更准确的回答是：**LangChain 偏通用 Agent 组装，LlamaIndex 偏数据与上下文增强，LangGraph 负责复杂状态编排；选型取决于项目最主要的工程风险。**

## 关联笔记

- [[2026-08-06-AI-Agent开发框架选型-LangChain-LangGraph-LlamaIndex-小林面试笔记]] — 同主题的框架全景选型
- [[2026-09-01-RAG核心知识全解析（范式演进）]] — RAG 全链路、检索与评测
- [[2026-08-04-GraphRAG与LightRAG-小林面试笔记]] — 图增强检索与 RAG 变体
- [[wiki/entities/langchain]] — LangChain 实体页
- [[wiki/entities/llamaindex]] — LlamaIndex 实体页

## 来源

- 原文：<https://mp.weixin.qq.com/s/-DkZ_pzXK-_PYnIhmE2loA>
- 公众号：小林面试笔记
- 图片图床：Gitee（通过 PicGo 上传）
