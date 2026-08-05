---
created: 2026-08-04
updated: 2026-08-04
tags: [Agent, RAG, GraphRAG, LightRAG, 知识图谱, 面试]
source:
  - "https://mp.weixin.qq.com/s/F8L7wlf-8fM3Aa6oX3qBCA"
  - "公众号：小林面试笔记"
evidence: article-summary
---

# GraphRAG 与 LightRAG：从传统 RAG 到图增强检索

> 原文标题：字节二面，我霸气反问：“别光吹RAG，说说GraphRAG的多跳推理，你们线上跑通了吗”
>
> 本笔记以原文为基础重新整理。文中的成本、性能和论文数据属于作者转述，使用前应回到论文或官方文档核验。

## 一句话总结

传统 RAG 擅长“找相似文本”；GraphRAG 通过实体、关系、社区和摘要增强多跳推理与全局归纳；LightRAG 则去掉昂贵的社区预计算，使用轻量图结构和双层检索，换取更低成本、更快查询和更好的增量能力。

## 一、传统 RAG 的边界

```text
文档 → 切块 → Embedding → 向量库 → Top-K 检索 → LLM 生成
```

![传统 RAG 流程](https://gitee.com/cheng-jiaqing/images/raw/master/graphrag-01-traditional.png)

### 主要痛点

1. **多跳推理弱**：只能召回相似文本，没有显式的 A→B→C 关系路径。
2. **全局问题弱**：主题、趋势和整体战略变化通常不在单个 Chunk 中。
3. **切块造成语义断裂**：实体、因果关系和上下文可能被拆散。
4. **检索噪声**：语义相似不代表真正相关。

根本原因：传统 RAG 主要在“找相似文本”，而复杂企业问答需要“理解实体关系”。

## 二、什么是 GraphRAG

GraphRAG 将文档中的实体和关系抽取为知识图谱，再结合图检索、向量检索和社区摘要回答问题。

```text
文档
 → Text Unit
 → 实体/关系抽取
 → 实体与关系归并
 → 知识图谱
 → 社区检测
 → 社区报告
 → Local / Global Search
 → LLM 回答
```

![GraphRAG 索引与查询流程](https://gitee.com/cheng-jiaqing/images/raw/master/graphrag-02-pipeline.png)

| 传统 RAG | GraphRAG |
|---|---|
| 知识以独立文本块存在 | 知识以实体、关系和社区存在 |
| 主要依赖向量相似度 | 结合向量检索和图遍历 |
| 偏局部检索 | 同时支持局部和全局分析 |
| 关系隐含在文本中 | 关系被显式建模 |
| 增量简单 | 增量可能触发级联更新 |

## 三、GraphRAG 的索引与查询

### 1. 索引阶段

1. **文档切块**：生成 Text Unit。
2. **实体与关系抽取**：识别人、组织、地点、事件及其关系。
3. **实体/关系摘要**：合并同一实体或关系在不同文本块中的描述。
4. **社区检测**：使用 Leiden 等算法，把关系紧密的节点划分成层次化社区。
5. **社区报告**：为每个社区生成主题、关键实体、关系和发现的摘要。

社区报告是 GraphRAG 处理全局问题的核心资产。

### 2. Local Search

适合具体实体和局部关系问题。

```text
问题 → 定位入口实体 → 扩展邻居和关系 → 关联原文/社区报告 → LLM 回答
```

![GraphRAG Local Search](https://gitee.com/cheng-jiaqing/images/raw/master/graphrag-03-local.png)

### 3. Global Search

适合整体趋势、主题归纳和跨文档总结。

```text
选择社区层级
 → Map：分别处理社区报告
 → Reduce：汇总并排序中间答案
 → 生成最终回答
```

![GraphRAG Global Search](https://gitee.com/cheng-jiaqing/images/raw/master/graphrag-04-global.png)

Global Search 的代价是需要处理大量社区报告，调用量和延迟都可能较高。

> 原文还提到 DRIFT Search，作为 Local 与 Global 的混合方式；具体实现应参考 GraphRAG 官方版本。

## 四、GraphRAG 的主要难点

### 1. 索引成本高

实体抽取、关系抽取、摘要和社区报告都可能调用 LLM，成本显著高于“切块 + Embedding”的传统 RAG。

### 2. 实体消歧困难

`IBM`、`国际商业机器`、`IBM Corp.` 可能是同一实体；“张三”也可能指不同的人。

错误消歧会导致：

- 节点重复；
- 关系碎片化；
- 查询召回不完整；
- 错误沿图结构传播。

### 3. Global Search 延迟高

Global Search 本质上是社区级 Map-Reduce，社区数量和 LLM 调用数会直接影响延迟。

### 4. 增量更新复杂

新增文档可能依次影响：

```text
实体合并 → 图结构 → 社区划分 → 社区报告 → 向量索引
```

因此 GraphRAG 常采用局部增量、周期性全量重建或时间分区等策略。

## 五、什么是 LightRAG

LightRAG 保留实体和关系图，但通常去掉 Leiden 社区检测和社区摘要，改为查询时动态组合上下文。

```text
文档
 → 切块
 → 实体/关系抽取
 → 去重与合并
 → 图存储 + 向量存储
 → 高层/低层关键词抽取
 → 双层检索
 → LLM 生成
```

### 三个核心设计

1. **轻量图索引**：保留实体和关系，不预生成社区报告。
2. **双层检索**：低层关键词找具体实体，高层关键词找抽象关系或主题。
3. **追加式增量更新**：新文档抽取后直接 upsert 节点、边和向量。

### 双层检索

| 层级 | 主要对象 | 关注内容 |
|---|---|---|
| 低层 | 实体、术语 | 具体事实和局部细节 |
| 高层 | 关系、主题 | 抽象概念和全局方向 |

可将其记为：**低层找“点”，高层找“线”**。

![LightRAG 双层检索](https://gitee.com/cheng-jiaqing/images/raw/master/lightrag-01-dual.png)

### 查询模式

| 模式 | 说明 |
|---|---|
| Naive | 传统向量检索 |
| Local | 低层实体检索与图扩展 |
| Global | 高层关系和主题检索 |
| Hybrid | 低层 + 高层联合检索 |

### LightRAG 的代价

- 去重和实体消歧可能较朴素；
- 同名异人、别名和跨语言实体需要额外治理；
- 新旧事实冲突时，需要时间、版本或有效期机制；
- 复杂全局归纳可能不如带社区摘要的 GraphRAG。

## 六、GraphRAG 与 LightRAG 对比

| 维度 | GraphRAG | LightRAG |
|---|---|---|
| 全局视角 | 社区摘要预计算 | 查询时动态组合 |
| 社区检测 | 有 | 通常没有 |
| 查询方式 | Local / Global / 其他混合模式 | 低层实体 + 高层关系 |
| 索引成本 | 高 | 较低 |
| 查询延迟 | Global Search 较高 | 通常更低 |
| 增量更新 | 复杂 | 更接近追加式更新 |
| 全局洞察 | 更强 | 可能较浅 |
| 工程复杂度 | 高 | 较低 |
| 主要风险 | 成本、延迟、社区失效 | 实体碎片、事实冲突 |

本质区别不是“一个有图、一个没图”，而是：

> GraphRAG 通过社区检测和社区摘要预计算全局视角；LightRAG 放弃这层预计算，在查询时动态检索和拼装。

![GraphRAG 与 LightRAG 对比](https://gitee.com/cheng-jiaqing/images/raw/master/lightrag-02-compare.png)

## 七、如何选型

### 选择传统 RAG

- FAQ、制度查询、产品说明书；
- 主要是单跳事实问答；
- 数据关系简单，成本和速度优先。

### 选择 LightRAG

- 文档频繁更新；
- 成本和延迟敏感；
- 需要一定的实体关系理解；
- 中小规模知识库或轻量部署。

![LightRAG 适用场景](https://gitee.com/cheng-jiaqing/images/raw/master/lightrag-03-scenarios.png)

### 选择 GraphRAG

- 数据相对静态；
- 跨文档多跳推理价值高；
- 需要深度全局分析；
- 对证据链、可解释性和分析质量要求高；
- 能接受更高的构建与维护成本。

### 推荐的生产路线

```text
传统 RAG MVP
  → 观察真实失败案例
  → 对关系类问题引入 LightRAG
  → 对稳定、高价值语料引入 GraphRAG
  → 用 Query Router 按问题类型分流
```

![GraphRAG 与 LightRAG 选型决策](https://gitee.com/cheng-jiaqing/images/raw/master/lightrag-04-decision.png)

## 八、面试版回答

### 什么是 GraphRAG？

GraphRAG 将文档中的实体和关系抽取成知识图谱，再结合图遍历、向量检索和社区摘要完成 RAG。相比只检索相似文本的传统 RAG，它更适合跨文档多跳推理和全局性问题，但代价是索引成本高、实体消歧难、Global Search 延迟高、增量维护复杂。

### 什么是 LightRAG？

LightRAG 是轻量化的图增强 RAG。它保留实体和关系图，通常去掉社区检测和社区摘要，改用高层关键词检索关系、低层关键词检索实体的双层检索机制。因此成本、延迟和增量更新都更友好，但复杂全局归纳和实体治理能力可能弱于 GraphRAG。

### 如何选择？

简单事实问答用传统 RAG；动态、成本敏感且需要关系检索时优先 LightRAG；静态、高价值、需要深度全局分析和多跳推理时考虑 GraphRAG。生产环境可以采用三者混合，并通过评测集比较召回、正确性、延迟和成本。

## 九、需要谨慎核验的内容

原文中的以下内容不宜脱离上下文直接引用：

- GraphRAG 与 LightRAG 的具体 F1、准确率和成本数据；
- “Token 降低 99%”“成本为原版 0.1%”等定量结论；
- 特定数据规模对应的选型阈值；
- 具体查询延迟、社区数量和索引价格；
- 不同论文、项目版本之间的性能比较。

核验时应确认：模型、数据集、Prompt、评测指标、实现版本、是否包含 Embedding/存储/并发成本。

## 十、最终记忆版

```text
传统 RAG：相似文本检索

GraphRAG：实体关系图 + 社区检测 + 社区摘要 + Local/Global Search

LightRAG：实体关系图 + 高低层关键词 + 双层检索 + 轻量增量更新
```

> **一句话选型：简单问题用传统 RAG；动态关系检索用 LightRAG；静态深度全局分析用 GraphRAG。**
