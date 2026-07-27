# CLAUDE.md（中文翻译注释版）

> 中文翻译自 [CLAUDE.md]

# CLAUDE.md

本文件为在此仓库中使用代码时向 Claude Code（claude.ai/code）提供指导。

## 仓库概述

这是一个 Claude Code 配置的最佳实践仓库，展示了 skills、subagents、hooks 和 commands 的模式。它作为参考实现而非应用代码库。

## 关键组件

### 天气系统（示例工作流）

通过**命令 → Agent → Skill** 架构演示两种不同的 skill 模式：
- `/weather-orchestrator` 命令（`.claude/commands/weather-orchestrator.md`）：入口点 — 询问用户摄氏度/华氏度，调用 agent，然后调用 SVG skill
- `weather-agent` agent（`.claude/agents/weather-agent.md`）：使用其预加载的 `weather-fetcher` skill（agent skill 模式）获取温度
- `weather-fetcher` skill（`.claude/skills/weather-fetcher/SKILL.md`）：预加载到 agent 中 — 从 Open-Meteo 获取温度的指令
- `weather-svg-creator` skill（`.claude/skills/weather-svg-creator/SKILL.md`）：Skill — 创建 SVG 天气卡片，写入 `orchestration-workflow/weather.svg` 和 `orchestration-workflow/output.md`

两种 skill 模式：agent skills（通过 `skills:` 字段预加载）vs skills（通过 `Skill` 工具调用）。参见 `orchestration-workflow/orchestration-workflow.md` 获取完整流程图。

### Skill 定义结构

`.claude/skills/<name>/SKILL.md` 中的 skills 使用 YAML frontmatter：
- `name`：显示名称和 `/slash-command`（默认为目录名）
- `description`：何时调用（推荐用于自动发现）
- `argument-hint`：自动补全提示（例如 `[issue-number]`）
- `disable-model-invocation`：设置为 `true` 可防止自动调用
- `user-invocable`：设置为 `false` 可从 `/` 菜单隐藏（仅作为背景知识）
- `allowed-tools`：skill 激活时无需权限提示即可使用的工具
- `model`：skill 激活时使用的模型
- `context`：设置为 `fork` 以在隔离的子 agent 上下文中运行
- `agent`：`context: fork` 时的子 agent 类型（默认：`general-purpose`）
- `hooks`：作用域为该 skill 的生命周期钩子

### 演示文稿系统

参见 `.claude/rules/presentation.md` — 演示文稿工作按演示分配，委托给 `presentation-vibe-coding`（用于 `presentation/vibe-coding-to-agentic-engineering/`）或 `presentation-claude-gemini`（用于 `presentation/2026-04-25-gdg-kolachi-cli-claude-code-gemini/`）。

### 钩子系统

`.claude/hooks/` 中的跨平台声音通知系统：
- `scripts/hooks.py`：Claude Code 钩子事件的主处理器
- `config/hooks-config.json`：共享团队配置
- `config/hooks-config.local.json`：个人覆盖（git 忽略）
- `sounds/`：按钩子事件组织的音频文件（通过 ElevenLabs TTS 生成）

`.claude/settings.json` 中配置的钩子事件：PreToolUse, PostToolUse, UserPromptSubmit, Notification, Stop, SubagentStart, SubagentStop, PreCompact, SessionStart, SessionEnd, Setup, PermissionRequest, TeammateIdle, TaskCompleted, ConfigChange。

特殊处理：git 提交触发 `pretooluse-git-committing` 声音。

## 关键模式

### Subagent 编排

Subagents **不能**通过 bash 命令调用其他 subagents。使用 Agent 工具（从 v2.1.63 起由 Task 重命名；`Task(...)` 仍然可作为别名使用）：
```
Agent(subagent_type="agent-name", description="...", prompt="...", model="haiku")
```

在 subagent 定义中明确说明工具使用方式。避免使用可能被误解为 bash 命令的模糊术语，如"launch"。

### Subagent 定义结构

