# Claude Code中15个隐藏且未被充分利用的功能——来自Boris Cherny（中文翻译）

> 中文翻译自 [tips/claude-boris-15-tips-30-mar-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-15-tips-30-mar-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年3月30日分享的技巧总结。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 背景

Boris分享了他最喜欢的一批Claude Code中隐藏且未被充分利用的功能，重点是他最常使用的那些。

<a href="https://x.com/bcherny/status/2038454336355999749"><img src="assets/boris-26-3-30/0.png" alt="Boris Cherny 介绍推文" width="50%" /></a>

---

## 1/ Claude Code有移动应用

你知道吗，Claude Code有移动应用？Boris的很多代码都是在iOS应用上编写的——这是一种无需打开笔记本电脑就能进行修改的便捷方式。

- 下载适用于iOS/Android的Claude应用
- 导航到左侧的 **Code** 标签页
- 你可以直接从手机审查变更、批准PR和编写代码

<a href="https://x.com/bcherny/status/2038454337811386436"><img src="assets/boris-26-3-30/1.png" alt="Claude Code移动应用" width="50%" /></a>

---

## 2/ 在移动端/Web端/桌面端和终端之间移动会话

运行 `claude --teleport` 或 `/teleport` 在你的机器上继续云端会话。或者运行 `/remote-control` 从手机/Web端控制本地运行的会话。

- **Teleport（传送）**：将云端会话拉取到本地终端
- **Remote Control（远程控制）**：让你从任何设备控制本地会话
- Boris在他的 `/config` 中设置了 **"Enable Remote Control for all sessions"（为所有会话启用远程控制）**

<a href="https://x.com/bcherny/status/2038454339933548804"><img src="assets/boris-26-3-30/2.png" alt="Teleport和远程控制" width="50%" /></a>

---

## 3/ /loop 和 /schedule —— 两个最强大的功能

使用这些功能让Claude按设定的间隔自动运行，最长可达一周。Boris在本地运行了一堆循环：

- `/loop 5m /babysit` —— 自动处理代码审查、自动变基，并引导PR到生产环境
- `/loop 30m /slack-feedback` —— 每30分钟自动提交PR以获取Slack反馈
- `/loop /post-merge-sweeper` —— 提交PR以处理他遗漏的代码审查评论
- `/loop 1h /pr-pruner` —— 关闭陈旧且不再需要的PR
- ……还有更多！

尝试将工作流转化为技能+循环。非常强大。

<a href="https://x.com/bcherny/status/2038454341884154269"><img src="assets/boris-26-3-30/3.png" alt="/loop 和 /schedule" width="50%" /></a>

---

## 4/ 使用钩子确定性地运行逻辑

使用钩子在代理生命周期中运行逻辑。例如：

- 每次启动Claude时**动态加载**上下文（`SessionStart`）
- **记录模型运行的每个bash命令**（`PreToolUse`）
- **将权限提示路由到WhatsApp**供你批准/拒绝（`PermissionRequest`）
- **提醒Claude**在它停止时继续前进（`Stop`）

<a href="https://x.com/bcherny/status/2038454343519932844"><img src="assets/boris-26-3-30/4.png" alt="使用钩子" width="50%" /></a>

---

## 5/ Cowork Dispatch

Boris每天使用Dispatch来跟进Slack和邮件、管理文件，以及在他不在电脑前时在笔记本电脑上做事。当他不编码时，他就在使用Dispatch。

- Dispatch是Claude桌面应用的**安全远程控制**
- 它可以在你的许可下使用你的MCP、浏览器和电脑
- 将其视为一种从任何地方将非编码任务委托给Claude的方式

<a href="https://x.com/bcherny/status/2038454345419936040"><img src="assets/boris-26-3-30/5.png" alt="Cowork Dispatch" width="50%" /></a>

---

## 6/ 使用Chrome扩展进行前端工作

使用Claude Code最重要的技巧：**给Claude一种验证其输出的方法。** 一旦你这样做，Claude会迭代直到结果令人满意。

- 想象一下，要求某人建一个网站但不允许他们使用浏览器——结果可能不会好看
- 给Claude一个浏览器，它会编写代码并迭代直到看起来不错
- Boris每次处理Web代码时都使用Chrome扩展——它往往比其他类似的MCP更可靠

<a href="https://x.com/bcherny/status/2038454347156398333"><img src="assets/boris-26-3-30/6.png" alt="用于前端的Chrome扩展" width="50%" /></a>

---

## 7/ 使用Claude桌面应用自动启动和测试Web服务器

同样的思路，桌面应用集成了让Claude**自动运行你的Web服务器甚至在内置浏览器中测试它的能力。**

- 你可以在CLI或VSCode中使用Chrome扩展设置类似的功能
- 或者直接使用桌面应用获得一体化体验

<a href="https://x.com/bcherny/status/2038454348804714642"><img src="assets/boris-26-3-30/7.png" alt="桌面应用Web服务器测试" width="50%" /></a>

---

## 8/ 派生（Fork）你的会话

人们经常问如何派生现有会话。有两种方式：

1. 在你的会话中运行 `/branch`
2. 从CLI运行 `claude --resume <session-id> --fork-session`

`/branch` 创建一个分支对话——你现在就在分支中。要恢复原始会话，使用 `claude -r <original-session-id>`。

<a href="https://x.com/bcherny/status/2038454350214041740"><img src="assets/boris-26-3-30/8.png" alt="派生你的会话" width="50%" /></a>

---

## 9/ 使用 /btw 进行旁路查询

Boris经常使用这个功能在代理工作时回答快速问题。`/btw` 让你在不中断代理当前任务的情况下提出一个旁路问题。

示例：
```
/btw 如何拼写dachshund？
> dachshund —— 德语中"獾狗"的意思（dachs = 獾，hund = 狗）。
↑/↓ 滚动 · Space、Enter或Escape关闭
```

<a href="https://x.com/bcherny/status/2038454351849787485"><img src="assets/boris-26-3-30/9.png" alt="/btw 旁路查询" width="50%" /></a>

---

## 10/ 使用Git Worktrees

Claude Code内置了对git worktrees的深度支持。Worktrees对于在同一仓库中做大量并行工作至关重要。Boris**随时运行着几十个Claude**，这就是他做到这一点的方式。

- 使用 `claude -w` 在worktree中启动新会话
- 或在Claude桌面应用中点击 **"worktree" 复选框**
- 对于非git VCS用户，使用 `WorktreeCreate` 钩子添加你自己的worktree创建逻辑

<a href="https://x.com/bcherny/status/2038454353787519164"><img src="assets/boris-26-3-30/10.png" alt="Git Worktrees" width="50%" /></a>

---

## 11/ 使用 /batch 展开大规模变更集

`/batch` 会对你的需求进行访谈，然后让Claude将工作分发给尽可能多的**worktree代理**（几十个、几百个，甚至几千个）来完成。

- 用于大型代码迁移和其他类型的可并行化工作
- 每个worktree代理在其代码库副本上独立工作

<a href="https://x.com/bcherny/status/2038454355469484142"><img src="assets/boris-26-3-30/11.png" alt="/batch 大规模变更集" width="50%" /></a>

---

## 12/ 使用 --bare 将SDK启动速度提升最多10倍

默认情况下，当你运行 `claude -p`（或TypeScript/Python SDK）时，Claude会搜索本地的CLAUDE.md、设置和MCP。但对于非交互式使用，大多数情况下你希望通过 `--system-prompt`、`--mcp-config`、`--settings` 等显式指定要加载的内容。

- 这是SDK最初构建时的设计疏忽
- 在未来的版本中，他们会将默认值改为 `--bare`
- 目前，使用该标志可以享受最高**10倍的启动速度**

```bash
claude -p "总结这个代码库" \
    --output-format=stream-json \
    --verbose \
    --bare
```

<a href="https://x.com/bcherny/status/2038454357088457168"><img src="assets/boris-26-3-30/12.png" alt="--bare 标志用于SDK启动" width="50%" /></a>

---

## 13/ 使用 --add-dir 让Claude访问更多文件夹

当在多个仓库之间工作时，Boris通常在一个仓库中启动Claude，并使用 `--add-dir`（或 `/add-dir`）让Claude看到其他仓库。

- 这不仅让Claude知道该仓库的存在，还**赋予它在该仓库中工作的权限**
- 或者将 `"additionalDirectories"` 添加到团队的 `settings.json`，以便在启动Claude Code时始终加载额外的文件夹

<a href="https://x.com/bcherny/status/2038454359047156203"><img src="assets/boris-26-3-30/13.png" alt="--add-dir 用于多个仓库" width="50%" /></a>

---

## 14/ 使用 --agent 给Claude Code自定义系统提示和工具

自定义代理是一个强大的原语，但常常被忽视。使用方法很简单：只需在 `.claude/agents/` 中定义一个新代理，然后运行：

```bash
claude --agent=<你的代理名称>
```

- 代理可以具有受限的工具、自定义描述和特定模型
- 它们非常适合创建只读代理、专门的审查代理或领域特定的工具

<a href="https://x.com/bcherny/status/2038454360418787764"><img src="assets/boris-26-3-30/14.png" alt="--agent 用于自定义系统提示" width="50%" /></a>

---

## 15/ 使用 /voice 启用语音输入

有趣的事实：Boris大部分编码工作是通过对Claude说话而不是打字完成的。

- 在CLI中运行 `/voice`，然后按住空格键说话
- 在桌面上按下语音按钮
- 或在iOS设置中启用听写功能

<a href="https://x.com/bcherny/status/2038454362226467112"><img src="assets/boris-26-3-30/15.png" alt="/voice 语音输入" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — March 30, 2026](https://x.com/bcherny/status/2038454336355999749)
