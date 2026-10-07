# RAG（Retrieval-Augmented Generation）学习笔记：从基础检索到知识工程

> [!summary]
> RAG（Retrieval-Augmented Generation，检索增强生成）的核心思想是：**在 LLM 生成回答之前，从外部知识源中检索与问题相关的信息，将这些信息加入当前 Context，再由 LLM 基于增强后的上下文生成答案。**
>
> 现代 RAG 已经不再只是“Embedding → Top-K → Prompt → LLM”，而逐渐发展为一套完整的 **Knowledge Engineering + Retrieval Engineering + Context Engineering** 体系。

---

## 1. RAG 到底是什么？

RAG 的全称是：

> **Retrieval-Augmented Generation**
>
> 检索增强生成

三个词可以分别理解为：

### 1.1 Retrieval：检索

从外部知识库、文档库、数据库或其他信息源中，找到与用户问题相关的内容。

```text
用户问题
  ↓
Retriever
  ↓
相关文档 / Chunk / 知识
```

本质是：

> **找资料。**

---

### 1.2 Augmented：增强

“增强”不是说修改或增强 LLM 本身，而是：

> **用检索得到的外部知识增强 LLM 当前的 Context。**

普通问答：

```text
用户问题
  ↓
LLM
```

RAG：

```text
用户问题
  +
检索到的外部知识
  ↓
增强后的 Context
  ↓
LLM
```

因此可以记成：

> **Augmented = 给生成模型临时提供额外的信息依据。**

---

### 1.3 Generation：生成

LLM 根据：

- 用户问题
- 系统指令
- 对话历史
- 检索到的知识
- 其他工具结果

组织语言、推理并生成最终回答。

因此：

> **Generation = 基于当前 Context 生成最终结果。**

---

## 2. RAG 的最基本流程

最简单的 RAG 可以理解成：

```text
用户问题
   ↓
Query
   ↓
Retriever
   ↓
找到相关知识
   ↓
Context Construction
   ↓
把知识放入 Context
   ↓
LLM
   ↓
最终回答
```

一个非常典型的 Prompt 结构可以抽象为：

```text
System:
你是公司的智能助手。

Context:
一线城市住宿标准：500 元/晚
二线城市住宿标准：400 元/晚
其他城市住宿标准：300 元/晚

User:
公司的差旅住宿标准是多少？
```

因此，从第一性原理看：

> **RAG 本质上就是一种“外部知识 → Context → LLM”的信息获取机制。**

但现代 RAG 的重点已经从“把文本塞进 Prompt”扩展到了：

- 如何组织知识
- 如何检索知识
- 如何融合多种检索信号
- 如何重排序
- 如何更新知识
- 如何控制 Context
- 如何让 Agent 自主选择检索策略

---

# 3. RAG、Fine-tuning、Tool Calling、Memory 的区别

| 技术 | 核心思想 |
|---|---|
| **RAG** | 不修改模型参数，把外部知识临时检索进 Context |
| **Fine-tuning** | 修改模型参数，让模型学习特定行为/知识 |
| **Tool Calling** | 让模型运行时调用外部能力 |
| **Memory** | 保存过去的信息，在之后需要时找回来 |

可以简单理解：

```text
Fine-tuning
→ 知识 / 能力“学进去”

RAG
→ 知识“拿过来给模型看”

Tool Calling
→ 能力“调用起来”

Memory
→ 信息“保存下来以后再找”
```

---

# 4. Embedding 是什么？

RAG 检索经常需要判断：

> “用户问题和哪些文本在语义上更相关？”

Embedding 模型可以把文本转换成数值向量。

例如：

```text
“苹果手机什么时候发布？”
       ↓
Embedding Model
       ↓
[0.12, -0.31, 0.52, ...]
```

知识库中的文档也会提前转换为向量：

```text
Document A → [0.10, -0.29, 0.50, ...]
Document B → [0.71,  0.12, -0.08, ...]
```

然后通过向量相似度进行召回。

---

## 4.1 RAG Embedding 与 Transformer 内部 Embedding

两者相关，但不能直接画等号。

Transformer 内部有 token embedding 等各种中间表示，而 RAG 中常说的 **Embedding Model** 通常指：

