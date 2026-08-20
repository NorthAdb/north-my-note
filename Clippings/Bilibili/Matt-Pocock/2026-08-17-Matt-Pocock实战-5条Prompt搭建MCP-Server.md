---
created: 2026-08-17
updated: 2026-08-17
article_date: 2026-08-17
tags: [Matt Pocock, MCP, MCP Server, Prompt工程, 三段式prompt, Cursor, Octokit, SSE, AI辅助开发, B站笔记]
source:
  - "https://www.bilibili.com/video/BV1mubY6jE4u/"
  - "翻译：ChHsich | 原作者：Matt Pocock"
---

# Matt Pocock 实战教程：5 条 Prompt 从零搭一个 MCP Server

> **UP 主**：ChHsich（搬运翻译） | **原作者**：Matt Pocock
> **时长**：25:40（分 P，英文字幕） | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI）
> **证据等级**：`bilibili-ai-subtitles`

---

## 一句话总论

> **用 5 条「三段式 prompt」从空目录搭出一个可用的 TypeScript MCP server。** 每条 prompt 都遵循 `Problem → Supporting Information → Steps To Complete` 结构；核心立场是区分 **vibe coding** 和**严肃的 AI 辅助开发**——做计划、写文档、管好文件结构，因为**代码就是喂给 LLM 的输入**。

---

## 学完你应该获得什么

- 掌握「三段式 prompt」的结构：Problem / Supporting Information / Steps To Complete，以及每部分写什么
- 看一条完整链路：初始化 → 接 GitHub（Octokit）→ stdio 换 SSE → 按领域重构拆分 → 加新功能
- 理解 `important-files.mdc` 为什么重要、为什么**必须保持更新**
- 理解「LLM 幻觉」的真正根因——context window 里没有对的东西，以及怎么用 prompt 显式修复
- 体会 vibe coding vs AI 辅助开发的理念差异（文件结构 = 喂给 LLM 的输入）

---

## 三段式 Prompt 结构（整套方法的核心）

每条 prompt 有三个顶层标题：

| 部分 | 作用 | 内容示例 |
| :--- | :--- | :--- |
| **Problem** | 描述这条 prompt 要解决的问题 | "从空目录新建一个 TypeScript MCP server" |
| **Supporting Information** | LLM 完成任务所需的全部信息 | 用 pnpm、期望的文件结构（important-files.mdc / package.json / tsconfig） |
| **Steps To Complete** | LLM 按部就班要做的具体步骤 | 明确到"先加载哪些文件再照着写" |

> 三层标题对**聚焦写作**和 prompt 结构很有用。Steps 不用全手写——可以自动补全，或直接让 LLM 生成这个结构。

---

## 5 条 Prompt 逐条拆解

### Prompt 1：初始化 TypeScript MCP server（从空目录）

- **Problem**：从空目录新建 TS MCP server，搭好基本文件系统、装依赖、建项目结构。
- **Supporting**：指定用 pnpm、期望的文件结构。
- **产出**：
  - `tsconfig` 基本原样搬来；`pnpm-lock`（说明它知道该用 pnpm）；
  - `package.json` 含 `type: module`、`bin`、`build`、`dev` 等配置和依赖；
  - `.gitignore` 写好了 `dist` 和 `node_modules`；
  - `src` 目录基本照搬 MCP SDK 的 readme，当前用 **stdio** 传输。
- **过程观察**：唯一卡点是 LLM 装了错误版本的 MCP SDK，被一条大报错纠正了——分步指令执行得非常好。

### Prompt 2：接上 GitHub（处理 issues 和 PR）

