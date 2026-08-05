# Skills 全览

> 本仓库所有 Skills、Agents、Commands 的完整梳理。
>
> 快捷跳转：[项目原生](#-项目原生-skills) · [Mattpocock Engineering](#-工程-engineering) · [Mattpocock Productivity](#-生产力-productivity) · [Mattpocock Misc](#-工具-misc) · [Mattpocock Personal](#-个人-personal) · [Mattpocock In-Progress](#-实验中-in-progress) · [全局安装](#-全局安装-skills) · [内置 Skills](#-claude-code-内置-skills) · [Agents](#-agents) · [Commands](#-commands)

---

## 安装来源统计

| 来源 | 数量 |
|------|:----:|
| 项目原生（手工编写） | 7 |
| `mattpocock/skills` | 31 |
| 全局安装（来自其他源） | 2 |
| Claude Code 内置 | 12+ |
| **Agents** | 11 |
| **Commands** | 10 |
| **总计** | **73+** |

---

## 📦 项目原生 Skills

项目自己编写、不来自任何包的技能。

| Skill | 调用方式 | 功能说明 | 属性 |
|-------|----------|----------|------|
| **agent-browser** | 上下文触发或 `/agent-browser` | 浏览器自动化 CLI，可导航页面、填写表单、截图、提取数据、测试 web 应用 | `allowed-tools: Bash(agent-browser:*)` |
| **time-skill** | 上下文触发或 `/time-skill` | 显示巴基斯坦标准时间（PKT, UTC+5） | `user-invocable: true` |
| **weather-fetcher** | 被 weather-agent 调用（非用户直接调用） | 从 Open-Meteo API 获取迪拜实时温度 | `user-invocable: false` |
| **weather-svg-creator** | 上下文触发或 `/weather-svg-creator` | 创建 SVG 天气卡片，写入 `orchestration-workflow/weather.svg` 和 `orchestration-workflow/output.md` | — |
| **vibe-to-agentic-framework** | 自动加载 | Vibe Coding → Agentic Engineering 演讲的概念框架 — 每张幻灯片如何融入叙事弧 | 后台知识 skill |
| **presentation-structure** | 自动加载 | 演示文稿的幻灯片格式、weight 系统、导航和章节结构知识 | 后台知识 skill |
| **presentation-styling** | 自动加载 | 演示文稿的 CSS 类、组件模式和语法高亮知识 | 后台知识 skill |

---

## 🛠 工程 (Engineering) — mattpocock/skills

来自 `skills/engineering/` 目录。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/code-review`** | `/code-review` 或上下文触发 | **双轴代码审查**：同时从 Standards（是否遵循编码标准）和 Spec（是否实现需求）两个维度审查变更。并行 sub-agent 执行 |
| **`/codebase-design`** | `/codebase-design` 或上下文 | **深度模块设计**共享词汇表。用于设计/改进模块接口、寻找深化机会、让代码更可测试或 AI 可导航 |
| **`/diagnosing-bugs`** | `/diagnosing-bugs` 或上下文 | **硬 BUG 诊断循环**。用于处理崩溃、异常、性能回退等疑难问题 |
| **`/domain-modeling`** | `/domain-modeling` 或上下文 | **领域建模**。用于固定领域术语、统一语言、记录架构决策 |
| **`/grill-with-docs`** | `/grill-with-docs` | **带文档的 Grilling** — 追问计划的同时创建 ADR 和术语表 |
| **`/implement`** | `/implement` | **根据 PRD 或 Issue 实现代码**。PRD/Issue 驱动的实现 |
| **`/improve-codebase-architecture`** | `/improve-codebase-architecture` | **扫描代码库**寻找深化机会，生成可视化 HTML 报告，然后逐个 grilling |
| **`/prototype`** | `/prototype` | **构建一次性原型**来验证设计问题 — 状态模型、逻辑可行性、UI 探索 |
| **`/research`** | `/research` | **深度研究** — 多源搜索、获取、对抗性验证、合成带引用的报告 |
| **`/resolving-merge-conflicts`** | `/resolving-merge-conflicts` | **解决进行中的 git 合并/变基冲突** |
| **`/setup-matt-pocock-skills`** | `/setup-matt-pocock-skills`（首次运行一次） | 为 engineering skills 配置 repo — Issue Tracker、分类标签词汇表、领域文档布局 |
| **`/tdd`** | `/tdd` | **测试驱动开发**。红-绿-重构循环 |
| **`/to-issues`** | `/to-issues` | **将计划/PRD 拆解为可独立领取的 Issue**，用 tracer-bullet 垂直切片 |
| **`/to-prd`** | `/to-prd` | **把当前对话合成为 PRD**，发布到项目 Issue tracker — 无访谈，直接合成已讨论内容 |
| **`/triage`** | `/triage` | **Issue/PR 分类** — 通过状态机进行归类、验证、grilling、编写 agent-ready 简报 |
| **`/ask-matt`** | `/ask-matt` | **技能路由** — 告诉 Matt 你想做什么，他推荐哪个 skill 最合适 |

---

## 🚀 生产力 (Productivity) — mattpocock/skills

来自 `skills/productivity/` 目录。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/grill-me`** | `/grill-me` | **无情面试官** — 追问计划/设计的所有细节，直到逻辑无懈可击（`/grilling` 的别名） |
| **`/grilling`** | `/grilling` | **追问式访谈** — 逐一遍历设计树的分支，逐个解决决策依赖 |
| **`/handoff`** | `/handoff` | **生成交接文档** — 把当前对话压缩为另一 agent 能接手的交接文件 |
| **`/teach`** | `/teach` | **教学** — 在当前工作区内教用户新技能或概念 |
| **`/writing-great-skills`** | `/writing-great-skills` | **Skill 编写参考** — 编写可预测 skill 的词汇表和原则 |

---

## 🔧 工具 (Misc) — mattpocock/skills

来自 `skills/misc/` 目录。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/git-guardrails-claude-code`** | `/git-guardrails-claude-code` | **Git 安全防护** — 设置 Claude Code hooks 拦截危险 git 命令（push, reset --hard, clean, branch -D 等） |
| **`/migrate-to-shoehorn`** | `/migrate-to-shoehorn` | **迁移测试断言** — 将 `as` 类型断言迁移到 `@total-typescript/shoehorn` |
| **`/scaffold-exercises`** | `/scaffold-exercises` | **搭建练习题目录** — 创建带 section、problem、solution、explainer 的目录结构 |
| **`/setup-pre-commit`** | `/setup-pre-commit` | **配置 pre-commit hooks** — 安装 Husky、lint-staged（Prettier）、类型检查和测试 |

---

## 📝 个人 (Personal) — mattpocock/skills

来自 `skills/personal/` 目录。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/edit-article`** | `/edit-article` | **编辑/改进文章** — 重构章节、提升清晰度、精简行文 |
| **`/obsidian-vault`** | `/obsidian-vault` | **管理 Obsidian 笔记库** — 搜索、创建、用 wikilink 组织笔记 |

---

## 🧪 实验中 (In-Progress) — mattpocock/skills

来自 `skills/in-progress/` 目录，尚未稳定。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/decision-mapping`** | `/decision-mapping` | 把模糊想法变成有序的调查 ticket 映射，然后逐个驱动解决 |
| **`/loop-me`** | `/loop-me` | 询问本工作区内想要构建的工作流的规格 |
| **`/wizard`** | `/wizard` | 生成交互式 bash 向导，引导人完成手动流程（第三方设置、一次性迁移等） |
| **`/writing-beats`** | `/writing-beats` | 写作 — 将原始素材编排成节拍旅程，每个术语在使用前先定义 |
| **`/writing-fragments`** | `/writing-fragments` | 写作 — 挖掘原始片段，尚未组织结构 |
| **`/writing-shape`** | `/writing-shape` | 写作 — 将原始材料逐段塑形为文章 |

---

## 🌐 全局安装 Skills

来自 `~/.claude/skills/`（用户级安装）。

| Skill | 来源 | 功能说明 |
|-------|------|----------|
| **`/web-access`** | `eze-is/web-access` | 所有联网操作：搜索、页面抓取、登录操作、社交媒体内容抓取、动态渲染页面等 |
| **`/agently-mail`** | `agently-mail` | 通过 agently-cli CLI 操作邮件：发送、回复、转发、搜索、读取、管理收件箱 |

---

## ⚡ Claude Code 内置 Skills

以下为 Claude Code 原生内置，无需安装。

| Skill | 调用方式 | 功能说明 |
|-------|----------|----------|
| **`/verify`** | `/verify` | 通过运行应用来验证代码变更是否符合预期 |
| **`/simplify`** | `/simplify` | 审查变更的复用性、简化度、效率和抽象层次，并自动应用修复 |
| **`/code-review`** | `/code-review` | 审查当前 diff 的正确性、复用性、简化度（注意：与 mattpocock 的 code-review 冲突，后者功能更丰富） |
| **`/review`** | `/review` | 审查 GitHub Pull Request |
| **`/security-review`** | `/security-review` | 对当前分支的变更进行安全审查 |
| **`/run`** | `/run` | 启动并驱动项目应用 |
| **`/init`** | `/init` | 初始化新的 CLAUDE.md 文件 |
| **`/loop`** | `/loop <interval> <command>` | 按固定间隔重复运行 prompt 或 slash command |
| **`/fewer-permission-prompts`** | `/fewer-permission-prompts` | 扫描透明并添加允许列表以减少权限提示 |
| **`/update-config`** | `/update-config` | 通过 settings.json 配置 Claude Code 行为、权限、环境变量 |
| **`/keybindings-help`** | `/keybindings-help` | 自定义键盘快捷键绑定 |
| **`/claude-api`** | `/claude-api` | Claude API / Anthropic SDK 参考 — model id、定价、参数等 |

> **注意**：mattpocock 的 `code-review` 和 Claude Code 内置的 `code-review` skill 重叠。内置版更轻量，mattpocock 版是双轴（Standards + Spec）并行审查，功能更强。

---

## 🤖 Agents

项目定义的 reusable agents，可通过 `Agent(subagent_type="...")` 调用。

| Agent | 类型 | 功能说明 |
|-------|------|----------|
| **presentation-claude-code** | 演讲专用 | 更新/修改 Claude Code 最佳实践演示文稿（`presentation/claude-code-best-practice/`） |
| **presentation-claude-gemini** | 演讲专用 | 更新/修改 GDG Kolachi 活动的 Claude-Gemini 演讲（`presentation/2026-04-25-gdg-kolachi-cli-claude-code-gemini/`） |
| **presentation-vibe-coding** | 演讲专用 | 更新/修改 Vibe Coding → Agentic Engineering 演讲（`presentation/vibe-coding-to-agentic-engineering/`） |
| **weather-agent** | 数据获取 | 获取迪拜实时天气。使用模型：inherit |
| **time-agent-pkt** | 时间工具 | 显示巴基斯坦标准时间（PKT, UTC+5） |
| **development-workflows-research-agent** | 研究 | 获取 GitHub repos，分析 Claude Code workflow 仓库。使用模型：sonnet |
| **workflow-claude-commands-agent** | 研究 | 获取 Claude Code 文档，分析 commands report drift。使用模型：opus |
| **workflow-claude-settings-agent** | 研究 | 获取 Claude Code 文档，分析 settings report drift。使用模型：opus |
| **workflow-claude-skills-agent** | 研究 | 获取 Claude Code 文档，分析 skills report drift。使用模型：opus |
| **workflow-claude-subagents-agent** | 研究 | 获取 Claude Code 文档，分析 subagents report drift。使用模型：opus |
| **workflow-concepts-agent** | 研究 | 获取 Claude Code 文档和 changelog，分析 CONCEPTS drift。使用模型：opus |

---

## 🔧 Commands

Slash commands 定义，可通过 `/command-name` 调用。

| Command | 功能说明 |
|---------|----------|
| **`/weather-orchestrator`** | 获取迪拜天气并创建 SVG 天气卡片（使用模型：haiku）— 完整的 Command → Agent → Skill 架构示例 |
| **`/time-command`** | 显示巴基斯坦标准时间（PKT, UTC+5） |
| **`/agent-collections`** | 通过并行研究所有 agent-collection 仓库来更新 AGENT COLLECTIONS 表格 |
| **`/skill-collections`** | 通过并行研究所有 5 个 skill-collection 仓库来更新 SKILL COLLECTIONS 表格 |
| **`/development-workflows`** | 通过并行研究所有 11 个 workflow 仓库来更新 DEVELOPMENT WORKFLOWS 表格 |
| **`/workflow-claude-commands`** | 跟踪 Claude Code commands report 变更并找出需要更新的内容 |
| **`/workflow-claude-settings`** | 跟踪 Claude Code settings report 变更并找出需要更新的内容 |
| **`/workflow-claude-skills`** | 跟踪 Claude Code skills report 变更并找出需要更新的内容 |
| **`/workflow-claude-subagents`** | 跟踪 Claude Code subagents report 变更并找出需要更新的内容 |
| **`/workflow-concepts`** | 根据最新的 Claude Code 特性和概念更新 README CONCEPTS 部分 |

---

## 架构总结

```
用户交互层
├── /command (Commands) ──→ Skill ──→ Agent
├── /skill (Skills)       ──→ 直接执行或 fork sub-agent
└── Agent(subagent_type)  ──→ 独立 sub-agent 运行

原生技能: .claude/skills/*/SKILL.md
通用技能: .agents/skills/*/SKILL.md  → symlink → .claude/skills/
全局技能: ~/.claude/skills/*/SKILL.md
Agents:   .claude/agents/*.md
Commands: .claude/commands/*.md
```

---

> 最后更新: 2026-07-06