> 专门训练来生成适合语义检索的文本表示。

例如：

```text
完整文本
   ↓
BGE / E5 等 Embedding Model
   ↓
文本向量
   ↓
向量数据库
```

---

# 5. BGE-M3 是什么？

**BGE-M3** 是 BGE（BAAI General Embedding）系列中的一个多语言、多功能检索模型。

它可以支持多种检索表示 / 检索方式，典型包括：

```text
BGE-M3
 ├── Dense Representation
 ├── Sparse Lexical Representation
 └── Multi-Vector / ColBERT-style Representation
```

因此它和 RAG 的关系主要在：

> **Retrieval，而不是最终 Generation。**

可以理解为：

```text
BGE-M3
→ 帮你“找什么”

LLM
→ 帮你“怎么回答”
```

---

# 6. 稠密嵌入（Dense）与稀疏表示（Sparse）

这是 RAG 检索中最重要的基础概念之一。

## 6.1 Dense Embedding：稠密嵌入

一整段文本被编码成一个固定长度的向量：

```text
Document
  ↓
Embedding Model
  ↓
[0.12, -0.34, 0.56, ...]
```

特征：

- 维度通常较高
- 大多数维度都有数值
- 关注**语义相似性**

例如：

```text
“苹果手机”
≈
“iPhone”
```

即使两个文本的词面并不完全一致，只要语义相近，Dense Retrieval 也可能找到它们。

核心问题：

> **“意思像不像？”**

---

## 6.2 Sparse Retrieval / Sparse Representation：稀疏

稀疏表示可以理解为：

> 在巨大的词汇维度空间中，只有少量词对应的维度真正有值。

抽象表示：

```text
{
  “苹果”: 1.8,
  “iPhone”: 2.4,
  “发布”: 1.5
}
```

其他大量词对应的权重接近 0。

传统 **BM25** 通常被归入这一类词法 / 稀疏检索思路。

它擅长：

- 专有名词
- 产品型号
- 版本号
- API 名称
- 错误码
- 精确短语
- 文档编号
- 代码符号

核心问题：

> **“关键词对不对得上？”**

---

## 6.3 Dense 与 Sparse 的直觉区别

例如 Query：

> 如何给 PostgreSQL 添加向量搜索能力？

Dense 可能找到：

> 如何使用 pgvector 构建语义搜索

因为：

```text
向量搜索 ≈ 语义搜索
```

而 BM25 更容易准确抓住：

```text
PostgreSQL
pgvector
```

因此：

```text
Dense
→ 解决“说法不同但意思相同”

Sparse
→ 解决“必须是这个词 / 标识符”
```

---

# 7. Multi-Vector：多向量怎么理解？

这是和普通 Dense Embedding 很容易混淆的概念。

## 单向量

```text
整段文本
   ↓
一个向量
```

即：

```text
Document → [1536 维]
```

---

## 多向量

不是把向量做得更长，而是：

```text
Document
  ↓
多个局部向量
```

例如抽象成：

```text
PostgreSQL → Vector₁
支持       → Vector₂
pgvector   → Vector₃
扩展       → Vector₄
```

实际上通常可以理解为保留 token / 局部语义级别的多个向量：

```text
Document
 ↓
[
  Vector₁,
  Vector₂,
  Vector₃,
  ...
]
```

因此：

```text
Single Vector
= 1 × D

Multi-Vector
= N × D
```

这里：

- `N` = 向量数量
- `D` = 每个向量的维度

### 为什么需要多向量？

因为把整个文档压缩成一个向量，会损失一些局部信息。

多向量可以实现更细粒度的 Query-Document 匹配。

经典思想可以用 **ColBERT** 来理解：

```text
Query tokens
    ↓
Q1 Q2 Q3

Document tokens
    ↓
D1 D2 D3 D4 D5

Q1 → 找最匹配的 D
Q2 → 找最匹配的 D
Q3 → 找最匹配的 D
```

因此可以记住：

> **Dense 看整体语义，Sparse 看词法匹配，Multi-Vector 看更细粒度的局部 / Token 级匹配。**

---

