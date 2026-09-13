# Mem0 v2 → v3：记忆架构与 Hybrid Retrieval

> 本笔记整理本轮对话中对 Mem0 两代 Memory 架构，以及 `Semantic / BM25 / Entity / Time` 四种检索信号的讨论。

---

## 1. 核心结论

Mem0 这两代方案最重要的变化，不只是“检索算法变多了”，而是 **Memory 的职责发生了变化**：

> **v2：写入时解决新旧记忆之间的冲突，维护一份较干净的当前状态。**  
> **v3：写入时尽量保留历史，不在写入阶段删除/覆盖旧事实；把“哪条记忆当前最相关”交给检索与排序阶段。**

可以概括为：

```text
v2
Dialogue
  ↓
LLM Extract
  ↓
Vector Search
  ↓
LLM Decide
  ↓
ADD / UPDATE / DELETE / NOOP
```

↓

```text
v3
Dialogue
  ↓
1× LLM Extract
  ↓
ADD-only Store
  ↓
Hybrid Retrieval
  ├─ Semantic
  ├─ BM25
  ├─ Entity
  └─ Time
  ↓
Top-k
```

因此可以把两代架构理解成：

```text
v2 = State-oriented Memory
v3 = History + Retrieval-oriented Memory
```

---

# 2. Mem0 v2（2025）：写入时解决冲突

## 2.1 流程

```text
Dialogue
   ↓
LLM extract candidate facts
   ↓
Vector search existing memory
   ↓
LLM compare new facts with existing facts
   ↓
ADD / UPDATE / DELETE / NOOP
```

核心思想：

> Memory Store 不是简单追加的日志，而是一份需要不断维护的“当前知识状态”。

---

## 2.2 示例

历史 Memory：

```text
User lives in Beijing.
```

新对话：

```text
I moved to Shanghai last month.
```

### Step 1：LLM 提取新事实

```text
User lives in Shanghai.
```

### Step 2：搜索相关旧记忆

找到：

```text
User lives in Beijing.
```

### Step 3：LLM 判断新旧事实关系

例如得到：

```text
Action = UPDATE
```

最后 Memory 变成：

```text
User lives in Shanghai.
```

原来的：

```text
User lives in Beijing.
```

被更新/覆盖。

---

## 2.3 ADD / UPDATE / DELETE / NOOP

v2 的写入决策大致可以理解为：

| 操作 | 含义 |
|---|---|
| `ADD` | 新信息以前不存在，直接加入 |
| `UPDATE` | 新信息和旧信息冲突，用新信息替换旧信息 |
| `DELETE` | 旧信息已经失效，需要移除 |
| `NOOP` | 新信息没有必要改变 Memory |

因此 v2 的核心任务是：

> **维护一个相对干净、一致的 Memory State。**

---

## 2.4 v2 的优点

- Memory 中重复内容较少
- 直接读取时容易理解“当前状态”
- 写入后数据比较干净
- 旧事实和新事实之间可以显式进行状态迁移

---

## 2.5 v2 的问题

### ① 写入流程复杂

一次 Memory 写入可能变成：

```text
LLM Extraction
+
Vector Search
+
LLM Decision
+
Mutation
```

其中第二次 LLM 判断负责：

> 新事实和旧事实到底是什么关系？

---

### ② LLM 可能误判

例如：

```text
Old:
User works at Apple.

New:
User met his friend at Apple yesterday.
```

如果模型错误理解为“用户换工作”，就可能错误地：

```text
UPDATE
```

从而破坏原本正确的 Memory。

---

### ③ 历史信息可能丢失

如果：

```text
2024: User lives in Beijing.
2025: User lives in Shanghai.
```

v2 倾向于把它理解成：

```text
Beijing → Shanghai
```

最终只留下当前状态。

但现实世界中的 Memory 经常不是简单的“旧值错误，新值正确”。

---

# 3. Mem0 v3（2026-04）：ADD-only + Retrieval 决策

v3 的架构发生了明显变化：

```text
Dialogue
   ↓
1× LLM extract
   ↓
ADD-only store
   ↓
Hybrid retrieval
   ↓
Top-k
```

其中最关键的是：

> **ADD-only Store**

即：

> 新 Memory 默认追加保存，而不是在写入时 UPDATE / DELETE 旧 Memory。

