# Goal 模式实现

> 中文翻译整理自 `implementation/claude-goal-implementation.md`

<table width="100%">
<tr>
<td><a href="../../">← 返回 Claude Code Best Practice</a></td>
<td align="right"><img src="../../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

<a href="#goal-tips-from-the-community"><img src="../../!/tags/implemented-hd.svg" alt="Implemented"></a>

`/goal` 让你的 agent 跨多次对话持续工作，直到条件满足——Claude Code、Codex 和 Hermes Agent 都支持此功能。社区正在形成几个高杠杆的提示技巧，与 `/goal` 配合效果极佳。

---

## 来自社区的 Goal 技巧

### 1. 让 Agent 自己提出目标

<p align="center">
  <img src="../../implementation/assets/impl-goal-claude.png" alt="Alex Finn 推文——/goal 是 2026 年最被低估的 AI 功能" width="50%">
</p>

> `/goal` 已经正式发布了。
>
> 2026 年最被低估的 AI 功能。
>
> 现在 Claude Code、Codex 和 Hermes Agent 都支持它。
>
> 它让你的 agent 可以完成长时间运行的任务，有时持续数天。
>
> 每个人都应该立刻运行这个提示：
>
> > "根据你对我的了解、我的目标和抱负，以及我们已经共同构建的成果，你认为我们现在可以运行哪 3 个 `/goal` 来产生最好的结果？"
>
> 选择一个，然后让它帮你构建一个提示。
>
> 你会得到几个超强的 goal 提示选项，让你选择的 agent 完成长期任务，带给你令人惊叹的结果。
>
> 今晚抽出 15 分钟做这件事，你会感谢我的。

**来源：** [Alex Finn (@AlexFinn) 在 X 上](https://x.com/AlexFinn/status/2053976411296452887)

---

### 2. 让 Agent 为你起草 /goal 提示

<p align="center">
  <img src="../../implementation/assets/impl-goal-codex.png" alt="Meta Alchemist 推文——/goal 技巧" width="50%">
</p>

> 想知道 Codex 上最好的 `/goal` 技巧吗？
>
> 只需告诉你的 Codex：
>
> > "阅读本次会话和仓库，深入分析我们想要达成的确切意图和目标，然后为我写出这个 `/goal` 提示。
> >
> > 确保深入研究历史和文档，做到 100% 清晰。"
>
> 你也可以补充：
>
> > "如果你对某些部分不确定，或者想问我几个问题来进一步明确目标，尽管问。"
>
> 然后复制粘贴 Codex 给你的内容，把开头改成 `/goal`。
>
> 它就会精确地完成你在这个会话或仓库中想做的事情，持续工作直到完成。

**来源：** [Meta Alchemist (@meta_alchemist) 在 X 上](https://x.com/meta_alchemist/status/2054214497443995694)

---

## ![如何使用](../../!/tags/how-to-use.svg)

```bash
$ claude
> /goal <完成条件>
> /goal clear
```

`/goal <完成条件>` 让 Claude 跨多次对话持续工作，直到 Haiku 评估的条件成立。它与 `/loop`（基于时间）和 auto 模式（基于每次工具调用）互补。需要 Claude Code v2.1.139+。

详细行为请参考[官方文档](https://code.claude.com/docs/en/goal)。

---

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| 命令参考 | [`../practice/03-命令-commands.md`](../practice/03-命令-commands.md) | `/goal` 命令在会话类命令中的说明 |
| 设置参考 | [`../practice/05-设置-settings.md`](../practice/05-设置-settings.md#一通用设置general-settings) | `effortLevel`、`alwaysThinkingEnabled` 等与 Goal 配合的设置 |
| 学习路径 | [`../practice/10-学习路径.md`](../practice/10-学习路径.md) | `/goal` 在第 3 周进阶技巧中 |
| CLI 参数 | [`../practice/08-CLI启动参数-cli-startup-flags.md`](../practice/08-CLI启动参数-cli-startup-flags.md) | `--max-turns` 等与 Goal 配合的启动参数 |
