# AI Research Hub Demo：Spring Boot + Supabase + RAG + Agent 学习路线

> 目标：从 0 构建一个“个人/团队 AI 知识库 + Agent”Demo，逐步演进出一套现代后端架构。
>
> 核心思想：**先跑通最小闭环，再逐层增加 Redis / Kafka / Elasticsearch / Agent / Docker / CI/CD / Monitoring。**
>
> 本文既是学习路线，也是可以直接交给 Coding Agent 执行的项目规格。

---

# 1. 项目定位

## 1.1 项目名称

**AI Research Hub**

一个支持以下能力的知识库应用：

- 用户注册 / 登录
- 创建知识库
- 上传 PDF / Markdown
- 文档解析与切分
- Embedding
- PostgreSQL + pgvector 向量检索
- BM25 / Hybrid Search
- Reranker
- RAG 问答
- 文档引用（Citation）
- Agentic RAG
- 异步文档处理
- 缓存、搜索、消息队列
- Docker 本地开发
- CI/CD
- Monitoring

---

# 2. 最终目标架构

```text
                                ┌──────────────────────┐
                                │        用户          │
                                └──────────┬───────────┘
                                           │
                              HTTPS / WebSocket
                                           │
                                ┌──────────▼──────────┐
                                │ Next.js / React     │
                                │ Web Frontend        │
                                └──────────┬──────────┘
                                           │
                             ┌─────────────┴─────────────┐
                             │                           │
                             ▼                           ▼
                      Supabase Auth               Spring Boot API
                      登录 / OAuth                  业务后端
                                                         │
                           ┌─────────────────────────────┼────────────────┐
                           │                             │                │
                           ▼                             ▼                ▼
                         Redis                         Kafka       Elasticsearch
                     缓存 / Session                异步任务 / 事件     全文搜索
                           │                             │
                           │                             ▼
                           │                          Worker
                           │                    解析 / Chunk / Embedding
                           │
                           ▼
                  ┌────────────────────────────┐
                  │ Supabase                   │
                  │                            │
                  │ PostgreSQL                 │
                  │ + pgvector                 │
                  │ Auth                       │
                  │ Storage                    │
                  │ Realtime                   │
                  │ RLS                        │
                  └──────────────┬─────────────┘
                                 │
                                 ▼
                      ┌─────────────────────┐
                      │ AI Pipeline         │
                      │ Embedding            │
                      │ Reranker             │
                      │ LLM                  │
                      │ Agent                │
                      └─────────┬───────────┘
                                │
                       ┌────────┼─────────┐
                       ▼        ▼         ▼
                    OpenAI   Claude   DeepSeek
```

---

# 3. 技术栈

## 前端

- Next.js
- React
- TypeScript
- Tailwind CSS（可选）
- Supabase JS Client

## 后端

- Java
- Spring Boot
- Spring Web
- Spring Validation
- Spring Security
- Spring Data JPA
- PostgreSQL JDBC

## 数据

- Supabase PostgreSQL
- pgvector
- JSONB
- HNSW
- PostgreSQL Full Text Search
- Elasticsearch（后期加入）

## 基础设施

- Redis
- Kafka
- Docker / Docker Compose
- Nginx

## AI

- Embedding API
- Reranker API
- LLM API
- Agent Tool Calling

## 工程化

- Git / GitHub
- GitHub Actions
- Prometheus
- Grafana
- Spring Boot Actuator

---

# 4. 非目标（第一版不要做）

为了避免 Coding Agent 过度设计，第一版明确不做：

- 微服务拆分
- Kubernetes
- Service Mesh
- 多 Region
- 复杂 RBAC
- 分布式事务
- 自建向量模型
- 自建 LLM
- 多租户企业级权限体系
- 高可用 Kafka 集群
- 高可用 Elasticsearch 集群

第一目标永远是：

> **一个开发者可以在本地和 Supabase 云环境中跑通完整 Demo。**

---

# 5. 项目演进路线

```text
V0  Next.js + Supabase
        ↓
V1  + Spring Boot
        ↓
V2  + pgvector + 基础 RAG
        ↓
V3  + BM25 + Hybrid Search + Reranker
        ↓
V4  + Redis
        ↓
V5  + Kafka + Worker
        ↓
V6  + Supabase Realtime
        ↓
V7  + Elasticsearch
        ↓
V8  + Agentic RAG
        ↓
V9  + Docker + CI/CD
        ↓
V10 + Monitoring / Observability
```

