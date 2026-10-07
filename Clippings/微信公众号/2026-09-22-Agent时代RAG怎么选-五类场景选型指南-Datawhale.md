---
created: 2026-09-22
updated: 2026-09-27
tags: [RAG, Agent, 上下文工程, GraphRAG, LightRAG, Agentic-RAG, LLM-Wiki, 上下文压缩, RAG评估, Ragas, 选型]
source:
  - "https://mp.weixin.qq.com/s/4XGijn-_H1KH05op--1kBg"
  - "公众号：Datawhale《Agent时代，RAG怎么选？这份指南一次讲清！》（2026-09-22）"
  - "作者：卞一岚（百度 ACG 实习生、河海大学研究生）"
---

# Agent 时代，RAG 怎么选？这份指南一次讲清

> **来源**：公众号「Datawhale」，作者卞一岚，2026-09-22。
>
> **一句话总结**：RAG 的核心不是 retrieval，而是 **context construction**——它只是在推理时补上下文的手段之一。选型应该由「**信息缺口长什么样**」反推技术，而不是先决定「要不要上 RAG」。

![信息缺口驱动的 RAG 技术选型全景：五类 RAG 场景 + 五类 RAG 替代方案](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig8-when-to-use-rag.png)

## 0. 立论：RAG 不是固定架构，而是一组补信息缺口的技术

给 Agent 的输入往往只有一句话（"这是我领导的要求 xxx""帮我修一下这个报错"），但完成任务真正需要的上下文远不止这一句：**具体目标、验收标准、以及停留在人脑/会议记录/聊天记录/历史决策里的背景知识**。

现在 Agent 越来越不像"大模型 + 一句 prompt"，而像一个围绕模型搭起来的**上下文系统**。以 Claude Code 为例，Anthropic 官方把可控入口拆成 CLAUDE.md、rules、skills、subagents、hooks、output styles、append system prompt 等[1]——共同点不是"把提示词写长"，而是**把不同类型的信息放到合适的位置**：有些常驻、有些按需加载、有些用 hook 保证执行、有些交给隔离的 subagent。

RAG 要解决的就是这条链路上最常见的一类问题：**用户的话太短，模型参数里的知识又不够新、不够私有、不够贴合当前任务**。

所以定义要抠两个关键词：

| 关键词 | 含义 | 反例 |
| --- | --- | --- |
| **"此刻需要"** | 不是把所有资料都塞给模型 | 用户问"上线后白屏怎么办"，真正需要的不是"白屏"附近的几段文档，而是环境差异、构建配置、资源加载、容器渲染、监控日志这条排查链路 |
| **"整理成上下文"** | 不是一搜到相似 chunk 就万事大吉 | 淘宝答疑系统用 CoT 做意图拆解、再并行生成多组查询，本质是**先规划信息需求，再填充上下文** |

> **结论**：RAG 的核心不是 retrieval，而是 **context construction**。检索只是手段，目标是让模型看到**足够相关、足够完整、足够有结构**的信息。

VimRAG / MMG 这类多轮检索记忆方案进一步佐证：多轮对话里如果检索系统没有"自己查过什么"的记忆，就会反复召回同一批内容。用图结构记录检索路径，或在工程上用 session 级 `consumed_ids` 去重，都是在让上下文构建**变得有状态**[2]。

因此本文不复述一套固定架构，而是给出一组技术选型：经典向量检索 / GraphRAG·LightRAG / Agentic RAG / 知识预编译 / 上下文压缩——**以及什么时候根本不该用 RAG**。

![Agent 为什么需要 Context System：一句话输入 → Agent 发现信息缺口 → Context System 补齐](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig1-agent-context-system.png)

## 1. 五类场景选型总览

| # | 技术 | 解决的问题 | 适合 | 不适合 |
| --- | --- | --- | --- | --- |
| 1 | **经典 RAG**<br>Naive → Advanced → Modular | 有文档，别让模型瞎编 | 文档问答、客服、内部知识库 | 需全局统计 / 跨文档结构推理 |
| 2 | **GraphRAG / LightRAG** | 答案散落在文档之间的**关系**里 | 离线分析、全局综述、研究型知识库 | 高频更新、强实时、成本敏感 |
| 3 | **Agentic RAG** | 一跳查不到，需要"查到 A 才知道去哪查 B" | 多跳、跨源、信息分散 | 固定知识库 FAQ |
| 4 | **LLM Wiki / 知识预编译** | 稳定知识被反复从头检索，太浪费 | 个人知识库、onboarding、竞品分析 | 强实时 / 强审计 / 强权限隔离 |
| 5 | **上下文压缩** | 内容很多，信息密度很低 | 长文档、长对话、长轨迹复用 | 每次都是临时网页、错误代价极高 |

> **共同判据**：不要先问"要不要上 RAG"，先问"**当前信息缺口到底适合被检索成 chunk 吗**"。

## 2. 经典 RAG：Naive → Advanced → Modular

### 2.1 Naive RAG：两条链路

