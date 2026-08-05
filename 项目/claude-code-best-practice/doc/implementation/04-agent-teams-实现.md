# Agent Teams 实现

> 中文翻译整理自 `implementation/claude-agent-teams-implementation.md`

<table width="100%">
<tr>
<td><a href="../../">← 返回 Claude Code Best Practice</a></td>
<td align="right"><img src="../../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

<a href="#time-orchestration"><img src="../../!/tags/implemented-hd.svg" alt="Implemented"></a>

<p align="center">
  <img src="../../implementation/assets/impl-agent-teams.png" alt="Agent Teams 实际运行——tmux 分屏模式" width="100%">
</p>

Agent Teams（Agent 团队）可以生成**多个独立的 Claude Code 会话**，通过共享任务列表进行协调。与子代理不同（子代理是一个会话内的隔离上下文分支），每个团队成员拥有独立的完整上下文窗口，自动加载 CLAUDE.md、MCP 服务器和技能。

---

## ![如何使用](../../!/tags/how-to-use.svg)

时间编排工作流完全由 agent 团队构建。要运行成品：

```bash
cd agent-teams
claude
/time-orchestrator
```

这会调用 **Command → Agent → Skill** 流程：agent 获取迪拜的当前时间，skill 将 SVG 时间卡片渲染到 `agent-teams/output/dubai-time.svg`。

---

## ![如何实现](../../!/tags/how-to-implement.svg)

你可以使用 agent 团队创建一个天气编排工作流的复刻版本——本例中的时间编排工作流完全由 agent 团队构建。

### 1. 安装 [iTerm2](https://iterm2.com/) 和 tmux

```bash
brew install --cask iterm2
brew install tmux
```

> **注意：** Windows 用户可以使用 Windows Terminal + WSL 或类似的终端多路复用器。

### 2. 启动 iTerm2 → tmux → Claude

```bash
tmux new -s dev
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

### 3. 提供团队结构提示

<a id="time-orchestration"></a>

将以下提示粘贴到 Claude，让 agent 团队自动搭建完整的时间编排工作流：

主提示文件：**[agent-teams-prompt.md](../../agent-teams/agent-teams-prompt.md)**

### 团队协调流程

```
┌──────────────────────────────────────────────────────────────┐
│                         领导者 (你)                            │
│             "创建一个 agent 团队来搭建时间编排"                   │
└──────────────────────────┬───────────────────────────────────┘
                           │ 生成团队（全部并行）
              ┌────────────┼────────────┐
              ▼            ▼            ▼
   ┌────────────────┐ ┌──────────┐ ┌──────────────┐
   │ Command        │ │ Agent    │ │ Skill        │
   │ 架构师         │ │ 工程师   │ │ 设计师       │
   │                │ │          │ │              │
   │ agent-teams/   │ │ agent-   │ │ agent-teams/ │
   │ .claude/       │ │ teams/   │ │ .claude/     │
   │ commands/      │ │ .claude/ │ │ skills/      │
   │ time-          │ │ agents/  │ │ time-svg-    │
   │ orchestrator.md│ │ time-    │ │ creator/     │
   │                │ │ agent.md │ │              │
   └───────┬────────┘ └────┬─────┘ └──────┬───────┘
           │               │              │
           ▼               ▼              ▼
   ┌──────────────────────────────────────────────────┐
   │                  共享任务列表                      │
   │  ☐ 约定数据合约：{time, tz, formatted}            │
   │  ☐ Command 使用 Agent 工具（不是 bash）            │
   │  ☐ Agent 预加载 time-fetcher skill                │
   │  ☐ Skill 从上下文读取时间（不重新获取）             │
   │  ☐ 所有文件放在 agent-teams/.claude/ 下            │
   └──────────────────────────────────────────────────┘
                       │
                       ▼
          ┌──────────────────────────────┐
          │  cd agent-teams && claude    │
          │    /time-orchestrator        │
          │   Command → Agent → Skill    │
          └──────────────────────────────┘
```

---

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Subagent 最佳实践 | [`../practice/02-子代理-subagents.md`](../practice/02-子代理-subagents.md) | Agent Teams 是 Subagent 的进阶协作模式 |
| Command 最佳实践 | [`../practice/03-命令-commands.md`](../practice/03-命令-commands.md) | 团队产出的 `/time-orchestrator` 命令 |
| Skill 最佳实践 | [`../practice/04-技能-skills.md`](../practice/04-技能-skills.md) | 团队产出的 `time-fetcher`、`time-svg-creator` Skills |
| 设置参考 | [`../practice/05-设置-settings.md`](../practice/05-设置-settings.md#六显示--ux) | `teammateMode` 显示模式配置 |
| 学习路径 | [`../practice/10-学习路径.md`](../practice/10-学习路径.md) | Agent Teams 在第 3 周学习计划中 |