# 8. 混合检索（Hybrid Retrieval）

当 Dense 和 Sparse 各有优势时，可以让它们并行工作。

典型架构：

```text
                    用户 Query
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
           Dense                  BM25
          Retrieval             Retrieval
              ↓                     ↓
            Top-K                 Top-K
              │                     │
              └──────────┬──────────┘
                         ↓
                     Fusion
                         ↓
                       RRF
                         ↓
                     Reranker
                         ↓
                       Top-N
                         ↓
                      Context
                         ↓
                        LLM
```

---

## 8.1 为什么需要 Hybrid？

### 只用 Dense

可能会出现：

```text
PostgreSQL 17
```

和：

```text
PostgreSQL 16
```

语义上非常接近。

但某些问题需要精确区分版本号。

### 只用 BM25

又可能无法很好理解：

```text
kitty
≈
cat
≈
feline
```

因此：

> **Dense 负责语义，Sparse 负责词面，两者结合通常比单一路线更稳健。**

---

## 8.2 Hybrid Retrieval 是不是现在更常见？

需要避免绝对化地说“所有生产 RAG 都是 Hybrid”。

更准确的判断是：

- Demo / 入门系统中，Dense-only 很常见
- 生产型 RAG 中，**Dense + Sparse Hybrid 已经是非常主流的方案之一**
- 对技术文档、企业知识库、代码、法律等需要精确关键词的场景尤其有价值
- 是否使用 Hybrid 应根据查询和数据特点决定

因此不要把：

> Hybrid = 所有 RAG 的硬性标准

作为结论。

---

# 9. RRF 是什么？

当 Dense 与 BM25 各自得到一份排序时，需要将两份结果融合。

例如：

```text
Dense:
1. A
2. C
3. B

BM25:
1. B
2. A
3. D
```

不能简单直接相加 cosine similarity 和 BM25 分数，因为二者量纲不同。

**RRF（Reciprocal Rank Fusion）** 是一种常见的排名融合方法。

它主要看：

> **一个文档在每个检索器中排第几。**

一个文档：

- Dense 排得高
- BM25 也排得高

那么它的 RRF 得分就更有优势。

因此：

```text
Dense Ranking
      \
       → RRF → 新的融合排序
      /
BM25 Ranking
```

### RRF 与 Reranker 的关系

非常容易混淆：

> **RRF 自己就会产生一个新的排序。**

然后如果需要，还可以：

```text
Dense / BM25
   ↓
RRF
   ↓
RRF 融合后的候选排序
   ↓
Reranker
   ↓
最终精排
```

所以不是说 RRF “排错了”。

而是：

> **RRF 更适合作为高效的粗粒度融合，神经 Reranker 再进行更精确的排序。**

---

# 10. Top-K / Top-N / Top-P

### Top-K / Top-N

在检索里，本质上都是：

> **取排名前 K / N 个结果。**

例如：

```text
Top-10
→ 取前 10 个文档
```

Top-K 和 Top-N 在 RAG 检索语境下基本可以视为同一类概念，只是符号不同。

### Top-P

Top-P 是 LLM 生成阶段的**概率采样**概念。

它不是：

> “取前 P 个”

而是选择累计概率达到阈值 P 的 token 候选集合。

因此：

```text
RAG Retrieval
→ Top-K / Top-N

LLM Generation
→ Top-P
```

---

# 11. 神经重排序（Neural Reranking）

Reranker 的核心思想：

> **先快速召回一批候选，再用更强的模型逐个判断 Query 与 Document 的真实相关性。**

流程：

```text
Query
 ↓
Dense / BM25 / Hybrid
 ↓
Top-100 candidates
 ↓
Reranker
 ↓
对 (Query, Document) 逐个打分
 ↓
重新排序
 ↓
Top-5
```

因此：

> **Recall 负责“别漏掉”，Rerank 负责“把最好的排到前面”。**

---

# 12. 双编码器（Dual Encoder）

双编码器是高效召回模型的常见架构。

```text
Query ──→ Encoder ──→ Query Vector
                           │
                           ├→ Similarity
                           │
Doc   ──→ Encoder ──→ Doc Vector
```