- **Problem**：让 MCP server 能处理 GitHub issues 和 PR。
- **Supporting**：用 **Octokit**（GitHub 官方 SDK）；需要了解 **TSX** 和 **.env** 用法；**要求给每个 tool 写描述**（MCP 客户端才知道 tool 是干什么的）。
- **产出**：`.env` 加进 `.gitignore`、`demo.env`、dev 脚本改用 `TSX --env-file` 加载 .env、一大批 Octokit functions（get pull request、create issue comment 等）+ 配套 Zod 参数。
- **三个亮点**：
  1. **TSX 不在预训练数据里**——把用法写进 supporting information，模型就会了；
  2. **pnpm 装依赖 = 自动拿最新版**，不用猜预训练数据里的旧版本号；
  3. **Octokit 几乎没给资料也写得干净**——GitHub REST API 存在太久，训练数据里多半有。

### Prompt 3：把 stdio 换成 SSE（WSL 踩坑实录）

- **Why**：在 WSL 上 Cursor 一直没能用 stdio 连上 MCP server，那玩意挺麻烦。
- **Supporting**：明确写下踩坑经验——**只在 Express 框架下跑通过，试过 Hono 没成**；并贴了 AI Hero 文章里的示例实现。
- **Steps**：开全新 chat（上下文什么都不带）→ 粘贴 → 跑起来。
- **过程**：模型知道装 Express，用 pnpm 拿最新版；把 supporting 里的代码搬进 Express 部分。它还自作主张加了个 `start` 脚本（没用，删掉）。
- **排障**：配置指向 `localhost:8080/sse` 后报 `client closed` 错误——删掉 `app.use(express.json)` 这行，刷新后一切正常。
- **结果**：Cursor 连上 MCP server，演示查询 "Everlight 上有哪些 open PR"，成功调 `list pull requests` 工具。

### Prompt 4：重构——把大 main.ts 拆成小块

- **Why**：main.ts 干的事太多（声明 transport、初始化 server、注册所有 tool），职责过载会让 LLM"乱加一个 tools 文件又绕回来"。
- **做法**：按领域拆文件；主要文件都登记进 `important-files.mdc`——**保持它更新**。
- **产出**：
  - `src/github/`：一堆 functions 和 tools；
  - `src/server/`：app 和 transport；
  - `main.ts` 只剩：建 MCP server、注册 GitHub tools、起 Express 监听。
- **关键观察**：
  - `important-files.mdc` 同步加好——文件结构直接影响 LLM 对代码库的理解；
  - 超大库不逐个列文件，LLM 看文件夹就能发现内容；
  - **cursor rules 可以按文件夹挂**：比如规则设成 auto attached，只在 `src/github` 下的文件生效；全局规则则每个对话都带上。

### Prompt 5：加新功能（查 GitHub Actions 状态）——翻车与修复

- **任务**：加一个查 GitHub Actions workflows 状态的 tool。
- **第一次翻车**：LLM 幻觉出仓库里有个 `createTool` 函数 → 凭空想象出 `createTool.js` 文件 → 用自己发明的 API 造出 `list workflow runs` tool。
- **根因分析**（Matt 主动背锅）：**这是 context window 问题**——上下文里没有对的东西，LLM 没法找到正确解法。也许模型该聪明到自己知道先去看那些文件，但说到底是他**没提前想清楚 LLM 的上下文该是什么**，那些文件必须在它的上下文里。
- **修复**：在 Steps To Complete 开头加一条**明确指令**——"先看对应目录里现成的 functions 和 tools，把它们加载进 context window"，写得很具体；然后清掉之前生成的代码（覆盖掉那条消息），重跑。
- **结果**：这次 LLM 先说"看看现有的 GitHub functions 和 tools"，读正确的文件加载进 context，actions 文件长得和其他 functions 一个样——**明确让它先加载再照着写，才是正解**。

---

## 核心理念：vibe coding vs AI 辅助开发

Matt 明确区分两种方式：

- **vibe coding**：完全不管代码；
- **AI 辅助开发（严肃的那种）**：**做计划、写文档这些"枯燥的事"极其重要**。

> **"我们确实在乎代码，因为代码就是喂给 LLM 的输入。"**

- **文件结构以前不在意，现在能快速告诉 LLM 各文件的用途**；结构一旦混乱、零散或奇怪，就会把 LLM 带偏、干出怪事。
- 目录结构的意义：**加功能时，LLM 知道自己该干什么**。