---

# 4. v3 为什么要保留历史？

因为很多所谓的“冲突”，其实并不是错误，而是 **时间不同导致的不同状态**。

例如：

```text
2024:
User lives in Beijing.

2025:
User moved to Shanghai.

2026:
User plans to move to Tokyo.
```

这些信息都可能是真的：

```text
Beijing  = 过去状态
Shanghai = 当前状态
Tokyo    = 未来计划
```

因此：

```text
Beijing
Shanghai
Tokyo
```

并不一定应该被视为“冲突”。

---

# 5. v3 的核心变化：从 Write-time Conflict Resolution 到 Read-time Resolution

这是理解 v2 → v3 的关键。

## v2

```text
新 Memory
   ↓
寻找旧 Memory
   ↓
比较
   ↓
写入时解决冲突
```

即：

> **Conflict Resolution at Write Time**

---

## v3

```text
新 Memory
   ↓
直接 ADD
   ↓
历史全部保留
   ↓
Query
   ↓
Hybrid Retrieval
   ↓
根据语义、关键词、实体、时间进行排序
```

即：

> **Conflict Resolution / Relevance Resolution at Retrieval Time**

因此：

```text
v2 = 写入时决定“谁覆盖谁”
v3 = 查询时决定“现在应该看谁”
```

---

# 6. v3 的 ADD-only Store

可以把 v3 的 Memory 更接近地理解成：

```text
Historical Fact Store
```

而不是：

```text
Current State Table
```

例如：

```text
2024 → Beijing
2025 → Shanghai
2026 → Tokyo plan
```

全部保留。

然后不同 Query 得到不同结果。

### Query 1

```text
Where do I live now?
```

结果更倾向：

```text
Shanghai
```

### Query 2

```text
Where did I live in 2024?
```

结果：

```text
Beijing
```

### Query 3

```text
Where am I planning to move?
```

结果：

```text
Tokyo
```

所以：

> Memory 本身保存“历史事实”，Query 决定“应该使用哪一个事实”。

---

# 7. Hybrid Retrieval

v3 的另一大变化是：

```text
Hybrid Retrieval
```

图中给出了四种重要信号：

```text
Semantic · BM25 · Entity · Time
```

它们可以理解为四种不同的“找 Memory 的依据”。

```text
Semantic → 意思
BM25     → 关键词
Entity   → 对象
Time     → 时间
```

最终再把这些信号综合起来做 Ranking。

---

# 8. Semantic：语义检索

## 8.1 它解决什么问题？

核心问题：

> **“这条 Memory 和 Query 的意思像不像？”**

通常会使用：

```text
Embedding
    ↓
Vector Similarity
```

---

## 8.2 示例

Memory：

```text
I moved to Shanghai last year.
```

Query：

```text
Where does the user live?
```

虽然字面不完全一致：

```text
moved
live
```

但语义相关，因此 Semantic Retrieval 可以召回它。

---

## 8.3 特点

### 优点

- 擅长理解语义
- 不要求 Query 和 Memory 使用完全相同的词
- 对自然语言表达变化比较鲁棒

### 局限

- 精确关键词可能不是它最强的场景
- 数字、版本号、专有名词等精确匹配可能需要其他信号补充

---

# 9. BM25：关键词检索

BM25 是经典的 **Lexical Retrieval（词法检索）** 方法。

它更关注：

> **Query 中的词，在文档/Memory 中出现得有多重要。**

---

## 9.1 示例

Query：

```text
What PostgreSQL version did I install?
```

Memory：

```text
I installed PostgreSQL 17.3 on my server.
```

这里：

```text
PostgreSQL
17.3
```

属于非常精确的关键词。

BM25 对这种场景非常有价值。

---

## 9.2 Semantic vs BM25

```text
Semantic
= “意思像不像？”

BM25
= “关键词对不对得上？”
```

例如：

```text
Query:
database version
```

Memory：

```text
PostgreSQL 17.3
```

Semantic 可能能够理解两者相关。

而：

```text
Query:
PostgreSQL 17.3
```

这种精确查询，BM25 更容易发挥优势。

---

## 9.3 为什么需要 BM25？

因为只用 Vector Search：

```text
Query
 ↓
Embedding
 ↓
Vector Search
```