Query 和 Document **分别编码**。

这样文档向量可以离线提前计算并存入向量数据库。

所以非常适合：

> **大规模检索 / 第一阶段召回。**

BGE-M3、E5 等 Embedding 模型可以放在这类体系中。

---

# 13. 跨编码器（Cross Encoder）

跨编码器则把 Query 和 Document **一起输入模型**：

```text
Query + Document
       ↓
Cross Encoder
       ↓
Relevance Score
```

例如：

```text
Query:
PostgreSQL 向量搜索

Document:
PostgreSQL + pgvector ...
       ↓
Reranker
       ↓
0.96
```

优点：

- Query 和 Document 能进行更深的交互
- 通常相关性判断更准确

缺点：

- 每一个候选都要过一次模型
- 计算成本更高

因此：

> **双编码器主要负责召回，跨编码器主要负责重排序。**

---

# 14. RAG 的一个典型生产级检索链

把前面的内容全部串起来：

```text
                     User Query
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
        Dense Retriever          Sparse Retriever
          Dual Encoder              BM25
             ↓                       ↓
           Top-K                   Top-K
             │                       │
             └──────────┬────────────┘
                        ↓
                      RRF
                        ↓
                 Candidate Set
                        ↓
               Cross-Encoder
                  Reranker
                        ↓
                     Top-N
                        ↓
                 Context Build
                        ↓
                       LLM
                        ↓
                     Answer
```

这是理解传统高质量 RAG 的核心架构之一。

---

# 15. 什么时候传统 RAG 已经足够？

如果问题主要是：

> **“帮我找到相关内容并回答。”**

例如：

> “SSE4.2 是什么？”

那么典型：

```text
Dense
 +
Sparse
 +
Reranker
```

往往已经足够。

可以记成：

```text
传统 RAG
= 检索 → 排序 → 生成
```

并不需要一开始就上 GraphRAG / RAPTOR。

---

# 16. 为什么需要 RAPTOR 和 GraphRAG？

随着知识库变大，会出现另外的问题：

> **不是“找不到相关 chunk”，而是“知识本身的组织方式不适合这个问题”。**

这就是结构化索引的价值。

---

# 17. RAPTOR

RAPTOR（Recursive Abstractive Processing for Tree-Organized Retrieval）的核心思想：

> **把原始 Chunk 聚类并总结，递归形成多层级的知识树。**

抽象结构：

```text
                         总体总结
                       /          \
                 主题 A            主题 B
                / |  \            / |  \
              C1 C2  C3         C4 C5  C6
```

底层：

```text
C1 C2 C3 ...
```

上层：

```text
这些 Chunk 的总结
```

再上层：

```text
这些主题的总体总结
```

因此可以支持不同抽象层次的检索。

### RAPTOR 适合的问题

例如：

> “从 CPU 的整体 SIMD 架构讲到 AVX 和 SSE，再讲到具体指令。”

这类问题有明显的层级导航需求。

可以理解为：

```text
整体概念
  ↓
子主题
  ↓
具体细节
```

因此：

> **RAPTOR 更偏向“从抽象到具体的层级知识导航”。**

---

# 18. GraphRAG

GraphRAG 的核心思想：

> **把知识组织为实体（Entity）和关系（Relation）构成的图。**

例如：

```text
OpenAI ──开发──→ GPT
   │
   └──合作──→ Microsoft
                    │
                    └──投资──→ OpenAI
```

这里重要的不是某一个 Chunk，而是：

> **实体之间如何连接。**

GraphRAG 特别适合：

- 多跳关系
- 实体关联
- 因果 / 关联网络
- “A 和 B 是什么关系”
- 需要跨多个文档拼接关系的问题

例如：

> “A 公司和 B 公司之间通过哪些项目产生联系？”

这显然比单纯找一个最相似文本更依赖关系结构。

---

# 19. RAPTOR 与 GraphRAG 的区别

| | RAPTOR | GraphRAG |
|---|---|---|
| 核心结构 | **树** | **图** |
| 重点 | 层级与抽象 | 实体与关系 |
| 适合 | 从概念深入细节 | 多跳关系查询 |
| 典型问题 | “整体 → 子主题 → 细节” | “A 和 B 有什么关系？” |

