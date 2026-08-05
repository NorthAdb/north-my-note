# Claude Code 概念总览

> 中文翻译自 README.md 的 CONCEPTS 表格
> 学习顺序：第 1 篇

> 📋 **翻译追踪**：完整翻译进度和缺口分析见 [`doc/翻译进度与文档缺口分析.md`](../翻译进度与文档缺口分析.md)

---

## 项目结构总览

以下是本仓库（`shanraisshan/claude-code-best-practice`）的完整目录结构，展示了 Claude Code 各项能力的实际配置方式：

```
claude-code-best-practice/
│
├── .claude/                          # ★ Claude Code 核心配置目录
│   ├── agents/                       #   子代理定义（Subagents）
│   │   ├── weather-agent.md          │   └ 示例：天气查询 Agent
│   │   ├── time-agent.md             │   └ 示例：时间查询 Agent
│   │   ├── presentation-*.md         │   └ 演示文稿专用 Agent（×3）
│   │   ├── development-workflows-*   │   └ 研究工作流 Agent
│   │   └── workflows/                │   └ 工作流专用 Agent 子目录
│   ├── agent-memory/                #   子代理的记忆存储
│   │   └── weather-agent/           │   └ 天气 Agent 的记忆
│   ├── commands/                    #   斜杠命令（/command）
│   │   ├── weather-orchestrator.md  │   └ 示例：/weather-orchestrator
│   │   ├── time-command.md          │   └ 示例：/time-command
│   │   └── workflows/               │   └ 工作流命令子目录
│   ├── hooks/                       #   钩子系统（25+ 种事件）
│   │   ├── config/                  │   └ hooks-config.json 配置
│   │   ├── scripts/                 │   └ hooks.py 主处理脚本
│   │   ├── sounds/                  │   └ 各事件音效（40+ 种）
│   │   └── HOOKS-README.md          │   └ 钩子使用说明
│   ├── rules/                       #   规则文件（按路径选择性加载）
│   │   ├── markdown-docs.md         │   └ Markdown 文档规范
│   │   └── presentation.md          │   └ 演示文稿工作规范
│   ├── skills/                      #   ★ Skills 技能库（可复用的领域知识包）
│   │   ├── weather-fetcher/         │   └ 【官方】天气数据获取
│   │   ├── weather-svg-creator/     │   └ 【官方】天气 SVG 生成
│   │   ├── agent-browser/           │   └ 【安装】浏览器自动化
│   │   ├── presentation/            │   └ 【安装】演示文稿
│   │   ├── time-skill/              │   └ 【安装】时间显示
│   │   ├── web-access/              │   └ 【安装】联网操作
│   │   ├── ask-matt/                │   └ 【Matt Pocock】AI 问答
│   │   ├── code-review/             │   └ 【Matt Pocock】代码审查
│   │   ├── codebase-design/         │   └ 【Matt Pocock】代码库设计
│   │   ├── diagnosing-bugs/         │   └ 【Matt Pocock】Bug 诊断
│   │   ├── domain-modeling/         │   └ 【Matt Pocock】领域建模
│   │   ├── tdd/                     │   └ 【Matt Pocock】测试驱动开发
│   │   ├── implement/               │   └ 【Matt Pocock】功能实现
│   │   ├── handoff/                 │   └ 【Matt Pocock】任务交接
│   │   ├── grilling/                │   └ 【Matt Pocock】方案质疑
│   │   ├── teach/                   │   └ 【Matt Pocock】教学
│   │   ├── ...（更多 skills）        │   └ 共 35+ 个 Matt Pocock skills
│   │   └── setup-*/                 │   └ 环境配置类 skills
│   ├── settings.json                #   ★ 项目级核心配置
│   └── settings.local.json          #   个人项目配置（git-ignored）
│
├── .agents/                          # ★ 子代理专用 Skills（镜像）
│   └── skills/                      #   与 .claude/skills/ 内容一致
│       ├── code-review/             │   └ 专供子代理使用的 skill 副本
│       ├── diagnosing-bugs/         │
│       └── ...（35+ skills）         │   └ 双目录模式：一个供主会话，一个供子代理
│
├── .github/                          # GitHub 配置
│   └── FUNDING.yml
│
├── best-practice/                    # ★ 最佳实践文档
│   ├── claude-subagents.md          │   └ 子代理最佳实践
│   ├── claude-commands.md           │   └ 命令最佳实践
│   ├── claude-skills.md             │   └ 技能最佳实践
│   ├── claude-settings.md           │   └ 设置最佳实践
│   ├── claude-mcp.md                │   └ MCP 最佳实践
│   ├── claude-memory.md             │   └ 记忆系统最佳实践
│   ├── claude-power-ups.md          │   └ 能力强化最佳实践
│   ├── claude-cli-startup-flags.md  │   └ CLI 启动参数最佳实践
│   └── assets/                      │   └ 文档用图
│
├── implementation/                   # ★ 实现参考文档
│   ├── claude-subagents-implementation.md
│   ├── claude-commands-implementation.md
│   ├── claude-skills-implementation.md
│   ├── claude-agent-teams-implementation.md
│   ├── claude-goal-implementation.md
│   ├── claude-scheduled-tasks-implementation.md
│   └── assets/
│
├── reports/                          # ★ 特性研究报告
│   ├── claude-advanced-tool-use.md
│   ├── claude-agent-command-skill.md
│   ├── claude-agent-memory.md
│   ├── claude-agent-sdk-vs-cli-system-prompts.md
│   ├── claude-global-vs-project-settings.md
│   ├── claude-skills-for-larger-mono-repos.md
│   ├── claude-in-chrome-v-chrome-devtools-mcp.md
│   ├── claude-spinner-verbs-and-tips.md
│   ├── claude-usage-and-rate-limits.md
│   ├── why-harness-is-important.md
│   └── learning-journey-weather-reporter-redesign.md
│
├── tips/                             # ★ 使用技巧
│   ├── claude-boris-*               │   └ Boris 的 Claude Code 技巧
│   ├── claude-thariq-*              │   └ Thariq 的 Claude Code 技巧
│   └── assets/
│
├── doc/                              # ★ 中文学习资料
│   ├── practice/                    │   └ 实践系列（10 篇，从概念到实践）
│   │   ├── 01-CONCEPTS-概念总览.md   │
│   │   ├── 02-子代理-subagents.md    │
│   │   ├── 03-命令-commands.md       │
│   │   ├── 04-技能-skills.md         │
│   │   ├── 05-设置-settings.md       │
│   │   ├── 06-MCP服务器-mcp.md       │
│   │   ├── 07-记忆系统-memory.md     │
│   │   ├── 08-CLI启动参数.md         │
│   │   ├── 09-能力强化-power-ups.md  │
│   │   └── 10-学习路径.md            │
│   ├── learn/                       │   └ 学习笔记
│   ├── implementation/              │   └ 实现记录
│   └── collected/                   │   └ 资料收集
│
├── orchestration-workflow/           # ★ 编排工作流演示
│   ├── orchestration-workflow.md    │   └ 流程图解
│   ├── orchestration-workflow.svg   │   └ 流程 SVG
│   ├── weather.svg                  │   └ 生成的天气卡片
│   └── output.md                    │   └ 工作流输出
│
├── development-workflows/            # 开发工作流研究
│   ├── best-practice/
│   ├── development-workflows/
│   ├── cross-model-workflow/
│   └── rpi/
│
├── agent-teams/                      # Agent Teams 研究
│   ├── agent-teams-prompt.md        │
│   └── output/                      │
│
├── presentation/                     # ★ 演示文稿
│   ├── claude-code-best-practice/   │   └ Claude Code 最佳实践
│   ├── vibe-coding-to-agentic-*/    │   └ Vibe Coding → Agentic 工程
│   └── 2026-04-25-gdg-kolachi-*/   │   └ GDG 演讲：Claude + Gemini
│
├── changelog/                        # 变更日志
│   ├── best-practice/
│   ├── development-workflows/
│   ├── skill-collections/
│   └── agent-collections/
│
├── feishu-doc/                       # 飞书文档备份
│   └── java-backend-interview-*.md
│
├── tutorial/                         # 教程
│   ├── day0/
│   └── day1/
│
├── videos/                           # 视频 / 播客资料
│   └── claude-*.md                  │   └ Boris、Karpathy、Matt Pocock 等
│
│─── !/                               # 媒体资源
│   ├── root/
│   ├── tags/
│   └── thumbnail/
│
├── CLAUDE.md                         # ★ 项目级记忆（AI 入职手册）
├── README.md                         # 主 README
├── README.html                       # README 的 HTML 版本
├── skills-lock.json                  # Skill 版本锁定
├── .mcp.json                         # MCP 服务器配置
├── .gitignore
└── LICENSE
```