适合从"我有一堆文档，想让大模型别瞎编"这个朴素需求起步。好处是**便宜、快、工程闭环短**；RAG 最早的核心思想也是把参数化记忆与外部非参数化记忆结合[3]。

![经典 RAG 全链路：离线索引（Load → Split → Embed → Store）+ 在线查询（Query → Retrieve → Rerank → Generate）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig2-classic-rag-pipeline.png)

| 阶段 | 链路 | 关键要点 |
| --- | --- | --- |
| **离线索引** | Load → Split → Embed → Store | 解析 PDF/Word/Markdown/HTML/JSON/Excel/语雀/钉钉文档，**顺手保留 metadata**（文件名、页码、章节、标题、作者、创建时间） |
| **在线查询** | Query → Retrieve → Rerank → Generate | 处理问题 → 召回 Top-K → 必要时精排 → 把问题与证据一起交给 LLM |

**每一步都有坑，且坑大多在检索侧而不在模型侧：**

- **文档加载**：PDF 文字层、扫描件 OCR、HTML 正文抽取、表格结构保留，都会影响后面能不能搜到正确内容。**很多"RAG 答错"的根因是文档入口就把正文、页码、标题层级或表格语义弄丢了。**
- **metadata 不是装饰品**：它决定了后面能不能做权限过滤、章节过滤、时间过滤、引用溯源。
- **切分是最容易被低估的环节**：

| 切分策略 | 特点 |
| --- | --- |
| 固定长度 | 最便宜，适合快速 baseline |
| **递归切分** | 按标题/段落/句子逐层切，通用文档的**稳健起点** |
| 语义切分 | 按相邻句 embedding 相似度找边界；研究显示计算成本不一定换来同比例收益[19] |

> **实践起步方案**：递归切分 + **10%~20% overlap** + 保留章节 metadata，然后用 Context Precision / Recall 看 bad case。

- **进阶切分**解决"chunk 太小答案被切碎、太大噪声太多"：

| 方案 | 思路 |
| --- | --- |
| **Late Chunking** | 先编码整篇文档再切 chunk，让每个 chunk 向量带上全局上下文[20] |
| Intent-Driven Dynamic Chunking | 切分边界不只看文档结构，而是预测用户可能的信息需求[21] |
| **Small-to-Big** | 检索用**小 chunk 精确命中**，生成时扩展到相邻段落/章节/父节点——工程上很实用的折中 |

- **Embedding 与索引**：稠密检索适合语义改写/同义表达/开放问题；稀疏检索（BM25）适合专有名词、编号、错误码、型号、法规条款；**混合检索通常比任何单一路线更稳**（融合可用 RRF，也可在验证集上调加权系数）。
  - ⚠️ 不要迷信"最强 embedding 模型"，要看语种、领域、文档长度、成本、延迟，以及**它是否真的提高了端到端答案质量**。
- **别把用户原话直接扔进向量库**：用户问题往往短、口语化、缺上下文，还带错别字和指代词。

| 查询增强 | 做法 | 代价 |
| --- | --- | --- |
| Query rewriting | 指代消解、纠错、术语对齐、结构简化 | 低 |
| Multi-Query | 从多个角度生成 3~5 个变体问题，提升召回 | 放大 rerank 成本 |
| **HyDE** | 先让 LLM 写"假设性答案"，再用答案向量搜真实文档——把"问题-文档匹配"变成"文档-文档匹配"[22] | 假设答案偏了检索方向就偏 |
| 反向 HyDE / Doc2Query | 离线给每个 chunk 生成它可能回答的问题，用问题倒排到 chunk | 降低在线延迟，但需离线成本 |

> **查询增强不是越多越好**：EAR 提醒查询扩展本身也需要被重排[23]。稳的做法是**按问题类型路由**——事实型走 BM25 + Dense，模糊开放问题走 HyDE / Multi-Query，强业务约束问题先做 metadata filter。

- **Rerank 是经典 RAG 里性价比很高的一步**：

| | Bi-Encoder（向量检索） | Cross-Encoder（Reranker） |
| --- | --- | --- |
| 编码方式 | query 和 document **分别**编码 | query 和候选文档**拼在一起**，逐 token 判断匹配 |
| 速度 | 快 | 慢 |
| 精度 | 只能说明"看起来相关" | 能缓解"资料看着像，但答非所问" |
| 典型用法 | 召回 Top-50 / Top-100 | 精排到 Top-3 / Top-5 交给 LLM |

- **生成阶段是"基于证据组织答案"，不是自由发挥**：prompt 要清楚区分用户问题、检索材料、回答规则和引用格式；事实型问题降 temperature；资料不足时**要求拒答**；知识冲突时**要求指出冲突来源**；关键结论要带引用。若提示词优化到极限仍不稳，才考虑 SFT。

### 2.2 Advanced RAG：不是换一套，而是每个环节补强

切分更聪明、检索更混合、query 更贴近知识库、rerank 更精确、生成更受约束。

