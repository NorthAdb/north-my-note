# MCP 与 Skill：机制、安装、调用方式全面对比

## 1. 核心定位区别

两者都是"扩展 Claude 能力"的机制，容易让人觉得功能重合，但解决的是完全不同层面的问题：

| | MCP | Skill |
|---|---|---|
| 解决什么问题 | **够不够得着**——能否连接到外部系统、拿到数据/操作权限 | **做得好不好**——如何用已有能力更专业地完成某类任务 |
| 本质 | 连接协议，接入外部系统（数据库、API、GitHub、Slack…） | 一份 Markdown 操作手册，封装领域知识和最佳实践 |
| 提供的是 | 新的数据源 / 操作权限 | 方法论 / 流程规范 |
| 类比 | 给电脑装新硬件驱动 | 给员工发公司 SOP |

**一句话判断标准**：需要"实时的、属于你个人/组织的、外部世界的数据或操作"→ 只能靠 MCP；需要"把一件事做得更专业、更规范"→ 靠 Skill。

### 典型只有 MCP 能做的事
- "我在 GitHub 上的 PR 现在什么状态？" —— 需要连接权限，Skill 编不出实时数据
- "把这个会议加到我的日历上" —— 需要实际写入操作
- "上次我存的实验设计文档里评分标准是什么" —— 需要读取你实际的私有数据

### 典型 Skill 负责的事
- Word/PPT 文档该怎么排版、目录怎么生成
- 代码审查该按什么规范来
- 某类任务的常见坑和最佳实践

### 一个组合场景
"把 Salesforce 里这季度销售数据整理成专业 PPT"：
1. MCP 连 Salesforce，拉取真实数据（Skill 做不到）
2. pptx skill 指导如何把数据排版成规范的演示文稿（MCP 不管这个）

---

## 2. 安装方式

### 2.1 Skill 的安装

**内置 Skill（Word/Excel/PPT/PDF 等）**
- claude.ai / Desktop：Settings → Features → Skills，逐个开关即可
- 前提：Settings → Capabilities 里的 **Code execution** 需开启

**自定义 Skill**
1. 准备文件夹，至少包含 `SKILL.md`（YAML frontmatter + 说明正文），可附带脚本、模板
2. 打包 zip（`SKILL.md` 必须在解压根目录）
3. Skills 设置页点 "+" 上传
4. 打开对应 toggle

**Claude Code 中**
```bash
# 个人级（所有项目通用）
mkdir -p ~/.claude/skills/crypto-lab-writeup
# 项目级（跟仓库走，可提交到 git 团队共享）
mkdir -p .claude/skills/kernel-module-review
```
验证：会话里输入 `/skills` 查看已加载的 skill

### 2.2 MCP 的安装

**claude.ai / Desktop（Connector）**
- 官方目录里现成的：Customize → Connectors → 浏览目录 → OAuth 授权
- 自定义/第三方远程服务器：Connectors → "+" → Add custom connector，填服务器 URL + 认证方式
- ⚠️ 自定义连接器未经 Anthropic 验证，只连信任的服务器

**Claude Code 中**
```bash
# 远程 HTTP 服务器
claude mcp add --transport http github https://mcp.github.com/mcp

# 本地 stdio 服务器
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem ~/crypto-lab-docs

# 带密钥的服务器
claude mcp add lab-grading --env API_KEY=sk-abc123 -- node /path/to/server.js

# 管理命令
claude mcp list
claude mcp remove github -s user
```

---

## 3. 项目级 / 个人级的区分

### Skill：两级
| 级别 | 位置 | 共享方式 |
|---|---|---|
| 个人级 | `~/.claude/skills/` | 本机所有项目通用 |
| 项目级 | `.claude/skills/`（仓库内） | 提交到 git，协作者 clone 后自动获得 |

### MCP：三级（比 Skill 多一层）
| Scope         | 存放位置                  | 共享范围       | 是否提交 git |
| ------------- | --------------------- | ---------- | -------- |
| **local**（默认） | 本地配置，绑定当前目录           | 仅你、仅当前项目目录 | 否        |
| **project**   | 仓库根目录 `.mcp.json`     | 提交后团队所有人共享 | 是        |
| **user**      | 全局配置 `~/.claude.json` | 本机所有项目通用   | 否        |

```bash
claude mcp add my-server -s project -- node server.js   # 团队共享
claude mcp add github -s user --transport http https://mcp.github.com/mcp  # 个人全局
```

