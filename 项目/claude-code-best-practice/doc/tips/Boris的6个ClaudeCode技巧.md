# 从Opus 4.7中获得更多价值的6个技巧——来自Boris Cherny（中文翻译）

> 中文翻译自 [tips/claude-boris-6-tips-16-apr-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-6-tips-16-apr-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年4月16日分享的一系列技巧——在亲身使用了Opus 4.7几周之后。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 背景

在亲身使用Opus 4.7几周后，Boris感觉"生产力爆棚"，并分享了从新模型中获得更多价值的六种方式——从权限自动化到精力调节再到验证模式。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/0.png" alt="Boris Cherny 介绍推文——亲身使用Opus 4.7" width="50%" /></a>

---

## 1/ 自动模式——不再有权限提示

Opus 4.7喜欢执行复杂、长时间运行的任务：深度研究、重构代码、构建复杂功能、迭代直到达到性能基准。过去，你要么需要在模型执行这类长任务时"保姆式"照看，要么使用 `--dangerously-skip-permissions`。

Anthropic最近推出了**自动模式**作为更安全的替代方案。在此模式下，权限提示被路由到基于模型的分类器，由它决定命令是否安全运行：

- 如果安全，自动批准
- 如果有风险，暂停并询问

这意味着模型运行时不再需要照看。更重要的是，这意味着你可以并行运行更多Claude——如果安全，你可以将注意力切换到下一个Claude。

自动模式现在可供Max、Teams和企业用户用于Opus 4.7。在CLI中按 **Shift+Tab** 在 `Ask permissions` → `Plan mode` → `Auto mode` 之间切换，或在桌面版或VS Code的下拉菜单中选择。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/1.png" alt="Boris Cherny 关于自动模式" width="50%" /></a>

---

## 2/ 新的 /fewer-permission-prompts 技能

Anthropic发布了一个新的 `/fewer-permission-prompts` 技能。它会扫描你的会话历史，找出那些安全但反复提示权限的常见bash和MCP命令。然后它会推荐一个命令列表，供你添加到权限允许列表中。

使用这个来优化你的权限设置，避免不必要的权限提示，尤其是如果你不使用自动模式的话。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/2.png" alt="Boris Cherny 关于 /fewer-permission-prompts 技能" width="50%" /></a>

---

## 3/ 回顾（Recaps）

Anthropic本周早些时候推出了**回顾**功能，为Opus 4.7做准备。回顾是代理做了什么以及下一步做什么的简短摘要。

当几分钟或几小时后返回到一个长时间运行的会话时非常有用：

```
* 思考了 6m 27s

* 回顾：修复提交后转录偏移bug。样式闪烁
  部分已作为PR #29869提交（自动合并已开启，已发布到stamps）。下一步：
  我需要一个 `cc -c` 上剩余水平重排问题的屏幕录制，
  以针对那个单独的原因进行处理。（在 /config 中禁用回顾）
```

如果你不需要回顾，可以在 `/config` 中禁用。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/3.png" alt="Boris Cherny 关于回顾" width="50%" /></a>

---

## 4/ 专注模式

Boris非常喜欢CLI中新的**专注模式**，它隐藏所有中间工作，只关注最终结果。模型已经达到了他通常可以信任它执行正确命令和做出正确编辑的程度。他只看最终结果。

使用 `/focus` 切换开关。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/4.png" alt="Boris Cherny 关于专注模式" width="50%" /></a>

---

## 5/ 配置你的Effort（精力）级别

Opus 4.7使用**自适应思考**而不是固定的思考预算。要调整模型思考更多或更少，调节effort即可。

- **较低的effort** —— 更快的响应和更低的token使用量
- **较高的effort** —— 最高的智能和能力

滑块提供五个级别：`low` · `medium` · `high` · `xhigh` · `max` —— 左边是速度，右边是智能。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/5.png" alt="Boris Cherny 关于精力级别" width="50%" /></a>

---

## 6/ 给Claude一种验证其工作的方法

最后，确保Claude有一种验证其工作的方法。这一点一直很重要——现在4.7的输出质量是之前的2-3倍，所以比以往任何时候都更重要。

验证因任务而异：

- **后端工作** —— 让Claude运行你的服务器/服务以进行端到端测试
- **前端工作** —— 使用 [Claude Chromium扩展](https://code.claude.com/docs/en/chrome) 让Claude控制你的浏览器
- **桌面应用** —— 使用Computer Use

Boris现在的提示词通常像 `Claude do blah blah /go`，其中 `/go` 是一个技能，它：

1. 使用bash、浏览器或computer use进行端到端自测
2. 运行 `/simplify`
3. 提交一个PR

对于长时间运行的工作，验证更加重要——当你回到任务时，你知道代码是能用的。

<a href="https://x.com/bcherny"><img src="assets/boris-26-4-16/6.png" alt="Boris Cherny 关于验证" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — April 16, 2026](https://x.com/bcherny)
