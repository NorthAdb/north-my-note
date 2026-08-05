# 使用 Claude Code 的 10 个技巧 — 来自 Claude Code 团队

> Boris Cherny（[@bcherny](https://x.com/bcherny)），Claude Code 的创建者，于 2026 年 2 月 1 日分享的团队技巧汇总。

---

## 背景

Boris 分享了直接来自 Claude Code 团队的使用技巧。团队使用 Claude 的方式与 Boris 个人的使用方式不同。记住：没有唯一正确的 Claude Code 使用方式——每个人的配置都不一样。你应该实验，找到适合你的方式！

---

## 1/ 并行做更多事

一次启动 3–5 个 git worktree，每个运行自己的 Claude 会话。这是最大的生产力解锁，也是团队的首推技巧。Boris 个人使用多个 git checkout，但 Claude Code 团队大多数更喜欢 worktree——这也是 `@amorisscode` 为 Claude Desktop 应用构建原生 worktree 支持的原因！

有些人还会给 worktree 命名并设置 shell 别名（`2a`、`2b`、`2c`），一键切换。另一些人有一个专门的"分析"worktree，只用于读取日志和运行 BigQuery。

参考：[Worktrees 文档](https://code.claude.com/docs/en/common...)

---

## 2/ 每个复杂任务都从 Plan 模式开始

把你的精力投入计划中，这样 Claude 可以一次完成实现。

有人让一个 Claude 写计划，然后启动第二个 Claude 作为资深工程师来审查计划。

另一个人说，一旦事情偏离轨道，就切回 plan 模式重新规划。不要硬推。他们也明确告诉 Claude 进入 plan 模式进行验证步骤，而不仅仅是构建。

---

## 3/ 投入精力到你的 CLAUDE.md

每次修正后，以这句话结尾："更新你的 CLAUDE.md，这样你就不会再犯同样的错误。"Claude 在为自己编写规则方面出奇地擅长。

随着时间的推移，无情地编辑你的 `CLAUDE.md`。持续迭代直到 Claude 的错误率明显下降。

一位工程师让 Claude 为每个任务/项目维护一个笔记目录，每个 PR 后更新。然后让 `CLAUDE.md` 指向它。

---

## 4/ 创建你自己的 Skills 并提交到 Git

跨每个项目复用。团队的建议：

- 如果某件事一天做超过一次，就把它变成 skill 或 command
- 建一个 `/techdebt` 斜杠命令，每次会话结束时运行，查找并删除重复代码
- 设置一个斜杠命令，将 7 天的 Slack、GDrive、Asana 和 GitHub 同步到一个上下文转储中
- 构建类似分析工程师的 agent，用于编写 dbt 模型、审查代码和在开发环境中测试更改

参考：[用 Skills 扩展 Claude — Claude Code 文档](https://code.claude.com/docs/en/skills)

---

## 5/ Claude 能自己修复大多数 Bug

团队的做法：

启用 Slack MCP，然后将 Slack 上的 bug 讨论粘贴给 Claude，只说"修复"。零上下文切换。

或者，只说"去修复失败的 CI 测试"。不要微观管理如何做。

把 Claude 指向 docker 日志来排查分布式系统——它在这方面出奇地有能力。

---

## 6/ 提升你的提示词技巧

a. **挑战 Claude。** 说"质疑我的这些更改，在你通过我的测试之前不要创建 PR。"让 Claude 做你的审查者。或者，说"向我证明这能工作"，让 Claude 比较 main 分支和你的功能分支之间的行为差异。

b. **在平庸的修复之后，** 说："根据你现在知道的一切，废弃这个，实现优雅的解决方案。"

c. **写详细的规格说明**，在交接工作前减少歧义。你越具体，输出越好。

---

## 7/ 终端和环境设置

团队热爱 Ghostty！多人喜欢它的同步渲染、24 位色彩和正确的 unicode 支持。

为了更方便地管理多个 Claude，使用 `/statusline` 自定义状态栏，始终显示上下文使用情况和当前 git 分支。许多人还会给终端标签着色和命名，有时使用 tmux——每个任务/worktree 一个标签。

使用语音输入。你说话比打字快 3 倍，你的提示词也会因此变得更详细。（在 macOS 上按 fn 两次）

参考：[终端设置文档](https://code.claude.com/docs/en/termin...)

---

## 8/ 使用 Subagent

a. 在任何你希望 Claude 投入更多计算资源的问题后附加"使用 subagent"。

b. 将单个任务卸载给 subagent，保持主 agent 的上下文窗口干净和专注。

c. 通过钩子将权限请求路由到 Opus 4.5——让它扫描攻击并自动批准安全的请求。参考：[Hooks 文档](https://code.claude.com/docs/en/hooks#...)

---

## 9/ 用 Claude 做数据和数据分析

让 Claude Code 使用 "bq" CLI 即时拉取和分析指标。团队在代码库中检入了一个 BigQuery skill，每个人都在 Claude Code 中直接运行分析查询。Boris 个人已经 6 个多月没写过一行 SQL 了。

这适用于任何有 CLI、MCP 或 API 的数据库。

---

## 10/ 用 Claude 学习

团队的一些使用 Claude Code 学习的技巧：

a. 在 `/config` 中启用 "Explanatory" 或 "Learning" 输出风格，让 Claude 解释其更改背后的"为什么"。

b. 让 Claude 生成可视化的 HTML 演示文稿，解释你不熟悉的代码。它做的幻灯片出奇地好！

c. 让 Claude 绘制新的协议和代码库的 ASCII 示意图，帮助你理解它们。

d. 构建一个间隔重复学习 skill：你解释你的理解，Claude 提出后续问题来填补空白，存储结果。

---

## 来源

- [Boris Cherny (@bcherny) 在 X 上 — 2026 年 2 月 1 日](https://x.com/bcherny/status/2017742741636321619)

---

### 关联文档

| 概念 | 文档 | 说明 |
|------|------|------|
| Subagent | [`doc/practice/02-子代理-subagents.md`](../practice/02-子代理-subagents.md) | 技巧 8 背后的概念详解 |
| CLAUDE.md | [`doc/practice/07-记忆系统-memory.md`](../practice/07-记忆系统-memory.md) | 技巧 3 的三层记忆体系 |
| Skill/Command | [`doc/practice/04-技能-skills.md`](../practice/04-技能-skills.md) | 技巧 4 的创建方法 |
| Plan 模式 | [`doc/practice/05-设置-settings.md`](../practice/05-设置-settings.md#21-权限模式) | 技巧 2 的权限模式配置 |
| CLI 参数 | [`doc/practice/08-CLI启动参数.md`](../practice/08-CLI启动参数-cli-startup-flags.md) | 技巧 1 的 `--worktree` 参数 |
| 更多技巧 | [`doc/tips/`](./) | Boris 和 Thariq 的更多技巧 |
