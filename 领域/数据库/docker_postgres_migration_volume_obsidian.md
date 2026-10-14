---
title: Docker、Volume 与 PostgreSQL 数据迁移
aliases:
  - Docker 数据持久化与数据库迁移
  - PostgreSQL 迁移笔记
tags:
  - Docker
  - PostgreSQL
  - 数据库迁移
  - Obsidian
created: 2026-10-11
---

# Docker、Volume 与 PostgreSQL 数据迁移

## 1. 核心概念

理解数据库迁移时，要区分三件事：**运行环境、数据文件、数据库结构**。

| 概念 | 主要职责 | 可以怎样理解 |
|---|---|---|
| Docker | 提供容器化运行环境，管理镜像、容器、网络和存储等 | 运行环境管理工具 |
| Image（镜像） | 提供启动容器所需的软件与依赖 | 软件环境模板 |
| Container（容器） | 实际运行 PostgreSQL 等程序 | 正在运行的数据库服务 |
| PostgreSQL | 管理数据库、表、索引、事务和 SQL 查询 | 数据库管理系统（DBMS） |
| Volume | 在容器生命周期之外持久化保存文件 | 由 Docker 管理的数据存储空间 |
| `pg_dump` / `pg_restore` | 导出和恢复数据库内容 | 逻辑备份与迁移工具 |
| Alembic 等迁移工具 | 追踪并应用数据库结构变更 | Schema 版本管理工具 |

一句话概括：

> **Docker 解决数据库怎么运行；Volume 解决数据文件如何持久保存；`pg_dump` / `pg_restore` 解决数据库内容如何搬迁；Alembic 解决数据库结构如何随应用演进。**

## 2. Docker、Volume 与数据库的关系

```text
宿主机（Host）
└── Docker
    └── PostgreSQL 容器（Container）
        └── PostgreSQL 进程
            └── 读写数据目录
                 ↕ 挂载
              Docker Volume
                 └── 数据文件保存在宿主机磁盘上
```

PostgreSQL 负责理解 SQL、管理表和索引；Volume 不理解 SQL，它只是提供持久化存储。

### 不挂载 Volume

数据库文件如果只写在容器自身的可写层中，删除容器后，这些文件通常也会一起消失。

### 挂载 Volume

将宿主机上的持久化存储挂载到容器内的数据库数据目录。删除或重建容器时，只要原来的 Volume 仍存在，并重新挂载到兼容的数据库环境，数据通常就能继续使用。

**关键点：容器可以重建，数据不必跟着重建。**

### Volume 不等于备份

Volume 可以防止数据仅依赖于容器本身，但不能自动防止磁盘损坏、误删、数据库逻辑错误或主机故障。重要数据仍需定期备份，并把备份放在独立的存储位置。

## 3. Docker Compose 示例：运行 PostgreSQL + pgvector

以下示例使用 PostgreSQL 17 兼容的 pgvector 镜像：

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: rag
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

在同一目录创建 `.env` 文件，例如：

```dotenv
POSTGRES_PASSWORD=请替换为强密码
```

注意：不要把包含真实密码的 `.env` 提交到 Git。应将它加入 `.gitignore`，生产环境则优先使用安全的密钥管理方式。

### 关键配置解读

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

表示把名为 `postgres_data` 的 Volume 挂载到容器中的 PostgreSQL 数据目录。

```yaml
volumes:
  postgres_data:
```

表示声明一个由 Docker 管理的命名 Volume。

`/var/lib/postgresql/data` 是此示例中使用的数据目录。不同镜像或数据库大版本的目录布局可能不同，应以对应镜像文档为准。

启动服务：

```bash
docker compose up -d
```

查看运行状态：

```bash
docker compose ps
```

对于使用 pgvector 的数据库，通常还需要在目标数据库中启用扩展：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

前提是所用镜像已包含 pgvector 扩展。

## 4. 数据库迁移的三种常见含义

“迁移数据库”可能指不同事情，需要先分清目标。

| 需求 | 常见做法 | 是否需要搬运数据 |
|---|---|---|
| 重建同一台机器上的容器 | 保留并重新挂载原 Volume | 通常不需要 |
| 把数据库搬到另一台机器 | `pg_dump` + 传输 + `pg_restore` | 需要 |
| 修改表、列或索引结构 | Alembic、Flyway 等 Schema Migration | 视变更而定，通常通过迁移脚本调整 |

## 5. 逻辑备份与恢复：`pg_dump` / `pg_restore`

这是本地开发环境迁移到云服务器时，一种通用且相对稳妥的方案。

### 5.1 从源数据库导出

```bash
pg_dump -Fc -h localhost -U postgres -d rag -f rag.dump
```

参数含义：

- `-Fc`：使用 PostgreSQL 自定义归档格式，适合用 `pg_restore` 恢复。
- `-h`：源数据库主机。
- `-U`：数据库用户名。
- `-d`：要导出的数据库。
- `-f`：备份文件路径。