---

# 6. Phase 0：项目骨架

## 目标

先让三个模块跑起来：

```text
frontend/
backend/
infra/
```

推荐结构：

```text
ai-research-hub/
├── frontend/
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── package.json
│
├── backend/
│   ├── src/main/java/.../
│   ├── src/main/resources/
│   └── pom.xml
│
├── infra/
│   ├── docker-compose.yml
│   └── nginx/
│
├── docs/
│   └── architecture.md
│
└── README.md
```

## 验收标准

- Next.js 能启动
- Spring Boot 能启动
- `/api/health` 返回 `{"status":"ok"}`
- 前后端可以正常通信
- README 能说明启动方法

---

# 7. Phase 1：Supabase 基础能力

## 目标

先不做复杂业务，让 Supabase 工作起来。

## 创建

- Supabase Project
- PostgreSQL
- Auth
- Storage

## 数据表

### users

```text
id
created_at
display_name
```

> 实际项目可以使用 Supabase Auth 的 user id 作为外键。

### knowledge_bases

```text
id
owner_id
name
description
created_at
updated_at
```

### documents

```text
id
knowledge_base_id
title
filename
storage_path
mime_type
status
created_at
updated_at
```

### document_chunks

```text
id
document_id
chunk_index
content
metadata JSONB
embedding vector(...)
created_at
```

### chat_sessions

```text
id
user_id
knowledge_base_id
title
created_at
```

### messages

```text
id
session_id
role
content
created_at
```

## 关键数据库能力

必须实践：

- 外键
- JSONB
- Index
- pgvector extension
- HNSW index
- RLS

---

# 8. Phase 2：Auth

## 用户流程

```text
用户
 ↓
Next.js
 ↓
Supabase Auth
 ↓
JWT
 ↓
Spring Boot
 ↓
识别 user_id
```

## 第一版支持

- Email / Password
- OAuth（可以后加）

## Spring Boot 要求

实现：

- JWT 校验
- 当前用户解析
- 用户级数据隔离

## 验收

用户 A：

```text
GET /api/knowledge-bases
```

只能得到自己的知识库。

---

# 9. Phase 3：Spring Boot 业务层

## 领域划分

```text
backend/
└── domain/
    ├── auth/
    ├── knowledge/
    ├── document/
    ├── search/
    ├── chat/
    └── agent/
```

## 第一批 API

### Knowledge Base

```http
POST   /api/knowledge-bases
GET    /api/knowledge-bases
GET    /api/knowledge-bases/{id}
DELETE /api/knowledge-bases/{id}
```

### Document

```http
POST   /api/documents
GET    /api/documents
GET    /api/documents/{id}
DELETE /api/documents/{id}
```

### Search

```http
POST /api/search
```

### Chat

```http
POST /api/chat
```

---

# 10. Phase 4：文件上传

## 推荐流程

```text
Next.js
   │
   ▼
Supabase Storage
   │
   ▼
文件上传完成
   │
   ▼
Spring Boot
   │
   ▼
documents.status = UPLOADED
```

数据库只保存 metadata：

```text
filename
storage_path
mime_type
size
status
```

不要把大型 PDF 二进制直接存 PostgreSQL。

---

# 11. Phase 5：基础 RAG

这是第一个真正的 AI 核心阶段。

## Ingestion Pipeline

```text
PDF / Markdown
       ↓
Parser
       ↓
Clean Text
       ↓
Chunking
       ↓
Embedding
       ↓
document_chunks
       ↓
pgvector
```

## Query Pipeline

```text
用户问题
   ↓
Query Embedding
   ↓
Vector Search
   ↓
Top K Chunks
   ↓
Prompt Builder
   ↓
LLM
   ↓
回答 + Citation
```

---

# 12. Chunking 最低要求

第一版支持：

- 固定 Token / 字符长度
- overlap
- metadata

例如：

```json
{
  "source": "paper.pdf",
  "page": 3,
  "section": "Introduction"
}
```

推荐默认：

```text
chunk_size = 500 ~ 1000 tokens
overlap = 50 ~ 150 tokens
top_k = 5 ~ 10
```

实际参数不要硬编码在业务逻辑中，统一配置。

---

# 13. Phase 6：Hybrid Search

从纯向量检索升级为：

```text
                     Query
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Vector Search       BM25
         pgvector            PostgreSQL
              │                 │
              └────────┬────────┘
                       ▼
                      RRF
                       ▼
                    Top 30
                       ▼
                   Reranker
                       ▼
                     Top 5
                       ▼
                      LLM
```

