# Skill 实现

> 中文翻译整理自 `implementation/claude-skills-implementation.md`

<table width="100%">
<tr>
<td><a href="../../">← 返回 Claude Code Best Practice</a></td>
<td align="right"><img src="../../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

<a href="#weather-svg-creator"><img src="../../!/tags/implemented-hd.svg" alt="Implemented"></a>

本仓库实现了两种 skill，作为 **Command → Agent → Skill** 架构模式的一部分，演示了两种不同的 skill 调用模式：**Agent Skill**（预加载）和 **Skill**（直接调用）。

---

## Weather SVG Creator（Skill—直接调用型）

**文件**: [`.claude/skills/weather-svg-creator/SKILL.md`](../../.claude/skills/weather-svg-creator/SKILL.md)

```yaml
---
name: weather-svg-creator
description: Creates an SVG weather card showing the current temperature for
  Dubai. Writes the SVG to orchestration-workflow/weather.svg and updates
  orchestration-workflow/output.md.
---

# Weather SVG Creator Skill

This skill creates a visual SVG weather card and writes the output files.

## Task
创建显示迪拜温度的可视化 SVG 天气卡片，并将输出写入文件。

## Instructions
接收来自调用上下文的温度值和单位。

### 1. 创建 SVG 天气卡片
生成简洁的 SVG 天气卡片...
### 2. 写入 SVG 文件
将 SVG 内容写入 `orchestration-workflow/weather.svg`
### 3. 写入输出摘要
写入 `orchestration-workflow/output.md`
```

这是 **skill** 模式——由命令通过 Skill 工具直接调用。它从对话上下文中获取温度数据，创建 SVG 天气卡片和输出摘要。

---

## Weather Fetcher（Agent Skill—预加载型）

**文件**: [`.claude/skills/weather-fetcher/SKILL.md`](../../.claude/skills/weather-fetcher/SKILL.md)

```yaml
---
name: weather-fetcher
description: Instructions for fetching current weather temperature data
  for Dubai, UAE from Open-Meteo API
user-invocable: false
---

# Weather Fetcher Skill

This skill provides instructions for fetching current weather data.

## Task
以请求的单位（摄氏度或华氏度）获取迪拜当前温度。

## Instructions
1. 获取天气数据：使用 WebFetch 工具获取当前天气数据
   - 摄氏度 URL: https://api.open-meteo.com/v1/forecast?latitude=25.2048&longitude=55.2708&current=temperature_2m&temperature_unit=celsius
   - 华氏度 URL: https://api.open-meteo.com/v1/forecast?latitude=25.2048&longitude=55.2708&current=temperature_2m&temperature_unit=fahrenheit
2. 提取温度：从 JSON 响应中提取 `current.temperature_2m`
3. 返回结果：清晰地返回温度值和单位
```

这是 **agent skill** 模式——通过 `skills:` frontmatter 字段预加载到 `weather-agent` 上下文中。它不直接调用，而是作为领域知识注入到 agent 的上下文。注意 `user-invocable: false` 使其不在 `/` 命令菜单中显示。

---

## 两种 Skill 模式对比

| 模式 | 调用方式 | 示例 | 关键区别 |
|------|----------|------|----------|
| **Skill** | `Skill(skill: "名称")` | `weather-svg-creator` | 通过 Skill 工具直接调用 |
| **Agent Skill** | 通过 `skills:` 字段预加载 | `weather-fetcher` | 启动时注入到 agent 上下文 |

---

## ![如何使用](../../!/tags/how-to-use.svg)

**Skill** — 通过斜杠命令直接调用：
```bash
$ claude
> /weather-svg-creator
```

---

## ![如何实现](../../!/tags/how-to-implement.svg)

让 Claude 帮你创建——它会自动在 `.claude/skills/my-skill/SKILL.md` 生成带 YAML frontmatter 的 skill 文件：

```markdown
# My Skill

该 skill 的指令说明。
```

---

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Skill 最佳实践 | [`../practice/04-技能-skills.md`](../practice/04-技能-skills.md) | Skill 定义、frontmatter 字段、设计原则、7 大最佳实践 |
| Subagent + Skill | [`../practice/02-子代理-subagents.md`](../practice/02-子代理-subagents.md) | Agent 如何通过 `skills:` 预加载 Agent Skill |
| Command + Skill | [`../practice/03-命令-commands.md`](../practice/03-命令-commands.md) | Command 如何通过 Skill 工具调用 Skill |
| 编排工作流 | [`../../orchestration-workflow/orchestration-workflow.md`](../../orchestration-workflow/orchestration-workflow.md) | 两种 Skill 模式在编排中的实际角色 |
| 本仓库 Skill 列表 | 项目结构，[`../practice/01-CONCEPTS-概念总览.md`](../practice/01-CONCEPTS-概念总览.md#项目结构总览) | 35+ 个 Matt Pocock Skills 安装目录 |
