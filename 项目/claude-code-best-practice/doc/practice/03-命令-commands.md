# 命令（Commands）最佳实践

> 中文翻译整理自 `best-practice/claude-commands.md`
> 学习顺序：第 3 篇
> 官方文档：[Slash Commands](https://code.claude.com/docs/en/slash-commands)

---

## 什么是 Command？

Command 是 Claude Code 中的**斜杠命令**（如 `/code-review`），用于快速执行常用工作流。你可以把每天重复做的事定义为一个命令，一键执行。

---

## 前置元数据字段（17 个）

每个命令定义文件（`.claude/commands/<名称>.md`）使用 YAML frontmatter 配置：

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | 否 | 显示名称和 `/命令` 标识。省略时使用目录名 |
| `description` | string | 推荐 | 命令功能描述。显示在自动补全中 |
| `when_to_use` | string | 否 | 何时调用的额外上下文——触发短语或示例请求 |
| `argument-hint` | string | 否 | 自动补全时的提示（如 `[issue-number]`） |
| `arguments` | string/list | 否 | 位置参数名，用于命令内容中的 `$name` 替换 |
| `disable-model-invocation` | boolean | 否 | `true` 则阻止 Claude 自动调用 |
| `user-invocable` | boolean | 否 | `false` 则从 `/` 菜单隐藏 |
| `paths` | string/list | 否 | Glob 模式，限制命令只在匹配文件路径时激活 |
| `allowed-tools` | string | 否 | 命令激活时免确认的工具 |
| `disallowed-tools` | string/list | 否 | 命令激活时禁止的工具 |
| `model` | string | 否 | 命令运行时的模型（`haiku`、`sonnet`、`opus`） |
| `effort` | string | 否 | 覆盖模型推理力度（`low`、`medium`、`high`、`xhigh`、`max`） |
| `context` | string | 否 | `fork` 则在独立子代理中运行 |
| `agent` | string | 否 | `context: fork` 时的子代理类型（默认 `general-purpose`） |
| `background` | boolean | 否 | 仅与 `context: fork` 配合使用。设为 `false` 则等待分支子代理的结果，而不是在后台运行。默认 `true`。需 v2.1.218+ |
| `shell` | string | 否 | `` !`command` `` 块的 shell：`bash`（默认）或 `powershell` |
| `hooks` | object | 否 | 生命周期钩子 |

---

## 系统内置命令大全（86 个）

> 💡 点击命令名可跳转到官方文档对应章节。

### 🔐 认证类

| 命令 | 说明 |
|------|------|
| `/login` | 登录你的 Anthropic 账户 |
| `/logout` | 登出 Anthropic 账户 |
| `/design-login` | 授权 design-system 访问 claude.ai 账户（用于 `/design-sync`） |
| `/setup-bedrock` | 交互式配置 Amazon Bedrock（需设 `CLAUDE_CODE_USE_BEDROCK=1`）|
| `/setup-vertex` | 交互式配置 Google Cloud's Agent Platform（需设 `CLAUDE_CODE_USE_VERTEX=1`）|
| `/upgrade` | 打开升级页面切换更高套餐 |

### ⚙️ 配置类

| 命令 | 说明 |
|------|------|
| `/color [color\|default]` | 设置当前会话的提示栏颜色（`red` `blue` `green` 等）|
| `/config [key=value ...]` | 打开设置界面调整主题、模型等；v2.1.181+ 支持直接传参修改 |
| `/focus` | 切换聚焦视图（仅显示最后提示 + 摘要 + 最终回复）|
| `/keybindings` | 打开或创建快捷键配置文件 |
| `/permissions` | 管理工具权限的 allow/ask/deny 规则 |
| `/privacy-settings` | 查看和更新隐私设置（Pro/Max 用户）|
| `/radio` | 在浏览器打开 Claude FM lo-fi 电台 |
| `/sandbox` | 切换沙盒模式 |
| `/scroll-speed` | 交互式调节鼠标滚轮速度 |
| `/statusline` | 配置状态栏内容 |
| `/stickers` | 订购 Claude Code 贴纸 |
| `/terminal-setup` | 配置终端快捷键（Shift+Enter 等）|
| `/theme` | 切换颜色主题（暗色、明色、色盲友好、ANSI、自定义）|
| `/tui [default\|fullscreen]` | 设置终端渲染器，`fullscreen` 启用无闪烁全屏模式 |
| `/voice [hold\|tap\|off]` | 切换语音输入模式（需 Claude.ai 账户）|

### 📊 上下文类

| 命令 | 说明 |
|------|------|
| `/context [all]` | 可视化展示当前上下文使用情况（彩色网格），可加 `all` 展开详细 |
| `/cost` | `/usage` 的别名 |
| `/insights` | 生成 Claude Code 会话分析报告 |
| `/stats` | `/usage` 的别名（打开在 Stats 标签页）|
| `/status` | 显示版本、模型、账户、连接状态（响应中也可用）|
| `/usage` | 显示会话费用、套餐限额和使用统计 |
| `/usage-credits` | 配置额度超限后继续使用的信用额度 |

### 🔧 调试类

| 命令 | 说明 |
|------|------|
| `/bug [report]` | 报告 Bug 或分享对话。可选择包含多少会话历史，在发送前有确认界面。别名：`/share` |
| `/doctor` | 运行完整的设置检查，诊断和修复 Claude Code 安装和设置。按 `f` 让 Claude 自动修复。别名：`/checkup` |
| `/feedback [report]` | 提交关于 Claude Code 的产品反馈。打开与 `/bug` 相同的对话框 |
| `/heapdump` | 导出 JS 堆快照用于诊断内存问题 |
| `/help` | 查看帮助和可用命令 |
| `/release-notes` | 查看版本更新日志 |
| `/tasks` | 查看和管理所有后台运行的任务 |
| `/copy [N]` | 复制上一条回复到剪贴板；指定 N 复制倒数第 N 条；含代码块时可选代码片段 |

### 📤 导出类

| 命令 | 说明 |
|------|------|
| `/export [filename]` | 导出当前会话为纯文本。带文件名直接写入，否则打开保存对话框 |

### 🔌 扩展类

| 命令 | 说明 |
|------|------|
| `/agents` | 查看子代理管理指南 |
| `/chrome` | 配置 Claude in Chrome 设置 |
| `/hooks` | 查看钩子配置 |
| `/ide` | 管理 IDE 集成和显示状态 |
| `/mcp [reconnect\|enable\|disable]` | 管理 MCP 服务器连接。无参数打开交互列表 |
| `/plugin [subcommand]` | 管理 Claude Code 插件（list, install, enable, disable）|
| `/reload-plugins [--force]` | 重载所有插件以应用更改 |
| `/reload-skills` | 重新扫描 skill 和命令目录 |
| `/skills` | 列出可用 skill。按 `t` 按 token 排序 |

### 🧠 记忆类

| 命令 | 说明 |
|------|------|
| `/memory` | 编辑 `CLAUDE.md` 记忆文件，管理自动记忆 |

### 🤖 模型类

| 命令 | 说明 |
|------|------|
| `/advisor [model\|off]` | 启用顾问模式（调用第二个模型提供建议）|
| `/effort [level]` | 设置模型推理力度（low/medium/high/xhigh/max/ultracode）|
| `/fast [on\|off]` | 切换快速模式 |
| `/model [model]` | 切换 AI 模型并保存为默认 |
| `/passes` | 分享免费周给朋友（仅限符合条件的账户）|
| `/plan [description]` | 进入计划模式。可附带描述直接开始 |
| `/ultraplan <prompt>` | 在超算计划会话中起草计划，浏览器审查后远程执行 |

### 📁 项目类

| 命令 | 说明 |
|------|------|
| `/add-dir <path>` | 添加工作目录供当前会话访问文件 |
| `/diff` | 打开交互式差异查看器（未提交更改 + 每轮更改）|
| `/init` | 初始化项目，生成 `CLAUDE.md` 指南 |
| `/review [PR]` | 按编号快速审查 GitHub PR（单轮审查）。需要更深入审查请用 `/code-review`。无参数列出打开的 PR |
| `/security-review` | 分析当前分支的待提交更改的安全漏洞 |
| `/team-onboarding` | 从使用历史生成团队入门指南 |
| `/ultrareview [PR]` | 在云端沙箱运行深度多代理代码审查（Pro/Max 含 3 次免费）|

### 🌐 远程类

| 命令 | 说明 |
|------|------|
| `/autofix-pr [prompt]` | 启动 web 会话监控当前 PR，CI 失败或收到评论时自动修复 |
| `/desktop` | 切换到 Claude Code 桌面应用（macOS/Windows）|
| `/install-github-app` | 安装 Claude GitHub App |
| `/install-slack-app` | 安装 Claude Slack App |
| `/mobile` | 显示二维码下载 Claude 移动 App |
| `/remote-control` | 使当前会话可从 claude.ai 远程控制 |
| `/schedule [description]` | 创建定时任务或日常例程 |
| `/teleport` | 将 web 端 Claude Code 会话拉取到本终端 |
| `/web-setup` | 用本地 `gh` CLI 凭证连接 GitHub 账户到 Claude Code Web |

### 💬 会话类

| 命令 | 说明 |
|------|------|
| `/background [prompt]` | 将当前会话转为后台运行，释放终端 |
| `/branch [name]` | 在当前位置创建会话分支 |
| `/btw <question>` | 问一个快速旁路问题，不加入会话记录 |
| `/cd <path>` | 切换工作目录，不破坏提示缓存 |
| `/clear [name]` | 开始新会话。可传 `name` 标记旧会话便于恢复 |
| `/compact [instructions]` | 压缩当前会话以释放上下文，可附带压缩重点 |
| `/exit` | 退出 CLI。附加到后台会话时仅分离 |
| `/fork [prompt]` | 将当前对话复制为新的后台会话，你可以继续在当前会话工作 |
| `/goal [condition\|clear]` | 设定目标——Claude 跨轮次持续工作直到条件满足 |
| `/recap` | 生成当前会话的一行摘要 |
| `/rename [name]` | 重命名当前会话 |
| `/resume [session]` | 按 ID 或名称恢复会话 |
| `/rewind` | 回退对话或代码到之前的状态 |
| `/stop` | 停止当前后台会话 |
| `/subtask <directive>` | 将当前会话委托给一个名为 Subtask 的 agent，专注于高级用例的独占任务 |
| `/workflows` | 打开工作流进度视图 |

> ⚠️ 注：部分内置 skill（如 `/debug`）也可能出现在 `/` 菜单中，但它们不属于内置命令。

---

### 命令分类速查图

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   🔐 认证   │  │  ⚙️ 配置   │  │  📊 上下文  │  │  🔧 调试   │  │  🤖 模型   │
│  (6 个)     │  │ (16 个)    │  │  (7 个)     │  │  (8 个)     │  │  (7 个)     │
├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤
│ 📤 导出     │  │ 🔌 扩展    │  │ 🧠 记忆     │  │ 📁 项目     │  │ 🌐 远程     │
│  (1 个)     │  │ (9 个)     │  │  (1 个)     │  │  (7 个)     │  │ (10 个)     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
                          ┌─────────────┐
                          │  💬 会话   │
                          │  (15 个)    │
                          └─────────────┘
```

---

## 最佳实践

### 何时创建自定义命令

- **每天多次执行的工作流** — 如代码审查、部署、测试
- **多步骤操作** — 需要多个提示才能完成的任务
- **团队共享流程** — 让团队用统一的命令执行标准流程

### 命令 vs Subagent vs Skill

| | Command | Subagent | Skill |
|--|---------|----------|-------|
| **最佳用途** | 快捷入口 | 并行任务 | 知识注入 |
| **是否可被 Claude 自动调用** | 是 | 是（PROACTIVELY） | 是 |
| **上下文隔离** | 可选（fork） | 是 | 可选（fork） |
| **适合** | 工作流触发 | 后台处理 | 领域知识 |

### Tips

- Boris Cherny（Claude Code 创建者）：**用 Command 代替 Subagent 来做你的工作流**
- 每天使用超过 1 次的操作，就应该做成 Command 或 Skill
- 一个 Command 定义文件通常不超过 20 行

---

## 一个自定义命令示例

```yaml
---
name: deploy
description: 部署当前分支到生产环境
argument-hint: [env]
arguments: env
allowed-tools: Bash(git *), Bash(npm run *)
---
执行部署流程：
1. 运行测试：`npm test`
2. 构建项目：`npm run build`
3. 部署到 $env 环境
4. 验证部署状态
```

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Subagent（子代理） | [`02-子代理-subagents.md`](./02-子代理-subagents.md) | `/agents`、`/fork`、`/subtask` 命令背后的 Subagent 概念 |
| Skill（技能） | [`04-技能-skills.md`](./04-技能-skills.md) | Skill 也可通过 `/` 调用，与 Command 的区别与联系 |
| 实现参考 | [`../implementation/02-command-实现.md`](../implementation/02-command-实现.md) | 本仓库 `/weather-orchestrator` 等命令的实现 |

---

> 下一篇：[04-技能-skills.md](./04-技能-skills.md)