## 第一版可以使用

- dense retrieval：pgvector
- sparse retrieval：PostgreSQL FTS
- fusion：RRF
- reranker：Cross Encoder / API

## 需要记录

每次搜索保存：

```text
query
dense_results
sparse_results
fused_results
reranked_results
latency
```

这些数据后面可以用于评估。

---

# 14. Phase 7：Redis

当以下问题出现时加入 Redis：

- 高频重复查询
- LLM 成本较高
- Session / 临时状态
- Rate Limit

第一版只做：

### Query Cache

```text
query
 ↓
normalize
 ↓
Redis GET
 ├── HIT → return
 └── MISS
       ↓
      RAG
       ↓
    Redis SET
```

不要一开始让 Redis 承担所有状态。

---

# 15. Phase 8：Kafka + 异步 Worker

文档处理不应该长期阻塞 HTTP 请求。

## 改造前

```text
POST /documents
      ↓
parse
      ↓
chunk
      ↓
embedding
      ↓
index
      ↓
HTTP response
```

## 改造后

```text
POST /documents
      ↓
Storage
      ↓
Kafka
      ↓
document.process
      ↓
Worker
 ├── Parser
 ├── Chunker
 ├── Embedding
 └── Indexer
      ↓
PostgreSQL
```

## 状态

```text
UPLOADED
   ↓
PROCESSING
   ↓
INDEXED
```

失败：

```text
PROCESSING
   ↓
FAILED
```

可以增加：

```text
retry_count
error_message
```

---

# 16. Phase 9：Realtime

利用 Supabase Realtime 做前端进度更新。

```text
Worker
   ↓
documents.status
   ↓
Supabase Realtime
   ↓
Next.js
   ↓
UI
```

前端显示：

```text
上传完成
↓
正在解析
↓
正在切分
↓
正在生成 Embedding
↓
索引完成
```

这样可以同时实践：

- 异步任务
- DB 状态机
- Realtime
- 前端状态管理

---

# 17. Phase 10：Elasticsearch

此时再引入 Elasticsearch。

## 为什么

PostgreSQL：

- 业务主数据
- 事务
- pgvector
- 基础全文搜索

Elasticsearch：

- 复杂全文搜索
- Search-as-you-type
- 高亮
- Facet / Aggregation
- 日志分析（后续）

## 架构

```text
                  Search Service
                       │
              ┌────────┴────────┐
              ▼                 ▼
         PostgreSQL        Elasticsearch
         业务 / vector     full-text / BM25
```

注意：

> Elasticsearch 不替代 PostgreSQL。

PostgreSQL 仍然是主要业务数据源。

---

# 18. Phase 11：Agentic RAG

此时开始做 Agent。

## Tools

第一版提供：

```text
search_knowledge_base
get_document
get_document_chunk
list_documents
summarize_document
```

## Agent Loop

```text
User
 ↓
LLM
 ↓
判断是否需要 Tool
 ├── No → Final Answer
 │
 └── Yes
       ↓
   Tool Call
       ↓
   Tool Result
       ↓
   LLM 再推理
       ↓
   ...
       ↓
 Final Answer
```

## 示例任务

```text
比较“PQC TLS”相关的 5 篇论文，
指出它们对证书、KEM、握手开销的不同观点。
```

Agent：

```text
1. 搜索关键词
2. 找到候选论文
3. 读取相关 chunk
4. 判断信息是否足够
5. 补充检索
6. 综合
7. 输出带 citation 的结果
```

---

# 19. Agent 设计原则

不要一开始做“万能 Agent”。

第一版只做：

```text
Research Agent
```

工具数量控制在：

```text
5 个以内
```

并要求记录：

```text
agent_run
 ├── run_id
 ├── user_id
 ├── input
 ├── steps
 ├── tool_calls
 ├── final_answer
 ├── token_usage
 └── latency
```

这部分后面可以用于研究：

- Agent Loop
- Memory
- Tool Calling
- Context Management
- Compaction
- Evaluation

---

# 20. Phase 12：Docker Compose

本地运行：

```text
docker compose up
```

建议包含：

```yaml
services:
  backend:
    # Spring Boot

  worker:
    # Document Worker

  redis:
    # Redis

  kafka:
    # Kafka

  elasticsearch:
    # Elasticsearch

  nginx:
    # API Gateway / Reverse Proxy
```

