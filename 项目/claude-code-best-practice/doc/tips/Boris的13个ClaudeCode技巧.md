# 我如何使用Claude Code——来自Boris Cherny的13个技巧（中文翻译）

> 中文翻译自 [tips/claude-boris-13-tips-03-jan-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-13-tips-03-jan-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年1月3日分享的设置技巧总结。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 背景

Boris分享了他个人的Claude Code设置，并指出它"出奇地普通"——Claude Code开箱即用效果很好，所以他并没有做太多定制。没有一种正确的使用方式：团队有意将其构建为你可以随心所欲地使用、定制和改造。Claude Code团队中的每个人使用方式都大相径庭。

<a href="https://x.com/bcherny/status/2007179832300581177"><img src="assets/boris-26-1-3/0.png" alt="Boris Cherny 介绍推文" width="50%" /></a>

---

## 1/ 并行运行5个Claude

在终端中并行运行5个Claude。将你的标签页编号为1–5，并使用系统通知了解Claude何时需要输入。

参见：[终端设置文档](https://code.claude.com/docs/en/terminal)

<a href="https://x.com/bcherny/status/2007179833990885678"><img src="assets/boris-26-1-3/1.png" alt="并行运行5个Claude" width="50%" /></a>

---

## 2/ 使用 claude.ai/code 实现更多并行

在 claude.ai/code 上并行运行5–10个Claude，与你本地的Claude同时工作。使用 `claude.ai/code` 将会话交接给Web会话，在Chrome中手动启动会话，并在两者之间来回穿梭。

<a href="https://x.com/bcherny/status/2007179836704600237"><img src="assets/boris-26-1-3/2.png" alt="claude.ai/code 并行" width="50%" /></a>

---

## 3/ 所有任务都使用带思考功能的Opus

所有任务都使用带思考功能的Opus 4.5。这是Boris用过的最好的编码模型——尽管它比Sonnet更大、更慢，但由于你需要引导它的次数更少，而且它的工具使用能力更强，最终它几乎总是比使用更小的模型更快。

<a href="https://x.com/bcherny/status/2007179838864666847"><img src="assets/boris-26-1-3/3.png" alt="带思考功能的Opus" width="50%" /></a>

---

## 4/ 与团队共享一个CLAUDE.md

为仓库共享一个 `CLAUDE.md`。将其检入git，让整个团队每周多次贡献。每当Claude做错了什么，将其添加到 `CLAUDE.md` 中，这样Claude下次就知道不要那么做。

<a href="https://x.com/bcherny/status/2007179840848597422"><img src="assets/boris-26-1-3/4.png" alt="共享CLAUDE.md" width="50%" /></a>

---

## 5/ 在PR上 @claude 以更新CLAUDE.md

在代码审查期间，在你的同事的PR上标记 `@claude`，以在PR中向 `CLAUDE.md` 添加内容。使用Claude Code GitHub操作（[install-@hub-action](https://github.com/apps/claude)）来实现——这是Boris版本的复合工程。

<a href="https://x.com/bcherny/status/2007179842928947333"><img src="assets/boris-26-1-3/5.png" alt="在PR上@claude" width="50%" /></a>

---

## 6/ 大多数会话以计划模式开始

大多数会话以计划模式开始（shift+tab两次）。如果目标是编写一个Pull Request，使用计划模式与Claude反复交流，直到你满意它的计划。然后切换到自动接受编辑模式，Claude通常可以一次性完成。一个好的计划非常重要。

<a href="https://x.com/bcherny/status/2007179845336527000"><img src="assets/boris-26-1-3/6.png" alt="计划模式" width="50%" /></a>

---

## 7/ 对内循环工作流使用斜杠命令

为你每天多次执行的每个"内循环"工作流使用斜杠命令。这省去了你重复提示的麻烦，并且使Claude也能使用这些工作流。命令被检入git，存放在 `.claude/commands/` 中。

示例：`/commit-push-pr` —— 提交、推送并打开一个PR。

<a href="https://x.com/bcherny/status/2007179847949500714"><img src="assets/boris-26-1-3/7.png" alt="斜杠命令" width="50%" /></a>

---

## 8/ 使用子代理自动化常用工作流

定期使用几个子代理：`code-simplifier` 在Claude完成工作后简化代码，`verify-app` 包含测试Claude Code端到端的详细指令，等等。将子代理视为自动化最常见工作流的方式——类似于斜杠命令。

子代理存放在 `.claude/agents/` 中。

<a href="https://x.com/bcherny/status/2007179850139000872"><img src="assets/boris-26-1-3/8.png" alt="子代理" width="50%" /></a>

---

## 9/ 使用PostToolUse钩子自动格式化代码

使用 `PostToolUse` 钩子来格式化Claude的代码。Claude通常开箱即用就能生成格式良好的代码，钩子处理最后10%，以避免后续CI中的格式错误。

```json
"PostToolUse": [
  {
    "matcher": "Write|Edit",
    "hooks": [
      {
        "type": "command",
        "command": "bun run format || true"
      }
    ]
  }
]
```

<a href="https://x.com/bcherny/status/2007179852047335529"><img src="assets/boris-26-1-3/9.png" alt="用于格式化的PostToolUse钩子" width="50%" /></a>

---

## 10/ 预允许权限，而不是--dangerously-skip-permissions

不要使用 `--dangerously-skip-permissions`。相反，使用 `/permissions` 预允许你环境中已知安全的常用bash命令，以避免不必要的权限提示。这些大多数被检入 `.claude/settings.json` 并与团队共享。

<a href="https://x.com/bcherny/status/2007179854077407667"><img src="assets/boris-26-1-3/10.png" alt="预允许权限" width="50%" /></a>

---

## 11/ 让Claude通过MCP使用你所有的工具

Claude Code使用你所有的工具。它经常搜索和发布到Slack（通过MCP服务器），运行BigQuery查询以回答分析问题（使用 `bq` CLI），从Sentry获取错误日志等等。Slack MCP配置被检入 `.mcp.json` 并与团队共享。

<a href="https://x.com/bcherny/status/2007179856266789204"><img src="assets/boris-26-1-3/11.png" alt="MCP工具" width="50%" /></a>

---

## 12/ 使用后台代理验证长时间运行的任务

对于非常长时间运行的任务，要么（a）提示Claude在完成后使用后台代理验证其工作，（b）使用代理Stop钩子更确定性地执行此操作，或（c）使用ralph-wiggum插件（最初由 @GeoffreyHuntley 构想）。

<a href="https://x.com/bcherny/status/2007179858435281082"><img src="assets/boris-26-1-3/12.png" alt="长时间运行任务的验证" width="50%" /></a>

---

## 13/ 给Claude一种验证其工作的方法

可能从Claude Code获得出色结果的最重要的事情——给Claude一种验证其工作的方法。如果Claude有这个反馈循环，最终结果的质量将提高2-3倍。

Claude测试Boris提交的每一个变更。

<a href="https://x.com/bcherny/status/2007179861115511237"><img src="assets/boris-26-1-3/13.png" alt="给Claude验证的方法" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — January 3, 2026](https://x.com/bcherny/status/2007179832300581177)