会弱化精确文本匹配。

因此：

```text
Semantic + BM25
```

可以互补：

```text
Semantic → Recall semantic similarity
BM25     → Recall exact lexical matches
```

---

# 10. Entity：实体匹配 / 实体关联

Entity Signal 关注：

> **“这些 Memory 说的是不是同一个人、物体、公司、地点、项目或产品？”**

---

## 10.1 示例

历史：

```text
I bought a MacBook Pro.

The MacBook has an M4 chip.

My laptop battery is great.
```

这三句话实际上可能都在描述：

```text
MacBook Pro
```

其中：

```text
MacBook Pro
MacBook
laptop
```

可以被理解为相关实体。

---

## 10.2 Entity 和 Semantic 的区别

可以简单记：

```text
Semantic
→ “意思相似吗？”

Entity
→ “是不是同一个对象？”
```

例如：

```text
Apple
Apple Inc.
the company
the Cupertino company
```

它们语义相关之外，还需要判断：

> 是否在指向同一个实体。

因此 Entity Signal 特别适合：

```text
人物
公司
地点
项目
产品
技术名称
```

等 Memory。

---

# 11. Time：时间 / 时序信号

Time Signal 关注：

> **“这条 Memory 发生在什么时候？它描述的是过去、现在还是未来？”**

这是 v3 架构里非常重要的部分。

---

## 11.1 示例

历史：

```text
2024:
I live in Beijing.

2025:
I moved to Shanghai.

2026:
I'm planning to move to Tokyo.
```

Query：

```text
Where do I live now?
```

单纯 Semantic Search 可能认为三条都相关。

但加入 Time 后，可以得到：

```text
Beijing
→ past

Shanghai
→ current

Tokyo
→ future plan
```

于是：

```text
Shanghai
↑
更符合“now”
```

---

## 11.2 为什么 Time 很重要？

因为：

> **“旧事实”不一定是“错误事实”。**

例如：

```text
Beijing
```

在 2024 年可能完全正确。

问题不在于：

```text
Beijing = Wrong
```

而在于：

```text
Beijing = Historical
Shanghai = Current
Tokyo = Future
```

因此时间信息可以帮助系统避免把历史事实错误地当成当前事实。

---

# 12. 四种 Retrieval Signal 的一句话理解

| Signal | 核心问题 | 主要作用 |
|---|---|---|
| **Semantic** | 意思像不像？ | 语义相关性 |
| **BM25** | 关键词对不对？ | 精确文本匹配 |
| **Entity** | 是不是同一个对象？ | 实体关联 |
| **Time** | 哪个时间更合适？ | 新旧 / 过去现在未来判断 |

可以直接记：

```text
Semantic = 意思
BM25     = 词
Entity   = 对象
Time     = 时间
```

---

# 13. 四个信号不是四选一，而是综合 Ranking

可以把 v3 的 Retrieval 理解成：

```text
                    ┌─ Semantic
                    │
Query ──────────────┼─ BM25
                    │
                    ├─ Entity
                    │
                    └─ Time
                           ↓
                     Fusion / Ranking
                           ↓
                          Top-k
```

也就是说：

> **不是“Semantic 找一次、BM25 找一次，然后四选一”，而是利用多个 retrieval signal 综合判断哪些 Memory 最值得返回。**

---

# 14. 一个完整的 v3 示例

假设 Memory Store 中保存了：

```text
2024
User lived in Beijing.

2025
User moved to Shanghai.

2025
User works at Company A.

2026
User plans to move to Tokyo next year.

2026
User uses PostgreSQL 17.3.
```

现在 Query：

```text
Where does the user live now?
```

系统可能得到：

### Semantic

召回：

```text
User lived in Beijing.
User moved to Shanghai.
User plans to move to Tokyo.
```

### BM25

如果 Query 中出现：

```text
live
```

会补充一些 lexical match。

### Entity

识别：

```text
Beijing
Shanghai
Tokyo
```

都是地点实体。

### Time

判断：

```text
Beijing → past
Shanghai → current
Tokyo → future
```

最终 Ranking：

```text
Shanghai
↑
Top result
```

---

# 15. 两代架构的本质对比

