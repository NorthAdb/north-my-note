# claude-code-best-practice 实践指南

> 基于 [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
> "from vibe coding to agentic engineering — practice makes claude perfect"

---

## 中文文档体系

本仓库自带全套中文学习文档（位于 `doc/` 目录），请**按以下顺序阅读**：

| 序号 | 文档 | 内容 | 建议时间 |
|------|------|------|----------|
| 1 | [doc/01-CONCEPTS-概念总览.md](../doc/01-CONCEPTS-概念总览.md) | Claude Code 所有核心概念的中文解释 | Day 1 |
| 2 | [doc/02-子代理-subagents.md](../doc/02-子代理-subagents.md) | Subagent 定义、字段、最佳实践 | Day 2 |
| 3 | [doc/03-命令-commands.md](../doc/03-命令-commands.md) | Command 定义、85 个系统命令列表 | Day 3 |
| 4 | [doc/04-技能-skills.md](../doc/04-技能-skills.md) | Skill 定义、设计原则、两种模式 | Day 3 |
| 5 | [doc/05-设置-settings.md](../doc/05-设置-settings.md) | 完整配置项说明（权限、模型、显示等） | Day 4 |
| 6 | [doc/06-MCP服务器-mcp.md](../doc/06-MCP服务器-mcp.md) | MCP 配置、推荐服务器、权限规则 | Day 5 |
| 7 | [doc/07-记忆系统-memory.md](../doc/07-记忆系统-memory.md) | 三层记忆体系（CLAUDE.md + rules + Auto Memory） | Day 5 |
| 8 | [doc/08-CLI启动参数-cli-startup-flags.md](../doc/08-CLI启动参数-cli-startup-flags.md) | 启动参数速查表 | Day 6 |
| 9 | [doc/09-能力强化-power-ups.md](../doc/09-能力强化-power-ups.md) | 10 个交互式教学课程 | Day 6 |
| 10 | [doc/10-学习路径.md](../doc/10-学习路径.md) | 从入门到精通的详细路线图 | Day 7 |

---

## 这个项目是什么

**它不是教程，是一本 Claude Code 的"使用说明书 + 配置宝典"。**

你可以把它理解为：拿到一把瑞士军刀，这个项目告诉你每一把刀片是干什么的、什么时候用、怎么组合最顺手。所有内容都可以直接用 Claude Code 打开后让 Agent 帮你读、帮你配。

---

## 这个仓库的核心架构

```
Command → Agent → Skill
```

这是这个仓库演示的最核心模式：

1. **Command**（入口）：你输入 `/weather-orchestrator`
2. **Agent**（执行者）：weather-agent 获取温度数据
3. **Skill**（知识包）：weather-fetcher 知道怎么调用天气 API，weather-svg-creator 知道怎么生成图片

每一层各司其职，组合成一个完整的工作流。

---

## 前置条件

```bash
# 安装 Claude Code（如果还没有）
npm install -g @anthropic-ai/claude-code

# 需要一个 Claude Pro/Max 订阅 或者 Anthropic API Key
# Claude Pro: $20/月（推荐入门）
# API Key: 按量付费（重度使用更划算）

# 验证安装
claude --version
```

---

## 四步上手

### Step 1 — 克隆仓库

```bash
git clone https://github.com/shanraisshan/claude-code-best-practice
cd claude-code-best-practice
```

### Step 2 — 阅读中文文档

按上面表格的顺序，从 [doc/01-CONCEPTS-概念总览.md](../doc/01-CONCEPTS-概念总览.md) 开始，每天 1-2 篇。

### Step 3 — 运行第一个工作流

```bash
claude
/weather-orchestrator
```

观察 **Command → Agent → Skill** 三级架构如何串联。

### Step 4 — 观看 `/powerup` 课程

```bash
/powerup
```

从第 1 课开始，10 个交互课程过一遍。

---

## 核心配置

### settings.json（位于 `.claude/settings.json`）

Claude Code 的"控制面板"，控制：模型选择、权限规则、MCP 服务器、显示选项等。

### CLAUDE.md（位于项目根目录）

Claude 的"入职手册"——每次会话启动时自动加载，告诉 Claude 项目的技术栈、代码风格、架构约定。

### 配置优先级

```
托管设置（组织策略，不可覆盖）
  ↓
命令行参数（单次会话）
  ↓
.claude/settings.local.json（个人项目，不提交 git）
  ↓
.claude/settings.json（团队共享）
  ↓
~/.claude/settings.json（全局默认）
```

---

## 配置到你自己的项目

```bash
# 在 Claude Code 中运行
"帮我把这个项目的 .claude/settings.json 和 CLAUDE.md
根据我的实际情况适配一份，放到我当前项目目录下。
不要直接复制，要理解每个配置项的含义后给我推荐。"
```

---

## 学习路线（4 周）

### 第 1 周：建立基础

| 天数 | 学习内容 | 练习 |
|------|----------|------|
| Day 1 | 阅读 [CONCEPTS 概念总览](../doc/01-CONCEPTS-概念总览.md) | 理解五大概念 |
| Day 2 | 阅读 [子代理](../doc/02-子代理-subagents.md) | 查看官方 agent 类型 |
| Day 3 | 阅读 [命令](../doc/03-命令-commands.md) + [技能](../doc/04-技能-skills.md) | 区分三者的不同 |
| Day 4 | 阅读 [设置](../doc/05-设置-settings.md) | 查看自己项目的 settings.json |
| Day 5 | 运行 `/weather-orchestrator` | 观察 Command→Agent→Skill |
| Day 6 | 运行 `/powerup` | 10 个交互课程 |
| Day 7 | 配置自己的 CLAUDE.md | 让 Claude 帮你生成 |

### 第 2 周：深入实践

| 天数 | 学习内容 | 练习 |
|------|----------|------|
| Day 8 | 阅读 [MCP 服务器](../doc/06-MCP服务器-mcp.md) | 配置 Context7 |
| Day 9 | 阅读 [记忆系统](../doc/07-记忆系统-memory.md) | 理解三层记忆 |
| Day 10 | 创建 Subagent | 创建一个 code-reviewer |
| Day 11 | 创建 Skill | 记录项目 API 设计模式 |
| Day 12 | 创建 Command | 创建 `/deploy` |
| Day 13 | 配置权限 | 使用 `/permissions` |
| Day 14 | 阅读 [CLI 启动参数](../doc/08-CLI启动参数-cli-startup-flags.md) | 学会 `-p` 一次性用法 |

### 第 3 周：进阶技巧

| 天数 | 学习内容 |
|------|----------|
| Day 15-16 | Agent Teams 配置 |
| Day 17-18 | 使用 `ultracode` 关键词触发动态工作流 |
| Day 19 | 定时任务 `/loop` |
| Day 20 | Remote Control 远程控制 |
| Day 21 | Goal 模式 |

### 第 4 周+：社区工作流

| 工作流 | ⭐ | 特点 |
|--------|---|------|
| [Superpowers](https://github.com/obra/superpowers) | 233k | 完整开发流程 |
| [ECC](https://github.com/affaan-m/everything-claude-code) | 218k | 全功能套件 |
| [SpecKit](https://github.com/github/spec-kit) | 114k | 需求到代码 |
| [gstack](https://github.com/garrytan/gstack) | 112k | 规划到部署 |

---

## 目录速查

| 目录 | 中文说明 |
|------|----------|
| `.claude/settings.json` | 项目配置文件 |
| `.claude/agents/*.md` | 子代理定义 |
| `.claude/commands/*.md` | 斜杠命令定义 |
| `.claude/skills/*/SKILL.md` | 技能定义 |
| `.claude/hooks/` | 钩子系统 |
| `.claude/rules/*.md` | 规则文件 |
| `best-practice/` | 最佳实践文档（英文原版） |
| `doc/` | **中文翻译文档** |
| `implementation/` | 实现参考 |
| `reports/` | 深度报告 |
| `tips/` | 技巧合集 |
| `orchestration-workflow/` | 编排工作流示例（天气系统） |
| `development-workflows/` | 社区工作流对比 |

---

## 常用命令速查

| 场景 | 命令 |
|------|------|
| 交互模式 | `claude` |
| 一次性任务 | `claude -p "..."` |
| 指定模型 | `claude --model opus` |
| 恢复会话 | `claude -r` |
| 自动模式 | `claude --permission-mode auto` |
| 后台运行 | `claude --bg "..."` |
| 隔离工作树 | `claude -w` |
| 诊断问题 | `/doctor` |

---

## 日常好习惯

1. **每天更新**：`claude update`
2. **早晨看更新日志**：阅读 [Changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
3. **~50% 上下文时手动 `/compact`**：比自动 compact 效果更好
4. **新任务 = 新会话**：不要让不同任务混在一个会话中
5. **复杂任务先从 Plan 开始**：先规划再执行
6. **遇见问题立刻截图**：分享给 Claude 让它分析

---

## 福利：操作速查

| 操作 | 效果 |
|------|------|
| `双按 Esc` | 撤销到上一步 |
| `Shift+Tab` | 循环切换权限模式（Ask → Plan → Auto） |
| `/rewind` | 回退到指定检查点 |
| `/compact` | 压缩上下文释放空间 |
| `/clear` | 清空当前会话（新任务用） |
| `/bg` | 当前会话转后台 |
| `@文件名` | 在提示中引用文件 |

---

> "from vibe coding to agentic engineering — practice makes claude perfect"
>
> 别把 Claude 当聊天机器人——学会用 Agent + Command + Skill 构建你的开发工作流。