### 目录说明

| 目录 | 用途 | 来源 |
|------|------|------|
| `.claude/` | Claude Code 核心配置（agents, commands, hooks, rules, skills, settings） | 核心 |
| `.agents/` | 子代理专用 skills 镜像（与 `.claude/skills/` 内容一致） | 核心 |
| `best-practice/` | Claude Code 各特性的最佳实践指南 | 文档 |
| `implementation/` | 各特性的实现参考配置 | 文档 |
| `reports/` | 深度研究报告（对比分析、特性评估） | 文档 |
| `tips/` | 社区专家使用技巧 | 文档 |
| `doc/practice/` | 中文实践系列教程（10 篇） | 中文 |
| `orchestration-workflow/` | Weather 编排工作流演示 | 演示 |
| `presentation/` | 多场演讲的演示文稿 | 演示 |
| `changelog/` | 各模块变更追踪 | 运维 |
| `development-workflows/` | 开发工作流研究 | 研究 |
| `agent-teams/` | Agent Teams 研究 | 研究 |

---

---

## 核心概念

Claude Code 提供了一套可扩展的框架，包含以下核心概念：

### 1. Subagents（子代理）`🤖`

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/agents/<名称>.md` |
| **是什么** | 独立的迷你 AI 助手，拥有自己的上下文窗口，可以在后台并行工作 |
| **最佳实践** | `best-practice/claude-subagents.md` |
| **实现参考** | `implementation/claude-subagents-implementation.md` |
| **官方文档** | [Sub-agents](https://code.claude.com/docs/en/sub-agents) |

**通俗理解：** Subagent 就像你雇了多个实习生，每人负责一个特定任务（代码审查、写测试、查文档），各自独立工作不干扰你。他们干完活把结果汇报给你。

**关键特性：**
- 每个 Subagent 有独立的上下文窗口
- 可以通过 `maxTurns` 控制最大对话轮数
- 可以预加载 Skills（技能）
- 支持 `isolation: "worktree"` 在隔离的 git worktree 中运行
- 支持 `background: true` 在后台运行

---

### 2. Commands（命令）`/`

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/commands/<名称>.md` |
| **是什么** | 可复用的斜杠命令，快速执行常用工作流 |
| **最佳实践** | `best-practice/claude-commands.md` |
| **实现参考** | `implementation/claude-commands-implementation.md` |
| **官方文档** | [Slash Commands](https://code.claude.com/docs/en/slash-commands) |

**通俗理解：** 就像游戏里的快捷键。你每天要重复做的事（比如"帮我审查代码"），定义一个命令 `/code-review` 一键执行。

**关键特性：**
- 支持参数（`arguments`）传递
- 支持 `context: fork` 在独立子代理中运行
- 可以限制只在特定文件路径下激活（`paths`）
- 系统内置 85+ 个官方命令

---

### 3. Skills（技能）`📋`

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/skills/<名称>/SKILL.md` |
| **是什么** | 可复用的领域知识包，按需加载到 Claude 的提示中 |
| **最佳实践** | `best-practice/claude-skills.md` |
| **实现参考** | `implementation/claude-skills-implementation.md` |
| **官方文档** | [Skills](https://code.claude.com/docs/en/skills) |
| **官方 Skills 仓库** | [anthropics/skills](https://github.com/anthropics/skills/tree/main/skills) |
| **Monorepo 报告** | `reports/claude-skills-for-larger-mono-repos.md` |

**通俗理解：** Skill 就像一本"专业领域手册"。当 Claude 需要做某个特定任务时，你把对应的手册给它看，它就知道该怎么做。用完就放下，不占脑子。

**关键特性：**
- 支持 `context: fork` 在隔离的子代理中执行
- 支持 `allowed-tools` 在白名单中免确认使用工具
- 可以包含子目录（`references/`、`scripts/`、`examples/`）
- 内置 10 个官方 Skill

---

### 4. Workflows（工作流）

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/commands/weather-orchestrator.md`（示例） |
| **是什么** | 将 Command → Agent → Skill 串联起来的完整工作流 |
| **官方文档** | [Common Workflows](https://code.claude.com/docs/en/common-workflows) |

**通俗理解：** 把多个工具组合成一个自动化流水线。比如：输入一个命令 → 启动一个 Agent 去查天气 → 调用一个 Skill 生成天气卡片。

这个仓库的示例：`/weather-orchestrator` 演示了完整的 **Command → Agent → Skill** 三级架构。

---

### 5. Hooks（钩子）

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/hooks/` |
| **是什么** | 在特定事件发生时自动触发的脚本 |
| **最佳实践** | [Claude Code Hooks 仓库](https://github.com/shanraisshan/claude-code-hooks) |
| **官方文档** | [Hooks](https://code.claude.com/docs/en/hooks) / [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) |

**通俗理解：** 就像"当...时，做..."的自动化规则。比如"当 Claude 写完代码后，自动格式化"、"当会话开始时，自动拉取最新代码"。

**支持的事件（25+ 种）：** PreToolUse、PostToolUse、UserPromptSubmit、SessionStart、SessionEnd、Stop 等。

**三种类型：**
- **Command hooks** — 运行 shell 命令
- **HTTP hooks** — 发送 HTTP 请求
- **On-demand hooks** — 在 Skill 中按需触发

---

### 6. MCP Servers（MCP 服务器）

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/settings.json`、`.mcp.json` |
| **是什么** | 通过 Model Context Protocol 连接外部工具、数据库和 API |
| **最佳实践** | `best-practice/claude-mcp.md` |
| **实现参考** | `.mcp.json` |
| **官方文档** | [MCP](https://code.claude.com/docs/en/mcp) |

**通俗理解：** MCP 是 Claude Code 的"USB 接口"——想用什么外设就插什么。需要浏览器自动化？插 Playwright MCP。需要查文档？插 Context7 MCP。

**常用 MCP 服务器：**
- **Context7** — 获取最新的库文档，防止幻觉 API
- **Playwright** — 浏览器自动化（截图、导航、表单测试）
- **Claude in Chrome** — 连接真实 Chrome 浏览器，查看控制台日志
- **DeepWiki** — 获取任意 GitHub 仓库的结构化文档

---

### 7. Plugins（插件）

| 项目 | 说明 |
|------|------|
| **位置** | 可分发的软件包 |
| **是什么** | 可安装的 Claude Code 扩展包 |
| **官方文档** | [Plugins](https://code.claude.com/docs/en/plugins) |

**Marketplace：** 可以创建和共享插件市场。

---

### 8. Settings（设置）

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/settings.json` |
| **是什么** | Claude Code 的全局和项目级配置 |
| **最佳实践** | `best-practice/claude-settings.md` |
| **实现参考** | `.claude/settings.json` |
| **官方文档** | [Settings](https://code.claude.com/docs/en/settings) |

**涵盖范围：** 权限控制、模型配置、显示/UX、Sandbox、MCP 服务器、环境变量等。

---

### 9. Status Line（状态栏）

| 项目 | 说明 |
|------|------|
| **位置** | `.claude/settings.json` |
| **是什么** | 自定义终端底部状态栏，显示当前上下文、模型、成本等信息 |
| **最佳实践** | [claude-code-status-line](https://github.com/shanraisshan/claude-code-status-line) |
| **实现参考** | `.claude/settings.json` |
| **官方文档** | [Status Line](https://code.claude.com/docs/en/statusline) |

---

### 10. Memory（记忆系统）

| 项目 | 说明 |
|------|------|
| **位置** | `CLAUDE.md`、`.claude/rules/`、`~/.claude/rules/`、`~/.claude/projects/<项目>/memory/` |
| **是什么** | Claude Code 的持久化记忆系统 |
| **最佳实践** | `best-practice/claude-memory.md` |
| **实现参考** | `CLAUDE.md` |
| **官方文档** | [Memory](https://code.claude.com/docs/en/memory) |

**三层记忆体系：**
1. **CLAUDE.md** — 项目级记忆（相当于项目的"入职手册"）
2. **.claude/rules/** — 规则级记忆（按路径选择性加载）
3. **Auto Memory** — 自动记忆（从对话中自动学习）

---

### 11. Checkpointing（检查点）

| 项目 | 说明 |
|------|------|
| **是什么** | 自动跟踪文件编辑历史，可以随时回退 |
| **官方文档** | [Checkpointing](https://code.claude.com/docs/en/checkpointing) |

每次编辑文件前自动创建快照，可以用 `/rewind` 回退到任意历史状态。

---

### 12. CLI Startup Flags（CLI 启动参数）

| 项目 | 说明 |
|------|------|
| **最佳实践** | `best-practice/claude-cli-startup-flags.md` |
| **官方文档** | [CLI Reference](https://code.claude.com/docs/en/cli-reference) |

涵盖：会话管理、模型配置、权限安全、输出格式、系统提示、代理、MCP、工作目录、预算限制等。

**常用示例：**
```bash
claude                          # 交互模式
claude -p "修复所有 TS 错误"      # 一次性任务
claude --model opus              # 指定模型
claude --permission-mode auto    # 自动权限模式
```

---

## 热门功能（🔥 Hot 表格）

| 功能 | 说明 |
|------|------|
| **Ultrareview** | 深度多代理代码审查（`/code-review ultra`） |
| **Devcontainers** | 开发容器配置 |
| **Channels** | 频道功能（Beta） |
| **Ultraplan** | 深度规划模式（Beta） |
| **No Flicker Mode** | 无闪烁全屏渲染 |
| **Auto Mode** | 自动权限模式，Shift+Tab 切换 |
| **Power-ups** | 交互式功能教学（`/powerup`，v2.1.218+ 已移除） |
| **Fast Mode** | 快速模式（`/fast`） |
| **Advisor** | 顾问模式，调用第二个模型提供建议 |
| **Computer Use** | Claude 操作电脑（Beta） |
| **Agent SDK** | 构建自定义 Agent 的 SDK |
| **Ralph Wiggum Loop** | 长期自主运行循环 |
| **Chrome 集成** | 浏览器自动化 |
| **Claude Code Web** | 浏览器版 Claude Code |
| **Slack 集成** | 在 Slack 中使用 @Claude |
| **Code Review** | GitHub App 代码审查 |
| **GitHub Actions** | CI/CD 集成 |
| **Remote Control** | 远程控制（`/remote-control`） |
| **Dynamic Workflows** | 动态工作流（`ultracode` 关键词触发） |
| **Agent Teams** | 多 Agent 团队协作 |
| **Agent View** | 后台 Agent 管理 |
| **Scheduled Tasks** | 定时任务（`/loop`、`/schedule`） |
| **Routines** | 日常任务自动化 |
| **Tasks** | 任务跟踪系统 |
| **Goal** | 持续工作模式（`/goal`） |
| **Voice Dictation** | 语音输入（`/voice`） |
| **Bundled Skills** | 内置 Skill（如 `/code-review`、`/batch`） |
| **Git Worktrees** | 隔离的 Git 工作树 |

---

> 下一篇：[02-子代理-subagents.md](./02-子代理-subagents.md)