- 淘宝答疑案例：用 CoT 做意图识别，再按排查步骤生成多组查询并行召回——用户问"小程序上线后白屏"，不是只搜"白屏原因"，而是拆成**环境差异 / 资源加载 / 渲染链路 / 监控日志**几条线同时找证据。

![从 Naive RAG 到 Advanced RAG：Query Rewrite → Hybrid Retrieval → Rerank → Context Optimization → LLM](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig3-naive-to-advanced-rag.png)

**适合**：知识库规模已起来、用户问法很飘、领域术语多、召回质量不稳定（企业内部文档、工单系统、研发答疑、售后知识库）。

> **最值得落地的不是"上最贵的 embedding"，而是先做最小评测集**：十几条 golden queries 覆盖**精确词 / 多跳 / 长尾 / 不可回答问题**；每次改 chunk、embedding、rerank、rewrite，都看 Recall@k、Context Precision、Faithfulness、Relevance 有没有变好。

### 2.3 Modular RAG：把 RAG 拆成可替换、可路由、可组合的模块

Advanced RAG 仍有边界：它主要是在"**更好地找片段**"，不是"真正理解整个知识系统"。需要全局统计、复杂实体关系、跨文档结构推理时，单纯的 query rewriting + rerank 只是把更多碎片塞给模型。

另一个误区是把所有优化堆进一条 pipeline：每次查询都 rewrite、multi-query、hybrid、rerank、compress、judge，最后准确率没涨多少，延迟和 token 成本先爆了。

Modular RAG 的关键**不是某个单点算法**，而是模块化：loader、splitter、embedder、retriever、reranker、query rewriter、fusion、compressor、generator、evaluator。它不再默认所有问题都走同一条 `retrieve-then-generate` 直线，而是允许 **conditional / branching / looping**[5]：

```
简单事实题 → dense retrieval
含精确术语 → BM25
多跳问题   → hybrid RRF
召回不足   → 触发 query rewrite
证据冲突   → 再检索一次
```

| 代表实践 | 做法 |
| --- | --- |
| **AutoRAG** | 自动搜索 chunk 策略 / embedding / retriever / reranker 的组合[24] |
| **QuIM-RAG** | 把"Query-Document 匹配"改成"Query-Query 匹配"，用问题倒排索引提升 QA 召回[25] |
| **OpenViking** | 把上下文数据库做成**文件系统范式**，让记忆/资源/技能/检索链路更可解释可调试[26] |
| **Experience-RAG Skill** | 把"选择检索策略"从业务代码里抽出来，按任务场景在 BM25 / Rewrite-BM25 / Dense / Hybrid RRF 间选择 |

> **经典 RAG 选型**：刚起步文档少问题简单 → Naive RAG（先跑通"有据可查"）；召回不稳、问法复杂 → Advanced RAG；出现多知识库、多检索器、多任务类型、多轮调用 → Modular RAG。**不要一上来就建宇宙飞船，RAG 第一原则：先证明文档能被找对，再讨论模型能不能答好。**

## 3. GraphRAG / LightRAG：当知识不再是一堆孤立 chunk

经典 RAG 最怕：答案不在某一个 chunk 里，而**散落在一堆文档的关系里**——"这个系统的核心设计模式是什么""A 组件和 B 组件为什么耦合""上线白屏可能涉及哪些链路"。向量检索能把相似片段捞出来，但它不知道这些片段之间**谁依赖谁、谁解释谁、谁只是同名路人**。最后模型拿到一盘散沙，只能硬拼：拼得好叫推理，拼不好就是一本正经地猜。

![RAG 演进：从文本检索（Naive）到知识关系网络（GraphRAG）再到轻量图 + 向量（LightRAG）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig4-graphrag-lightrag-evolution.png)

| | **GraphRAG** | **LightRAG** |
| --- | --- | --- |
| 核心 | 先抽实体/关系构建知识图谱，再对关联实体做社区发现并生成社区摘要[6] | 保留"实体-关系"主线，砍掉最贵的部分；图结构 + 向量共同索引[7] |
| 检索 | Local Search（围绕实体）+ 社区摘要答全局问题 | **双层检索**：低层事实 + 高层关系 |
| 真正解决的 | 不是"召回更多文本"，而是"**让知识之间的结构先显形**" | 把"知识有关联"做得更便宜、更快、更能上线 |
| 代价 | ⚠️ 构建成本高（大量 LLM 调用）、社区发现/摘要继续吃 token、**增量更新不友好** | 用**增量更新**替代全量重建：新数据先形成局部图再合并 |
| 适合 | 离线分析、全局综述、研究型知识库、低频更新但要求综合理解 | 企业答疑、技术支持、产品文档助手（80% 是具体事实或局部排障） |

> 淘宝 AI 答疑的典型判断：GraphRAG 在复杂问答和全局理解上更强，但在线答疑更在意**秒级响应、频繁更新和可控成本**，于是 LightRAG 成了更接近生产的折中方案。

