# Claude Code 完全学习指南

> 🔗 **本指南围绕 `claude-code-best-practice` 仓库编写**，章节顺序与 `doc/practice/` 文档编号一致。
>
> 学习方式：**按章节顺序，边读边打开项目中对应的文件**，理论和实际对照着看。

---

## 📋 目录

- [第零章：项目全景](#第零章项目全景)
- [第一章：概念总览](#第一章概念总览)
- [第二章：子代理 Subagents](#第二章子代理-subagents)
- [第三章：命令 Commands](#第三章命令-commands)
- [第四章：技能 Skills](#第四章技能-skills)
- [第五章：配置体系 Settings](#第五章配置体系-settings)
- [第六章：MCP 服务器](#第六章mcp-服务器)
- [第七章：记忆系统 Memory](#第七章记忆系统-memory)
- [第八章：工作流编排](#第八章工作流编排)
- [第九章：钩子系统 Hooks](#第九章钩子系统-hooks)
- [第十章：CLI 启动参数](#第十章cli-启动参数)
- [第十一章：能力强化 Power Ups](#第十一章能力强化-power-ups)
- [第十二章：实用技巧合集](#第十二章实用技巧合集)
- [第十三章：学习路径](#第十三章学习路径)

---

## 第零章：项目全景

### 0.1 这个仓库是什么

`claude-code-best-practice` 是一个 **Claude Code 配置示范仓**——它不是一个应用项目，而是一个**最佳实践参考实现**。

```
┌─────────────────────────────────────────────────┐
│ 你可以把它当作：                                   │
│  • 一个 Claude Code 配置的百科全书                  │
│  • 一个可以直接复用的技能/命令/代理模板库               │
│  • 一个学习 Claude Code 的实操环境                    │
└─────────────────────────────────────────────────┘
```

### 0.2 项目目录结构

```
📁 claude-code-best-practice/
│
├── 📄 CLAUDE.md              ← 项目级指令（给 Claude 读的 README）
├── 📄 README.md              ← 导航中心
│
├── 📁 .claude/               ← Claude Code 配置目录
│   ├── 📁 skills/    ~40个   ← 每个子目录是一个 skill
│   ├── 📁 commands/  10个    ← 每个 .md 文件是一个 command
│   ├── 📁 agents/    11个    ← 每个 .md 文件是一个 subagent
│   ├── 📁 hooks/             ← 钩子脚本和配置
│   ├── 📁 rules/             ← 懒加载规则
│   ├── 📄 settings.json      ← 共享配置
│   └── 📄 mcp.json           ← MCP 配置
│
├── 📁 .agents/skills/        ← 跨工具通用 skill 格式（npx skills）
│
├── 📁 doc/                   ← 中文文档（本指南的核心参考目录）
│   ├── 📁 practice/    10篇  ← 各功能的中文最佳实践，编号 01-10  ← 📍 主学习路径
│   ├── 📁 implementation/ 6篇 ← 中文实现说明
│   ├── 📁 reports/     12篇  ← 中文研究报告
│   ├── 📁 tips/          9篇  ← 中文技巧合集
│   └── 📁 rules/         2篇  ← 中文规则
│
├── 📁 best-practice/         ← 英文原版（`doc/practice/` 为中文翻译）
├── 📁 implementation/        ← 英文原版
├── 📁 reports/               ← 英文原版
├── 📁 tips/                  ← 英文原版
│
├── 📁 orchestration-workflow/  ← 天气编排系统演示
└── 📁 presentation/            ← 演讲幻灯片
```

> 💡 **中文文档索引**：英文原版在 `best-practice/`、`implementation/` 等目录，中文翻译版统一在 `doc/` 下。
> 本指南优先使用中文版，并标注对应的英文原版位置。

### 0.3 文档编号体系

`doc/practice/` 下的文档按编号 01-10 顺序排列，这也是**推荐的学习顺序**：

```
01 → 概念总览      06 → MCP 服务器
02 → 子代理         07 → 记忆系统
03 → 命令           08 → CLI 启动参数
04 → 技能           09 → 能力强化
05 → 设置           10 → 学习路径
```

本指南的章节顺序完全对应这个编号。

### 0.4 先了解 "Harness"

**"Harness"** 是 Claude Code 的**上下文构建与编排架构**。Skills、Commands、Subagents、Hooks 都是它的组成部分。它不是简单的"提示词模板"，而是：

```
用户输入的 prompt
  + CLAUDE.md（项目指令）
  + 被激活的 Skill 指令集
  + 项目 settings 配置
  + Hooks 的前后处理
  + 被调用的 Subagent 上下文
  ────────────────────────────────
  = Claude 实际收到的完整上下文
```

📄 **`doc/reports/为什么Harness如此重要.md`** — 深入理解 Harness 的价值

### 0.5 学习起点文件

| 文件 | 用途 |
|------|------|
| 📄 **`doc/CLAUDE中文注释.md`** | CLAUDE.md 的中文注释版，方便理解 |
| 📄 **`doc/skills-inventory.md`** | 所有已安装的 skills、agents、commands 完整清单 |
| 📄 **`doc/practice/01-CONCEPTS-概念总览.md`** | 核心概念快速总览 |

---

## 第一章：概念总览

> 对应文档：**`doc/practice/01-CONCEPTS-概念总览.md`**
>
> 英文原版：README.md（CONCEPTS 表格部分）

这个文件是**整个仓库的索引**。读完它你应该理解：

| 概念 | 一句话理解 |
|------|-----------|
| **Subagent（子代理）** | 独立的 Claude 会话，隔离上下文，适合并行任务 |
| **Command（命令）** | 用 `/name` 一键触发的快捷入口 |
| **Skill（技能）** | 教 Claude 怎么做某件事的指令集 |
| **Settings（设置）** | 5 层配置体系，控制权限/行为/扩展 |
| **MCP** | 让 Claude 能调用外部服务的协议 |
| **Memory（记忆）** | 跨会话持久化信息 |
| **Hooks（钩子）** | 事件驱动的前后处理脚本 |
| **Workflow（工作流）** | 多 Agent 并行编排的工具 |

**三种扩展机制的简要对比：**

| 维度 | Subagent | Command | Skill |
|------|----------|---------|-------|
| **上下文隔离** | ✅ 独立 | ❌ 共享 | ❌ 共享（除非 `fork`） |
| **被 Claude 主动调用** | ✅ | ❌ | ✅ |
| **被用户 `/` 调用** | ❌ | ✅ | ✅ |
| **参数支持** | ✅ | ✅ | ❌ |
| **适合** | 并行独立任务 | 一键流程入口 | 行为指令 |

📄 **`doc/reports/Agent-vs-Command-vs-Skill.md`** — 三种机制的深度对比

---

## 第二章：子代理 Subagents

> 对应文档：**`doc/practice/02-子代理-subagents.md`**
>
> 英文原版：`best-practice/claude-subagents.md`
>
> 项目实例：`.claude/agents/*.md`

### 2.1 先看理论

Subagent 是一个**独立的 Claude 会话**，有自己的上下文。可以把它理解成"给你派了个分身去干活"。

> 💡 **为什么要用 Subagent？**
> 主会话有 1M token 上限，复杂任务会让上下文膨胀。
> Subagent 把任务隔离出去，主会话只做协调，大幅降低上下文压力。

### 2.2 核心字段

| 字段 | 作用 | 示例 |
|------|------|------|
| `name` | Agent 名字 | `weather-agent` |
| `description` | 何时使用 | 含 `PROACTIVELY` 则自动触发 |
| `tools` | 允许的工具 | `Read, Grep, Edit, Bash` |
| `disallowedTools` | 禁止的工具 | `Write` |
| `model` | 模型选择 | `haiku`, `sonnet`, `opus`, `inherit` |
| `skills` | 预载的 skill | `[weather-fetcher]` |
| `permissionMode` | 权限模式 | `acceptEdits`, `plan`, `bypassPermissions` |
| `maxTurns` | 最大交互轮数 | `20`（防跑飞） |
| `memory` | 记忆范围 | `user`, `project`, `local` |
| `effort` | 推理努力度 | `low`, `medium`, `high`, `max` |
| `isolation` | 是否隔离 worktree | `worktree`（并行修改文件安全） |
| `color` | CLI 输出颜色 | `green`, `blue`, `yellow`... |
| `background` | 是否后台运行 | `true` |
| `hooks` | 作用域钩子 | 只在该 agent 生命周期生效 |

### 2.3 看项目实例

📄 **`doc/implementation/01-subagent-实现.md`**（中文）← 实现说明

英文原版 `implementation/claude-subagents-implementation.md`

**核心示例：`weather-agent`**

📄 **`.claude/agents/weather-agent.md`**

```yaml
---
name: weather-agent
description: Use this agent PROACTIVELY when you need to fetch weather data for Dubai, UAE
allowedTools:
  - Skill(weather-fetcher)
  - Bash
model: inherit
skills:
  - weather-fetcher
---
```

注意 `skills: [weather-fetcher]` — agent 启动时自动把 weather-fetcher skill 的指令加载到上下文。

**更多 Agent 示例**：

📄 **`.claude/agents/time-agent.md`** — 简单 agent 示例

📄 **`.claude/agents/workflows/best-practice/`** — 5 个研究用 agent，都使用 `model: opus`

```yaml
# workflow-claude-commands-agent.md
---
name: workflow-claude-commands-agent
description: Research agent that fetches Claude Code docs...
model: opus
---
```

> ✅ **对照看**：打开 `weather-agent.md` 和 `time-agent.md`，比较 `allowedTools` 和 `skills` 字段。

### 2.4 如何在对话中调用 Subagent

```javascript
Agent(subagent_type="weather-agent",
      description="Get Dubai weather",
      prompt="Fetch the current temperature in Dubai")

// 带高级参数
Agent(subagent_type="agent-name",
      model="haiku",
      permissionMode="bypassPermissions",
      maxTurns=10)
```

### 2.5 内置 Agent 类型

| Agent 类型 | 用途 | 可用工具 |
|-----------|------|---------|
| `general-purpose` | 通用（默认） | 全部 |
| `Explore` | 只读搜索 | Read, Grep, Glob 等 |
| `Plan` | 架构设计 | 全部（只读模式） |
| `claude-code-guide` | Claude Code 用法问答 | 搜索 + 网页 |
| `statusline-setup` | 状态栏配置 | Read, Edit |

### 实操练习

1. 打开 `weather-agent.md` 看完整 frontmatter
2. 打开 `time-agent.md` 对比
3. 试着在对话里用 `Agent()` 调用一个子代理

---

## 第三章：命令 Commands

> 对应文档：**`doc/practice/03-命令-commands.md`**
>
> 英文原版：`best-practice/claude-commands.md`
>
> 项目实例：`.claude/commands/*.md`

### 3.1 先看理论

Command 是**快捷入口**，用 `/command-name` 触发。和 Skill 最大的区别是：

> **Command 不能被 Claude 主动调用**，必须用户输入 `/` 触发。它适合做流程入口。

### 3.2 核心结构

```markdown
---
name: command-name      # / 后面用的名字
description: 一句话说明  # 在 / 列表里显示
model: haiku            # 可选，指定模型
---
```

### 3.3 看项目实例

📄 **`doc/implementation/02-command-实现.md`**（中文）← 实现说明

英文原版 `implementation/claude-commands-implementation.md`

**核心示例：`/weather-orchestrator`**

📄 **`.claude/commands/weather-orchestrator.md`**

这个 Command 演示了完整的 **Command → Agent → Skill** 三阶段流程：

```
1. 问用户要摄氏还是华氏
2. 调用 weather-agent 获取温度
     └── 预载 weather-fetcher skill 获取实时数据
3. 调用 weather-svg-creator 生成 SVG
```

📄 **`.claude/commands/time-command.md`** — 最简 command 示例

📄 **`.claude/commands/workflows/`** — 自动化工作流命令（使用 `Workflow` 工具）

### 3.4 内置 Slash Commands（86+ 个）

常用内置命令：

| 命令 | 用途 |
|------|------|
| `/help` | 查看帮助 |
| `/doctor` | 诊断问题 |
| `/compact` | 手动压缩上下文 |
| `/clear` | 清屏 |
| `/usage` | 查看用量 |
| `/model` | 切换模型 |
| `/fast` | 切换到快速模式 |
| `/init` | 初始化 CLAUDE.md |
| `/cost` | 查看本轮花费 |

### 实操练习

1. 打开 `time-command.md` — 最简单的 command，看结构
2. 打开 `weather-orchestrator.md` — 看编排类 command
3. 看看 `workflows/` 目录下的 command 有什么不同

---

## 第四章：技能 Skills

> 对应文档：**`doc/practice/04-技能-skills.md`**
>
> 英文原版：`best-practice/claude-skills.md`
>
> 项目实例：`.claude/skills/*/SKILL.md`

### 4.1 先看理论

Skill 本质上是一个 **Markdown 指令文件**，教 Claude 如何完成特定类型的工作。它放在 `.claude/skills/<name>/SKILL.md` 里。

### 4.2 核心字段

| 字段 | 作用 |
|------|------|
| `name` | Skill 名称，也是 `/` 命令名 |
| `description` | 告诉 Claude 什么时候该用 |
| `disable-model-invocation: true` | 设为 true 后只能手动 `/` 调用 |
| `allowed-tools` | 激活时允许的工具白名单 |
| `model` | 指定模型（sonnet/opus/haiku） |
| `context: fork` | 在独立子会话中运行 |
| `user-invocable: false` | 隐藏，不让用户看到 |

### 4.3 看项目实例

📄 **`doc/implementation/03-skill-实现.md`**（中文）← 实现说明

英文原版 `implementation/claude-skills-implementation.md`

项目里演示了两种 Skill 模式：

**模式 A：Agent Skill（预载模式）**

Skill 作为 Agent 的预载知识，Agent 启动时自动加载。

📄 **`.claude/skills/weather-fetcher/SKILL.md`**

```markdown
---
name: weather-fetcher
description: Instructions for fetching current weather temperature data for Dubai, UAE
user-invocable: false   # ← 关键：用户不可见，只给 agent 用
---
```

```yaml
# .claude/agents/weather-agent.md 中通过 skills 字段引用
skills:
  - weather-fetcher
```

**模式 B：独立 Skill（调用模式）**

Skill 被用户或主会话通过 `Skill` 工具显式调用。

📄 **`.claude/skills/weather-svg-creator/SKILL.md`**

```markdown
---
name: weather-svg-creator
description: Creates an SVG weather card showing the current temperature for Dubai
---
```

### 4.4 内置 Skills（12+ 个）

| Skill | 用途 | 调用方式 |
|-------|------|----------|
| `/code-review` | 审查当前 diff | `/code-review` |
| `/verify` | 运行并验证 | `/verify` |
| `/run` | 启动项目 | `/run` |
| `/init` | 创建 CLAUDE.md | `/init` |
| `/loop` | 定时重复 | `/loop 5m /command` |
| `/simplify` | 简化代码 | `/simplify` |
| `/claude-api` | API 参考查询 | `/claude-api` |
| `/review` | 审查 GitHub PR | `/review` |
| `/security-review` | 安全审查 | `/security-review` |

> ⚠️ **注意**：mattpocock 的 `code-review` 和 Claude Code 内置的 `code-review` 功能重叠。
> 内置版是单维审查，mattpocock 版是双轴（Standards + Spec）并行审查。

### 4.5 本项目所有已安装 Skills（31 个）

📄 **`doc/skills-inventory.md`** — 详细清单，每个 skill 有功能说明和调用方式

### 实操练习

1. 选一个 skill，打开它的 `SKILL.md`
2. 看 frontmatter（`---` 之间的 YAML）
3. 看正文怎么组织指令
4. 对比 `weather-fetcher`（预载模式）和 `weather-svg-creator`（独立模式）的区别

---

## 第五章：配置体系 Settings

> 对应文档：**`doc/practice/05-设置-settings.md`**
>
> 英文原版：`best-practice/claude-settings.md`
>
> 项目实例：`.claude/settings.json`

### 5.1 先看理论

📄 **`doc/practice/05-设置-settings.md`**（中文）— 80+ 个设置项、200+ 环境变量

英文原版 `best-practice/claude-settings.md`

### 5.2 配置层级（5 层）

每一层覆盖上一层，数字越小的优先级越高：

```
1️⃣  Managed Settings（组织强制）→ 不能被覆盖
2️⃣  CLI Arguments（启动参数）→ 单次会话有效
3️⃣  .claude/settings.local.json（个人项目）→ git-ignored
4️⃣  .claude/settings.json（团队共享）→ 版本控制
5️⃣  ~/.claude/settings.json（全局默认）
```

### 5.3 看项目配置

📄 **`.claude/settings.json`** — 项目共享配置

```json
{
  "permissions": {
    "allow": ["git", "npm", "npx"],
    "deny": ["rm -rf"]
  },
  "hooks": {
    "PreToolUse": "scripts/hooks.py",
    "PermissionRequest": "scripts/hooks.py"
  }
}
```

### 5.4 全局 vs 项目配置

📄 **`doc/reports/全局vs项目设置对比.md`**（中文）← 推荐

英文原版 `reports/claude-global-vs-project-settings.md`

- **全局配置**（`~/.claude/settings.json`）：通用偏好（主题、默认模型、个人 MCP）
- **项目配置**（`.claude/settings.json`）：团队约束（权限白名单、钩子、项目 MCP）

### 实操练习

1. 打开 `.claude/settings.json`，看项目配置了什么
2. 打开 `~/.claude/settings.json`，看全局默认配置
3. 比较两者的差异

---

## 第六章：MCP 服务器

> 对应文档：**`doc/practice/06-MCP服务器-mcp.md`**
>
> 英文原版：`best-practice/claude-mcp.md`
>
> 项目实例：`.claude/mcp.json`

### 6.1 先看理论

MCP（Model Context Protocol）让 Claude Code 可以**访问外部服务**。每个 MCP 服务器提供一组工具，Claude 可以在对话中按需使用。

### 6.2 服务器类型

| 类型 | 说明 | 配置方式 |
|------|------|---------|
| `stdio` | 本地启动进程 | `"command": "npx", "args": [...]` |
| `http` | 远程 HTTP 服务 | `"url": "https://...", "headers": {...}` |

### 6.3 推荐 MCP 服务器

| MCP | 用途 | 推荐度 |
|-----|------|--------|
| **Context7** | 查框架/库的实时文档 | ⭐⭐⭐⭐⭐ |
| **Playwright** | 浏览器自动化 | ⭐⭐⭐⭐⭐ |
| **Chrome DevTools** | 前端调试 | ⭐⭐⭐⭐ |
| **DeepWiki** | 开源项目知识检索 | ⭐⭐⭐⭐ |
| **Excalidraw** | 画架构图 | ⭐⭐⭐ |

📄 **`doc/reports/浏览器自动化MCP对比.md`**（中文）— Chrome DevTools vs Playwright 对比

### 6.4 配置示例

```json
{
  "mcpServers": {
    "context7": {
      "type": "http",
      "url": "https://api.context7.com/v1",
      "headers": { "Authorization": "Bearer $CONTEXT7_API_KEY" }
    }
  }
}
```

### 实操练习

1. 打开 `.claude/mcp.json` 看 MCP 配置
2. 试在当前对话里用 Context7 查一个库的文档

---

## 第七章：记忆系统 Memory

> 对应文档：**`doc/practice/07-记忆系统-memory.md`**
>
> 英文原版：`best-practice/claude-memory.md`

### 7.1 先看理论

记忆系统让 Claude **跨会话记住信息**。它不是写在 CLAUDE.md 里的固定指令，而是动态积累的知识。

📄 **`doc/reports/子代理记忆系统.md`**（中文）— Agent 记忆深度报告

### 7.2 记忆类型

| 类型 | 用途 | 示例 |
|------|------|------|
| `user` | 你的角色和偏好 | "我是全栈 TypeScript 开发者" |
| `feedback` | 你给 Claude 的反馈 | "先写测试再写实现" |
| `project` | 项目决策和约束 | "用 pnpm 而非 npm" |
| `reference` | 外部资源链接 | "API 地址、工单链接" |

### 7.3 记忆文件格式

记忆文件存放在 `~/.claude/projects/<project-hash>/memory/`：

```markdown
---
name: project-tech-stack
description: 项目技术栈决策记录
metadata:
  type: project
---

使用 Next.js 15 + pnpm + Vitest
**Why:** 团队 React 背景，性能优先
**How to apply:** 新功能优先 Server Component
```

### 7.4 CLAUDE.md 的两种加载机制

1. **Ancestor 加载**：启动时从当前目录向上找 CLAUDE.md，所有祖先目录的都加载
2. **Descendant 懒加载**：`.claude/rules/*.md` 只在触及匹配文件时才加载

这对 monorepo 特别有用——根目录写全局规则，子项目文件触发子规则。

📄 **`doc/rules/Markdown文档规范.md`**（中文）— 文档编写规范
📄 **`doc/rules/演示文稿规范.md`**（中文）— 演示文稿维护规范

---

## 第八章：工作流编排

> 对应文档：**`doc/orchestration-workflow/编排工作流.md`**
>
> 英文原版：`orchestration-workflow/orchestration-workflow.md`

### 8.1 Command → Agent → Skill 模式

这是本仓库展示的**核心架构模式**，也是可以在你项目里复用的设计。

```
用户输入 /weather-orchestrator
        │
        ▼
┌─────────────────────────────────┐
│  Command                         │
│  weather-orchestrator.md        │
│  ───────────────────────────    │
│  1. 问用户要摄氏还是华氏          │
│  2. 调用 weather-agent 获取温度   │
│  3. 调用 weather-svg-creator     │
└─────────────────────────────────┘
        │
        ├──→ Agent + Skill（预载模式）
        │    weather-agent
        │    └── weather-fetcher skill
        │         └── Open-Meteo API
        │
        └──→ Skill（独立调用模式）
             weather-svg-creator
             └── 写 weather.svg + output.md
```

### 8.2 三个实现文档对照看

这三个中文实现文档描述的是**同一个流程的三个视角**：

📄 **`doc/implementation/02-command-实现.md`** — Command 视角：我是入口，我调用了谁

📄 **`doc/implementation/01-subagent-实现.md`** — Agent 视角：我被调用了，我用 skill 干了活

📄 **`doc/implementation/03-skill-实现.md`** — Skill 视角：我被预载到 agent 里，或者被直接调用

> 💡 **建议**：三个文档一口气读完，你会发现它们在描述同一个流程。

### 8.3 Weather 系统设计演进

📄 **`doc/reports/Weather工作流重构历程.md`**（中文）— 从头到尾的设计演进

### 8.4 Workflow 高级编排

对于更复杂的多 Agent 并行任务，项目使用 **Workflow** 工具：

📄 **`.claude/commands/workflows/`** — 6 个自动维护 workflow

```
/agent-collections         → 并行研究所有 agent 集合仓库
/skill-collections         → 并行研究所有 skill 集合仓库
/development-workflows     → 并行研究所有 workflow 仓库
/workflow-concepts         → 更新 CONCEPTS 表
/workflow-claude-commands  → 分析 commands drift
/workflow-claude-settings  → 分析 settings drift
```

### 实操练习

1. 执行 `/weather-orchestrator` 看完整流程
2. 打开 `编排工作流.md` 看架构图
3. 一口气读完三个 implementation 文档

---

## 第九章：钩子系统 Hooks

> 英文原版参考：`best-practice/claude-subagents.md`（搜索 hooks 部分）
>
> 项目实例：`.claude/hooks/`

### 9.1 先看理论

Hooks 是 Claude Code 的**事件系统**——在特定时刻自动执行脚本。比如每次 Claude 使用工具前播放一个声音、每次会话结束时记录日志。

### 9.2 支持的事件

| 事件 | 触发时机 | 典型用途 |
|------|----------|----------|
| `UserPromptSubmit` | 用户提交 prompt 后 | 记录输入 |
| `PreToolUse` | 工具调用前 | 安全检查 |
| `PostToolUse` | 工具调用后 | 记录输出 |
| `SessionStart` | 会话启动时 | 播放入场音效 |
| `SessionEnd` | 会话结束时 | 记录总结 |
| `SubagentStart` | 子代理启动时 | 通知用户 |
| `SubagentStop` | 子代理停止时 | 通知结果 |
| `PermissionRequest` | 请求权限时 | 自动审批 |
| `PreCompact` | 上下文压缩前 | 保存状态 |

### 9.3 看项目实现

📄 **`.claude/hooks/`** — 完整的钩子系统

```
.claude/hooks/
├── config/
│   ├── hooks-config.json          ← 团队共享的钩子配置
│   └── hooks-config.local.json    ← 个人覆盖（git-ignored）
├── scripts/
│   └── hooks.py                   ← 主处理脚本
└── sounds/                        ← 各种音效文件
```

📄 **`.claude/hooks/config/hooks-config.json`**

```json
{
  "events": {
    "SessionStart": { "sound": "session-start.mp3" },
    "SessionEnd": { "sound": "session-end.mp3" },
    "PreToolUse": { "sound": "tool-use.mp3" }
  }
}
```

特殊处理：git 提交时触发 `pretooluse-git-committing` 音效。

### 9.4 如何禁用

```json
// .claude/settings.local.json
{
  "disableAllHooks": true
}
```

### 实操练习

1. 打开 `hooks-config.json`，看事件配置
2. 打开 `hooks.py`，了解结构
3. 打开 `sounds/` 目录，看看有哪些音效

---

## 第十章：CLI 启动参数

> 对应文档：**`doc/practice/08-CLI启动参数-cli-startup-flags.md`**
>
> 英文原版：`best-practice/claude-cli-startup-flags.md`

### 10.1 先看理论

Claude Code 提供了丰富的 CLI 启动参数，按功能分类：

| 类别 | 示例 |
|------|------|
| **会话管理** | `claude`（新会话）、`claude --resume`（恢复） |
| **模型/配置** | `--model opus-4-8`、`--temperature 0.5` |
| **权限/安全** | `--permissionMode auto`、`--allowedTools git,npm` |
| **单次模式** | `claude -p "Describe this code"` |
| **Agent 控制** | `--agent agent-name`、`--maxTurns 20` |
| **MCP/插件** | `--mcp-servers`、`--mcp-config` |
| **文件作用域** | `--allowedPaths src/`、`--disallowedPaths node_modules/` |

### 10.2 常用示例

```bash
claude                          # 启动新会话
claude -p "问一句就走"           # 单次模式
claude --model opus-4-8        # 指定模型
claude --resume                 # 恢复上次会话
claude --permissionMode auto   # 自动权限
claude --debug                  # 调试模式
claude --version                # 查看版本
```

---

## 第十一章：能力强化 Power Ups

> 对应文档：**`doc/practice/09-能力强化-power-ups.md`**
>
> 英文原版：`best-practice/claude-power-ups.md`

### 11.1 先看理论

Power Ups 是 Claude Code v2.1.90 引入的 **10 个交互式教程**，通过 `/powerup` 命令启动。

| # | 名称 | 作用 |
|---|------|------|
| 1 | **Talk to your codebase** | 学会与代码库对话 |
| 2 | **Steer with modes** | 用计划/自动模式引导 |
| 3 | **Undo anything** | 撤销任何操作 |
| 4 | **Run in background** | 后台运行任务 |
| 5 | **Teach rules** | 教 Claude 你的规则 |
| 6 | **Extend with tools** | 用 MCP 扩展工具 |
| 7 | **Automate workflow** | 自动化工作流 |
| 8 | **Multiply yourself** | 用子代理复制自己 |
| 9 | **Code from anywhere** | 在任何地方编码 |
| 10 | **Dial the model** | 调整模型 |

### 11.2 高级工具使用模式

📄 **`doc/reports/高级工具使用模式.md`**（中文）

三个 API 级特性，可节省 24-85% token：
- **程序化工具调用**：工具返回结构化数据而非 markdown
- **动态工具过滤**：根据上下文允许/禁止特定工具
- **工具搜索**：按需发现工具

📄 **`doc/reports/使用量与速率限制.md`**（中文）— Pro / Max 5x / Max 20x 三档订阅

---

## 第十二章：实用技巧合集

### 12.1 Boris Cherny 的核心技巧

📄 **`doc/tips/Boris的10个ClaudeCode技巧.md`**（中文）— 基础 10 条

**最重要的几条：**

1. 🔥 **并行 git worktree** — 开 3-5 个 worktree，每个跑一个 Claude 会话
   ```bash
   git worktree add ../project-feature-b feature-branch
   ```
2. **50% 时 `/compact`** — 不要等到上下文快满了才压缩
3. **先 Plan 再执行** — 复杂任务先让 Claude 出计划
4. **给 Subagent 设置 `maxTurns`** — 防止子代理跑飞
5. **不要把 Opus 用在小事上** — 简单任务切 Haiku

📄 **`doc/tips/Boris的12个ClaudeCode技巧.md`**（中文）— 进阶 12 条
（hooks 音效、plugins、LSP、MCP、skills、自定义 agent、effort 级别等）

📄 **`doc/tips/Boris的13个ClaudeCode技巧.md`**（中文）
📄 **`doc/tips/Boris的15个ClaudeCode技巧.md`**（中文）
📄 **`doc/tips/Boris的2个ClaudeCode技巧-0310.md`**（中文）
📄 **`doc/tips/Boris的2个ClaudeCode技巧-0325.md`**（中文）
📄 **`doc/tips/Boris的6个ClaudeCode技巧.md`**（中文）

### 12.2 Thariq 的上下文管理

📄 **`doc/tips/Thariq的ClaudeCode技巧-0416.md`**（中文）

- 1M token 上下文在 **300-400k 时开始腐烂**（遗忘早期内容）
- **主动 `/compact`** 比 Claude 自动压缩效果好得多
- 复杂任务**拆给 subagent**，主会话只做协调

📄 **`doc/tips/Thariq的ClaudeCode技巧-0317.md`**（中文）

### 12.3 更多深度报告

📄 **`doc/reports/Agent-SDK-vs-CLI-系统提示对比.md`**（中文）— SDK 和 CLI 的 prompt 差异
📄 **`doc/reports/LLM日常退化问题.md`**（中文）— LLM 日常退化
📄 **`doc/reports/Spinner动词与提示.md`**（中文）— 加载动画中的滚动动词彩蛋
📄 **`doc/reports/大型Monorepo的Skill策略.md`**（中文）— 大型仓库的 Skill 策略

📄 **`doc/翻译进度与文档缺口分析.md`** — 中英文文档的翻译进度与缺口

---

## 第十三章：学习路径

> 对应文档：**`doc/practice/10-学习路径.md`**（中文）
>
> 英文原版：README.md（尾部）

### 🟢 第一阶段：熟悉项目（第 1 天）

```
目的：了解仓库结构，知道什么文件在哪里
```

| 步骤 | 做什么 | 打开什么文件 |
|------|--------|-------------|
| 1 | 看项目整体结构 | 浏览根目录和 `.claude/` |
| 2 | 看 README 总览 | `README.md` |
| 3 | 看概念总览（中文） | `doc/practice/01-CONCEPTS-概念总览.md` |
| 4 | 看 Skills 目录 | `ls .claude/skills/` |
| 5 | 看 Commands 目录 | `ls .claude/commands/` |
| 6 | 看 Agents 目录 | `ls .claude/agents/` |
| 7 | 看设置文件 | `.claude/settings.json` |
| 8 | 看钩子配置 | `.claude/hooks/config/hooks-config.json` |
| 9 | 看 skills 全览（中文） | `doc/skills-inventory.md` |

### 🟡 第二阶段：按文档顺序逐个深入（第 2-5 天）

```
目的：按 doc/practice/ 的编号顺序，逐个理解每个概念
```

| 步骤  | 主题     | 读最佳实践（中文）                                      | 看实现说明（中文）                              | 看项目文件                                      |
| --- | ------ | ---------------------------------------------- | -------------------------------------- | ------------------------------------------ |
| 1   | 子代理    | `doc/practice/02-子代理-subagents.md`             | `doc/implementation/01-subagent-实现.md` | `.claude/agents/weather-agent.md`          |
| 2   | 命令     | `doc/practice/03-命令-commands.md`               | `doc/implementation/02-command-实现.md`  | `.claude/commands/weather-orchestrator.md` |
| 3   | 技能     | `doc/practice/04-技能-skills.md`                 | `doc/implementation/03-skill-实现.md`    | `.claude/skills/weather-fetcher/SKILL.md`  |
| 4   | 设置     | `doc/practice/05-设置-settings.md`               | —                                      | `.claude/settings.json`                    |
| 5   | MCP    | `doc/practice/06-MCP服务器-mcp.md`                | —                                      | `.claude/mcp.json`                         |
| 6   | 记忆     | `doc/practice/07-记忆系统-memory.md`               | —                                      | `~/.claude/projects/*/memory/`             |
| 7   | CLI 参数 | `doc/practice/08-CLI启动参数-cli-startup-flags.md` | —                                      | 命令行运行 `claude --help`                      |
| 8   | 能力强化   | `doc/practice/09-能力强化-power-ups.md`            | —                                      | 运行 `/powerup`                              |

### 🔴 第三阶段：实操体验（第 6-7 天）

```
目的：亲手操作，跑通完整流程
```

| 步骤  | 做什么                                   |                                       |
| --- | ------------------------------------- | ------------------------------------- |
| 1   | 跑 `/time-skill` 或 `/time-command` 看效果 |                                       |
| 2   | 跑 `/weather-orchestrator` 看完整编排流程     |                                       |
| 3   | 读编排工作流说明（中文）                          | `doc/orchestration-workflow/编排工作流.md` |
| 4   | 跑 `/grill-me` 体验追问模式                  |                                       |
| 5   | 一口气读完三个 implementation 文档             | `doc/implementation/01`、`02`、`03`     |
| 6   | 读 Workflow 重构历程（中文）                   | `doc/reports/Weather工作流重构历程.md`       |

### 🟣 第四阶段：深入进阶（第 2 周+）

```
目的：理解高级模式，开始自己创造
```

| 步骤  | 做什么           | 参考文件（中文优先）                                 |
| --- | ------------- | ------------------------------------------ |
| 1   | 理解全局 vs 项目设置  | `doc/reports/全局vs项目设置对比.md`                |
| 2   | 理解子代理记忆       | `doc/reports/子代理记忆系统.md`                   |
| 3   | 研究 Hook 系统    | `.claude/hooks/config/hooks-config.json`   |
| 4   | 读高级工具模式       | `doc/reports/高级工具使用模式.md`                  |
| 5   | 读 Harness 重要性 | `doc/reports/为什么Harness如此重要.md`            |
| 6   | 读深度对比报告       | `doc/reports/Agent-vs-Command-vs-Skill.md` |
| 7   | 读浏览器 MCP 对比   | `doc/reports/浏览器自动化MCP对比.md`               |

### ⭐ 第五阶段：融会贯通（第 3 周+）

```
目的：按照自己的需求定制 Claude Code
```

| 项目              | 参考什么                                                               |
| --------------- | ------------------------------------------------------------------ |
| 编写自己的 Skill     | 参考 `doc/practice/04-技能-skills.md` + `tdd`/`code-review` 的 SKILL.md |
| 编写自己的 Command   | 参考 `doc/practice/03-命令-commands.md` + `weather-orchestrator.md`    |
| 编写自己的 Agent     | 参考 `doc/practice/02-子代理-subagents.md` + `weather-agent.md`         |
| 搭建 MCP 服务器      | 参考 `doc/practice/06-MCP服务器-mcp.md` + `.claude/mcp.json`            |
| 创建 Workflow 自动化 | 参考 `doc/orchestration-workflow/编排工作流.md`                           |
| 管理社区 Skills     | 用 `npx skills add <repo>` 安装社区技能                                   |
| 文档规范            | 参考 `doc/rules/Markdown文档规范.md`                                     |

---

## 附录

### 文件位置速查

| 主题 | 中文最佳实践 | 中文实现说明 | 对应英文原版 |
|------|-------------|-------------|-------------|
| 概念总览 | `doc/practice/01-CONCEPTS-概念总览.md` | — | README.md |
| 子代理 | `doc/practice/02-子代理-subagents.md` | `doc/implementation/01-subagent-实现.md` | `best-practice/claude-subagents.md` |
| 命令 | `doc/practice/03-命令-commands.md` | `doc/implementation/02-command-实现.md` | `best-practice/claude-commands.md` |
| 技能 | `doc/practice/04-技能-skills.md` | `doc/implementation/03-skill-实现.md` | `best-practice/claude-skills.md` |
| 设置 | `doc/practice/05-设置-settings.md` | — | `best-practice/claude-settings.md` |
| MCP | `doc/practice/06-MCP服务器-mcp.md` | — | `best-practice/claude-mcp.md` |
| 记忆 | `doc/practice/07-记忆系统-memory.md` | — | `best-practice/claude-memory.md` |
| CLI 参数 | `doc/practice/08-CLI启动参数-cli-startup-flags.md` | — | `best-practice/claude-cli-startup-flags.md` |
| 能力强化 | `doc/practice/09-能力强化-power-ups.md` | — | `best-practice/claude-power-ups.md` |
| 学习路径 | `doc/practice/10-学习路径.md` | — | — |
| 编排工作流 | — | `doc/orchestration-workflow/编排工作流.md` | `orchestration-workflow/orchestration-workflow.md` |
| Skills 全览 | — | `doc/skills-inventory.md` | — |

### 常用命令速查

| 你想 | 输入 |
|-----|------|
| 审查代码 | `/code-review` |
| 诊断 bug | `/diagnosing-bugs` |
| 追问计划 | `/grill-me` |
| 写测试 | `/tdd` |
| 简化代码 | `/simplify` |
| 检查健康 | `/doctor` |
| 查看用量 | `/usage` |
| 切换模型 | `/model` |
| 压缩上下文 | `/compact` |
| 初始化项目 | `/init` |
| 获取天气（演示） | `/weather-orchestrator` |

---

> 📍 **项目根目录**: `D:\AI-Tech\claude-code-best-practice`
>
> 📍 **中文文档根目录**: `doc/`
>
> 📍 **文档编号规则**: `doc/practice/` 和 `doc/implementation/` 按 01-10 编号，章节顺序与之对齐
>
> 本指南最后更新: 2026-07-06
