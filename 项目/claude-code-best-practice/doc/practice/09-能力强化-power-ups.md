# 能力强化（Power-ups）最佳实践

> 中文翻译整理自 `best-practice/claude-power-ups.md`
> 学习顺序：第 9 篇

---

## 什么是 Power-ups？

Power-ups 是 Claude Code 内置的**交互式功能教学**。每个 Power-up 通过生动的动画演示教会你一个 Claude Code 的特性。

**启动方式：** 在 Claude Code 中输入 `/powerup`

> ⚠️ **v2.1.218+ 已移除**：`/powerup` 命令和 `/remote-env` 已从 Claude Code 中移除。此文档保留供参考，但 `/powerup` 已不可用。

---

## 10 个 Power-up 课程

| # | 课程名 | 主题 | 关联文档 |
|---|--------|------|---------|
| 1 | **与代码库对话** | `@` 文件引用、行引用 | — |
| 2 | **用模式掌控** | Shift+Tab 切换 Plan/Auto 模式 | [`05-设置-settings.md` → 权限模式](./05-设置-settings.md#21-权限模式) |
| 3 | **撤销一切** | `/rewind`、双击 Esc | — |
| 4 | **后台运行** | 任务管理、`/tasks` | [`02-子代理-subagents.md`](./02-子代理-subagents.md) |
| 5 | **教 Claude 你的规则** | `CLAUDE.md`、`/memory` | [`07-记忆系统-memory.md`](./07-记忆系统-memory.md) |
| 6 | **用工具扩展** | MCP、`/mcp` | [`06-MCP服务器-mcp.md`](./06-MCP服务器-mcp.md) |
| 7 | **自动化工作流** | Skills、Hooks | [`04-技能-skills.md`](./04-技能-skills.md) |
| 8 | **复制你自己** | Subagents、`/agents` | [`02-子代理-subagents.md`](./02-子代理-subagents.md) |
| 9 | **随处编码** | `/remote-control`、`/teleport` | [`08-CLI启动参数.md`](./08-CLI启动参数-cli-startup-flags.md) |
| 10 | **调节模型** | `/model`、`/effort` | [`05-设置-settings.md` → 模型配置](./05-设置-settings.md#五模型配置) |

---

## 学习建议

1. **新用户**：从头到尾过一遍 10 个课程
2. **老用户**：挑不熟悉的课程重点学习
3. **团队培训**：让新成员从 Power-ups 开始

---

> 下一篇：[10-学习路径.md](./10-学习路径.md)