**别混淆的方向——Graphify**[8]：更像给 Agent 准备的"**项目导航图**"，把代码、SQL、文档、论文、图片映射成可查询知识图谱，输出 `graph.html` / `GRAPH_REPORT.md` / `graph.json` 这类人和 Agent 都能消费的产物。它不一定是直接回答用户问题的 RAG 后端，而是让 Agent 不必每次从原始文件开始瞎逛——**"先有结构化地图，再决定读哪里"**，比单纯 embedding 检索更接近真实工程师的工作方式。

> **一句话**：别一上来就建图，先看问题是不是需要**跨文档关联、全局摘要、实体关系推理**；如果只是查一个接口参数、找一条制度原文，Naive/Advanced RAG 加重排可能更便宜、更稳。

## 4. Agentic RAG：让 Agent 决定要不要再查一次

传统 RAG 像一个"**只准搜一次的实习生**"：拿到问题，查一把，拼上下文，开始回答。但很多真实问题不是一跳能解决的——问"项目 X 用的服务器规格是什么"，第一次检索可能只找到项目文档里的**服务器 ID**，真正的规格藏在另一套资产库里。

**Agentic RAG 的核心价值**：让 Agent 在发现信息不够时，自己决定——**拆问题、换 query、换数据源、再查一次**，直到上下文足够或明确不可答。

![Agentic RAG：让 AI 主动寻找缺失信息（Planner → Retrieve → Context Check → Enough Context? → 再检索 / 生成）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig5-agentic-rag-loop.png)

Google 的 Agentic RAG 方案把检索拆进 Agent workflow：Orchestrator 判断是否一跳可做 → Planner 规划信息路径 → Query Rewriter 改写多个可检索问题 → Search Fanout Agent 分发到不同语料/系统 → 综合回答。

> **真正有意思的不是"多 Agent"，而是引入了 sufficient context agent**[9]：先看召回片段、草稿答案和原始问题，判断"现在的信息够不够回答"。如果缺关键信息，**就别急着生成答案**，而是带着明确缺口回去搜更具体的线索。

**选型逻辑**：如果答案路径需要"**查到 A 后才能知道去哪里查 B**"，就该考虑它；如果只是固定知识库 FAQ，一次混合检索加 rerank 就够了——**别把小问题装修成指挥中心**。

### 4.1 副作用一：重复查

多轮对话里用户经常追问、改问、绕同一主题问，检索系统没有记忆就会一遍遍召回同一批 chunk。

| 层级 | 解法 |
| --- | --- |
| **轻量（工程现实）** | 按 session 维护 `consumed_ids`，每轮记录已用过的 doc/chunk 并在下轮过滤；单次检索内按 doc 去重（避免同一篇文档的三个相似 chunk 挤满 Top-K） |
| **重（VimRAG 类）** | 多模态记忆图：把推理过程建成 DAG，记录每步 action、query、证据节点和路径[2] |

但 VimRAG 不是拿来就能用的插件——Multimodal Memory Graph、图调制视觉记忆编码、Graph-Guided Policy Optimization 都是围绕**训练后的多模态 agentic RAG 体系**设计的，适合多图片/多视频/长链路推理。

> **普通团队更现实的起步方式**：把 retrieval 做成 **tool**，而不是普通 function。function 适合"输入确定、输出可预期"的逻辑；retrieval 的结果不确定，更适合交给 Agent loop 做"**决策 → 检索 → 反思 → 再检索**"。这就是 Agentic RAG 的工程最小闭环。

### 4.2 副作用二：查得更多 ≠ 引用更可信

Deep research 类系统最容易出现"**引用标签看起来很正规，点进去完全不支持原句**"。**引用存在和引用正确是两回事。**

靠谱做法是生成后加 Citation Resolver，三层递进：

1. **规则校验**：citation ID 是否存在；
2. **chunk hash 校验**：确认引用内容没有因索引重建或文档更新而漂移；
3. **语义蕴含校验**：对高风险 claim 做 NLI 或 LLM-as-Judge。

> 普通场景可抽样做第三层，高风险场景别省这点钱。

### 4.3 边界：擅长找证据，不擅长验证全局量词

"所有门店都没有差评吗""超过 50% 的评论是否提到服务慢""哪个 group 排名第一"——这些不是多搜几次就能稳的，而是**语义聚合与全局验证**问题。Evergreen 的思路是把自然语言 claim 编译成可执行的 semantic verification query，再用 query plan 和执行优化降低成本，并返回支撑 verdict 的 citations[10]。

> **记住**：Agentic RAG 解决"**路径没走完**"，不解决"**全局统计不可靠**"。前者让 Agent 再查一次，后者需要查询计划、采样、聚合和验证。

## 5. LLM Wiki / 知识预编译：稳定知识不要每次从头检索

经典 RAG 像"**开卷考试**"：问题来了先去翻书，翻到几个相似片段再临场组织答案。适合事实查询，但遇到稳定、长期、反复使用的知识就显得浪费——每次问跨文档问题都让模型重新检索、拼接、综合，**等于每次都把同一锅汤从生水开始煮**。真正缺的不是再多一个向量库，而是把已经读过的材料**沉淀成可复用的中间层**。