可以记成：

```text
RAPTOR
→ 如何分层组织知识？

GraphRAG
→ 知识之间如何连接？
```

---

# 20. RAPTOR / GraphRAG 是否一定比传统 RAG 高级？

不要这么理解。

它们不是：

```text
普通 RAG
 ↓
RAPTOR
 ↓
GraphRAG
```

这样的线性升级路线。

更准确地说：

```text
传统 RAG
→ 解决“找相关内容”

RAPTOR
→ 解决“按层级组织和检索知识”

GraphRAG
→ 解决“利用关系组织和检索知识”
```

是否需要它们取决于：

> **问题是否真的需要这种结构。**

如果数据简单、问题简单，传统 Hybrid Retrieval 可能已经很好。

---

# 21. RAPTOR / GraphRAG / OpenViking 的位置

三者都可以帮助 Agent 更有效地获取知识，但解决的问题并不完全相同。

| 技术 | 核心思路 | 主要解决什么 |
|---|---|---|
| **RAPTOR** | 树 / 层级摘要 | 多层次知识导航 |
| **GraphRAG** | 图 / 实体 / 关系 | 关系型、多跳检索 |
| **OpenViking** | 文件系统式上下文组织 | 上下文统一管理、定位、逐层加载 |

---

# 22. OpenViking：文件系统范式

OpenViking 可以理解为：

> **用文件系统的思想来组织 Agent 的上下文与知识。**

传统 RAG 更像：

```text
Document
 ↓
Chunk
 ↓
Vector DB
 ↓
Top-K
```

知识基本是扁平的。

OpenViking 更强调：

```text
viking://
├── resources/
├── memories/
└── skills/
```

可以把 Agent 的：

- Resources
- Memories
- Skills
- 其他上下文

统一放进一个可导航的知识空间。

### 它解决的问题

1. **上下文碎片化**
2. **知识缺乏可导航结构**
3. **不希望一次性把大量完整内容加载进 Context**

因此可以使用类似：

```text
L0：摘要
 ↓
L1：概览
 ↓
L2：完整内容
```

的分层加载方式。

Agent 可以：

```text
先知道“这是什么”
      ↓
判断“要不要深入”
      ↓
需要时再加载完整内容
```

因此更准确地说：

> **RAPTOR / GraphRAG / OpenViking 都在改善“知识获取”，但 OpenViking 更偏向 Context / Knowledge Management Layer，而不是单纯的 Retrieval Algorithm。**

---

# 23. 为什么现代 RAG 不只是“向量搜索”？

因为真实知识系统至少有以下问题：

```text
① 怎么组织知识？
② 怎么检索知识？
③ 怎么排序知识？
④ 怎么更新知识？
⑤ 怎么控制上下文？
⑥ 怎么处理关系？
⑦ 怎么处理层级？
⑧ 怎么让 Agent 决定检索策略？
```

因此：

> **现代 RAG 已逐渐从单纯的 Retrieval 问题，扩展成完整的知识工程与上下文工程问题。**

---

# 24. 原始数据为什么不能直接塞进知识库？

这是现代 RAG 非常重要的一条原则：

> **“存进去” ≠ “知识已经可以被可靠利用”。**

一个简单的方案：

```text
原始文档
 ↓
Chunk
 ↓
Embedding
 ↓
Vector DB
```

看起来完成了 RAG，但原始数据可能存在：

- 冗余
- 噪声
- 重复案例
- 隐含规则
- 缺乏结构
- 表述不一致
- 上下文缺失

即使拥有很大的 Context，也不代表模型就能稳定从这些原始材料中归纳出正确知识。

因此：

> **Context 很大 ≠ 知识已经组织好了。**

---

# 25. 原始案例 → 可用知识

假设有 100 个客户案例：

```text
案例 1
案例 2
...
案例 100
```

直接放进去，用户问：

> “这个问题通常发生在什么情况下？”

RAG 可能只是返回：

```text
案例 17
案例 42
案例 83
```

但真正有价值的信息可能是：