---

## 方法论提炼（可直接复用）

1. **三段式 prompt**：Problem → Supporting Information → Steps To Complete。先说明要解决的问题，再给全部支撑信息，最后给出明确执行步骤——**这三点非常重要**。
2. **每条 prompt 开新 chat、不带上下文**：避免污染；需要上下文时（如 Prompt 5）就**显式写进 Steps**。
3. **模型不知道的东西写进 Supporting Information**（TSX 用法、Express 踩坑经验、AI Hero 示例实现）。
4. **用 pnpm 装依赖**：自动拿最新版，不猜旧版本号。
5. **重要文件登记进 `important-files.mdc` 并保持更新**——这是 LLM 认识代码库的索引。
6. **按领域拆文件 + cursor rules 按文件夹挂**：职责单一，LLM 不容易跑偏。
7. **幻觉的根因常常是 context window**：不是怪模型不聪明，而是先想清楚"LLM 的上下文里应该有什么"，再在 prompt 里显式要求它加载。

---

## 自测题

1. 三段式 prompt 的三个部分是？各写什么？
2. 为什么每条 prompt 要开新 chat？
3. Prompt 2 里 TSX 为什么需要写进 supporting information？
4. 为什么用 pnpm 装依赖而不是写死版本号？
5. `important-files.mdc` 的作用是什么？为什么必须保持更新？
6. Prompt 4 为什么要拆 main.ts？拆完后的结构是什么样？
7. Prompt 5 第一次翻车的根因是什么？怎么修复的？
8. Matt 说"代码就是喂给 LLM 的输入"，指的是什么？

<details>
<summary>参考答案</summary>

1. Problem（要解决的问题）/ Supporting Information（LLM 需要的全部信息）/ Steps To Complete（具体执行步骤）
2. 避免上下文污染；需要已有代码时显式写进 Steps 让它加载
3. TSX 不在预训练数据里，写进去模型才会用
4. 自动装最新版，避免猜预训练数据里的旧版本号
5. LLM 认识代码库的索引（哪些文件重要）；结构变了不更新会把 LLM 带偏
6. main.ts 职责过载会让 LLM 乱加文件绕回来；拆成 src/github、src/server，main.ts 只留三件事
7. context window 里没有现成实现，LLM 只能凭空想象；修复=在 Steps 开头显式要求"先加载现成 functions/tools 再照着写"
8. 代码质量、文件结构直接决定 LLM 读到的上下文质量，进而决定它能不能干对活

</details>

---

## 证据与原文位置

- 字幕来源：agent-reach / OpenCLI `opencli bilibili subtitle BV1mubY6jE4u`（183 段逐句；分 P）
- 视频元数据：opencli video（2026-08-17 发布，25m40s，ChHsich 搬运，中英字幕）

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1mubY6jE4u/
- **BVID**：BV1mubY6jE4u | **翻译**：ChHsich | **原作者**：Matt Pocock | **发布**：2026-08-17
- **字幕**：agent-reach OpenCLI 提取 B站 AI 字幕
- ⚠️ 分 P 视频，字幕可能只覆盖完整内容的一部分；建议对照原视频
- ⚠️ B站 AI 字幕为英文音轨的翻译字幕，个别术语可能因音译有偏差

## 关联笔记

- [[2026-07-31-Matt-Pocock-Skills原作者手把手教程]] — skills 的安装配置与主流流程
- [[2026-07-31-Matt-Pocock实战-teach-skill让AI像真老师]] — 有状态 skill + 教学工作区，同一个"代码是 LLM 输入"的理念
- [[2026-07-31-Matt-Pocock实战-从原型到spec-driven开发]] — 保真度、spec 驱动的开发方法论
- [[2026-08-05-Matt-Pocock实战-用Claude-Code从零开发真实项目]] — AI 辅助开发的另一条实战路径
- [[2026-08-06-Matt-Pocock-Agent-vs-Workflow科普]] — agent 与工作流的边界
