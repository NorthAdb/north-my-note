# 定制Claude Code的12种方式——来自Boris Cherny（中文翻译）

> 中文翻译自 [tips/claude-boris-12-tips-12-feb-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-12-tips-12-feb-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年2月12日分享的定制技巧总结。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 背景

Boris Cherny指出，可定制性是工程师们最喜爱Claude Code的地方——包括钩子、插件、LSP、MCP、技能、精力、自定义代理、状态栏、输出风格等等。他分享了开发者和团队定制其设置的12种实用方式。

<a href="https://x.com/bcherny/status/2021699851499798911"><img src="assets/boris-26-2-12/0.webp" alt="Boris Cherny 介绍推文" width="50%" /></a>

---

## 1/ 配置终端

设置终端以获得最佳的Claude Code体验：

- **主题**：运行 `/config` 设置浅色/深色模式
- **通知**：为iTerm2启用通知，或使用自定义通知钩子
- **换行**：如果在IDE终端、Apple Terminal、Warp或Alacritty中使用Claude Code，运行 `/terminal-setup` 启用shift+enter换行（这样你就不需要输入 `\`）
- **Vim模式**：运行 `/vim`

<a href="https://x.com/bcherny/status/2021699859359883608"><img src="assets/boris-26-2-12/1.webp" alt="配置你的终端" width="50%" /></a>

---

## 2/ 调整Effort（精力）级别

运行 `/model` 选择你偏好的精力级别：

- **低** —— 更少的token，更快的响应
- **中** —— 平衡行为
- **高** —— 更多token，更高智能

Boris的偏好：所有任务都使用High。

<a href="https://x.com/bcherny/status/2021699860869902424"><img src="assets/boris-26-2-12/2.webp" alt="调整精力级别" width="50%" /></a>

---

## 3/ 安装插件、MCP和技能

插件让你安装LSP（适用于所有主流语言）、MCP、技能、代理和自定义钩子。

从官方的Anthropic插件市场安装，或为你的公司创建自己的市场。将 `settings.json` 检入代码库，自动为团队添加市场。

运行 `/plugin` 开始使用。

<a href="https://x.com/bcherny/status/2021699862522364149"><img src="assets/boris-26-2-12/3.webp" alt="安装插件、MCP和技能" width="50%" /></a>

---

## 4/ 创建自定义代理

在 `.claude/agents` 中放置 `.md` 文件以创建自定义代理。每个代理可以有自定义名称、颜色、工具集、预允许和预禁止的工具、权限模式和模型。

你也可以使用 `settings.json` 中的 `"agent"` 字段或 `--agent` 标志设置主会话的默认代理。

运行 `/agents` 开始使用。

<a href="https://x.com/bcherny/status/2021700144039903699"><img src="assets/boris-26-2-12/4.webp" alt="创建自定义代理" width="50%" /></a>

---

## 5/ 预批准常用权限

Claude Code使用一个结合了提示注入检测、静态分析、沙箱和人工监督的权限系统。

开箱即用，一组小型的安全命令已被预批准。要预批准更多命令，运行 `/permissions` 并添加到允许和阻止列表。将这些检入你团队的 `settings.json`。

支持完整的通配符语法——例如 `Bash(bun run *)` 或 `Edit(/docs/**)`。

<a href="https://x.com/bcherny/status/2021700332292911228"><img src="assets/boris-26-2-12/5.webp" alt="预批准常用权限" width="50%" /></a>

---

## 6/ 启用沙箱

选择使用Claude Code的开源沙箱运行时，在提高安全性的同时减少权限提示。

运行 `/sandbox` 启用。沙箱在你的机器上运行，并支持文件和网络隔离。

<a href="https://x.com/bcherny/status/2021700506465579443"><img src="assets/boris-26-2-12/6.webp" alt="启用沙箱" width="50%" /></a>

---

## 7/ 添加状态栏

自定义状态栏显示在编辑器正下方，显示模型、目录、剩余上下文、成本以及你在工作时想看到的任何其他信息。

每个团队成员可以有不同状态栏。使用 `/statusline` 让Claude基于你的 `.bashrc`/`.zshrc` 生成一个。

<a href="https://x.com/bcherny/status/2021700784019452195"><img src="assets/boris-26-2-12/7.webp" alt="添加状态栏" width="50%" /></a>

---

## 8/ 自定义你的快捷键

Claude Code中的每个按键绑定都是可自定义的。运行 `/keybindings` 重新映射任何按键。设置实时加载，因此你可以立即看到效果。

<a href="https://x.com/bcherny/status/2021700883873165435"><img src="assets/boris-26-2-12/8.webp" alt="自定义你的快捷键" width="50%" /></a>

---

## 9/ 设置钩子

钩子让你确定性地接入Claude的生命周期：

- 自动将权限请求路由到Slack或Opus
- 当Claude到达一轮结束时，提示它继续（你甚至可以启动一个代理或使用提示来决定Claude是否应该继续）
- 预处理或后处理工具调用，例如添加你自己的日志记录

让Claude添加一个钩子来开始。

<a href="https://x.com/bcherny/status/2021701059253874861"><img src="assets/boris-26-2-12/9.webp" alt="设置钩子" width="50%" /></a>

---

## 10/ 自定义加载动画动词

自定义你的加载动画动词，添加或替换默认列表为你自己的动词。将 `settings.json` 检入源代码管理，与团队共享动词。

<a href="https://x.com/bcherny/status/2021701145023197516"><img src="assets/boris-26-2-12/10.webp" alt="自定义你的加载动画动词" width="50%" /></a>

---

## 11/ 使用输出风格

运行 `/config` 并设置输出风格，让Claude使用不同的语气或格式回复。

- **解释型** —— 推荐在熟悉新代码库时使用，让Claude在工作的同时解释框架和代码模式
- **学习型** —— 让Claude指导你进行代码更改
- **自定义** —— 创建自定义输出风格以调整Claude的语气

<a href="https://x.com/bcherny/status/2021701379409273093"><img src="assets/boris-26-2-12/11.webp" alt="使用输出风格" width="50%" /></a>

---

## 12/ 定制一切！

Claude Code开箱即用效果很好，但当你进行定制时，将你的 `settings.json` 检入git，这样你的团队也能受益。在多个层级支持配置：

- 针对你的代码库
- 针对子文件夹
- 仅针对你自己
- 通过企业级策略

拥有37个设置和84个环境变量（使用 `settings.json` 中的 `"env"` 字段以避免包装脚本），你想要的任何行为很可能都是可配置的。

<a href="https://x.com/bcherny/status/2021701636075458648"><img src="assets/boris-26-2-12/12.webp" alt="定制一切" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — February 12, 2026](https://x.com/bcherny)
- [Claude Code Terminal Setup Docs](https://code.claude.com/docs/en/terminal)
- [Claude Code Plugins & Discovery Docs](https://code.claude.com/docs/en/discover-plugins)
- [Claude Code Sub-agents Docs](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Permissions Docs](https://code.claude.com/docs/en/permissions)
- [Claude Code Sandbox Docs](https://code.claude.com/docs/en/sandbox)
- [Claude Code Status Line Docs](https://code.claude.com/docs/en/statusline)
- [Claude Code Keyboard Shortcuts Docs](https://code.claude.com/docs/en/keybindings)
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Output Styles Docs](https://code.claude.com/docs/en/output-styles)
- [Claude Code Settings Docs](https://code.claude.com/docs/en/settings)