Supabase 可以继续使用云端开发项目。

---

# 21. Phase 13：Nginx / API Gateway

统一入口：

```text
Client
  ↓
Nginx
  ├── /api/* → Spring Boot
  └── /*     → Next.js
```

以后可以增加：

- Rate Limit
- CORS
- TLS termination
- Header control
- Request logging

---

# 22. Phase 14：CI/CD

GitHub：

```text
push
 ↓
GitHub Actions
 ↓
Lint
 ↓
Unit Test
 ↓
Build
 ↓
Docker Build
 ↓
Deploy
```

最低要求：

### Backend

```text
mvn test
mvn package
```

### Frontend

```text
npm run lint
npm run build
```

---

# 23. Phase 15：Observability

Spring Boot：

- Actuator
- Micrometer

Prometheus：

```text
metrics
 ↓
Prometheus
 ↓
Grafana
```

重点监控：

```text
HTTP QPS
HTTP latency
HTTP error rate

RAG latency
Embedding latency
LLM latency

Kafka consumer lag
Redis hit rate
Postgres slow query

JVM memory
JVM GC
CPU
```

---

# 24. 最终数据库设计

建议最终保留：

```text
users
knowledge_bases
documents
document_chunks
chat_sessions
messages
agent_runs
agent_steps
search_logs
```

关键关系：

```text
users
  │
  ├── knowledge_bases
  │       │
  │       └── documents
  │               │
  │               └── document_chunks
  │
  └── chat_sessions
          │
          └── messages

agent_runs
   │
   └── agent_steps
```

---

# 25. 最终服务边界

不要过早拆微服务，但逻辑上保持模块化：

```text
Spring Boot
│
├── Auth Module
├── Knowledge Module
├── Document Module
├── Search Module
├── Chat Module
├── Agent Module
└── Infrastructure Module
```

只有在真正出现独立扩展需求时，再考虑拆成：

```text
API Service
Document Worker
AI Service
Search Service
```

---

# 26. 推荐的实际开发顺序

## Sprint 1：MVP

```text
[ ] 创建 GitHub Repo
[ ] 创建 Next.js
[ ] 创建 Spring Boot
[ ] 创建 Supabase
[ ] PostgreSQL Schema
[ ] Auth
[ ] Knowledge Base CRUD
[ ] Document CRUD
```

## Sprint 2：RAG

```text
[ ] Storage Upload
[ ] PDF Parser
[ ] Chunking
[ ] Embedding
[ ] pgvector
[ ] Vector Search
[ ] RAG Chat
[ ] Citation
```

## Sprint 3：Search

```text
[ ] PostgreSQL FTS
[ ] BM25
[ ] Hybrid Search
[ ] RRF
[ ] Reranker
[ ] Search Evaluation
```

## Sprint 4：异步化

```text
[ ] Redis
[ ] Kafka
[ ] Worker
[ ] Processing State Machine
[ ] Realtime Progress
```

## Sprint 5：Agent

```text
[ ] Tool abstraction
[ ] Agent Loop
[ ] Research Agent
[ ] Agent Run Log
[ ] Citation
[ ] Error Recovery
```

## Sprint 6：工程化

```text
[ ] Elasticsearch
[ ] Docker Compose
[ ] Nginx
[ ] CI/CD
[ ] Prometheus
[ ] Grafana
[ ] API Documentation
```

---

# 27. Coding Agent 实现时的约束

把下面这些原则直接交给 Coding Agent：

## 原则 1：先做能运行的最小版本

每一个 Phase 必须：

```text
实现
 ↓
运行
 ↓
验证
 ↓
再进入下一阶段
```

不要一次性生成整个系统。

## 原则 2：不要无理由增加依赖

每加入一个组件必须回答：

```text
为什么需要？
解决什么问题？
没有它会怎样？
```

## 原则 3：数据库优先于“伪造的数据层”

必须真实连接 PostgreSQL。

禁止第一版大量使用：

- Mock Repository
- In-memory database
- Fake vector search

## 原则 4：配置外置

例如：

```text
DATABASE_URL
SUPABASE_URL
SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY
REDIS_URL
KAFKA_BOOTSTRAP_SERVERS
ELASTICSEARCH_URL
LLM_API_KEY
EMBEDDING_API_KEY
```

敏感信息不要进入 Git。

## 原则 5：保留可替换接口

例如：