```text
100 个案例
 ↓
分析
 ↓
72% 发生在高并发
18% 与配置错误有关
10% 与网络异常有关
```

因此：

> **100 个个体案例 → 统计摘要**

这是一种知识提炼。

---

# 26. 工单 → 规则

例如几百张工单：

```text
工单 1
工单 2
...
工单 500
```

它们真正有价值的可能不是逐条历史记录，而是：

```text
条件 A + 条件 B
   ↓
允许退款

条件 C
   ↓
仅部分退款

条件 D
   ↓
不允许退款
```

也就是：

> **散落的个案 → 带边界的明确规则**

“带边界”尤其重要。

例如：

```text
错误总结：
拆封商品一般不能退款。

更好的规则：
超过 7 天
AND
商品已拆封
→ 不支持无理由退款。
```

真正可用的知识需要尽量明确：

- 条件
- 阈值
- 适用范围
- 例外
- 边界

---

# 27. 现代 RAG 的“知识加工”流水线

因此，一个更完整的数据处理过程应该是：

```text
                 原始数据
                     ↓
              ① 数据清洗
                     ↓
              ② 内容解析
                     ↓
          ③ 知识提炼 / 结构化
                     ↓
        ┌────────────┼────────────┐
        ↓            ↓            ↓
      摘要          规则        实体/关系
        │            │            │
        └────────────┼────────────┘
                     ↓
              ④ Chunk 切分
                     ↓
          ⑤ Context Enrichment
                     ↓
        ⑥ Dense / Sparse / Multi-Vector
                     ↓
                 ⑦ Index
                     ↓
                Retrieval
                     ↓
                  Rerank
                     ↓
                 Context
                     ↓
                    LLM
```

---

# 28. Chunking：传统 RAG 最基础、也最容易被忽视的问题

最简单的切法：

```text
每 500 tokens 切一次
```

但它可能切出：

```text
“它在这种情况下性能更好。”
```

这个 chunk 单独看几乎没有意义。

更好的做法是考虑：

- 标题
- 章节
- 段落
- 父级主题
- 上下文关系

例如把它增强成：

```text
PostgreSQL 的 MVCC 机制在高并发事务环境下性能更好。
```

于是 Embedding 得到的语义表示更完整。

---

# 29. Context-aware Retrieval

Context-aware Retrieval 不是“比 Agentic RAG 更高一级”。

这是一个非常重要的理解。

错误的理解：

```text
普通 RAG
 ↓
Hybrid
 ↓
RAPTOR
 ↓
Agentic RAG
 ↓
Context-aware Retrieval
```

实际上 Context-aware Retrieval 是：

> **回到 RAG 最基础的 Chunk 层，改善 Chunk 本身的可检索性。**

典型过程：

```text
原文上下文
   ↓
理解 Chunk 所在位置
   ↓
给 Chunk 增加上下文
   ↓
Context-aware Chunk
   ↓
Embedding
```

所以它更像：

> **修补最底层的数据表示问题。**

---

# 30. Agentic RAG

传统 RAG 往往固定为：

```text
Query
 ↓
Embedding
 ↓
Vector Search
 ↓
Top-K
 ↓
LLM
```

但是复杂查询可能需要多个步骤：

```text
先查 A
 ↓
根据结果决定查 B
 ↓
再查 C
 ↓
综合
```

Agentic RAG 把“如何搜索”交给 Agent：

```text
                     Agent
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Vector         BM25        Graph
       Search        Search       Search
          │            │            │
          └────────────┼────────────┘
                       ↓
                     判断
                       ↓
                   再次检索
```

因此：

> **Agentic RAG 的重点是“让 Agent 自主决定检索策略”。**

---

# 31. 知识应该如何更新？

知识库不仅需要“建”，还需要“维护”。

可以分成两个基本方向。

## 31.1 增量更新（Incremental Update）

来了新数据：

```text
新文档 / 新案例 / 新工单
       ↓
只处理变化部分
       ↓
更新对应索引
```

优点：

- 快
- 成本低
- 适合持续变化的知识

核心思想：

> **新知识来了就及时吸收。**

---

## 31.2 全量整理（Full Reorganization）

定期重新审视整个知识库：

