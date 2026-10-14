---
title: JSON、JSONL、JSONB 的区别与选型
tags:
  - JSON
  - JSONL
  - JSONB
  - PostgreSQL
  - 数据存储
  - Agent
  - RAG
---

# JSON、JSONL、JSONB 的区别与选型

## 一、核心结论

JSON、JSONL、JSONB 有联系，但不属于完全相同的层级：

- **JSON**：一种数据表示格式，用于描述对象、数组、字符串、数字、布尔值等数据。
- **JSONL**：一种文本文件组织方式，每一行放置一个独立的 JSON 值，常见用途是日志、数据集和流式处理。
- **JSONB**：PostgreSQL 提供的一种数据类型，将 JSON 内容解析后以适合数据库处理的二进制表示形式存储，适合查询和索引。

可以简单记为：

> **JSON 关注数据怎么表示；JSONL 关注多条数据怎么按行组织；JSONB 关注 PostgreSQL 如何存储和查询 JSON 数据。**

## 二、JSON：数据表示格式

JSON（JavaScript Object Notation）是一种常见的轻量级数据交换格式。

### 示例

```json
{
  "id": 1,
  "name": "Alice",
  "tags": ["AI", "RAG"]
}
```

JSON 可以表示一个对象，也可以表示一个数组：

```json
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]
```

### 常见用途

- 前后端 API 请求与响应
- 程序配置文件
- Agent 工具调用的参数与结构化输出
- 不同服务之间的数据交换

### 特点

- 人类可读，结构清晰。
- 支持对象、数组、字符串、数字、布尔值和 `null` 等数据类型。
- JSON 是数据格式，不限定数据必须存储在文件中；它也可以通过网络传输，或存入数据库。

## 三、JSONL：一行一条 JSON 数据

JSONL 通常指 JSON Lines。它把多条独立的 JSON 值放在同一个文本文件中，每行一条记录。

### 示例

```jsonl
{"id":1,"name":"Alice"}
{"id":2,"name":"Bob"}
{"id":3,"name":"Charlie"}
```

每一行都应该是一个独立、完整、可解析的 JSON 值。常见用法是一行表示一条对象记录。

### 与 JSON 数组的区别

普通 JSON 文件可以把多条记录放在同一个数组中：

```json
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]
```

而 JSONL 则逐行保存：

```jsonl
{"id":1,"name":"Alice"}
{"id":2,"name":"Bob"}
```

主要区别：

- JSON 数组是一个整体的 JSON 文档。
- JSONL 是多条独立的 JSON 记录按行排列，通常没有外围数组，也不需要在记录之间加逗号。
- JSONL 可以逐行读取、追加和处理，不必将整个文件一次性读入内存。

### 常见用途

- LLM 训练数据和评测数据集
- Agent 运行日志与事件记录
- 大规模数据的批量导入
- 流式生成、逐条处理的程序输出

常见扩展名是 `.jsonl`，也常见 `.ndjson`（Newline Delimited JSON）。两者通常采用相同的逐行 JSON 组织方式，但具体工具对格式细节的要求可能不同。

## 四、JSONB：PostgreSQL 的 JSON 数据类型

JSONB 是 PostgreSQL 提供的一种数据类型。它接收 JSON 文本，解析其内容，再以适合数据库处理的二进制形式存储。