```java
interface EmbeddingService
interface RerankerService
interface LlmService
interface SearchService
interface DocumentParser
```

这样以后替换 OpenAI / Claude / DeepSeek 不需要大范围修改业务代码。

---

# 28. 每个阶段必须输出的文档

每完成一个 Phase，Coding Agent 至少更新：

```text
docs/
├── architecture.md
├── database.md
├── api.md
├── deployment.md
└── decisions.md
```

其中 `decisions.md` 记录：

```text
为什么用 Supabase？
为什么 Spring Boot？
为什么 pgvector？
为什么后来加入 Redis？
为什么后来加入 Kafka？
为什么需要 Elasticsearch？
```

这份文档对学习价值非常高。

---

# 29. 最终验收标准

整个 Demo 完成后，必须能够演示：

## 用户

```text
注册
 ↓
登录
```

## 知识库

```text
创建知识库
 ↓
上传 PDF
```

## 文档处理

```text
上传
 ↓
解析
 ↓
Chunk
 ↓
Embedding
 ↓
Index
```

## 搜索

```text
Query
 ↓
Dense + Sparse
 ↓
RRF
 ↓
Reranker
```

## RAG

```text
Question
 ↓
Retrieve
 ↓
Context
 ↓
LLM
 ↓
Answer + Citation
```

## Agent

```text
Question
 ↓
Agent
 ↓
Search Tool
 ↓
Read Tool
 ↓
Reasoning
 ↓
Final Answer
```

## 工程化

```text
Docker
CI/CD
Monitoring
```

---

# 30. 最终你应该真正理解的东西

完成这个 Demo 后，不是只得到一个项目，而应该形成下面这张“技术地图”：

```text
                          Web Application
                                │
                                ▼
                         Spring Boot API
                                │
           ┌────────────────────┼───────────────────┐
           │                    │                   │
           ▼                    ▼                   ▼
       PostgreSQL              Redis              Kafka
           │                                        │
     ┌─────┼─────┐                                  ▼
     │     │     │                                Worker
     │     │     │                                  │
     ▼     ▼     ▼                                  ▼
   SQL   JSONB pgvector                         AI Pipeline
                │                                  │
                ▼                                  ▼
         Dense Retrieval                     Embedding/Reranker
                │                                  │
                └──────────────┬───────────────────┘
                               ▼
                         Hybrid Search
                               │
                               ▼
                              RAG
                               │
                               ▼
                             Agent
                               │
                               ▼
                              LLM
```

最终形成的认知应该是：

> **Supabase 不等于 Spring Boot。**
>
> Supabase 负责 PostgreSQL、Auth、Storage、Realtime 等通用后端能力；
> Spring Boot 负责复杂业务逻辑和应用层编排；
> Redis / Kafka / Elasticsearch 分别解决缓存、异步消息、复杂搜索；
> pgvector / RAG / Agent 构成 AI 应用层；
> Docker / CI/CD / Monitoring 负责工程化交付。

---

# 31. 推荐最终项目形态

```text
AI Research Hub
│
├── Frontend
│   └── Next.js
│
├── Backend
│   └── Spring Boot
│
├── AI
│   ├── RAG
│   ├── Hybrid Search
│   ├── Reranker
│   └── Agent
│
├── Data
│   ├── Supabase PostgreSQL
│   ├── pgvector
│   └── Elasticsearch
│
├── Infra
│   ├── Redis
│   ├── Kafka
│   ├── Docker
│   └── Nginx
│
├── Cloud
│   └── Supabase
│
└── DevOps
    ├── GitHub Actions
    ├── Prometheus
    └── Grafana
```

---

# 32. 给 Coding Agent 的任务描述

请基于本文从 **Phase 0** 开始实现，不要一次性实现全部功能。

每完成一个 Phase：

1. 修改代码
2. 启动并验证
3. 运行测试
4. 更新 README / docs
5. 明确当前完成内容、未完成内容、下一阶段目标

第一阶段只要求：

```text
Next.js
+
Spring Boot
+
Supabase PostgreSQL
+
Supabase Auth
```

先实现：

```text
用户登录
 ↓
创建 Knowledge Base
 ↓
查看 Knowledge Base
```

只有 Phase 0 / Phase 1 稳定后，才进入文档上传与 RAG。

**禁止为了“看起来完整”而提前引入 Kafka、Elasticsearch、Kubernetes 等组件。**

目标不是一次生成一个庞大 Demo，而是让项目能够沿着本文的架构路线持续演进。
