---
created: 2026-08-07
updated: 2026-08-07
tags: [Codex, AGENTS.md, Skills, Subagents, MCP, Hooks, 工程化, AI编程, 小林coding]
source:
  - "https://mp.weixin.qq.com/s/Bmt8ygJoMEnXGD2d02BHVA"
  - "公众号：小林coding"
---

# Codex 工程化五件套详解：AGENTS.md / Skills / Subagents / MCP / Hooks

> **作者**：小林coding
> **来源**：微信公众号
> **日期**：2026-08-07

![Codex 模型选择界面：5.6 Sol、5.6 Terra、5.6 Luna 等模型](https://gitee.com/cheng-jiaqing/images/raw/master/cover.jpg)

---

## 背景：从 Claude Code 到 Codex

用 Claude 三个月后账号被封，转而研究 Codex。封号邮件称，Anthropic 的 Safeguards Team 在内部调查中发现与账号相关的可疑信号，认定违反 Usage Policy，随后撤销 Claude 的访问权限，并提供了申诉入口。

![Claude 封号邮件：账号因违反 Usage Policy 被撤销访问权限](https://gitee.com/cheng-jiaqing/images/raw/master/img-01.png)

聊天截图里，读者问「林哥不用 cc 了吗」，作者回答「看最近一直 codex」，并用「被封了」解释了迁移原因。Claude Code 和 Codex 的工程化能力**很多功能是相通的**——`AGENTS.md` ≈ `CLAUDE.md`，其余 Skills、Subagents、MCP、Hooks 的概念也基本一致。

![聊天截图：从 Claude Code 转向 Codex](https://gitee.com/cheng-jiaqing/images/raw/master/img-02.png)

**Codex 五件套总览**：

| 解决的问题 | 对应的能力 |
|-----------|-----------|
| 每次新任务都要重新交代项目规则 | **AGENTS.md** |
| 专项知识重要但平时用不上 | **Skills** |
| 主上下文被中间过程挤占 | **Subagents** |
| 需要连接仓库外的数据和系统 | **MCP** |
| 检查动作要固定时机自动执行 | **Hooks** |

---

## 01｜AGENTS.md — 项目入职手册

### 为什么需要

假设团队文档仓库有一组固定规则：正文中文、一篇文档只能有一个一级标题、图片放 `images/`、`published/` 只读。如果每个新任务都要重新交代，既麻烦又容易遗漏。

`AGENTS.md` 就是放在仓库里的**「项目入职手册」**——Codex 开始任务前会读取它，后续的分析、修改和验收都遵守其中规则。

### 怎么创建

两种方式：

1. **手动创建**：在项目根目录新建 `AGENTS.md`，写入团队约定、修改规则和检查命令
2. **让 Codex 生成**：按 `Cmd/Ctrl + O` 打开项目后，发送提示词描述规则

![在 Finder 中打开 product-docs 项目根目录](https://gitee.com/cheng-jiaqing/images/raw/master/img-03.png)

示例提示词：

```
请在当前项目根目录创建 AGENTS.md。
这是一个团队技术文档仓库。
所有正文使用中文，一篇文档只能有一个一级标题。
文件名使用小写英文和连字符，图片统一放在 `images/`。
`published/` 中的文档只读，不要直接修改。
完成修改后执行 `python3 scripts/check_docs.py` 检查标题和图片引用。
先只创建规则文件，不修改任何文档，也不执行检查命令。
```

![Codex 创建 AGENTS.md 后的任务结果与审核入口](https://gitee.com/cheng-jiaqing/images/raw/master/img-04.png)

Codex 生成后可在右侧审核产出内容：

![Codex 审核 AGENTS.md：右侧显示新增规则的 diff](https://gitee.com/cheng-jiaqing/images/raw/master/img-05.png)

**验证规则是否被读到**：新任务中输入「只总结当前生效的项目规则，并列出规则来自哪些 AGENTS.md 文件」，返回结果应包含中文正文、文件命名、图片目录、`published/` 只读和检查命令等规则。

![Codex 总结当前规则并列出来源文件](https://gitee.com/cheng-jiaqing/images/raw/master/img-06.png)

### 写法要点

最小示例：

```markdown
# 项目说明

这是一个团队技术文档仓库。

## 文档规则
- 所有正文使用中文
- 一篇文档只能有一个一级标题
- 文件名使用小写英文和连字符
- 图片统一放在 `images/` 目录

## 修改约束
- `published/` 中的文档只读
- 不要创建正文没有引用的图片
- 完成修改后执行 `python3 scripts/check_docs.py`
```

> **没有神秘语法，就是 Markdown。真正影响效果的是内容够不够具体。**
>
> 「注意文档质量」这种话基本等于没说。换成「只能有一个一级标题」「图片统一放进哪个目录」「修改后执行哪条检查命令」，Codex 才知道下一步该做什么，也知道怎么证明自己做完了。

![对比模糊规则与可执行规则：目录、命令和验收标准都要明确](https://gitee.com/cheng-jiaqing/images/raw/master/img-07.png)

### 规则怎么加载

```
~/.codex/
└── AGENTS.md                    # 个人全局规则

product-docs/
├── AGENTS.md                    # 整个文档仓库的规则
├── drafts/
├── published/
├── images/
└── api/
    ├── AGENTS.override.md       # API 文档的覆盖规则
    └── payment-api.md
```

- **个人级**：`~/.codex/AGENTS.md`（所有项目通用偏好，通过 Settings > Personalization 编辑）
- **项目级**：根目录写团队共识，子目录放模块专属规则
- **加载方式**：从项目根目录一路走到当前工作目录，每层最多选一个规则文件，按**从远到近**顺序拼接；离当前目录越近优先级越高
- **override 优先级**：同一层同时存在 `AGENTS.override.md` 和 `AGENTS.md` 时，只取 override 文件，不会两份都读
- **大小上限**：默认 32 KiB，长资料通过引用架构文档/专项说明而非全文

> 注意：如果当前打开的是仓库根目录，让 Codex 去改 `api/`，它不会自动补读更深层的规则。要测试子目录规则，就把该子目录作为本地项目打开，或把通用规则写到根目录。

图示对比了两种打开方式：只打开 `product-docs/` 时，规则链主要是个人级规则加项目根目录规则；打开 `product-docs/api/` 时，才会继续合并 `api/AGENTS.override.md`，并以更近的规则覆盖上层约定。

![AGENTS.md 规则加载层级：越靠近当前目录优先级越高，override 同层优先](https://gitee.com/cheng-jiaqing/images/raw/master/img-08.png)

> 每次都要遵守的放 `AGENTS.md`；只有做发布、Review、迁移时才需要的长流程，交给 Skills。

---

## 02｜Skills — 专项知识按需加载

### 为什么不能全塞进 AGENTS.md

有些知识很重要，但不是每次任务都用得上（比如发布 Release Notes 的整套流程）。全塞进 `AGENTS.md` 会让每个任务都背完整份手册，**上下文被占用，注意力被摊薄**。

那能不能等用到时再读？可以，这就是 **Skill**——渐进式加载。

![AGENTS.md 是随身薄手册，Skill 则像任务需要时才取出的专业手册](https://gitee.com/cheng-jiaqing/images/raw/master/img-09.png)

### 怎么创建和使用

Codex 内置 `$skill-creator`，直接描述流程即可生成：

```
$skill-creator 为当前项目创建一个 release-notes Skill。
它用于根据指定的 Git Tag 到当前分支之间的真实改动生成发布说明。
先确认起始版本，再读取提交和改动文件。
过滤合并提交、纯格式调整和 CI 配置更新。
把内容分成新功能、问题修复和破坏性变更。
使用面向用户的语言，每一项都要能追溯到提交或文件证据。
只生成草稿，不创建 Tag，也不发布版本。
同时提供 Release Notes 模板和收集提交的脚本。
生成项目共享 Skill，先只创建文件，不开始生成发布说明。
```

![Skill Creator 创建 release-notes Skill 的实际任务界面](https://gitee.com/cheng-jiaqing/images/raw/master/img-10.png)

图中的一次生成结果还展示了 Skill 的工程化产物：`SKILL.md` 负责流程正文，模板和收集提交的脚本作为配套资源，另有 `openai.yaml` 这类 UI 元数据；截图中实际使用的是 `assets/release-notes-template.md` 和 `scripts/collect-commits.py`。文章里的 `references/`、`.sh` 只是示例，目录名和脚本语言可以按项目约定调整，关键是让 Skill 自包含、可复用、可验证。

创建后打开侧边栏「Skills」页面即可看到：

![技能管理页面：已安装的发布说明 Skill 出现在技能列表中](https://gitee.com/cheng-jiaqing/images/raw/master/img-11.png)

**显式调用**：新建任务输入 `$release-notes` 即可触发。

![显式调用 Skill：任务输入框中使用 `$release-notes`](https://gitee.com/cheng-jiaqing/images/raw/master/img-12.png)

**自动匹配**：不写 `$release-notes`，直接说「根据 v1.4.0 之后的改动生成发布说明」，只要任务与 Skill 描述匹配，Codex 也会自动选中。

![Skill 自动匹配流程：创建、出现在 Skills 页面，再由新任务调用](https://gitee.com/cheng-jiaqing/images/raw/master/img-13.png)

### Skill 是怎么工作的

```
release-demo/
└── .agents/
    └── skills/
        └── release-notes/
            ├── SKILL.md
            ├── references/
            │   └── release-template.md
            └── scripts/
                └── collect-commits.sh
```

最小版本只要一个 `SKILL.md`：

```markdown
---
name: release-notes
description: 准备版本发布说明时使用，根据指定 Git Tag 到当前分支之间的真实改动，生成可追溯的 Release Notes 草稿。
---

# Release Notes 流程

1. 确认用户给出的起始 Git Tag 存在
2. 运行 `scripts/collect-commits.sh <tag>` 收集提交和改动文件
3. 过滤合并提交、纯格式调整和 CI 配置更新
4. 按新功能、问题修复和破坏性变更分类
5. 使用面向用户的语言，不直接复制提交信息
6. 每一项标出可追溯的提交或文件证据
7. 按 `references/release-template.md` 生成草稿

只生成发布说明草稿，不创建 Tag，不调用发布接口。
```

- **关键字段是 `description`**：要写清它做什么、什么时候用、产出什么，Codex 靠它判断何时使用
- **位置**：项目共享 Skill 放 `.agents/skills/`，个人跨项目 Skill 放 `~/.agents/skills/`（注意是 `.agents`，不是 `.codex`）
- **加载机制**：启动时只看名称和描述；任务匹配或显式点名时才读取完整正文，长资料和脚本按需读取/执行

> 简单来说，Codex 平时只记住「书名和简介」，接到任务后才去书架拿对应的书——这就是 Skill 的渐进式加载。

![Skill 渐进式加载：先读名称和描述，再读 SKILL.md，最后按需读取 references 与 scripts](https://gitee.com/cheng-jiaqing/images/raw/master/img-14.png)

### AGENTS.md 与 Skill 的分工

| 内容 | 放在哪里 | 加载策略 |
|------|----------|----------|
| 每次任务都必须遵守的规则 | `AGENTS.md` | 常驻、优先加载 |
| 某类任务的专业流程 | `SKILL.md` | 匹配任务后加载 |
| 长篇参考资料、模板、脚本 | Skill 目录 | 执行到需要的步骤时再读取或运行 |

实践中，`description` 应同时回答三个问题：**什么时候触发、解决什么问题、产出什么结果**。如果只写「发布工具」之类的宽泛描述，自动匹配容易误触发或漏触发。

---

## 03｜Subagents — 变成一支小团队

### 为什么需要

评估新日志平台需要同时调查产品能力、Java 接入方式和迁移风险。全部由主 Agent 处理，搜索结果和中间资料很快挤占主上下文。

Subagent 把多路调查**拆到独立上下文并行完成**，主 Agent 只负责分工、收集结果和整理结论。主任务派出后界面会显示每个 Subagent 线程，可点进去查看过程，再回到主任务看合并结果。

### 怎么安排

> **关键区分**：Subagent 是临时派出去执行子任务的**独立线程**；自定义 Agent 是**可反复使用的「岗位模板」**。
>
> 普通并行任务不创建任何配置也能直接派 Subagent；只有某类分工经常出现，才值得写进 `.codex/agents/` 复用。

先创建三个自定义 Agent（岗位模板）：

```
请在当前项目创建三个只读的自定义 Agent。
capability_researcher 只负责调查查询、告警、权限和运维能力。
integration_researcher 只负责调查 Java、OpenTelemetry 和现有监控的接入成本。
migration_researcher 只负责调查旧查询、历史数据和迁移风险。
每个 Agent 都要区分官方资料、项目现状和自己的推断，并保留来源。
先只创建 `.codex/agents/` 下的配置文件，不开始调研。
```

![创建三个只读 Agent 配置：分别负责能力、接入和迁移调研](https://gitee.com/cheng-jiaqing/images/raw/master/img-15.png)

图中的实际结果进一步体现了岗位模板应包含的边界：生成 `capability-researcher.toml`、`integration-researcher.toml` 和 `migration-researcher.toml`，统一设置 `sandbox_mode = "read-only"`，明确调研范围，区分「官方资料、项目现状、推断」，要求保留可追溯来源，并让模型和任务设置保持可继承。也就是说，自定义 Agent 不只是改一个名字，更重要的是把权限和输出标准固定下来。

再派发并行任务：

```
评估当前项目是否适合采用候选日志平台。
请并行安排三个 Subagent：
1. capability_researcher 调查产品能力和限制
2. integration_researcher 调查 Java 接入方式和改造成本
3. migration_researcher 调查迁移步骤、数据风险和回退方案
等三路结果回来后合并去重，输出适用条件、主要风险和待确认问题。
只做调研，不修改当前项目。
```

![三个只读 Subagent 同时启动，分别承担能力、接入和迁移调研](https://gitee.com/cheng-jiaqing/images/raw/master/img-16.png)

### Subagent 是怎么工作的

**独立上下文是核心机制**：Subagent 的调查过程（搜索结果、命令输出、中间笔记）不进入主上下文，主 Agent 拿到的是整理过的结论。Subagent 默认继承主任务权限模式，在当前工作区使用工具。

![Subagent 工作阶段表：拆分、派发、执行、回传、合并](https://gitee.com/cheng-jiaqing/images/raw/master/img-17.png)

可以把一次并行任务拆成五个阶段：

| 阶段 | 主 Agent 与 Subagent 的工作 |
|------|-----------------------------|
| **拆分** | 主 Agent 找出边界清晰、可以独立处理的子任务 |
| **派发** | 为每个 Subagent 指定目标、边界和输出格式 |
| **执行** | Subagent 在独立上下文中使用模型和工具完成任务 |
| **回传** | 只返回结论与必要证据，而不是全部搜索过程 |
| **合并** | 主 Agent 等待各路结果，去重、交叉核验后生成回答 |

![Subagent 并行流程图：主 Agent 拆分任务，子 Agent 独立执行并返回结论与证据](https://gitee.com/cheng-jiaqing/images/raw/master/img-18.png)

- **适合并行**：代码探索、资料整理、测试、问题归类等互相独立的任务
- **不适合**：多个 Subagent 同时修改同一批文件——可能产生冲突
- 需要几路同时改代码 → 创建多个顶层任务 + 独立 **Worktree**（文件层面隔离）
- Worktree 和 Subagent 解决的不是同一问题：前者是文件隔离，后者是上下文与分工拆分

图示的关键区别是：Subagent 虽然隔离了各自的上下文，但仍共享同一个工作目录和文件；顶层任务配合 Worktree 则同时隔离上下文与文件目录，适合并行改动代码、避免相互覆盖。

![Subagent 与 Worktree 对比：前者上下文隔离但共享文件，后者连工作目录也隔离](https://gitee.com/cheng-jiaqing/images/raw/master/img-19.png)

> Codex 内置 `default`、`worker`、`explorer` 等通用 Agent。普通并行任务不必先定义岗位，提示词里说「请并行安排三个 Subagent」即可。

---

## 04｜MCP — 统一的外部系统插座

### 为什么需要 MCP

真实任务往往需要仓库之外的信息：云端文档、GitHub Issue、数据库、协作工具……如果每个外部系统都要单独定义工具说明和参数格式，连接方式会非常碎片化。

**MCP（Model Context Protocol）** 提供一套统一通信规则——AI 工具世界的「通用插座」。外部服务通过 MCP 告诉 Codex 自己能提供什么数据和操作，Codex 根据任务选择对应能力。

![MCP 统一插座示意：Codex 通过 MCP 连接云端文档、GitHub、数据库和协作工具](https://gitee.com/cheng-jiaqing/images/raw/master/img-20.png)

### 怎么接入

通过插件市场安装常见 MCP 工具，安装完成后即可在新任务中调用。

以 Google Drive 为例：先创建测试文档 `Codex MCP Demo`，写入模拟内容（项目：会员中心改版 / 上线时间：8 月 20 日 / 待办：确认页面文案、补充登录异常监控、完成上线前回归测试），然后打开侧边栏「Plugins」搜索 `Google Drive` 安装并授权。

![插件市场示例：在 Plugins 页面搜索并安装 Google Drive](https://gitee.com/cheng-jiaqing/images/raw/master/img-21.png)

新建任务直接使用：

```
使用 Google Drive 工具查找名为「Codex MCP Demo」的文档。
读取文档内容，输出项目名称、上线时间和待办事项。
只读取和整理，不要修改或创建任何文件。
```

Codex 会根据任务内容自动选择 Google Drive 提供的工具，无需额外命令。图示把过程拆成五步：搜索 Google Drive、点击安装、连接 Google 账号、新任务读取 `Codex MCP Demo`，最后整理出项目名称、上线时间和待办事项。

![Google Drive MCP 接入流程：搜索、安装、授权、新任务读取、整理结果](https://gitee.com/cheng-jiaqing/images/raw/master/img-22.png)

### MCP 工具是怎么工作的

MCP 像一套统一的「工具说明书和通信规则」：Google Drive 接入后先告诉 Codex 自己有哪些工具、每个工具能做什么、调用要填什么参数。你说「找到 Codex MCP Demo 并读取内容」时，Codex 会：

1. 从工具说明中找到匹配工具
2. 生成查找文档所需的参数
3. 把请求发给 Google Drive
4. 对方返回结果后，发现还需读取正文 → 继续调用读取工具
5. 拿到内容后整理成回答

**角色对应**：

| MCP 角色 | 职责 |
|---------|------|
| **Host**（Codex） | 理解任务、选择工具 |
| **MCP Client**（Codex 内部） | 传话员，发送请求、取回结果 |
| **MCP Server**（外部服务） | 真正执行查找、读取、写入操作 |

这套流程不限于 Google Drive——换成数据库、GitHub 或其他系统，Codex 都用同一套方式理解和调用。MCP Server 除了提供可执行的 **Tools**，还可以提供可读取的 **Resources** 和可复用的 **Prompts**。

![MCP 调用链：用户任务 → Codex Host → MCP Client → MCP Server → 外部操作，再原路返回结果](https://gitee.com/cheng-jiaqing/images/raw/master/img-23.png)

可以用一句话区分 **Skill 和 MCP**：Skill 提供「应该怎样完成任务」的知识与流程，MCP 提供「怎样对外部系统动手」的工具与连接。比如 Skill 规定先审查再回写，真正读取 GitHub PR 或更新工单的动作由 MCP 完成。

---

## 05｜Hooks — 固定时机的自动触发

### 为什么需要 Hooks

提示词和 `AGENTS.md` 是「告诉 Codex 应该怎么做」，适合放项目背景、代码规范和修改约束。但测试、格式化、安全扫描这类动作，更适合由**程序在固定时机自动执行**——写进提示词既重复，执行时机也不固定。

**Hooks = 任务生命周期中的自动触发器**：运行到某个节点时，Codex 自动调用预先配置的 Handler，不需要再次发送提示词。

> Hooks 就像安装在 Codex 工作流程中的感应门。Codex 运行到对应位置，就会触发门后的脚本。

![Hooks 生命周期示意：加载上下文、检查命令、记录结果，并自动执行测试与安全扫描](https://gitee.com/cheng-jiaqing/images/raw/master/img-24.png)

### 一个简单的例子：Stop Hook

已有 `python3 scripts/check_docs.py` 检查脚本，让 Codex 创建 Stop Hook：

```
请为当前项目创建一个 Stop Hook。
任务准备结束时，自动运行 `python3 scripts/check_docs.py`。
检查通过时正常结束任务。
检查失败时把摘要交给 Codex 继续处理。
请创建所需的 Hook 配置和 Handler，不修改现有文档。
```

之后每次任务准备结束，Stop Hook 都会自动运行文档检查，无需在提示词中重复提醒。检查通过就正常结束；检查失败则把失败摘要交回 Codex，Codex 继续处理问题，随后再次经过结束检查。

![Stop Hook 闭环：任务结束前检查，失败返回摘要让 Codex 继续处理，通过后正常结束](https://gitee.com/cheng-jiaqing/images/raw/master/img-25.png)

### Hooks 是怎么工作的

Hook 配置说明三件事：**监听哪个生命周期事件、什么条件下命中、命中后执行哪个 Handler**。

| 生命周期节点 | 触发时机 | 常见用法 |
|-------------|---------|---------|
| `SessionStart` | 任务会话启动时 | 加载环境信息或项目上下文 |
| `UserPromptSubmit` | 用户提交任务时 | 记录或补充输入上下文 |
| `PreToolUse` | 工具执行之前 | 检查命令或阻止不允许的操作 |
| `PostToolUse` | 工具执行之后 | 记录结果或补充校验 |
| `Stop` | 任务准备结束时 | 运行测试、格式化或安全扫描 |

图示还区分了两个时间尺度：`SessionStart` 在会话启动时触发一次；每轮任务通常经历 `UserPromptSubmit → PreToolUse → PostToolUse → Stop`，其中工具调用前后的节点可能在一轮任务中反复触发。以上是常见节点，并非完整事件列表。

![Hooks 事件时间线：SessionStart 会话级一次触发，每轮任务包含提交、工具前后和 Stop 节点](https://gitee.com/cheng-jiaqing/images/raw/master/img-26.png)

运行到节点 → 找到匹配 Hook → 调用 Handler。Codex 通过**标准输入**把当前工作目录、事件名称（工具事件还含工具名称和参数）交给 Handler。Handler 执行后通过**退出码或 JSON 结果**告诉 Codex：继续运行、停止当前动作，或把新上下文交回给模型。

![Hook 执行链：事件经过 Matcher 命中 command Handler，JSON 通过 stdin 输入，结果决定继续、阻止或补充上下文](https://gitee.com/cheng-jiaqing/images/raw/master/img-27.png)

从图中可以把 Handler 的执行链记成四步：

1. 生命周期事件产生后，由 **Matcher** 判断当前 Hook 是否命中
2. 命中后启动 command Handler，并通过 `stdin` 传入 JSON 事件数据
3. Handler 返回退出码或 JSON 结果；图中 `exit 0` 表示放行继续
4. Codex 按结果继续任务、阻止当前动作，或把补充上下文交回模型

---

## 总结：五件套对比

| 能力 | 作用 |
|------|------|
| `AGENTS.md` | 让 Codex 在项目中持续遵守固定规则 |
| Skills | 封装可复用的专项知识和操作流程，按需加载 |
| Subagents | 拆分独立子任务，并行调查并汇总结果 |
| MCP | 连接仓库外的数据、工具和外部系统 |
| Hooks | 在 Codex 的固定生命周期节点自动执行动作 |

## 五件套如何协同

以「从外部系统获取资料、并行分析、修改项目、最后验收」为例，完整链路可以是：

1. **会话启动**：Codex 加载全局、项目和当前目录适用的 `AGENTS.md`，先确定工作边界
2. **任务匹配**：根据 Skill 的 `description` 选择对应 Skill，再按需读取 `SKILL.md`、参考资料和脚本
3. **获取外部信息**：通过 MCP 查询 Google Drive、GitHub、数据库等系统
4. **并行处理**：把边界清晰的调查或检查任务派给多个 Subagent，各自返回结论与证据
5. **执行修改**：主 Agent 汇总结果并修改项目，`PreToolUse` / `PostToolUse` Hook 负责过程中的拦截和校验
6. **结束验收**：`Stop` Hook 自动运行测试、格式化或安全扫描；失败摘要返回给 Codex，修复后再验收

这五项能力分别覆盖了**规则、知识、分工、连接、纪律**五个层面，不是五个互相独立的按钮，而是一套逐层叠加的工作流。

## 实践边界与风险

- **AGENTS.md**：只放每次任务都要遵守的规则；长篇流程放 Skill，且要在正确的项目/子目录中打开以加载对应规则
- **Skills**：把触发条件写清楚，明确脚本的副作用、输入输出和停止条件，避免自动匹配误触发
- **Subagents**：优先拆分只读、边界清晰的工作；要求「结论 + 证据 + 来源」，不要让多个子任务同时写同一批文件
- **Worktree**：需要并行改代码时，用 Worktree 做文件隔离；Subagent 本身只解决上下文和分工问题
- **MCP**：外部连接应遵循最小权限原则，读写权限、账号授权和数据范围都要先审核；插件来源不明时不要直接安装
- **Hooks**：Handler 应尽量快速、确定、可重复执行，失败信息要能指导 Codex 修复；不要把耗时很长或结果不稳定的流程无条件挂在每次工具调用上

## 核心洞见

> 每次都要遵守的规则 → `AGENTS.md`；专项流程 → **Skills**；并行分工 → **Subagents**；外部系统 → **MCP**；固定时机的强制动作 → **Hooks**。

这五件套与 Claude Code 六件套的底层思路完全一致——入职手册、按需查资料、团队分工、标准接口、门禁系统，全是软件工程和团队管理里玩了几十年的老经验。

> **AI 变强了，但让 AI 好好干活的方法，还是那些让人好好干活的方法。**

---

## 相关阅读

- [[2026-07-28-Claude-Code工程化六件套详解-小林coding]] — Claude Code 版本（多一个 Plugins：打包分发）
- [[2026-07-21-Claude-Code-Skill机制源码剖析]] — Skill 机制源码级剖析
