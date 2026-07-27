# CLI 启动参数最佳实践

> 中文翻译整理自 `best-practice/claude-cli-startup-flags.md`
> 学习顺序：第 8 篇
> 官方文档：[CLI Reference](https://code.claude.com/docs/en/cli-reference)

---

## 什么是 CLI 启动参数？

从终端启动 Claude Code 时，你可以通过启动参数控制其行为——用什么模型、什么权限模式、是否恢复之前的会话等。

---

## 常用启动方式

```bash
claude                                    # 交互模式
claude "修复这个 bug"                     # 交互模式 + 初始提示
claude -p "运行所有测试"                   # 一次性任务（非交互）
claude --model opus                       # 指定模型
claude -w                                 # 在工作树中启动
```

---

## 参数速查表（按类别）

### 📁 会话管理

| 参数 | 简写 | 说明 |
|------|------|------|
| `--continue` | `-c` | 继续当前目录最近的会话 |
| `--resume` | `-r` | 按 ID 或名称恢复指定会话，或显示交互选择器 |
| `--from-pr <编号\|URL>` | | 恢复链接到指定 GitHub PR 的会话 |
| `--fork-session` | | 恢复时创建新的会话 ID（与 `--resume` 或 `--continue` 配合）|
| `--session-id <UUID>` | | 使用指定会话 ID（必须是有效 UUID）|
| `--no-session-persistence` | | 禁止会话持久化（仅 print 模式）|
| `--remote` | | 在 claude.ai 创建新网页会话 |
| `--teleport` | | 将网页会话拉取到本地终端 |

### 🤖 模型 & 配置

| 参数 | 说明 |
|------|------|
| `--model <名称>` | 设置模型（`sonnet`、`opus`、`haiku` 或完整模型 ID）|
| `--fallback-model <名称>` | 主模型不可用时的自动备用（仅 print 模式）|
| `--betas <列表>` | API 请求的 Beta 头（仅 API key 用户）|

### 🔒 权限 & 安全

| 参数 | 说明 |
|------|------|
| `--dangerously-skip-permissions` | ⚠️ 跳过所有权限检查（使用需极度谨慎）|
| `--allow-dangerously-skip-permissions` | 允许绕过权限作为选项，但不激活 |
| `--permission-mode <模式>` | 设置初始权限模式：`default`、`plan`、`acceptEdits`、`bypassPermissions` |
| `--allowedTools <工具>` | 免确认执行的工具（权限规则语法）|
| `--disallowedTools <工具>` | 从模型上下文中完全移除的工具 |
| `--tools <工具>` | 限制 Claude 可使用的内置工具（用 `""` 禁用所有）|
| `--permission-prompt-tool <工具>` | 非交互模式下处理权限提示的 MCP 工具 |

### 📤 输出 & 格式

| 参数 | 简写 | 说明 |
|------|------|------|
| `--print` | `-p` | 非交互模式（一次性任务 / 无头模式）|
| `--output-format <格式>` | | 输出格式：`text`、`json`、`stream-json` |
| `--input-format <格式>` | | 输入格式：`text`、`stream-json` |
| `--json-schema <SCHEMA>` | | 获取符合 JSON Schema 的验证输出（仅 print 模式）|
| `--include-partial-messages` | | 包含部分流式事件（需 `--print` + `--output-format=stream-json`）|
| `--verbose` | | 启用详细日志输出（完整逐轮输出）|

### 📝 系统提示

| 参数 | 说明 |
|------|------|
| `--system-prompt <文本>` | 用自定义文本替换整个系统提示 |
| `--system-prompt-file <路径>` | 从文件加载系统提示（仅 print 模式）|
| `--append-system-prompt <文本>` | 在默认系统提示后追加自定义文本 |
| `--append-system-prompt-file <路径>` | 在默认提示后追加文件内容（仅 print 模式）|

### 🤝 Agent & Subagent

| 参数 | 说明 |
|------|------|
| `--agent <名称>` | 为当前会话指定 agent |
| `--agents <JSON>` | 通过 JSON 动态定义自定义子代理 |
| `--teammate-mode <模式>` | 设置 Agent 团队显示模式：`auto`、`in-process`、`tmux` |

### 🔌 MCP & 插件

| 参数 | 说明 |
|------|------|
| `--mcp-config <路径\|JSON>` | 从 JSON 文件或字符串加载 MCP 服务器 |
| `--strict-mcp-config` | 仅使用 `--mcp-config` 中的 MCP 服务器，忽略所有其他 |
| `--plugin-dir <路径>` | 从目录加载插件（仅当前会话，可重复使用）|

### 📂 目录 & 工作区

| 参数 | 简写 | 说明 |
|------|------|------|
| `--add-dir <路径>` | | 添加额外工作目录供 Claude 访问 |
| `--worktree` | `-w` | 在隔离的 git worktree 中启动 Claude（从 HEAD 分支）|

### 💰 预算 & 限制

| 参数 | 说明 |
|------|------|
| `--max-budget-usd <金额>` | API 调用的最大美元金额（仅 print 模式）|
| `--max-turns <次数>` | 限制 agentic 轮数（仅 print 模式）|

### 🔗 集成

| 参数 | 说明 |
|------|------|
| `--chrome` | 启用 Chrome 浏览器集成 |
| `--no-chrome` | 禁用 Chrome 浏览器集成 |
| `--ide` | 启动时自动连接到 IDE（如果恰好有一个有效 IDE）|

### ⚙️ 初始化 & 维护

| 参数 | 说明 |
|------|------|
| `--init` | 运行初始化钩子后进入交互模式 |
| `--init-only` | 仅运行初始化钩子后退出（不进入交互模式）|
| `--maintenance` | 运行维护钩子后退出 |

### 🐛 调试 & 诊断

| 参数 | 说明 |
|------|------|
| `--debug <类别>` | 启用调试模式，可选类别过滤（如 `"api,hooks"`）|

### 🎛️ 设置覆盖

| 参数 | 说明 |
|------|------|
| `--settings <路径\|JSON>` | 加载设置 JSON 文件或字符串 |
| `--setting-sources <列表>` | 逗号分隔的配置源：`user`、`project`、`local` |
| `--disable-slash-commands` | 禁用所有 skill 和斜杠命令 |

### ℹ️ 版本 & 帮助

| 参数 | 简写 | 说明 |
|------|------|------|
| `--version` | `-v` | 输出版本号 |
| `--help` | `-h` | 显示帮助信息 |

---

## 子命令

通过 `claude <子命令>` 使用：

| 子命令 | 说明 |
|--------|------|
| `claude` | 启动交互式 REPL |
| `claude "查询"` | 带初始提示启动 REPL |
| `claude agents` | 列出已配置的 agent |
| `claude auth` | 管理认证 |
| `claude doctor` | 从命令行运行诊断 |
| `claude install` | 安装或切换 Claude Code 原生构建版本 |
| `claude mcp` | 配置 MCP 服务器（add, remove, list, get, enable）|
| `claude plugin` | 管理插件 |
| `claude remote-control` | 管理远程控制会话 |
| `claude setup-token` | 创建订阅使用的长期令牌 |
| `claude update` / `claude upgrade` | 更新到最新版本 |

---

## 仅启动时可用的环境变量

这些变量需要在启动 Claude 之前设置，**不能**通过 `settings.json` 的 `env` 字段配置：

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_SIMPLE=1` | 简易模式（只有 Bash + 编辑工具）|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | 启用实验性 Agent 团队 |
| `CLAUDE_BASH_NO_LOGIN=1` | 跳过 BashTool 的 login shell |
| `USE_BUILTIN_RIPGREP=0` | 使用系统 ripgrep（Alpine Linux 等环境）|
| `CLAUDE_CODE_TMPDIR` | 覆盖内部文件的临时目录 |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` | 启用附加目录的 CLAUDE.md 加载 |
| `CCR_FORCE_BUNDLE=1` | 使用 `claude --remote` 时强制打包本仓库 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 控制推理力度（也可通过 `env` 字段设置）|
| `DISABLE_AUTOUPDATER=1` | 禁用自动更新（也可通过 `env` 字段设置）|

> 可通过 `env` 字段配置的环境变量（如 `MAX_THINKING_TOKENS`、`CLAUDE_CODE_SHELL` 等）详见[设置文档](./05-设置-settings.md#八环境变量通过-env-字段)。

---

## 实操场景

### 场景 1：日常开发
```bash
claude --model sonnet --permission-mode acceptEdits
```

### 场景 2：代码审查
```bash
claude -p "审查当前分支的所有变更" --model opus
```

### 场景 3：批量处理
```bash
claude -p "修复 src/ 目录下所有 TypeScript 错误" --model sonnet
```

### 场景 4：后台任务
```bash
claude --bg "重构 auth 模块" --model haiku
```

### 场景 5：隔离工作
```bash
claude -w
```

---

> 参考：设置文档 [`05-设置-settings.md`](./05-设置-settings.md) 包含所有 `env` 字段可配置的环境变量。

> 下一篇：[09-能力强化-power-ups.md](./09-能力强化-power-ups.md)