| 维度 | Mem0 v2 | Mem0 v3 |
|---|---|---|
| 核心理念 | State-based | History + Retrieval |
| 写入模式 | ADD / UPDATE / DELETE / NOOP | **ADD-only** |
| 新旧 Memory 是否比较 | 写入时比较 | 查询时综合判断 |
| 冲突处理 | LLM 决策 | Retrieval / Ranking |
| 历史记录 | 可能被覆盖 | **尽量保留** |
| Vector Search | 寻找冲突候选 | 检索信号之一 |
| BM25 | 非核心 | **重要检索信号** |
| Entity | 非核心 | **实体检索信号** |
| Time | 相对弱 | **重要检索信号** |
| LLM 调用 | Extraction + Decision | **Single-pass Extraction** |
| 主要复杂度 | 写入侧 | **检索侧** |
| Memory 的角色 | 当前知识状态 | 历史事实集合 |
| Conflict 观 | 新旧信息互相覆盖 | 新旧信息可能都是正确的，只是时间不同 |

---

# 16. 一个非常直观的类比

## v2：像 UPDATE 型数据库

```text
user_location = Beijing
```

后来：

```text
user_location = Shanghai
```

执行：

```text
UPDATE
```

最终：

```text
user_location = Shanghai
```

---

## v3：更像 History / Event Store + Retrieval

保存：

```text
2024 → Beijing
2025 → Shanghai
2026 → Tokyo plan
```

查询：

```text
Current location?
```

再根据：

```text
Semantic
+ BM25
+ Entity
+ Time
```

得到：

```text
Shanghai
```

> 这不是严格意义上的 Event Sourcing，只是为了帮助理解其架构倾向。

---

# 17. 与 Agent 架构趋势的关联

这一变化和 Agent 中经常讨论的：

```text
LLM-driven
vs
Runtime-driven
```

存在一定相似性。

## v2

更多事情交给 LLM：

```text
LLM
 ├─ extract fact
 └─ decide whether UPDATE / DELETE / NOOP
```

可以理解为：

```text
LLM = Memory State Manager
```

---

## v3

LLM 主要负责：

```text
Dialogue → Fact Extraction
```

而大量后续工作交给系统基础设施：

```text
ADD-only Store
Retrieval
BM25
Entity Linking
Temporal Ranking
```

因此可以理解为：

> **把 Memory 的部分“决策复杂性”从 LLM 驱动的写入 mutation，转移到了 Retrieval Infrastructure。**

这和现代 Agent 越来越强调：

```text
Runtime
Orchestration
Retrieval
Routing
State Management
```

的趋势具有相似的架构思路。

---

# 18. 最终心智模型

理解 Mem0 v2 → v3，最重要的是记住下面这条演化线：

```text
┌─────────────────────────────────────────────┐
│                  Mem0 v2                    │
│                                             │
│ Dialogue                                    │
│    ↓                                        │
│ Extract                                     │
│    ↓                                        │
│ Search old memory                           │
│    ↓                                        │
│ LLM compare                                 │
│    ↓                                        │
│ ADD / UPDATE / DELETE / NOOP                │
│                                             │
│     → 写入时维护“当前状态”                  │
└─────────────────────────────────────────────┘

                     ↓ 演化

┌─────────────────────────────────────────────┐
│                  Mem0 v3                    │
│                                             │
│ Dialogue                                    │
│    ↓                                        │
│ 1× LLM Extract                              │
│    ↓                                        │
│ ADD-only Store                               │
│    ↓                                        │
│ Hybrid Retrieval                            │
│    ├─ Semantic                              │
│    ├─ BM25                                  │
│    ├─ Entity                                │
│    └─ Time                                  │
│    ↓                                        │
│ Top-k                                       │
│                                             │
│     → 写入时保留历史，读取时决定相关性      │
└─────────────────────────────────────────────┘
```

---

# 19. 一句话总结

> **Mem0 v2 的思路是“写入时把 Memory 整理干净”；Mem0 v3 的思路是“写入时尽量不丢历史，查询时通过 Semantic + BM25 + Entity + Time 决定哪些历史最相关”。**

进一步压缩成四个关键词：

```text
v2 → Conflict Resolution at Write Time
v3 → History Preservation + Retrieval-time Resolution

Semantic → 意思
BM25     → 关键词
Entity   → 对象
Time     → 时间
```