![LLM Wiki 三层架构：Raw Sources / Schema → Wiki 层（concepts、entities、synthesis、journal、index.md、log.md…）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig6-llm-wiki-three-layers.png)

Karpathy 的 LLM Wiki 设计有三层[11]：

| 层 | 内容 |
| --- | --- |
| **Raw sources** | 只读原始材料 |
| **Wiki** | LLM 维护的摘要页、实体页、主题页、综合页 |
| **Schema** | 规定目录结构、引用规则、更新流程和校验方式 |

> 这和编译型语言有点像：**RAG 是运行时解释，LLM Wiki 是先把常用知识编译成更适合模型读取的形态。**

| 适合 | 不适合 |
| --- | --- |
| 知识稳定、问题会反复来、跨文档综合很多：个人知识库、团队 onboarding、研究选题、产品资料库、竞品分析、项目复盘 | 强实时、强审计、强权限隔离但工程底座没准备好的生产问答 |

⚠️ **Wiki 页是二级知识**——哪怕保留来源，也不是严格的原文 chunk 级引用。企业合同、金融合规、医疗法规这类"答案必须逐字追溯到原文"的场景，LLM Wiki 不能替代引用链、权限系统和评测闭环。

> **更现实的落地方式**：把它当作 RAG 的**预处理层**——先用 Wiki 做主题组织、实体归档、矛盾发现和问题路由，最后回答仍然回到原文证据。

新的研究把它描述成一种 **agent-native retrieval**[12]：外部知识不是扁平 chunk，而是**可搜索、可阅读、可沿链接遍历、可自我修正**的结构。代表实践上，Obsidian-Wiki、GBrain 在 Karpathy 的通用模式上进一步加入 delta 追踪、来源可信度、热缓存、可见性标签等工程机制。

> **最重要的判断标准不是"有没有向量库"，而是"上次查询产生的洞察有没有留下来"**。每次问完只剩聊天记录 = 一次性消费；好答案能回写成页面、被后续问题复用 = 知识库开始复利。

## 6. 上下文压缩：不是所有上下文都值得原样塞进去

长文档、长对话、RAG 召回结果、Agent 执行轨迹，常见毛病都是"**内容很多，信息密度很低**"。模型窗口变长后问题没有消失，只是**变贵了**：输入越长，延迟越高，注意力计算越重，模型越容易被无关细节带偏。

压缩的目标不是"把文本变短"，而是在 **token 预算 / 答案质量 / 延迟 / 可复用性**之间做交易。

| | **硬压缩**（人类可读符号空间） | **软压缩**（连续向量 / 特殊 token / KV 状态 / 前缀表示） |
| --- | --- | --- |
| 代表 | SelectiveContext、LLMLingua、LLMLingua-2 | Prompt Compression Survey 分 hard/soft prompt 两大路线[13] |
| 好处 | 工程友好：压缩后仍是自然语言，可直接喂闭源 API，跨模型迁移容易 | 想象空间大：像图片模型把百万像素压成少量视觉 token 一样，把长文本压成 LLM 能读懂的高密度前缀 |
| 坏处 | 删错一个约束答案就偏；压太碎语法/语义分布变怪；每个 query 重压一遍则压缩器本身变成新延迟源 | **绑定模型**：某个开源模型空间训练出的向量换到另一模型可能是随机噪声；依赖 GPT/Claude 这类闭源 API 时甚至不能注入自定义 soft prompt |

软压缩的 encoder-decoder 路线可粗略分四派：COCOM / LLOCO（前后模型都微调）、ICAE / 500xCompressor / QGC（冻结解码大模型，只训练编码器）、xRAG（借用 embedding encoder）、UniICL（冻结两端只训练 projector）。

> **软压缩适合**模型内核可控、调用量足够大、愿意长期维护压缩器的系统；**不适合**今天换模型、明天换供应商、后天还要接多个 API 的轻量应用。对大多数工程团队，**自然语言硬压缩虽然不酷，但更稳、更通用**。

**LLMLingua-2 是很实用的方向**[14]：把提示词压缩建模成 **token 二分类任务**，用 GPT-4 蒸馏出抽取式压缩数据，再用 XLM-RoBERTa-large / mBERT 这类双向编码器学"保留还是丢弃"。关键不是单次压得多狠，而是 **task-agnostic**——一份长文档先压成高信息密度版本，再被多个 RAG 查询或多轮对话**复用**。相比 LongLLMLingua 这类更 query-aware 的方案，它牺牲部分针对性，换来更好的复用性和系统吞吐。

**选型时问三个问题：**

1. **压缩对象会不会复用？** 同一份说明书被反复问 → 先离线压缩很划算；每次都是临时网页 → 压缩成本可能抵不过收益。
2. **错误代价高不高？** 合同、财报、代码补丁这类任务，压缩必须保留**引用、数字、否定词、条件边界**，宁可少压一点。
3. **瓶颈到底是 token 费、延迟，还是注意力被噪音污染？** 只是贵 → 硬压缩和摘要就够；是长任务状态管理 → 可能要像 Agent Memory 那样**逐级折叠**：完整材料留在外部，当前上下文只保留任务状态、关键证据和可回溯索引。

