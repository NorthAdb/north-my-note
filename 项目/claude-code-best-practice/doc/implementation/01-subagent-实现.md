# Subagent 实现

> 中文翻译整理自 `implementation/claude-subagents-implementation.md`

<table width="100%">
<tr>
<td><a href="../../">← 返回 Claude Code Best Practice</a></td>
<td align="right"><img src="../../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

<a href="#weather-agent"><img src="../../!/tags/implemented-hd.svg" alt="Implemented"></a>

天气 agent 在此仓库中实现，作为 **Command → Agent → Skill** 架构模式的示例，演示了两种不同的技能调用模式。

---

## Weather Agent（天气 Agent）

**文件**: [`.claude/agents/weather-agent.md`](../../.claude/agents/weather-agent.md)

```yaml
---
name: weather-agent
description: Use this agent PROACTIVELY when you need to fetch weather data for
  Dubai, UAE. This agent fetches real-time temperature from Open-Meteo
  using its preloaded weather-fetcher skill.
allowedTools:
  - "Read"
  - "Skill"
model: sonnet
color: green
maxTurns: 5
permissionMode: acceptEdits
memory: project
skills:
  - weather-fetcher
---

# Weather Agent

You are a specialized weather agent that fetches weather data for Dubai, UAE.

## Your Task

Execute the weather workflow by following the instructions from your preloaded skill:

1. **Fetch**: Follow the `weather-fetcher` skill instructions to fetch the current temperature
2. **Report**: Return the temperature value and unit to the caller
3. **Memory**: Update your agent memory with the reading details for historical tracking
```

该 agent 预加载了一个 skill（`weather-fetcher`），其中包含从 Open-Meteo API 获取数据的指令。它向调用方命令返回温度值和单位。

---

## ![如何使用](../../!/tags/how-to-use.svg)

```bash
$ claude
> 迪拜的天气怎么样？
```

---

## ![如何实现](../../!/tags/how-to-implement.svg)

你可以通过 `/agents` 命令创建 agent：

```bash
$ claude
> /agents
```

或者让 Claude 直接帮你创建——它会自动在 `.claude/agents/<name>.md` 生成带 YAML frontmatter 的 markdown 文件。

---

## 编排工作流

天气 agent 在 **Command → Agent → Skill** 编排模式中扮演 **Agent** 角色。它从 `/weather-orchestrator` 命令接收工作流，使用预加载的 skill（`weather-fetcher`）获取温度数据，命令随后调用独立 skill（`weather-svg-creator`）来创建可视化输出。

<p align="center">
  <img src="../../orchestration-workflow/orchestration-workflow.svg" alt="Command → Agent → Skill 三层架构" width="100%">
</p>

| 组件 | 角色 | 本仓库实现 |
|------|------|------------|
| **Command** | 入口、用户交互 | [`/weather-orchestrator`](../../.claude/commands/weather-orchestrator.md) |
| **Agent** | 用预加载 skill 获取数据 | [`weather-agent`](../../.claude/agents/weather-agent.md) + [`weather-fetcher`](../../.claude/skills/weather-fetcher/SKILL.md) |
| **Skill** | 独立创建最终输出 | [`weather-svg-creator`](../../.claude/skills/weather-svg-creator/SKILL.md) |

---

## 完整执行流程

当用户问"迪拜的天气怎么样？"时，背后发生了完整的**三层链路**：

