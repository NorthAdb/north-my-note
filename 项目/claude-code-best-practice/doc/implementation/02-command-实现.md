# Command 实现

> 中文翻译整理自 `implementation/claude-commands-implementation.md`


|                                          |     |
| ---------------------------------------- | --- |
| [← 返回 Claude Code Best Practice](../../) |     |


---



天气编排命令在此仓库中实现，作为 **Command → Agent → Skill** 架构模式的入口，演示了命令如何编排多步骤工作流。

---

## Weather Orchestrator（天气编排命令）

**文件**: `[.claude/commands/weather-orchestrator.md](../../.claude/commands/weather-orchestrator.md)`

```yaml
---
description: Fetch weather data for Dubai and create an SVG weather card
model: haiku
---

# Weather Orchestrator Command

Fetch the current temperature for Dubai, UAE and create a visual SVG weather card.

## Workflow

### Step 1: Ask User Preference
使用 AskUserQuestion 工具询问用户想要摄氏度还是华氏度。

### Step 2: Fetch Weather Data
使用 Agent 工具调用 weather agent：
- subagent_type: weather-agent
- prompt: 获取迪拜当前的温度，单位：[用户选择的单位]

### Step 3: Create SVG Weather Card
使用 Skill 工具调用 weather-svg-creator skill：
- skill: weather-svg-creator
```

该命令编排了整个工作流：它询问用户偏好的温度单位，通过 Agent 工具调用 `weather-agent` 获取数据，再通过 Skill 工具调用 `weather-svg-creator` skill 创建可视化卡片。

---



## 如何使用

```bash
$ claude
> /weather-orchestrator
```

---



## 如何实现

让 Claude 帮你创建——它会自动在 `.claude/commands/<name>.md` 生成带 YAML frontmatter 的命令文件。

---



天气编排命令在 Command → Agent → Skill 编排模式中扮演 **Command** 角色。它作为入口——处理用户交互（温度单位选择），将数据获取委托给 `weather-agent`，再调用 `weather-svg-creator` skill 生成可视化输出。




| 组件          | 角色              | 本仓库实现                                                                                                                         |
| ----------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Command** | 入口、用户交互         | `[/weather-orchestrator](../../.claude/commands/weather-orchestrator.md)`                                                     |
| **Agent**   | 用预加载 skill 获取数据 | `[weather-agent](../../.claude/agents/weather-agent.md)` + `[weather-fetcher](../../.claude/skills/weather-fetcher/SKILL.md)` |
| **Skill**   | 独立创建最终输出        | `[weather-svg-creator](../../.claude/skills/weather-svg-creator/SKILL.md)`                                                    |


---



### 关联文档


| 相关概念          | 文档                                                                                                                 | 说明                                               |
| ------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| Command 最佳实践  | `[../practice/03-命令-commands.md](../practice/03-命令-commands.md)`                                                   | 86 个内置命令参考表、frontmatter 字段、自定义命令示例               |
| Subagent 最佳实践 | `[../practice/02-子代理-subagents.md](../practice/02-子代理-subagents.md)`                                               | Subagent 定义，Command 如何通过 Agent 工具调用子代理           |
| Skill 最佳实践    | `[../practice/04-技能-skills.md](../practice/04-技能-skills.md)`                                                       | Skill 定义，Command 如何通过 Skill 工具调用 skill           |
| 编排工作流         | `[../../orchestration-workflow/orchestration-workflow.md](../../orchestration-workflow/orchestration-workflow.md)` | Command → Agent → Skill 三层架构流程图                  |
| 设置参考          | `[../practice/05-设置-settings.md](../practice/05-设置-settings.md)`                                                   | Command 的 `allowed-tools`、`permissionMode` 等配置详解 |