## 7. RAG 的评估：没有指标，就没有优化方向

落地时最容易掉进的坑：**系统答错了，第一反应是"再调一下 prompt"**。这很危险，因为 RAG 的错误不一定发生在生成端——很多时候模型不是不会答，而是**检索阶段根本没召回正确材料**；也有时候材料召回对了但模型没有忠实使用；还有时候答案看起来相关，但夹杂了冗余和幻觉。

### 7.1 第一原则：把检索层和生成层拆开看

![RAGAS Score：Generation（Faithfulness / Answer Relevancy）+ Retrieval（Context Precision / Context Recall）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis/rag-fig7-ragas-score.png)

| 层 | 问题 | 指标 |
| --- | --- | --- |
| **检索层** | 有没有把正确上下文找回来，且排在前面 | **Context Precision**：排在前面的上下文是不是相关（"捞上来的东西干不干净"）<br>**Context Recall**：参考答案需要的事实有没有被召回（"关键证据有没有漏"） |
| **生成层** | 有没有基于上下文回答，是否贴合用户问题 | **Faithfulness**：回答中的 claim 能否被检索上下文支撑（"有没有根据材料说话"）<br>**Answer Relevancy**：是否跑题、是否缺失用户真正关心的信息 |
| 附加 | 会不会被噪声带偏 | **Noise Sensitivity**：面对相关但冗余、或完全不相关的上下文时的稳定性 |

> 只有拆开以后，优化才有方向：**Context Recall 低，先别调 prompt**，应看切分、索引、query 改写、召回策略；**Faithfulness 低**，说明拿到了材料但没忠实使用，才该看提示词、引用约束、生成参数或模型能力。

Ragas 是常用的 RAG 自动化评估框架，核心思路是 **LLM-as-a-Judge 近似人工评测**，把 RAG 输出拆成可量化指标[17]。它不是取代人工验收，而是让团队在每次改 chunk / 换 embedding / 加 rerank / 改 prompt 后都能跑同一套测试集。**没有这个闭环，RAG 优化就很容易变成"我感觉这版更好了"。**

### 7.2 测试集：至少覆盖四类问题

Ragas 的测试集生成思路：把文档切分成节点 → 通过信息提取器和关系构建器形成知识图谱 → 按场景生成问题。**场景**可包含节点、查询长度、查询风格和用户 persona。

| 类型 | 检验什么 | 例子 |
| --- | --- | --- |
| 单跳具体 | 基础事实召回 | "某条规则是什么" |
| 单跳抽象 | 总结解释能力 | "这个规则背后的原则是什么" |
| 多跳具体 | 跨文档串联 | "A 的导师的导师是谁" |
| 多跳抽象 | 跨来源综合 | "某个理论从提出到现在如何演变" |

> ⚠️ **如果测试集只包含简单事实题，系统很可能在评测里很好看，一上线遇到复杂问法就崩。**

### 7.3 指标 → 动作对照表

| 指标低 | 尝试的优化动作 |
| --- | --- |
| Context Recall | query rewriting、多查询生成、HyDE、Doc2Query、chunk 策略调整 |
| Context Precision | metadata filter、标签过滤、rerank、缩小 Top-K |
| Faithfulness | 强化引用约束、降低 temperature、要求 evidence-only、增加 Citation Resolver |
| Answer Relevancy | 按问题类型切 prompt 模板 |
| Noise Sensitivity 高 | 减少无关上下文，或训练模型学会拒绝使用噪声 |

> **RAG 工程化的关键不是"技术栈堆得多"，而是"可测、可调、可信赖"**：可测 = 每次改动都有指标；可调 = 指标能指向具体模块；可信赖 = 系统不仅会回答，还能说明答案来自哪里、哪些证据支撑了它、哪些场景下应该拒绝回答。**没有评估闭环的 RAG，只是一个看起来会检索的聊天机器人。**

## 8. 什么时候不该用 RAG

RAG 不是"让模型知道更多"的唯一方式，它只是"在推理时找一小段上下文"的方式。**只要问题的关键不在"找几段相关证据"，RAG 就可能变成错误抽象**——多一套向量库、多一套索引更新、多一层召回误差，最后还要靠 prompt 去修补检索没拿到的东西。

### 边界 1：MB 级本地代码库 → 用 grep/glob/read

代码库不是一堆语义相似的段落，而是有**文件名、符号名、调用关系、提交历史和测试反馈**的工程对象。Claude Code、Codex 这类 CLI Agent 的思路更接近"让模型驱动 grep/glob/read/git 逐步探索"：先文件名定位 → `rg` 找匹配行 → 按需读取局部代码 → 必要时开子 Agent 隔离搜索上下文。

> 优势不是"更原始"，而是保留了代码世界里最强的信号：**精确字符串、路径结构、局部上下文和可执行验证**。对几 MB / 几十 MB 的项目，预先 embedding 往往是在给一个本来可以动态搜索的问题制造同步、切块和召回问题。