```
用户: "迪拜的天气怎么样？"
  │
  │  ─── 第 1 层：主会话（Opus 4.8）───
  │  负责：意图识别 + agent 匹配
  │
  ├─ 扫描系统提示中的 agent 列表（11 个 agent，约 1.8k tokens）
  │  → 发现 weather-agent 的 description：
  │    "Use this agent PROACTIVELY when you need to fetch weather data for Dubai"
  │  → 关键词 "PROACTIVELY" + "Dubai" 匹配用户问题
  │    （attention 并行处理，非顺序遍历，约 1.8k tokens 一次推理完成）
  │
  ├─ 决策：启动 weather-agent 子代理
  │  Agent(subagent_type="weather-agent", prompt="获取迪拜当前的温度")
  │  → 不带 model 参数，继承 agent 文件 frontmatter 中的 model: sonnet
  │
  │  ─── 第 2 层：weather-agent 子代理（Sonnet）───
  │  负责：隔离执行天气获取逻辑
  │  独立上下文窗口，不污染主会话
  │
  ├─ 加载子代理上下文（独立会话，约 47k tokens）
  │  ├─ weather-agent.md 的指令：必须使用 Skill 工具获取天气
  │  ├─ 预加载 weather-fetcher skill（skills: ["weather-fetcher"]）
  │  │   skill 通过 description 注入到子代理上下文
  │  │   user-invocable: false，不在 / 菜单中显示
  │  ├─ hooks：PreToolUse / PostToolUse 触发声音提示
  │  └─ model: sonnet（独立于主会话的模型，由 frontmatter 指定）
  │
  ├─ Step 1: 调用 Skill 工具
  │  Skill(skill: "weather-fetcher")
  │  → 触发 weather-fetcher SKILL.md 中的指令
  │  → SKILL.md 要求使用 WebFetch 从 Open-Meteo API 获取数据
  │  → skill 有 allowed-tools: ["WebFetch(*)"] 放行 WebFetch
  │
  ├─ Step 2: WebFetch 请求天气数据
  │  → 向 Open-Meteo API 发送 HTTP 请求
  │  → 解析 JSON 响应，提取 current.temperature_2m
  │  → 返回温度值 + 单位
  │
  ├─ Step 3: 报告结果 + 更新 project memory
  │  → 将温度读数存入 agent 记忆（memory: project）
  │  → 返回给主会话
  │
  └─ Step 4: 子代理结束，结果返回主会话
     │
     │  ─── 回到第 1 层：主会话（Opus 4.8）───
     │
     ├─ 收到子代理返回的温度值
     └─ 最终回复："迪拜现在 36.7°C"

```

### 关键设计点

| 设计 | 说明 |
|------|------|
| **PROACTIVELY 自动触发** | agent 的 description 中含 `PROACTIVELY` 关键词，Claude 在匹配到相关问题时自动启动，无需用户显式 `/call` |
| **模型隔离** | 子代理使用 `model: sonnet`，独立于主会话的 Opus。两种模型同时运行，各自消费 token |
| **上下文隔离** | 子代理内部的所有工具调用（Skill → WebFetch → JSON 解析）都在独立上下文窗口中完成，不污染主会话 |
| **预加载 skill** | `skills: ["weather-fetcher"]` 在子代理启动时把 skill 内容注入上下文，而非延迟加载。skill 的 `user-invocable: false` 隐藏了斜杠命令 |
| **Fail-closed 设计** | `weather-agent.md` 第 69 行的 "Fail-closed guardrail"：如果 Skill 工具没有返回有效温度值，禁止自己用其他方式获取，直接报告失败 |
| **内存追踪** | `memory: project` 让每次温度读数都存入项目记忆，后续查询可对比历史数据 |

<p align="center">
  <img src="../../orchestration-workflow/orchestration-workflow.svg" alt="Command → Agent → Skill 三层架构" width="100%">
</p>

| 组件 | 角色 | 本仓库实现 |
|------|------|------------|
| **Command** | 入口、用户交互 | [`/weather-orchestrator`](../../.claude/commands/weather-orchestrator.md) |
| **Agent** | 用预加载 skill 获取数据 | [`weather-agent`](../../.claude/agents/weather-agent.md) + [`weather-fetcher`](../../.claude/skills/weather-fetcher/SKILL.md) |
| **Skill** | 独立创建最终输出 | [`weather-svg-creator`](../../.claude/skills/weather-svg-creator/SKILL.md) |

---

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Subagent 最佳实践 | [`../practice/02-子代理-subagents.md`](../practice/02-子代理-subagents.md) | Subagent 定义、frontmatter 字段、内置 Agent 参考 |
| Command 最佳实践 | [`../practice/03-命令-commands.md`](../practice/03-命令-commands.md) | 86 个内置命令参考表，Command vs Subagent 对比 |
| Skill 最佳实践 | [`../practice/04-技能-skills.md`](../practice/04-技能-skills.md) | Skill 定义、设计原则，Agent Skill 模式详解 |
| MCP 最佳实践 | [`../practice/06-MCP服务器-mcp.md`](../practice/06-MCP服务器-mcp.md) | 通过 `mcpServers` 为 Agent 配置外部工具 |
| 编排工作流 | [`../../orchestration-workflow/orchestration-workflow.md`](../../orchestration-workflow/orchestration-workflow.md) | Command → Agent → Skill 三层架构流程图 |