`.claude/agents/*.md` 中的 subagents 使用 YAML frontmatter：
- `name`：Subagent 标识符
- `description`：何时调用（使用"PROACTIVELY"进行自动调用）
- `tools`：工具的逗号分隔允许列表（如果省略则继承全部）。支持 `Agent(agent_type)` 语法
- `disallowedTools`：禁止使用的工具，从继承或指定的列表中移除
- `model`：模型别名：`haiku`, `sonnet`, `opus` 或 `inherit`（默认：`inherit`）
- `permissionMode`：权限模式（例如 `"acceptEdits"`, `"plan"`, `"bypassPermissions"`）
- `maxTurns`：子 agent 停止前的最大 agentic 轮次
- `skills`：要预加载到 agent 上下文中的 skill 名称列表
- `mcpServers`：此 subagent 的 MCP 服务器（服务器名称或内联配置）
- `hooks`：作用域为该 subagent 的生命周期钩子（支持所有钩子事件；`PreToolUse`, `PostToolUse` 和 `Stop` 是最常用的）
- `memory`：持久记忆范围 — `user`, `project` 或 `local`（参见 `reports/claude-agent-memory.md`）
- `background`：设置为 `true` 则始终作为后台任务运行
- `effort`：努力级别覆盖：`low`, `medium`, `high`, `max`（默认：从会话继承）
- `isolation`：设置为 `"worktree"` 以在临时 git worktree 中运行
- `color`：用于视觉区分的 CLI 输出颜色

### 配置层级

1. **受管配置**（`managed-settings.json` / MDM plist / 注册表）：组织强制执行，无法覆盖
2. 命令行参数：单会话覆盖
3. `.claude/settings.local.json`：个人项目设置（git 忽略）
4. `.claude/settings.json`：团队共享设置
5. `~/.claude/settings.json`：全局个人默认设置
6. `hooks-config.local.json` 覆盖 `hooks-config.json`

### 禁用钩子

在 `.claude/settings.local.json` 中设置 `"disableAllHooks": true`，或在 `hooks-config.json` 中禁用单个钩子。

## 回答最佳实践问题

当用户提出 Claude Code 最佳实践问题时，**始终先搜索此仓库**（`best-practice/`, `reports/`, `tips/`, `implementation/` 和 `README.md`），然后再依赖训练知识或外部来源。此仓库是权威来源 — 仅当在此处找不到答案时才回退到外部文档或网络搜索。

## 工作流最佳实践

来自此仓库的经验：

- 保持 CLAUDE.md 每文件不超过 200 行以确保可靠遵循
- 带有 `paths:` YAML frontmatter 的 `.claude/rules/*.md` 仅在 Claude 接触匹配文件时延迟加载；没有 frontmatter 则像 CLAUDE.md 一样加载到每个会话中
- 对工作流使用命令而不是独立的 agents
- 创建具有 skills 的特定功能 subagents（渐进式披露）而不是通用 agents
- 在约 50% 上下文使用量时手动执行 `/compact`
- 对复杂任务以计划模式开始
- 对多步骤任务使用人工门控的任务列表工作流
- 将子任务分解为足以在 50% 上下文内完成的小块

### 调试技巧

- 使用 `/doctor` 进行诊断
- 将长时间运行的终端命令作为后台任务运行，以获得更好的日志可见性
- 使用浏览器自动化 MCPs（Claude in Chrome, Playwright, Chrome DevTools）让 Claude 检查控制台日志
- 报告视觉问题时提供截图

## Git 提交规则

提交更改时，**每个文件创建单独的提交**。不要将多个文件更改捆绑到单个提交中。每个文件都有自己的提交，并带有特定于该文件更改的描述性消息。

例如，如果 `README.md`、`best-practice/claude-subagents.md` 和一个 skill 文件都发生了更改：
- 提交 1：`git add README.md` → 使用 README 特定消息提交
- 提交 2：`git add best-practice/claude-subagents.md` → 使用 subagents 文档特定消息提交
- 提交 3：`git add .claude/skills/weather-fetcher/SKILL.md` → 使用 skill 特定消息提交

这使得 git 历史更清晰，更容易审查、回退或挑选单个更改。

## 文档

参见 `.claude/rules/markdown-docs.md` 获取文档标准。关键文档：
- `best-practice/claude-subagents.md`：Subagent frontmatter、hooks 和仓库 agents
- `best-practice/claude-commands.md`：Slash 命令模式和内置命令参考
- `orchestration-workflow/orchestration-workflow.md`：天气系统流程图
