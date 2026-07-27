# MCP 服务器最佳实践

> 中文翻译整理自 `best-practice/claude-mcp.md`
> 学习顺序：第 6 篇
> 官方文档：[MCP Servers](https://code.claude.com/docs/en/mcp)

---

## 什么是 MCP？

MCP（Model Context Protocol）是 Claude Code 的**扩展协议**，让 Claude 可以连接外部工具、数据库和 API。

**通俗理解：** MCP 就像 Claude Code 的"USB 接口"——需要什么功能就插什么外设。

---

## 日常推荐的 MCP 服务器

> 社区共识：不要贪多，15 个不如 4 个用得精。

| MCP 服务器 | 功能 | 推荐理由 |
|------------|------|----------|
| **Context7** | 获取最新库文档 | 防止 Claude 用过时的训练数据生成虚假 API |
| **Playwright** | 浏览器自动化 | 截图、导航、表单测试，前端开发必备 |
| **Claude in Chrome** | 连接真实 Chrome | 查看控制台日志、网络请求、DOM |
| **DeepWiki** | GitHub 仓库文档 | 获取任何仓库的架构文档 |
| **Excalidraw** | 架构图生成 | 画流程图、系统设计图 |

**工作流：** Research（Context7/DeepWiki）→ Debug（Playwright/Chrome）→ Document（Excalidraw）

---

## 配置方式

MCP 的配置分散在两个文件中，各有不同职责：

### 配置文件分工

| 文件 | 职责 | 例子 |
|------|------|------|
| **`.mcp.json`** | 定义 MCP 服务器**怎么安装/启动** | `command: "npx", args: ["@playwright/mcp"]` |
| **`settings.json`** | 控制 MCP 服务器**能不能用 + 谁可以用** | 服务器启停、工具权限白名单 |

### 服务器类型

| 类型 | 传输方式 | 示例 | 说明 |
|------|----------|------|------|
| **stdio** | 启动本地进程 | `npx`、`python`、二进制文件 | 🔸 **默认值，可省略 `type`**。用 `command` + `args` 指定启动命令 |
| **http** | 连接远程 URL | HTTP/SSE 端点 | 必须写明 `"type": "http"`，用 `url` 指定端点 |

### 一、定义安装 → `.mcp.json`

这里只关心"这个 MCP 怎么启动"，不关心权限。

```json
{
  "mcpServers": {
    "playwright": {
      // type 省略 = stdio（默认）
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    },
    "remote-api": {
      "type": "http",                  // 远程连接，必须写 type
      "url": "https://api.example.com/mcp?token=${MY_TOKEN}"
    }
  }
}
```

### 二、服务器启停控制 → `settings.json`

决定哪些 MCP 服务器能运行（服务器级别，不是工具级别）：

```json
{
  "enableAllProjectMcpServers": true,      // 全部放行
  "enabledMcpjsonServers": ["memory"],     // 只放行白名单
  "disabledMcpjsonServers": ["old-server"] // 黑名单屏蔽
}
```

### 三、工具调用权限 → `settings.json`

决定 MCP 的每个具体工具能不能被 Claude 调用。这里可以**细粒度到单个工具级别**：

- `mcp__服务器名__*` — 通配符，放行该服务器的**所有工具**
- `mcp__服务器名__具体工具` — 精确到**某个具体工具**

```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",                  // 放行 context7 所有工具
      "mcp__playwright__browser_snapshot"  // 只放行 playwright 的截图，其他不放行
    ],
    "deny": [
      "mcp__dangerous-server__*"           // 禁止某个服务器的所有工具
    ]
  }
}
```

### 三层关系总结

```
.mcp.json（安装）          settings.json（管控）
                ┌─────────────────────────────────┐
┌──────────┐   │  enableAllProjectMcpServers       │
│ Playwright├───→  这个服务器能启动吗？            │
│ 定义启动   │   │  enabledMcpjsonServers           │
└──────────┘   ├─────────────────────────────────┤
               │  permissions.allow:              │
┌──────────┐   │  mcp__playwright__*              │
│ Context7 ├───→  这个工具能被 Claude 用吗？       │
│ 定义启动   │   │  permissions.deny:              │
└──────────┘   │  mcp__context7__dangerous-tool    │
               └─────────────────────────────────┘
```

---

## MCP 上下文加载机制

MCP 和 Skill 的上下文加载方式有本质区别：

### Skill：渐进式披露

```
每次会话加载到系统提示：
  skill 列表 + 描述（受 skillListingBudgetFraction 限制，默认 1%）
       ↓
Claude 决定调用某个 skill
       ↓
完整 SKILL.md 内容注入上下文
       ↓
（可选）context: fork → 子代理隔离执行
```

先看"菜单"（描述），点菜后才上"菜"（完整内容），不常用的 skill 自动折叠为仅名称，节省 token。

### MCP：全部注册到工具列表

```
每次会话：
  所有 MCP 服务器的所有工具定义
  （工具名 + 描述 + 参数 Schema）→ 全部加载到系统提示
       ↓
Claude 在工具列表中看到全部工具，需要时直接调用
```

**不是渐进式**——每个工具的定义（名称、描述、参数 Schema）从一开始就在系统提示里占 token。MCP 越多，token 消耗越大。

### 优化：MCPSearch 自动模式（v2.1.7+）

当 MCP 工具定义超过上下文 10% 时，自动启用按需查找：

```
工具少（< 10% 上下文）：全部工具定义直接加载
工具多（> 10% 上下文）：工具定义延迟加载，通过 MCPSearch 按需查找
```

可通过环境变量调整阈值：`ENABLE_TOOL_SEARCH=auto:N`（N=0-100，百分比）。

### 对比总结

| | **Skill** | **MCP** |
|--|----------|---------|
| **加载时机** | 列表描述在系统提示，完整内容按需加载 | 工具定义**默认全部加载**到系统提示 |
| **控制预算** | `skillListingBudgetFraction`（默认 1%） | MCPSearch 自动阈值（10%，超出后按需查找） |
| **渐进式披露** | ✅ 是（描述→内容→fork 隔离） | ⚠️ 部分（超过阈值才按需加载） |
| **执行隔离** | `context: fork` | 无内置隔离机制 |
| **token 消耗** | 只加载描述，不常用的自动折叠 | 每个工具定义（名称+描述+参数 Schema）都占 token |

**简单说：** Skill 天然就是渐进式的——"菜单"占很少 token，"菜"用的时候才上。MCP 更像工具箱——所有工具摆在台面上随取随用，但摆得越多占地方越大。MCPSearch 是补救措施，工具太多时把不常用的收起来，需要时再找。

---

## MCP 作用域

MCP 服务器可以在三个层级定义：

| 层级 | 位置 | 用途 |
|------|------|------|
| **项目级** | `.mcp.json`（项目根目录） | 团队共享，提交到 git |
| **用户级** | `~/.claude.json`（`mcpServers` 键） | 个人跨项目配置 |
| **Subagent 级** | Agent frontmatter（`mcpServers` 字段） | 特定 agent 专用 |

**优先级：** Subagent > Project > User

---

## 最佳实践

1. **精选少量** — 5 个以内最佳，太多会降低性能和准确性
2. **用环境变量保护密钥** — 使用 `${VAR}` 语法，不要硬编码 API key
3. **上下文感知** — Context7 + DeepWiki 一起用覆盖一切文档需求
4. **浏览器自动化二选一** — Playwright 或 Claude in Chrome，不要同时用
5. **定期清理** — 不再需要的 MCP 服务器及时移除

---

## 更多阅读

- [Browser Automation MCP 对比报告](../../reports/claude-in-chrome-v-chrome-devtools-mcp.md)
- [MCP 规范](https://modelcontextprotocol.io/)
- [Reddit 讨论：5 个真正让我快 10 倍的 MCP](https://reddit.com/r/mcp/comments/1qarjqm/)

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Settings 配置 | [`05-设置-settings.md`](./05-设置-settings.md#三mcp-服务器设置) | MCP 相关设置项详解（`enableAllProjectMcpServers` 等）|
| Subagent | [`02-子代理-subagents.md`](./02-子代理-subagents.md) | 为 Subagent 配置专用 MCP 服务器 |
| 项目 `.mcp.json` | 仓库根目录 `.mcp.json` | 本仓库实际使用的 MCP 配置 |
| 实现参考 | [`../implementation/01-subagent-实现.md`](../implementation/01-subagent-实现.md) | 应用中 MCP 与 Subagent 组合的实例 |

---

> 下一篇：[07-记忆系统-memory.md](./07-记忆系统-memory.md)