**但索引会重新有价值**：代码库到 GB 级、跨仓库、用户查询经常是"哪里实现了某个概念"时（Cursor 会做 tree-sitter 切块、Merkle Tree 增量同步、embedding + 倒排索引结合）。

> **判断标准不是"代码能不能 RAG"，而是：关键词搜索和结构化工具是否已经足够覆盖主要工作流。** 已经够 → RAG 是复杂化；跨语言、跨仓库、概念召回成为瓶颈 → 索引才是合理成本。

### 边界 2：聚合查询和全局量词验证 → 用 query engine

RAG 擅长"有没有证据支持这句话"，不擅长"所有 tuple 是否都满足条件""满足比例是否超过 50%""哪个 group 排第一"。核心不是 evidence retrieval，而是 **global verification**。Evergreen[10] 把语义聚合后的自然语言 claim 拆成可执行的 semantic verification query；UQE[15] 把非结构化数据分析转成 query engine 问题，用 UQL、采样和优化处理条件聚合、语义检索、抽象聚合。

> ⚠️ 硬用 RAG 最危险的是：模型召回几个看似相关的样本，然后把**局部证据误读成全局结论**。

### 边界 3：个人助手 → 用 Memory，而不是 RAG

个人助手需要的是长期可用、可审计、可更新的**个人状态**。**Memory 和 RAG 的差别**：RAG 的输出通常是上下文片段，Memory 则是**可被决策利用的 external state**。

一个严肃的 Memory 系统至少要有 **Raw Ledger / Derived Views / Policy**：原始事件可追溯、派生视图面向使用、读写策略可记录可回放。否则所谓"长期记忆"很容易退化成**越存越乱的向量库**——模型每次从里面捞几段相似旧话，再把偶然相似误认为用户偏好。

> 这解释了一个现象：**"grep beats RAG"**。像 nanobot 这类小型个人助手，核心代码和知识规模都不大，直接用文件系统管理 prompt、技能和上下文，懒加载详细内容，比上来搭一套向量检索更稳。个人资料天然有**目录、日期、项目、联系人、任务**这些结构信号——Markdown、Wiki、配置文件、事件日志和显式工具调用，比"切块 embedding 后召回"更贴近真实需求。**RAG 可以作为读取 Derived Views 的一种方式，但不该冒充整个 Memory 系统。**

### 边界 4：持续监控 → 用 Continuous RAG

传统 RAG 是 **episodic** 的：问一次、检索一次、回答一次。但告警、舆情、股票新闻、客服风险、数据质量监控不是一次性问答，而是**长时间运行的语义管道**。Continuous Prompts / Continuous RAG 把 LLM 放进流处理系统：Semantic Filter、Map、Aggregate、Top-k、Join、Window、Group-By 持续运行，并用 batching、operator fusion、shadow runs、优化器在吞吐和准确率之间选 plan[16]。

> 在这类场景里，把每次新事件都包装成一次 RAG 问答，会**丢掉系统状态、窗口语义和增量优化空间**。

### 边界 5：复杂业务流程 → 用 Plan-and-Execute + 经验沉淀

小红书服务端端到端测试的案例里，团队面对跨域、长链路、组合爆炸的问题，并没有把 RAG 当主解法，而是用 **Plan-and-Execute + ReAct**：让 Agent 逆向推导依赖链、渐进式加载知识库、Debug-first 调通接口，再把成功链路沉淀为脚本。

> 这里 **RAG 召回不准的代价太高**——一次错误召回可能直接让 Agent 调错接口、造错数据、污染后续步骤。复杂流程里知识库可以存在，但更关键的是**计划、工具、验证和经验沉淀**，而不是一次检索命中率。

### 不该用 RAG 的简化判断

| 问题需要 | 用什么 |
| --- | --- |
| 精确定位 | grep / glob / read |
| 全局统计 | Query Engine / Semantic Operator |
| 长期个性化 | Memory / LLM Wiki |
| 持续运行 | Continuous RAG |
| 可控执行 | Workflow / Agent loop / Harness |

> RAG 仍然重要，但它**不应该是默认答案**。它适合**补一段证据**，不适合替代搜索工具、数据库、记忆系统、流处理系统和工程闭环。

## 9. 复习版结论

> **按信息缺口选技术，而不是按技术潮流选技术。**

- 问题简单、文档规模小、关键词明确 → Naive RAG 或 grep/glob 可能已经够用；
- 问题表达模糊、召回不稳定 → query rewriting + 混合检索 + 重排 + 评测闭环（Advanced RAG）；
- 出现多知识库/多检索器/多任务类型/多轮调用 → Modular RAG，把策略路由和证据打包做成独立层；
- 知识跨文档关联很强 → GraphRAG（离线/全局）或 LightRAG（在线/高频更新）；
- 需要多跳搜索、反复确认、跨数据源联动 → Agentic RAG；
- 知识稳定且被反复使用 → LLM Wiki / GBrain 这类预编译方案；
- 上下文太长、成本太高 → 压缩与路由（先问复用性、错误代价、真实瓶颈）；
- 系统答错 → 先用 Context Precision / Recall / Faithfulness / Answer Relevancy 定位问题在检索层还是生成层。