```text
整个知识库
 ↓
重新分析
 ↓
发现冲突
 ↓
去重
 ↓
合并
 ↓
重新建立知识结构
```

优点：

- 能发现历史问题
- 能处理长期积累的冗余
- 能重新发现全局规律

因此：

```text
增量更新
= 持续吸收

全量整理
= 定期重审
```

现实系统中两者往往需要组合。

---

# 32. 从结构化数据提取深度知识

RAG 不只有：

- PDF
- Markdown
- 网页
- 文本

还有：

- Excel
- SQL
- 数据库
- 表格
- 业务数据集

例如：

```text
客户 | 产品 | 金额 | 时间
A    | P1   | 100  | 2025
B    | P2   | 200  | 2026
```

最简单的做法只是检索某一行。

但更高级的知识系统希望从结构化数据中得到：

- 趋势
- 异常
- 聚合关系
- 变量之间的关系
- 统计结论
- 业务规则

因此：

```text
Structured Dataset
       ↓
Analysis / Aggregation
       ↓
Knowledge Extraction
       ↓
更高层次的知识
```

这和“把每一行当成一个 Chunk”是完全不同的思路。

---

# 33. 六个方向到底在解决什么？

可以用一张表建立整体地图：

| 方向 | 主要问题 |
|---|---|
| **RAPTOR** | 知识如何形成层级结构？ |
| **GraphRAG** | 知识之间有什么关系？ |
| **OpenViking** | 知识和 Agent Context 如何统一管理、导航、分层加载？ |
| **增量 / 全量更新** | 知识发生变化后怎么维护？ |
| **Agentic RAG** | 谁来决定如何检索？ |
| **Context-aware Retrieval** | 每个 Chunk 如何本身就更容易被正确检索？ |
| **结构化数据知识提取** | 如何从表格 / 数据集提炼深层知识？ |

它们**不是一条从低级到高级的直线**。

更准确的关系是：

```text
                 RAG / Knowledge System
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
   知识怎么组织       知识怎么维护       怎么检索
       │                 │                 │
 RAPTOR / Graph      增量 / 全量       Hybrid / Agentic
       │                                   │
       ├────────── Context-aware ──────────┤
       │
       └──── Structured Data → Knowledge ──┘
```

---

# 34. RAG 最值得形成的整体认知

可以把现代 RAG 分成三个层面。

## Layer 1：Knowledge Engineering

解决：

> **原始信息如何变成可用知识？**

包括：

- 清洗
- 解析
- Chunking
- 摘要
- 规则提取
- Entity / Relation
- RAPTOR
- GraphRAG
- Structured Data Knowledge Extraction
- 知识更新

---

## Layer 2：Retrieval Engineering

解决：

> **需要时如何找到正确知识？**

包括：

```text
Dense
+
Sparse / BM25
+
Hybrid
+
RRF
+
Metadata Filter
+
Reranker
+
Multi-Vector
```

---

## Layer 3：Context / Agent Engineering

解决：

> **找到以后，怎么让 Agent 有效使用？**

包括：

- Context Construction
- Context-aware Retrieval
- Agentic RAG
- Memory
- Tool Calling
- Context Compression
- 分层加载
- Runtime 编排

最终：

```text
Knowledge
    ↓
Retrieve
    ↓
Rerank
    ↓
Context
    ↓
LLM / Agent
    ↓
Answer / Action
```

---

# 35. 一张完整的 RAG 全景图

```text
                              ┌──────────────────────┐
                              │      原始数据         │
                              │ PDF / Web / DB / ... │
                              └──────────┬───────────┘
                                         ↓
                              ┌──────────────────────┐
                              │  Knowledge Engineering│
                              ├──────────────────────┤
                              │ 清洗 / 解析 / Chunk   │
                              │ 摘要 / 规则 / 实体    │
                              │ RAPTOR / GraphRAG    │
                              │ Structured Knowledge │
                              └──────────┬───────────┘
                                         ↓
                              ┌──────────────────────┐
                              │        Index         │
                              ├──────────────────────┤
                              │ Dense                │
                              │ Sparse               │
                              │ Multi-Vector         │
                              │ Metadata             │
                              └──────────┬───────────┘
                                         ↓
                                      Query
                                         ↓
                 ┌───────────────────────┼───────────────────────┐
                 ↓                       ↓                       ↓
              Dense                   Sparse                  Graph
                 ↓                       ↓                       ↓
               Top-K                   Top-K                 Graph Search
                 └───────────────────────┼───────────────────────┘
                                         ↓
                                       RRF
                                         ↓
                                   Candidate Set
                                         ↓
                                  Neural Reranker
                                         ↓
                                      Top-N
                                         ↓
                               Context Construction
                                         ↓
                              Context-aware / Memory
                                         ↓
                                      LLM / Agent
                                         ↓
                                Answer / Tool Action
```