### 示例：建立 JSONB 字段

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    profile JSONB
);
```

插入数据：

```sql
INSERT INTO users (id, profile)
VALUES (
    1,
    '{"name": "Alice", "tags": ["AI", "RAG"]}'
);
```

这里的 `profile` 是数据库字段，不是一个 JSON 文件。

### 查询 JSONB 内部字段

取出 `name` 字段并作为文本比较：

```sql
SELECT *
FROM users
WHERE profile ->> 'name' = 'Alice';
```

### 建立 GIN 索引

如果查询模式适合，可以对 JSONB 字段建立 GIN 索引：

```sql
CREATE INDEX users_profile_idx
ON users USING GIN (profile);
```

索引能否被使用、采用哪种索引，以及查询是否加速，仍取决于查询表达式、索引类型、数据分布和执行计划。

### 常见用途

- PostgreSQL 中的半结构化数据
- 可变属性、配置和扩展字段
- 文档或业务记录的 metadata
- RAG chunk 的元数据过滤

## 五、JSON 与 JSONB 的区别

JSON 与 JSONB 都是 PostgreSQL 数据类型，差异主要在于存储方式和查询特性。

| 对比维度 | `JSON` | `JSONB` |
|---|---|---|
| 存储方式 | 保存输入文本 | 解析后以二进制表示形式存储 |
| 原始格式 | 保留输入中的空白和文本表示 | 不保留原始空白格式 |
| 对象键顺序 | 保留输入文本中的顺序 | 不保证保留原顺序 |
| 重复对象键 | 原始文本会保留重复键 | 重复键只保留最后一个值 |
| 查询 | 支持 JSON 操作 | 支持 JSON 操作，通常更适合反复查询 |
| 索引 | 不支持直接对 JSON 值建立 GIN 索引 | 支持 GIN 等索引方式 |
| 典型用途 | 需要保留输入文本的情况 | 需要查询、筛选和索引的情况 |

### 如何选择 JSON 还是 JSONB？

- **优先考虑 JSONB**：数据要经常按内部字段查询、过滤、做包含关系判断，或需要索引。
- **考虑 JSON**：需要保留输入文本本身，例如原始空白、键的排列顺序或重复键等文本细节。
- **考虑普通文本字段**：如果需求是精确保留原始字节或原始文本，且不需要 PostgreSQL 的 JSON 操作，使用文本字段可能更直观；需要精确保留原始字节时，还要考虑编码等要求。

实践中，PostgreSQL 应用通常更常用 JSONB，但不代表所有场景都必须使用 JSONB。

## 六、快速选型表

| 需求 | 推荐选择 | 主要原因 |
|---|---|---|
| 前后端 API 传输数据 | JSON | 通用数据交换格式 |
| 保存易读、易编辑的配置 | JSON | 层级结构清晰 |
| 保存海量逐条日志 | JSONL | 易于逐行追加、读取和处理 |
| 构建 LLM 训练集或评测集 | JSONL | 每条样本独立，适合批量处理 |
| Agent 工具输入、输出 | JSON | 便于定义和校验结构化参数 |
| PostgreSQL 存储可变元数据 | JSONB | 便于查询和建立索引 |
| PostgreSQL 存储 RAG chunk 的 metadata | JSONB | 可以结合 SQL 条件过滤检索结果 |
| 保留 JSON 原始文本表示 | JSON 或文本字段 | JSONB 不保留原始文本格式 |

## 七、放到 Agentic RAG 系统中理解

以 Agentic RAG 为例，三者可以在同一个系统中承担不同职责：

```text
原始数据 / 配置 / API 数据
        │
        ├── JSON：配置、接口传输、结构化工具参数
        │
        └── JSONL：批量样本、评测集、逐条运行日志
                         │
                         ▼
                解析、清洗、切块
                         │
                         ▼
                生成 embedding 与 metadata
                         │
                         ▼
              PostgreSQL + pgvector
              ├── embedding：向量检索
              └── JSONB：metadata 存储与过滤
                         │
                         ▼
              向量检索 + 条件过滤 + Agent 推理
```

例如，RAG 数据表可能包含：

- `id`：chunk 的标识
- `doc_id`：来源文档标识
- `embedding`：用于向量检索的向量
- `meta JSONB`：页码、来源、标签、文档类型等元数据

这里的 `JSONB` 负责存储和查询 metadata；它本身不负责生成 embedding，也不替代向量索引。向量检索与 JSONB 条件过滤属于不同职责，可以组合使用。

## 八、常见误区

1. **把三者当成同一层级的格式。** JSON 是数据格式；JSONL 是逐行组织方式；JSONB 是 PostgreSQL 的数据类型。
2. **认为 JSONL 就是带换行的普通 JSON 数组。** JSONL 每行是一条独立的 JSON 值，通常没有外围数组和行间逗号。
3. **认为 JSONB 会原样保存 JSON 文本。** JSONB 会解析并重新组织数据，不保留原始空白和对象键顺序，也会折叠重复键。
4. **认为 JSONB 一定比 JSON 快。** 对需要反复查询的场景，JSONB 通常更合适；实际性能依赖访问模式、索引和查询计划。
5. **认为 JSONB 能替代所有数据库列。** 对经常用于关联、约束、排序或统计的核心字段，通常仍适合使用明确的关系型列；JSONB 更适合灵活、半结构化的属性。

## 九、最终记忆口诀

> **JSON 表示数据，JSONL 按行组织数据，JSONB 在 PostgreSQL 中存储并高效查询 JSON 数据。**

选择时先问自己：

1. 我要定义或交换数据吗？——用 JSON。
2. 我要保存大量独立记录并逐条处理吗？——考虑 JSONL。
3. 我要在 PostgreSQL 里查询或筛选 JSON 内部字段吗？——通常优先考虑 JSONB。
