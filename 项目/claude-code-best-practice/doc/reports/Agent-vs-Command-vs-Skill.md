# Agent vs Command vs Skill — 何时用什么

> Claude Code 三种扩展机制（子代理、命令、技能）的深度对比。

![显示 time-skill、time-command 和 time-agent 的斜杠菜单](assets/agent-command-skill-1.jpg)

---

## 一图概览

| | Agent | Command | Skill |
|---|---|---|---|
| **位置** | `.claude/agents/<名称>.md` | `.claude/commands/<名称>.md` | `.claude/skills/<名称>/SKILL.md` |
| **上下文** | 独立的子代理进程 | 内联（主会话） | 内联（主会话） |
| **用户可调用** | 无 `/` 菜单——由 Claude 或 Agent 工具调用 | 是——`/命令名` | 是——`/技能名`（除非 `user-invocable: false`） |
| **Claude 自动调用** | 是——通过 `description` 字段 | 否 | 是——通过 `description` 字段（除非 `disable-model-invocation: true`） |
| **接受参数** | 通过 `prompt` 参数 | `$ARGUMENTS`、`$0`、`$1` | `$ARGUMENTS`、`$0`、`$1` |
| **动态上下文注入** | 否 | 是——`` !`command` `` | 是——`` !`command` `` |
| **独立上下文窗口** | 是——隔离 | 否——共享主会话 | 否——共享主会话（除非 `context: fork`） |
| **模型覆盖** | `model:` frontmatter | `model:` frontmatter | `model:` frontmatter |
| **工具限制** | `tools:` / `disallowedTools:` | `allowed-tools:` | `allowed-tools:` |
| **钩子** | `hooks:` frontmatter | — | `hooks:` frontmatter |
| **记忆** | `memory:` frontmatter (user/project/local) | — | — |
| **可预加载技能** | 是——`skills:` frontmatter | — | — |
| **MCP 服务器** | `mcpServers:` frontmatter | — | — |

---

## 何时使用哪种

### 使用 Agent（子代理），当：

- 任务是**自主且多步骤的**——agent 需要探索、决策和行动，无需持续指导
- 你需要**上下文隔离**——工作不应污染主会话窗口
- Agent 需要跨会话的**持久记忆**（例如，一个学习代码模式的审查 agent）
- 你想通过技能**预加载领域知识**，而不弄乱主上下文
- 任务受益于**后台运行**或在 **git worktree** 中运行
- 你需要**工具限制**或**不同的权限模式**（如 `acceptEdits`、`plan`）

**示例**：`weather-agent`——使用预加载的 `weather-fetcher` skill 自主获取天气数据，在独立上下文中运行，工具受限。

### 使用 Command（命令），当：

- 你需要一个**用户发起的入口点**——用户显式触发的 workflow
- 工作流涉及**编排**其他 agent 或 skill
- 你想**保持上下文精简**——命令内容直到用户触发时才注入会话上下文

**示例**：`weather-orchestrator`——用户触发，询问 C/F 偏好，调用 agent，再调用 SVG skill。

### 使用 Skill（技能），当：

- 你想让 **Claude 根据用户意图自动调用**——skill 的描述被注入会话上下文用于语义匹配
- 任务是**可复用的流程**，可从多个地方调用（命令、agent 或 Claude 本身）
- 你需要 **agent 预加载**——在启动时将领域知识嵌入特定 agent

**示例**：`weather-svg-creator`——用户要求天气卡片时 Claude 自动调用；也可以从命令调用。

---

## Command → Agent → Skill 三层架构

本仓库演示了一个分层编排模式：

```
用户触发 /command
    ↓
Command 编排工作流
    ↓
Command 调用 Agent（独立上下文、自主运行）
    ↓
Agent 使用预加载的 Skill（领域知识）
    ↓
Command 调用 Skill（内联，用于输出生成）
```

**具体示例**——天气系统：

```
/weather-orchestrator (command — 入口点，询问 C/F)
    ↓
weather-agent (agent — 自主获取温度)
    ├── weather-fetcher (agent skill — 预加载的 API 指令)
    ↓
weather-svg-creator (skill — 内联创建 SVG)
```

---

## Frontmatter 对比

### Agent Frontmatter

```yaml
---
name: my-agent
description: Use this agent PROACTIVELY when...
tools: Read, Write, Edit, Bash
model: sonnet
maxTurns: 10
permissionMode: acceptEdits
memory: user
skills:
  - my-skill
---
```

### Command Frontmatter

```yaml
---
description: Do something useful
argument-hint: [issue-number]
allowed-tools: Read, Edit, Bash(gh *)
model: sonnet
---
```

### Skill Frontmatter

