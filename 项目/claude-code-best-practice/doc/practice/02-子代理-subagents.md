# 子代理（Subagents）最佳实践

> 中文翻译整理自 `best-practice/claude-subagents.md`
> 学习顺序：第 2 篇
> 官方文档：[Create custom subagents](https://code.claude.com/docs/en/sub-agents)

---

## 什么是 Subagent？

Subagent 是 Claude Code 中的**独立迷你助手**。它拥有自己的上下文窗口，可以在后台并行工作，完成后将结果汇报给你。

**想象一下：** 你正在写代码，同时需要一个助手去查文档、另一个去跑测试、第三个去审查代码。Subagent 就是做这个的——而且它们互不干扰。

---

## 前置元数据字段（16 个）

每个 Subagent 定义文件（`.claude/agents/<名称>.md`）使用 YAML frontmatter 配置：

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 唯一标识符，使用小写字母和连字符 |
| `description` | string | 是 | 何时调用此 agent。设为 `"PROACTIVELY"` 可让 Claude 自动调用 |
| `tools` | string/list | 否 | 允许使用的工具列表（逗号分隔）。不设置则继承所有工具 |
| `disallowedTools` | string/list | 否 | 禁止使用的工具 |
| `model` | string | 否 | 使用的模型：`sonnet`、`opus`、`haiku` 或完整模型 ID。默认 `inherit` |
| `permissionMode` | string | 否 | 权限模式：`default`、`acceptEdits`、`auto`、`dontAsk`、`bypassPermissions`、`plan` |
| `maxTurns` | integer | 否 | 最大对话轮数 |
| `skills` | list | 否 | 预加载的 Skill 列表（启动时注入完整内容） |
| `mcpServers` | list | 否 | 此 agent 可用的 MCP 服务器 |
| `hooks` | object | 否 | 生命周期钩子 |
| `memory` | string | 否 | 持久记忆范围：`user`、`project`、`local` |
| `background` | boolean | 否 | `true` 则在后台运行（默认 `false`） |
| `effort` | string | 否 | 推理力度：`low`、`medium`、`high`、`xhigh`、`max` |
| `isolation` | string | 否 | `"worktree"` 在临时 git worktree 中运行 |
| `initialPrompt` | string | 否 | 当此 agent 作为主会话 agent 运行时，自动提交的首条提示 |
| `color` | string | 否 | 显示颜色：`red`、`blue`、`green`、`yellow`、`purple`、`orange`、`pink`、`cyan` |

---

## 官方内置 Agent（5 个）

| # | Agent | 模型 | 工具 | 用途 |
|---|-------|------|------|------|
| 1 | `general-purpose` | inherit | 全部 | 通用任务——默认类型，适合研究和代码搜索 |
| 2 | `Explore` | haiku | 只读 | 快速代码库搜索和探索 |
| 3 | `Plan` | inherit | 只读 | 计划模式下的预研——探索代码库并设计方案 |
| 4 | `statusline-setup` | sonnet | Read, Edit | 配置 Claude Code 状态栏 |
| 5 | `claude-code-guide` | haiku | Glob, Grep, Read, WebFetch, WebSearch | 回答关于 Claude Code 功能的问题 |

---

## 最佳实践

### 何时使用 Subagent

- **并行任务** — 需要同时做多件事时
- **上下文隔离** — 某任务会占用大量上下文时
- **后台运行** — 不需要实时监控的任务
- **专业分工** — 为特定功能创建专用 agent（而非通用 agent）

### 配置建议

1. **description 要精准** — 让 Claude 知道"什么时候该用这个 agent"
2. **合理限制 tools** — 不是所有 agent 都需要写文件权限
3. **善用 skills 预加载** — 给 agent 注入领域知识
4. **设置 maxTurns** — 防止 agent 无限循环
5. **使用 isolation: worktree** — 需要修改文件但又不想污染主工作区时

### 一个典型的 Subagent 定义

```yaml
---
name: code-reviewer
description: PROACTIVELY review code changes for bugs and security issues
tools: Read, Grep, Glob, Bash
model: sonnet
maxTurns: 10
skills:
  - security-review
---
帮我审查代码变更，重点关注：
1. 逻辑错误和边界情况
2. 安全漏洞（注入、认证、数据暴露）
3. 性能问题
```

---

## 关键区别：Subagent vs 其他概念

| 对比 | Subagent | Command | Skill |
|------|----------|---------|-------|
| **上下文** | 独立 | 共享 | 共享或独立（fork） |
| **运行方式** | 后台/前台 | 当前会话 | 当前会话 |
| **用途** | 并行任务 | 快捷操作 | 知识注入 |
| **配置位置** | `.claude/agents/` | `.claude/commands/` | `.claude/skills/` |

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Commands（命令） | [`03-命令-commands.md`](./03-命令-commands.md) | Command 定义和 86 个内置命令参考 |
| Skills（技能） | [`04-技能-skills.md`](./04-技能-skills.md) | Skill 定义和预加载到 Agent 的方法 |
| MCP 服务器 | [`06-MCP服务器-mcp.md`](./06-MCP服务器-mcp.md) | 通过 `mcpServers` 为 Agent 配置外部工具 |
| 权限模式 | [`05-设置-settings.md`](./05-设置-settings.md#二权限系统permissions) | `permissionMode` 字段的可用模式详解 |
| 记忆系统 | [`07-记忆系统-memory.md`](./07-记忆系统-memory.md#第三层auto-memory自动记忆) | Agent 的 `memory` 字段对应的自动记忆范围 |
| 实现参考 | [`../implementation/01-subagent-实现.md`](../implementation/01-subagent-实现.md) | 本仓库的 Subagent 实际配置示例 |

---

> 下一篇：[03-命令-commands.md](./03-命令-commands.md)
