# 设置（Settings）最佳实践

> 中文翻译整理自 `best-practice/claude-settings.md`
> 学习顺序：第 5 篇
> 官方文档：[Settings](https://code.claude.com/docs/en/settings)

---

## 什么是 Settings？

Settings 是 Claude Code 的**配置文件体系**。通过 `settings.json`，你可以控制 Claude 的行为、权限、模型、外观等几乎所有方面。

截至 v2.1.215，Claude Code 提供了 **80+ 个设置项** 和 **200+ 个环境变量**（使用 `"env"` 字段在 settings.json 中配置，无需 wrapper 脚本）。

---

## 配置优先级

从高到低（高优先级覆盖低优先级）：

| 优先级 | 位置 | 范围 | 共享？ | 用途 |
|--------|------|------|--------|------|
| 1 | **托管设置**（MDM/注册表） | 组织 | 是（IT 部署） | 安全策略，不可覆盖 |
| 2 | **命令行参数** | 会话 | 不适用 | 临时单次覆盖 |
| 3 | `.claude/settings.local.json` | 项目 | 否（git-ignored） | 个人项目配置 |
| 4 | `.claude/settings.json` | 项目 | 是（git 提交） | 团队共享配置 |
| 5 | `~/.claude/settings.json` | 用户 | 不适用 | 全局默认配置 |

> **关键规则：**
> - `deny` 规则拥有最高安全优先级，不能被低优先级的 allow/ask 规则覆盖
> - 数组配置（如 `permissions.allow`）是跨作用域**合并并去重**的
> - 托管设置可以通过 MDM（macOS）、注册表（Windows）或 `managed-settings.json` 文件部署
> - v2.1.126+：`/config` 的更改持久化到 `~/.claude/settings.json`，重启后保留

---

## 一、通用设置（General Settings）

### 1.1 基本配置

| 键 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `$schema` | string | - | JSON Schema URL，用于 IDE 校验和自动补全 |
| `model` | string | `"default"` | 默认模型（支持别名 `sonnet`、`opus`、`haiku` 或完整模型 ID） |
| `agent` | string | - | 主对话的默认 agent，值为 `.claude/agents/` 中的 agent 名称 |
| `language` | string | `"english"` | Claude 首选响应语言 |
| `outputStyle` | string | `"default"` | 输出风格（如 `"Explanatory"`） |
| `plansDirectory` | string | `~/.claude/plans` | `/plan` 输出目录 |
| `autoMemoryEnabled` | boolean | `true` | 启用自动记忆 |
| `autoMemoryDirectory` | string | - | 自定义自动记忆存储目录 |
| `fileCheckpointingEnabled` | boolean | `true` | 编辑前自动创建快照（支持 `/rewind` 回退）|

### 1.2 工作区与 Worktree

| 键 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `worktree.symlinkDirectories` | array | `[]` | 从主仓库符号链接到 worktree 的目录（避免重复） |
| `worktree.sparsePaths` | array | `[]` | worktree 中通过 sparse checkout 检出的路径 |
| `worktree.baseRef` | string | `"fresh"` | worktree 分支来源：`"fresh"` 从远端分支创建，`"head"` 从本地 HEAD |
| `worktree.bgIsolation` | string | `"worktree"` | 后台会话隔离模式 |
| `claudeMdExcludes` | array | - | 跳过加载的 CLAUDE.md 文件 glob 模式 |

### 1.3 归属与认证

| 键 | 类型 | 说明 |
|-----|------|------|
| `attribution.commit` | string | Git commit 归属信息（支持 trailers） |
| `attribution.pr` | string | PR 描述归属信息 |
| `attribution.sessionUrl` | boolean | 在 web/Remote Control 会话的 commit 中加入会话 URL 链接 |
| `prUrlTemplate` | string | PR 徽章链接的 URL 模板（自托管 GitLab/Bitbucket 等） |
| `apiKeyHelper` | string | 动态生成认证 token 的脚本路径（token 以 `X-Api-Key` 和 `Authorization: Bearer` 两个头发送）|
| `forceLoginMethod` | string | 限制登录方式：`"claudeai"`、`"console"`、`"gateway"` |
| `forceLoginOrgUUID` | string/array | 限制登录到指定组织 UUID |
| `gcpAuthRefresh` | string | GCP 凭证刷新脚本 |

### 1.4 技能与模型控制

| 键 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `maxSkillDescriptionChars` | number | `1536` | 技能描述的字符上限 |
| `skillListingBudgetFraction` | number | `0.01` | 技能列表占上下文窗口的比例（1%） |
| `skillOverrides` | object | - | 技能可见性覆盖。按 skill 名设置：`"on"`（全显）、`"name-only"`（仅显示名称，不加载描述）、`"user-invocable-only"`（只显示用户可调用的 skill，过滤 agent 内部 skill）、`"off"`（完全隐藏）。见 [`04-技能-skills.md`](./04-技能-skills.md#skill-可见性控制) 的完整说明 |
| `disableBundledSkills` | boolean | `false` | 隐藏内置技能 |
| `dynamicWorkflowSize` | string | - | 动态工作流默认 agent 数量的建议值：`"small"`、`"medium"`、`"large"`。通过 `/config` 的 "Workflow size" 设置。v2.1.202+/v2.1.205 规范化 |
| `alwaysThinkingEnabled` | boolean | `false` | 默认启用扩展思考 |
| `thinkingBudgetTokens` | number | - | 每次响应的固定思考 token 预算 |
| `showThinkingSummaries` | boolean | `false` | 在交互会话中显示思考摘要 |
| `advisorModel` | string | - | 顾问模式使用的模型（`opus`、`sonnet`、`fable`）|
| `fastMode` | boolean | `false` | 全局启用快速模式（所有会话使用较快模型）。用户也可通过 `/fast` 按会话切换。与 `fastModePerSessionOptIn` 配合使用 |
| `askUserQuestionTimeout` | string | `"never"` | AskUserQuestion 对话框等待超时。值：`"60s"`、`"5m"`、`"10m"`、`"never"`（默认，永不超时）。通过 `/config` 的 "Ask user question timeout" 设置。v2.1.200+ |

---

## 二、权限系统（Permissions）

### 2.1 权限模式

| 模式 | 行为 |
|------|------|
| `"default"` | 标准权限检查，需要确认 |
| `"manual"` | `"default"` 的别名，v2.1.200 引入。在 CLI、VS Code、JetBrains UI 中名称更清晰 |
| `"acceptEdits"` | 自动接受文件编辑和常见文件系统命令（v2.1.160 起对构建工具配置文件仍会提示）|
| `"dontAsk"` | 自动拒绝未预批准的调用 |
| `"bypassPermissions"` | 跳过所有权限检查（⚠️ 危险） |
| `"auto"` | 自动安全检查 + 确认（Shift+Tab 切换，v2.1.111 起默认加入循环） |
| `"plan"` | 只读探索模式（v2.1.136 起即使有 Edit allow 规则也会阻止写入） |

### 2.2 权限结构

```json
{
  "permissions": {
    "allow": ["Edit(*)", "Write(*)", "Bash(npm run *)"],
    "ask": ["Bash(rm *)", "Bash(git push *)"],
    "deny": ["Read(.env)", "Read(./secrets/**)"],
    "additionalDirectories": ["../shared-libs/"],
    "defaultMode": "acceptEdits",
    "disableBypassPermissionsMode": "disable"
  }
}
```

### 2.3 工具权限语法

| 工具 | 语法 | 示例 |
|------|------|------|
| `Bash` | `Bash(命令模式)` | `Bash(npm run *)`、`Bash(git * main)` |
| `PowerShell` | `PowerShell(cmd *)` | `PowerShell(Get-ChildItem *)` |
| `Read` | `Read(路径模式)` | `Read(.env)`、`Read(./secrets/**)` |
| `Edit` | `Edit(路径模式)` | `Edit(src/**)`、`Edit(*.ts)` |
| `Write` | `Write(路径模式)` | `Write(*.md)` |
| `WebFetch` | `WebFetch(domain:模式)` | `WebFetch(domain:example.com)` |
| `Agent` / `Task` | `Agent(agent-name)` | `Agent(researcher)`、`Agent(*)` |
| `Skill` | `Skill(名称)` | `Skill(weather-fetcher)` |
| `MCP` | `mcp__server__tool` | `mcp__memory__*` |
| `Tool` | `Tool(参数:值)` | `Agent(model:opus)` — 匹配工具的输入参数 |
| `Cd` | `Cd(路径模式)` | `Cd(/home/*)` — 控制 `/cd` 的导航范围 |

**路径前缀：**
- `//` — 文件系统绝对路径
- `~/` — 用户 home 目录
- `/` — 项目根目录
- `./` — 相对路径

**Bash 通配符说明：**
- `*` 可出现在任意位置：`Bash(* install)`、`Bash(npm *)`
- `Bash(ls *)`（空格 + `*`）匹配 `ls -la` 但不匹配 `lsof`
- 复合命令（`&&`、`||`、`|`、`;`）的每个子命令必须独立匹配

### 2.4 自动模式配置（autoMode）

```json
{
  "autoMode": {
    "classifyAllShell": false,
    "environment": ["可信环境描述"],
    "allow": ["$defaults", "例外规则"],
    "soft_deny": ["$defaults", "有条件的阻止"],
    "hard_deny": ["无条件阻止（不可覆盖）"]
  }
}
```

> ⚠️ `autoMode` 仅可在 **用户级**（`~/.claude/settings.json`）和 **托管设置** 中配置，不可在项目级或本地设置中配置，以防止仓库注入（v2.1.207 起也不再读取 `.claude/settings.local.json`）。

---

## 三、MCP 服务器设置

| 键 | 类型 | 作用域 | 说明 |
|-----|------|--------|------|
| `enableAllProjectMcpServers` | boolean | 任意 | 自动批准所有 `.mcp.json` 服务器 |
| `enabledMcpjsonServers` | array | 任意 | 白名单特定服务器名称 |
| `disabledMcpjsonServers` | array | 任意 | 黑名单特定服务器名称 |
| `allowedMcpServers` | array | 托管 | 带名称/命令/URL 匹配的白名单 |
| `deniedMcpServers` | array | 托管 | 带匹配的黑名单 |
| `allowManagedMcpServersOnly` | boolean | 托管 | 只允许托管白名单中的 MCP 服务器 |

> 详细 MCP 配置见 [`06-MCP服务器-mcp.md`](./06-MCP服务器-mcp.md)
>
> ⚠️ 保留的 MCP 服务器名（v2.1.128+）：`workspace`、`Claude Browser`、`Claude Preview`。用户定义的服务器使用这些名称会被跳过并记录警告。

### 配置关联

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| 权限模式命令行 | [`08-CLI启动参数.md`](./08-CLI启动参数-cli-startup-flags.md#-权限--安全) | `--permission-mode`、`--allowedTools` 等 CLI 参数 |
| 环境变量（完整） | [`08-CLI启动参数.md`](./08-CLI启动参数-cli-startup-flags.md#仅启动时可用的环境变量) | 仅启动时可用的环境变量 |
| MCP 服务器 | [`06-MCP服务器-mcp.md`](./06-MCP服务器-mcp.md) | MCP 配置详解和推荐服务器 |
| 权限模式实操 | [`07-记忆系统-memory.md`](./07-记忆系统-memory.md#配置优先级总结) | 配置优先级与记忆系统的关系 |

---

## 四、沙盒（Sandbox）

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": false,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["git", "docker", "gh"],
    "allowUnsandboxedCommands": true,
    "network": {
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowLocalBinding": false,
      "allowedDomains": [],
      "deniedDomains": []
    },
    "filesystem": {
      "allowWrite": [],
      "denyWrite": [],
      "denyRead": [],
      "allowRead": []
    }
  }
}
```

`sandbox.credentials` 支持两种模式：
- **`"deny"`（默认）**：阻止文件读取，从环境变量中剥离凭证
- **`"mask"`**：配合 `injectHosts` 数组，选择性向特定受信主机暴露凭证

完整配置示例参考 `best-practice/claude-settings.md`。

---

## 五、模型配置

### 模型别名

| 别名 | 说明 |
|------|------|
| `"default"` | 账户推荐模型 |
| `"sonnet"` | 最新 Sonnet（Anthropic API 上为 Sonnet 5，Bedrock/Vertex 为 Sonnet 4.6）|
| `"opus"` | 最新 Opus（Anthropic API 上为 Opus 4.8，默认快速模式）|
| `"haiku"` | 快速 Haiku |
| `"sonnet[1m]"` / `"opus[1m]"` | 1M token 上下文 |
| `"opusplan"` | Opus 规划 + Sonnet 执行 |
| `"fable"` | Claude Fable 5 — 长程推理模型（Anthropic API 专用）|

### 模型覆盖（Model Overrides）

用于将模型选择器映射到 Bedrock/Vertex/Foundry 的特定模型 ID：

```json
{
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:...:inference-profile/anthropic.claude-opus-4-6-v1:0"
  }
}
```

### 推理力度（Effort Level）

| 级别 | 说明 |
|------|------|
| Max | 最大推理深度（Opus 4.6 专用） |
| XHigh | 超强推理（Opus 4.7/4.8） |
| High | 完整推理（默认） |
| Medium | 平衡模式 |
| Low | 最少推理，最快响应 |

---

## 六、显示 & UX

| 键 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `statusLine` | object | - | 自定义状态栏（type, command, padding, refreshInterval） |
| `viewMode` | string | - | 默认视图模式：`"default"`、`"verbose"`、`"focus"` |
| `tui` | string | `"default"` | 渲染模式：`"fullscreen"` 或 `"default"` |
| `spinnerTipsEnabled` | boolean | `true` | 等待时显示提示 |
| `spinnerVerbs` | object | - | 自定义加载动词（mode: "append"/"replace", verbs: []）|
| `spinnerTipsOverride` | object | - | 自定义加载提示（tips, excludeDefault）|
| `respectGitignore` | boolean | `true` | 文件选择器遵循 .gitignore |
| `prefersReducedMotion` | boolean | `false` | 减少动画效果 |
| `axScreenReader` | boolean | `false` | 屏幕阅读器友好模式 |
| `syntaxHighlightingDisabled` | boolean | `false` | 禁用语法高亮 |
| `autoScrollEnabled` | boolean | `true` | 全屏模式自动滚动 |
| `editorMode` | string | `"normal"` | 输入模式：`"normal"` 或 `"vim"` |
| `preferredNotifChannel` | string | `"auto"` | 通知方式（terminal_bell, iterm2, kitty 等）|
| `teammateMode` | string | `"in-process"` | Agent 团队显示模式（auto, tmux, iterm2）|

---

## 七、AWS & 云凭证

| 键 | 类型 | 说明 |
|-----|------|------|
| `awsAuthRefresh` | string | AWS 认证刷新脚本 |
| `awsCredentialExport` | string | 输出 AWS 凭证 JSON 的脚本 |
| `otelHeadersHelper` | string | 动态 OpenTelemetry 头生成脚本 |

---

## 八、环境变量（通过 `env` 字段）

在 `settings.json` 的 `env` 字段中设置，避免 wrapper 脚本：

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "...",
    "NODE_ENV": "development"
  }
}
```

### 认证与 API

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_API_KEY` | API 密钥 |
| `ANTHROPIC_AUTH_TOKEN` | OAuth token |
| `ANTHROPIC_BASE_URL` | 自定义 API 端点 |
| `ANTHROPIC_MODEL` | 模型名称（覆盖 `model` 设置） |
| `CLAUDE_CODE_USE_BEDROCK` | 使用 AWS Bedrock（`1` 启用） |
| `CLAUDE_CODE_USE_VERTEX` | 使用 Google Vertex AI（`1` 启用） |
| `CLAUDE_CODE_USE_FOUNDRY` | 使用 Microsoft Foundry（`1` 启用） |
| `ANTHROPIC_BEDROCK_SERVICE_TIER` | Bedrock 服务层级（default, flex, priority）|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude.ai OAuth 访问 token |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | Claude.ai OAuth 刷新 token |

### 模型与推理

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子代理模型覆盖 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 推理力度（low/medium/high/xhigh/max） |
| `MAX_THINKING_TOKENS` | 最大思考 token 数 |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 禁用 1M token 上下文（`1` 禁用）|
| `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | 在所有模型上强制启用 effort 参数 |

### 上下文与压缩

| 变量 | 说明 |
|------|------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 自动压缩阈值百分比（1-100），默认 ~95% |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 自动压缩计算用的上下文容量（token）|
| `DISABLE_AUTO_COMPACT` | 禁用自动压缩 |
| `DISABLE_COMPACT` | 禁用所有压缩（包括手动 `/compact`）|
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | 覆盖模型上下文窗口大小 |

### 超时与输出限制

| 变量 | 说明 |
|------|------|
| `API_TIMEOUT_MS` | API 请求超时（默认 600000ms） |
| `BASH_MAX_TIMEOUT_MS` | Bash 命令超时 |
| `BASH_MAX_OUTPUT_LENGTH` | Bash 输出最大长度 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 每次响应最大输出 token（默认 32000，Opus 4.6 为 64000）|
| `CLAUDE_CODE_MAX_RETRIES` | API 请求重试次数（默认 10）|
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | 最大并行只读工具数（默认 10）|
| `MCP_TOOL_TIMEOUT` | MCP 工具执行超时 |

### MCP 与工具

| 变量 | 说明 |
|------|------|
| `MCP_TIMEOUT` | MCP 启动超时 |
| `MAX_MCP_OUTPUT_TOKENS` | MCP 输出 token 上限（默认 25000）|
| `ENABLE_TOOL_SEARCH` | MCP 工具搜索阈值 |
| `CLAUDE_CODE_DISABLE_MCP` | 禁用所有 MCP 服务器（`1` 禁用）|
| `CLAUDE_CODE_MCP_ALLOWLIST_ENV` | 使用安全基线环境运行 stdio MCP 服务器 |

### 显示与终端

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_NO_FLICKER` | 启用无闪烁全屏渲染 |
| `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` | 禁用全屏渲染 |
| `CLAUDE_CODE_ACCESSIBILITY` | 保持终端光标可见（辅助功能）|
| `CLAUDE_CODE_AX_SCREEN_READER` | 屏幕阅读器友好输出 |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | 禁用 diff 语法高亮（`0` 禁用）|
| `CLAUDE_CODE_HIDE_CWD` | 隐藏启动横幅中的当前目录 |
| `CLAUDE_CODE_TMUX_TRUECOLOR` | 在 tmux 中启用真彩色 |
| `CLAUDE_CODE_DISABLE_MOUSE` | 禁用鼠标跟踪 |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | 禁用终端标题更新 |

### 特性开关

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | 禁用文件检查点 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务 |
| `CLAUDE_CODE_DISABLE_AGENT_VIEW` | 禁用后台 agent |
| `CLAUDE_CODE_DISABLE_WORKFLOWS` | 禁用动态工作流 |
| `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` | 禁用内置技能 |
| `CLAUDE_CODE_DISABLE_ARTIFACT` | 禁用 Artifact 发布工具 |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | 禁用 git 系统提示 |
| `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` | 禁用顾问模式 |
| `DISABLE_TELEMETRY` / `DO_NOT_TRACK` | 禁用遥测 |
| `DISABLE_ERROR_REPORTING` | 禁用错误报告 |
| `DISABLE_AUTOUPDATER` | 禁用自动更新检查 |

### 开发者与调试

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | 调试日志文件路径 |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | 调试日志级别 |
| `CLAUDE_CODE_EXTRA_BODY` | 注入到每个 API 请求体的 JSON 对象 |
| `CLAUDE_CODE_PROPAGATE_TRACEPARENT` | 传播 W3C traceparent 头 |
| `CLAUDE_CODE_SESSION_ID` | 只读，当前会话 ID |
| `CLAUDE_CODE_CHILD_SESSION` | 在子进程中设为 `1` |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | 跳过写入提示历史和会话记录 |

### 网络与代理

| 变量 | 说明 |
|------|------|
| `HTTP_PROXY` / `HTTPS_PROXY` | HTTP/HTTPS 代理 URL |
| `NO_PROXY` | 绕过代理的主机列表 |
| `CLAUDE_CODE_CLIENT_CERT` | mTLS 客户端证书路径 |
| `CLAUDE_CODE_CLIENT_KEY` | mTLS 客户端密钥路径 |
| `CLAUDE_CODE_CERT_STORE` | CA 证书源（`bundled`, `system`）|

### 插件与 Marketplace

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | 插件 git clone 超时 |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | 插件缓存目录 |
| `CLAUDE_CODE_PLUGIN_PREFER_HTTPS` | 使用 HTTPS 而非 SSH 克隆插件 |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 只读插件种子目录 |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | 启用 Claude.ai MCP 服务器 |

---

## 九、一个完整配置示例

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "advisorModel": "fable",
  "language": "english",
  "outputStyle": "Explanatory",
  "alwaysThinkingEnabled": true,
  "showThinkingSummaries": true,
  "tui": "fullscreen",
  "plansDirectory": "./plans",

  "worktree": {
    "symlinkDirectories": ["node_modules"],
    "sparsePaths": ["packages/my-app"],
    "baseRef": "fresh"
  },

  "skillOverrides": {
    "legacy-context": "name-only"
  },

  "permissions": {
    "allow": [
      "Edit(*)",
      "Write(*)",
      "Bash(npm run *)",
      "Bash(git *)",
      "WebFetch(domain:*)",
      "mcp__*",
      "Agent(*)"
    ],
    "deny": ["Read(.env)", "Read(./secrets/**)"],
    "additionalDirectories": ["../shared/"],
    "defaultMode": "acceptEdits"
  },

  "enableAllProjectMcpServers": true,

  "attribution": {
    "commit": "Co-Authored-By: Claude <noreply@anthropic.com>",
    "pr": ""
  },

  "statusLine": {
    "type": "command",
    "command": "git branch --show-current"
  },

  "spinnerTipsEnabled": true,
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["分析代码", "思考方案", "优化性能"]
  },

  "env": {
    "NODE_ENV": "development",
    "CLAUDE_CODE_EFFORT_LEVEL": "high",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "80"
  }
}
```

---

## 十、常用相关命令

| 命令 | 说明 |
|------|------|
| `/config [key=value]` | 交互式配置界面（v2.1.181+ 支持直接传参） |
| `/model [model]` | 切换模型并调整推理力度 |
| `/effort [level]` | 直接设置推理力度 |
| `/permissions` | 管理权限规则 |
| `/statusline` | 配置状态栏 |
| `/theme` | 切换颜色主题 |
| `/sandbox` | 切换沙盒模式 |
| `/doctor` | 诊断配置问题 |

---

> 下一篇：[06-MCP服务器-mcp.md](./06-MCP服务器-mcp.md)