---

# 36. 最终形成一个非常重要的认识

传统 RAG 可以简单理解成：

```text
“找到相关文本，然后让 LLM 看。”
```

但真正成熟的 RAG 更接近：

```text
原始数据
 ↓
知识提炼
 ↓
知识结构化
 ↓
索引
 ↓
多路检索
 ↓
结果融合
 ↓
精确重排
 ↓
上下文构建
 ↓
LLM / Agent 推理
```

因此，RAG 的核心问题已经逐渐从：

> **“怎么做向量搜索？”**

扩展成：

> **“如何把庞杂的外部信息加工成模型能够可靠检索、理解和使用的知识？”**

这是理解后续 **RAPTOR、GraphRAG、Agentic RAG、Context-aware Retrieval、OpenViking、Memory、Knowledge Engineering** 等技术的共同基础。

---

## 37. 建议的学习路线

对于初学阶段，不建议一开始就同时学习所有高级 RAG。

更适合按以下顺序建立知识体系：

```text
① RAG 基础
   ↓
② Embedding
   ↓
③ Dense Retrieval
   ↓
④ BM25 / Sparse Retrieval
   ↓
⑤ Hybrid Retrieval
   ↓
⑥ RRF
   ↓
⑦ Reranker
   ↓
⑧ Chunking / Context-aware Retrieval
   ↓
⑨ 知识加工与数据清洗
   ↓
⑩ RAPTOR
   ↓
⑪ GraphRAG
   ↓
⑫ Agentic RAG
   ↓
⑬ Memory / Context Management
   ↓
⑭ Structured Data Knowledge Extraction
```

其中最值得自己动手实现的第一个完整项目，可以是：

```text
PDF / Markdown
      ↓
Chunking
      ↓
BGE-M3
      ↓
Vector DB
      ↓
BM25
      ↓
Hybrid + RRF
      ↓
Reranker
      ↓
LLM
```

先把这条主干跑通，再逐步加入：

```text
Context-aware Chunk
RAPTOR
GraphRAG
Agentic Retrieval
```

这样可以非常清楚地看到每种技术到底解决了传统 RAG 的哪个具体问题。

---

## 38. 一句话速记表

```text
RAG
→ 把外部知识放进 Context

Embedding
→ 把文本转换为适合检索的表示

Dense
→ 看语义

Sparse / BM25
→ 看关键词

Multi-Vector
→ 看局部 / Token 级匹配

Hybrid
→ Dense + Sparse

RRF
→ 融合多个检索器的排名

Reranker
→ 对候选结果进行精排

Dual Encoder
→ 快速召回

Cross Encoder
→ 精确判断 Query-Document 相关性

RAPTOR
→ 树 / 层级 / 抽象

GraphRAG
→ 图 / 实体 / 关系

OpenViking
→ 文件系统式的 Context / Knowledge 管理

Agentic RAG
→ Agent 决定怎么检索

Context-aware Retrieval
→ 让 Chunk 本身携带足够上下文

Incremental Update
→ 新知识及时吸收

Full Reorganization
→ 定期重新整理全库

Structured Knowledge Extraction
→ 从结构化数据中提炼高层知识
```

> [!tip]
> 最重要的总框架：
>
> **Knowledge Engineering → Retrieval Engineering → Context / Agent Engineering → Generation**
>
> RAG 并不是一个单独的“向量数据库技术”，而是一整套让 LLM 能够可靠利用外部知识的工程方法。