### 5.2 将备份文件传到目标服务器

例如，可以通过 `scp`、SFTP 或其他安全的文件传输方式，将 `rag.dump` 上传到服务器。

### 5.3 在目标环境创建数据库并恢复

先启动目标数据库：

```bash
docker compose up -d
```

确认目标数据库已经创建，且为空或适合恢复的状态；然后执行：

```bash
pg_restore -h localhost -p 5432 -U postgres -d rag rag.dump
```

如果目标数据库尚不存在，可以先通过 `createdb` 或其他方式创建。恢复前应确认目标数据库版本、扩展和权限配置符合要求。

如果希望由恢复工具创建数据库，可采用 `pg_restore --create`，但要确保连接的初始数据库和权限设置正确。具体选项应根据迁移场景决定，不要直接对生产数据库盲目执行。

### 5.4 重要注意事项

- `pg_dump` 一般导出单个数据库的内容；角色等集群级对象需要根据需求另行迁移，例如使用 `pg_dumpall --globals-only` 导出全局对象。
- 源数据库与目标数据库的 PostgreSQL 版本、扩展版本和权限配置应提前确认。
- 使用 pgvector 时，目标环境必须具备相应扩展，并确保向量维度、索引和依赖对象能正确恢复。
- 生产迁移要安排停写或设计增量同步方案，避免导出期间仍在变化的数据造成切换不一致。
- 迁移后应检查表行数、索引、扩展、权限和应用读写功能，并保留可回滚方案。

## 6. 为什么不总是直接复制 Volume？

Volume 中存放的是数据库的物理文件，并不是可随意搬运的通用导出格式。

直接复制物理数据通常需要考虑：

- PostgreSQL 版本、数据目录布局和平台兼容性；
- 数据库是否已安全停止，或是否使用了合适的一致性备份机制；
- 文件权限、所有者和挂载位置；
- 目标环境是否与源环境兼容。

因此，跨机器迁移时，通常优先考虑 `pg_dump` / `pg_restore`。对于大型数据库或要求低停机时间的场景，可以再评估物理备份、流复制或其他专门方案。

## 7. Volume 的生命周期与危险操作

常见命令：

```bash
# 启动服务
docker compose up -d

# 停止并移除 Compose 创建的容器、网络等；命名 Volume 通常会保留
docker compose down

# 危险：同时删除 Compose 管理的命名 Volume，可能导致数据库数据丢失
docker compose down -v
```

不要把“删除容器”和“删除持久化数据”当成同一件事。执行清理命令前，务必确认 Volume 是否需要保留，并确认已经有可用备份。

## 8. Schema Migration：数据库结构如何演进

数据迁移和结构迁移不是同一件事。

例如，一个 RAG 项目最初创建了 `rg_chunks` 表，后来要为它增加 `strategy` 列：

```sql
ALTER TABLE rg_chunks
ADD COLUMN strategy TEXT;
```

项目通常不会每次都手动执行 SQL，而是把变更写成有版本记录的迁移脚本。Python 项目常用 Alembic；其他技术栈也有 Flyway、Liquibase、Django Migrations 等选择。

典型结构：

```text
migrations/
├── 001_create_rg_chunks
├── 002_add_strategy
├── 003_add_hnsw_index
└── 004_add_meta_gin_index
```

将迁移脚本与应用代码一起纳入 Git，可以让不同环境按顺序升级数据库结构，并追踪结构变更历史。

## 9. 推荐的 RAG / Agent 项目部署结构

```text
Git 仓库
├── 应用代码（例如 FastAPI）
├── Docker Compose 配置
└── 数据库结构迁移脚本（例如 Alembic）
          │
          ▼
     云服务器 Docker
     ├── FastAPI 容器
     └── PostgreSQL 容器
          ├── pgvector 扩展
          └── Docker Volume
               └── 持久化数据库文件

数据库备份：pg_dump 生成备份文件
               └── 存放到独立位置，而非只放在同一个 Volume
```

建议把工作拆成三条线：

1. **环境部署**：用 Docker Compose 统一数据库镜像和运行配置。
2. **数据迁移与备份**：用 `pg_dump` / `pg_restore` 处理逻辑备份、跨机器迁移和恢复演练。
3. **结构演进**：用 Alembic 等工具管理表、列和索引的版本化变更。

## 10. 最终心智模型

- **Image**：软件和依赖的模板。
- **Container**：镜像启动后运行的实例。
- **PostgreSQL**：真正负责数据库管理的软件。
- **Volume**：独立于容器生命周期的持久化存储。
- **`pg_dump` / `pg_restore`**：逻辑数据迁移与恢复。
- **Alembic 等工具**：数据库结构变更管理。
- **备份**：用于应对误删、损坏和灾难恢复；Volume 本身不能替代备份。