**选型时不要先问"我要不要上 RAG"，而要先问三个更具体的问题：**

1. **模型缺的是什么信息？**
2. **这些信息是稳定的、实时的、结构化的，还是分散在文档和工具里？**
3. **补上这些信息的最低成本方式是什么？**

答案可能是向量检索，也可能是知识图谱、Agent 工具调用、文件系统搜索、Memory、Wiki、SQL-like 语义查询，或者只是**一个更清楚的业务流程**。

> **最终**：RAG 的价值不在于"让模型多看点资料"，而在于**把用户的一句话，扩展成模型完成任务真正需要的上下文**。工程上真正要优化的，也不是某个单点算法，而是从**问题理解 → 信息获取 → 上下文组织 → 回答生成 → 评测反馈**的整条链路。

## 参考资料（原文 26 条，按序号）

| # | 文献 |
| --- | --- |
| 1 | Anthropic: Steering Claude Code: CLAUDE.md files, skills, hooks, rules, subagents and more |
| 2 | VimRAG: Navigating Massive Visual Context in RAG via Multimodal Memory Graph (arXiv:2602.12735) |
| 3 | Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks (arXiv:2005.11401) |
| 4 | RAG for Large Language Models: A Survey (arXiv:2312.10997) |
| 5 | Modular RAG: Transforming RAG Systems into LEGO-like Reconfigurable Frameworks (arXiv:2407.21059) |
| 6 | From Local to Global: A Graph RAG Approach to Query-Focused Summarization (arXiv:2404.16130) |
| 7 | LightRAG: Simple and Fast Retrieval-Augmented Generation (arXiv:2410.05779) |
| 8 | Graphify — https://github.com/safishamsi/graphify |
| 9 | Google Research: Unlocking dependable responses with Gemini Enterprise Agent Platform's Agentic RAG |
| 10 | Evergreen: Efficient Claim Verification for Semantic Aggregates (arXiv:2604.26180) |
| 11 | Andrej Karpathy: LLM Wiki (gist) |
| 12 | Retrieval as Reasoning: Self-Evolving Agent-Native Retrieval via LLM-Wiki (arXiv:2605.25480) |
| 13 | Prompt Compression for Large Language Models: A Survey (arXiv:2410.12388) |
| 14 | LLMLingua-2: Data Distillation for Efficient and Faithful Task-Agnostic Prompt Compression (arXiv:2403.12968) |
| 15 | UQE: A Query Engine for Unstructured Databases (arXiv:2407.09522) |
| 16 | Continuous Prompts: LLM-Augmented Pipeline Processing over Unstructured Streams (arXiv:2512.03389) |
| 17 | Ragas Documentation — https://docs.ragas.io/en/latest/getstarted/ |
| 18 | Traversal-as-Policy: Log-Distilled Gated Behavior Trees (arXiv:2603.05517) |
| 19 | Is Semantic Chunking Worth the Computational Cost? (arXiv:2410.13070) |
| 20 | Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models (arXiv:2409.04701) |
| 21 | Intent-Driven Dynamic Chunking (arXiv:2602.14784) |
| 22 | Precise Zero-Shot Dense Retrieval without Relevance Labels（HyDE，arXiv:2212.10496） |
| 23 | Expand, Rerank, and Retrieve: Query Reranking for Open-Domain QA（EAR，arXiv:2305.17080） |
| 24 | AutoRAG (arXiv:2410.20878) |
| 25 | QuIM-RAG: Inverted Question Matching for Enhanced QA (arXiv:2501.02702) |
| 26 | OpenViking — https://github.com/volcengine/OpenViking |

## 关联笔记

- [[Clippings/微信公众号/2026-09-24-给项目建Agent知识库-一套完整方法来了-Datawhale]] — 知识库的另一半：RAG 负责「找得到」，专家底座负责「第一眼看对」
- [[Clippings/微信公众号/2026-09-01-RAG核心知识全解析（范式演进）]] — RAG 面试向基础 + 范式演进（Vector RAG / BM25 / GraphRAG / SAG / PageIndex）
- [[Clippings/微信公众号/2026-08-04-GraphRAG与LightRAG-小林面试笔记]] — GraphRAG / LightRAG 对比
- [[RAG与LLM Wiki：原理、架构与开源实现]] — LLM Wiki 三层结构的本地沉淀
- [[rag_学习资料_从基础检索到知识工程]] — RAG 学习资料索引
- [[Agent 上下文压缩机制：六类核心机制与主流 Agent 实现]] — 上下文压缩机制（对应本文第 6 节）
- [[Claude Code记忆系统与Agent记忆架构]] — Memory vs RAG 的区别（对应本文边界 3）
- [[Agent_Skill_注入机制_Progressive_Disclosure与Zero_Resident]] — 上下文按需加载
- [[wiki/index]] — Karpathy LLM Wiki（对应本文第 5 节的实践载体）