⚠️ **MCP 独有的坑**：`.mcp.json` 提交到公开仓库时不能明文写密钥，要用环境变量占位符：
```json
{
  "mcpServers": {
    "lab-grading": {
      "type": "http",
      "url": "https://internal.xidian.edu/mcp",
      "headers": { "Authorization": "Bearer ${LAB_API_KEY}" }
    }
  }
}
```

---

## 4. 调用方式

### 共同点
默认都是**模型自主判断**是否调用，不需要用户手动触发。

### Skill 的调用
- **自动触发**：Claude 读取所有已装 Skill 的 name/description，语义匹配就自动加载
- **显式触发**：Claude Code 中 Skill 与斜杠命令合并，可直接 `/skill名` 强制调用
- **禁止自动触发**：在 `SKILL.md` frontmatter 加 `disable-model-invocation: true`，之后只能靠手动 `/名称` 触发（适合有副作用、不可逆的操作）

### MCP 的调用（三种显式方式，确定性由弱到强）

| 方式 | 确定性 | 前提 |
|---|---|---|
| 自然语言点名（"用 github MCP 查一下…"） | 模型仍有裁量权，但命中率高 | 无 |
| `@resource` 引用（如 `@github:issue://123`） | 完全确定性 | 服务器暴露了 resource |
| `/mcp__server__prompt`（如 `/mcp__github__pr_review 456`） | 完全确定性，跳过模型判断 | 服务器暴露了 prompt |

⚠️ `/mcp` 命令本身不是调用工具的入口，是**连接管理面板**（查看状态、重连、OAuth 认证）。

---

## 5. 上下文加载机制的不同

### Skill：渐进式加载（Progressive Disclosure），从设计第一天就是分层的
1. **常驻层**：所有已装 Skill 的 name + description，体积很小（几十 token 级别）
2. **触发后加载**：命中后读入完整 `SKILL.md` 正文（几千 token 级别）
3. **按需加载**：`SKILL.md` 引用的脚本/模板/参考文档，只在真正用到时才读

本质是**纯文本知识注入**，不需要协议转换。

### MCP：结构化 Schema，早期全量注册，后期才补上检索优化
- 每连一个服务器，其工具的完整 JSON Schema（参数类型、说明、返回格式）都要注册进模型的工具列表——这是**结构化数据**，模型需要精确的 schema 才能正确调用
- 早期问题：工具定义本身占用大量上下文（如单个 Chrome DevTools 服务器可能占用两万 token 级别），连接多个服务器容易context 膨胀
- 后来优化：默认开启 **Tool Search**，会话开始只加载工具名，真正需要时才检索完整定义——这一步实际上是在向 Skill 的渐进式加载思路靠拢，只是起点不同

### 为什么设计成两套不同机制

| 维度 | Skill | MCP |
|---|---|---|
| 单元本质 | 知识/流程，靠语义理解使用 | 外部系统的调用接口，需要严格 schema |
| 状态性 | 静态文本，不会自己变化 | 连接活的外部系统，工具列表可能动态变化 |
| 失败模式 | 顶多按错误流程做事，可随时纠正 | 可能造成不可逆副作用（删数据、发错邮件） |
| 对应的权限设计 | 仅有"是否允许自动触发"开关 | 有强制人工确认机制（`requiresUserInteraction`），还需专门的连接管理（OAuth、健康检查、重连） |

**一句话总结**：Skill 负责传递"知道该怎么做"的经验，MCP 负责获得"能不能做到"的权限——知识可以静态加载、按需展开，权限必须精确定义、动态管理、且对高风险操作加一道人工确认。

---

## 6. 附：两个实用的第三方文档类 MCP

| 工具 | 解决什么 | 用法 |
|---|---|---|
| **Context7** | 库/框架最新版本的官方文档和 API，避免用到过时或虚构的接口 | 提示词里加 "use context7"，或指定库 ID 精确匹配 |
| **DeepWiki** | 对具体 GitHub 仓库做索引分析，生成结构化文档（即使仓库本身没有像样文档） | 直接询问对某个仓库的理解，如 "分析一下这个仓库的模块结构" |

**区别**：Context7 面向"官方发布的文档"，DeepWiki 面向"任意仓库的源代码本身"。两者互补但不是"覆盖一切"——都依赖公开可索引的内容，接触不到私有代码或闭源文档，且都不能替代对复杂设计意图（如某个安全设计的深层原因）的人工理解。