```yaml
---
name: my-skill
description: Do something when the user asks for...
argument-hint: [file-path]
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob
model: sonnet
context: fork
agent: general-purpose
---
```

---

## 关键区别

### 自动调用

| 机制 | Claude 能否自动调用？ | 如何阻止 |
|------|---------------------|---------|
| Agent | 是——通过 `description`（用 "PROACTIVELY" 鼓励自动调用） | 移除或弱化 description |
| Command | 否——始终通过 `/` 由用户发起 | 不适用 |
| Skill | 是——通过 `description` | 设置 `disable-model-invocation: true` |

### `/` 菜单可见性

| 机制 | 出现在 `/` 菜单？ | 如何隐藏 |
|------|------------------|---------|
| Agent | 否 | 不适用 |
| Command | 是——始终 | 无法隐藏 |
| Skill | 是——默认 | 设置 `user-invocable: false` |

### 上下文隔离

| 机制 | 在自己的上下文中运行？ | 如何配置 |
|------|---------------------|---------|
| Agent | 始终 | 内置行为 |
| Command | 从不 | 不适用 |
| Skill | 可选 | 设置 `context: fork` |

---

## 实战示例："现在几点？"

本仓库为同一个任务（显示 PKT 当前时间）定义了全部三种机制。当用户输入 **"现在几点？"** 但没有显式调用任何 `/` 命令时，会发生什么：

| 机制 | 会触发吗？ | 为什么 / 为什么不 |
|------|-----------|-----------------|
| `time-command` | 否 | Command **从不自动调用**。用户需要显式输入 `/time-command` 才能运行。Command 没有自动发现路径——它们严格由用户发起 |
| `time-agent` | **可能**会 | Agent 的 `description` 说 *"Use this agent to display the current time in Pakistan Standard Time"*。Claude 将这与用户意图匹配，可能通过 Agent 工具启动它。但 agent 运行在**独立的上下文窗口**中，对于这个简单的任务来说太重了 |
| `time-skill` | **最可能**会 | Skill 的 `description` 说 *"Display the current time in Pakistan Standard Time (PKT, UTC+5). Use when the user asks for the current time, Pakistan time, or PKT."* Claude 匹配并调用它。由于它**内联**运行，没有上下文开销，是最有效的匹配 |

### 解析顺序

当多个机制匹配同一意图时，Claude 优先选择**最轻量级**的选项：

```
1. Skill（内联，无上下文开销）    ← 首选
2. Agent（独立上下文，自主运行）   ← 如果 skill 不可用或任务复杂
3. Command（从不——需要显式 /）   ← 仅当用户输入 /time-command
```

### 如果 skill 上设置了 `disable-model-invocation: true` 会怎样？

那么 Claude **不能**自动调用该 skill。Agent 成为唯一可自动调用的选项，所以 Claude 会改为启动 `time-agent`——代价是为一个单行 bash 命令启动一个独立的上下文窗口。

### 如果 skill 和 agent 都禁用了自动调用会怎样？

那么**什么都不会自动触发**。Claude 会退回到自己的通用知识，很可能直接运行 `TZ='Asia/Karachi' date`——不涉及任何扩展机制。用户需要显式输入 `/time-command` 或 `/time-skill` 来使用它们。

![Claude 在用户问"现在几点？"时自动调用 time-skill](assets/agent-command-skill-2.png)

---

## 相关阅读

- [Claude Code Skills — 官方文档](https://code.claude.com/docs/en/skills)
- [Claude Code Sub-agents — 官方文档](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Slash Commands — 官方文档](https://code.claude.com/docs/en/slash-commands)
- [Skills 最佳实践](../best-practice/claude-skills.md)
- [Commands 最佳实践](../best-practice/claude-commands.md)
- [Sub-agents 最佳实践](../best-practice/claude-subagents.md)

### 关联文档

| 概念 | 文档 | 说明 |
|------|------|------|
| Harness 架构 | [`doc/reports/为什么Harness如此重要.md`](./为什么Harness如此重要.md) | 理解为什么这些功能不只是"提示词" |
| Subagent 实践 | [`doc/practice/02-子代理-subagents.md`](../practice/02-子代理-subagents.md) | Subagent 定义和 frontmatter 字段详解 |
| Command 实践 | [`doc/practice/03-命令-commands.md`](../practice/03-命令-commands.md) | 86 个内置命令参考和自定义命令示例 |
| Skill 实践 | [`doc/practice/04-技能-skills.md`](../practice/04-技能-skills.md) | Skill 设计原则和两种模式详解 |
| 编排工作流 | [`../orchestration-workflow/orchestration-workflow.md`](../orchestration-workflow/orchestration-workflow.md) | 三层架构流程图和具体实现 |
