---
created: 2026-07-31
updated: 2026-08-02
tags: [Matt Pocock, 方法论, 精华, 汇总, Skills, Wayfinder, Agent工作流, DDD, 教学]
source:
  - "五篇 Matt Pocock 系列笔记汇总"
---

# Matt Pocock 方法论精华：从想法到交付的完整 Agent 工作流

> **本页是五篇 Matt Pocock 系列笔记的汇总精华**，整合了他「如何用 AI 从零构建项目」的完整方法论——从探索需求、到设计架构、到交付实现、再到教学传承。

---

## 📚 五篇笔记索引

| # | 笔记 | 主题 | 核心内容 |
|:-:|:-----|:-----|:---------|
| 1 | [[Matt-Pocock/2026-07-31-Matt-Pocock直播-SlopWatch从零搭建完整实录]] | GrillMe + DDD 完整实战 | 1:47 直播实录：点子收集 → GrillMe 诘问 → 研究 5 个 Agent → 统一语言 → 架构定型（SlopWatch） |
| 2 | [[Matt-Pocock/2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] | Wayfinder | grilling 会话编排器，解决 spec 之前探索阶段 |
| 3 | [[Matt-Pocock/2026-07-31-Matt-Pocock实战-从原型到spec-driven开发]] | Prototype + 保真度 | 别急着写 spec，用原型提保真度 |
| 4 | [[Matt-Pocock/2026-07-31-Matt-Pocock-Skills原作者手把手教程]] | Skills 主流流程 | 安装配置 + grill-with-docs → to-spec → to-tickets → implement → code-review |
| 5 | [[Matt-Pocock/2026-07-31-Matt-Pocock实战-teach-skill让AI像真老师]] | teach skill | 有状态教学 skill、最近发展区、HTML 课程 |

---

## 🧭 完整方法论地图

```
【探索】─────────────【设计】─────────────【交付】─────────────【传承】
 口述需求             保真度决策            to-spec               teach skill
   ↓                   ↕                    ↓                    ↓
 GrillMe /            prototype            to-tickets          教学区
 grill-with-docs ──→ (提保真度) ──→         ↓                  最近发展区
   ↓                   ↕                  implement            HTML 课程
 Wayfinder            DDD                 code-review
 (编排器+map)          统一语言              (子agent)
```

---

## ⚡ 五大核心心法

### 1️⃣ Agent = Model + Harness + 环境

```
        [ 环境 ]    ← 文件系统、代码库
    [   Harness  ]  ← Claude Code、Codex、Claude AI、ChatGPT
      [  Model ]    ← Opus、GPT 5.6
```

> **「换 model 就好了」的说法收益有限——在环境和 harness 上下功夫，能拿到的收益多得多。** Model 只是图中一小块，Harness（工具、状态、权限、审计）和环境（代码库、文件系统）才是工程化落地的关键。

### 2️⃣ 聪明区（Smart Zone）≈ 140K

> Matt 把 **140K 上下文**当作「聪明区」上限。超过 140K，注意力退化，模型开始变笨、产生幻觉。

**推论**：
- 每个 ticket 控制在「一个上下文窗口/聪明区」的粒度
- ticket 之间清空上下文
- 日常任务保持在 150K 以内

### 3️⃣ 保真度（Fidelity）决定一切

| 保真度 | 适用问题 | 方式 |
|:------|:---------|:-----|
| 低保真 | 基础问题（弹窗有取消/确认按钮） | 讨论里直接定 |
| 中保真 | 数据怎么展示 | 稍微提高 |
| 高保真 | 「真看看它跑起来什么样」 | **必须做原型** |

> **「我写了这么漂亮的 spec，结果做出来的东西完全跑偏」——大概率是讨论的保真度不够高，当时就该做原型。**

### 4️⃣ 有状态 vs 无状态 Skill

| | 无状态 | 有状态 |
|:--|:------|:-------|
| 记忆 | 不保留 | 存文件系统/MCP，记住进度 |
| 例子 | grill-me | grill-with-docs、teach |
| 适用 | 一次性任务 | 跨会话推进的学习/项目 |

> 设计 skill 时要想清楚该选哪种——没有谁更好，只是场景不同。

### 5️⃣ 从讨论到 production 是大跨步，从原型到 production 是小平步

> 从「讨论+spec」跨到 production-ready 代码，这一步非常大。但如果你有一个**能跑的原型**，把它变成 production 就挺简单。

---

## 🛠️ 完整技能体系（Matt Pocock / skills 仓库）

> 📋 **全量清单见** [[Matt-Pocock-Skills仓库全量分类汇总未命名]]（工程 17 + 生产力 5 + 杂项 4 + 个人 2 + 实验 9 + 废弃 4 = 41 个）

仓库规模：**16 万 star、750 万下载、38+ 个 skills**（官方 + 实验）。全部 skills 只占 **660 token** 上下文（手动调用 + 描述精简）。

### 探索类

| Skill               | 用途                                           | 有状态? |
| :------------------ | :------------------------------------------- | :--: |
| **grill-me**        | 无状态盘问，不断考你直到能上手                              |  ❌   |
| **grill-with-docs** | 有状态盘问，保存 ADR + 术语表到 repo                     |  ✅   |
| **domain model**    | GrillMe 进化版，用 DDD 敲定统一语言（SlopWatch 实战首次亮相）   |  ✅   |
| **Wayfinder**       | GrillMe 进化版，grilling 会话**编排器**，拆成多个聚焦 ticket |  ✅   |
| **zoom-out**        | 遇到不懂的概念时 explain-like-I'm-5                  |  ❌   |

### 设计类

| Skill               | 用途                     |
| :------------------ | :--------------------- |
| **prototype**       | 做便宜粗糙但具体的产物，把讨论保真度提上来  |
| **domain-modeling** | 用 DDD 敲定统一语言，让架构从语言中浮现 |

### 交付类

| Skill           | 用途                                               |
| :-------------- | :----------------------------------------------- |
| **to-spec**     | 把全部讨论压缩成 spec 文档（2PRD 的新名字）                      |
| **to-tickets**  | 把 spec 拆成实现 ticket（1 ticket = 1 上下文窗口）           |
| **implement**   | 逐个 ticket 实现，两个 ticket 间清空上下文                    |
| **code-review** | 对照 spec + 仓库标准检查，**用子 agent**（主 agent 不擅长审自己的代码） |

### 教学/支持类

| Skill                       | 用途                                       |
| :-------------------------- | :--------------------------------------- |
| **teach**                   | 有状态私人老师，建教学工作区（mission/lessons/glossary） |
| **ask-matt**                | 把 Matt 本人做成 skill，回答「该怎么开始」              |
| **setup-mattpocock-skills** | 配置 issue 跟踪器/领域文档/标签                     |
| **triage**                  | 用标签传达生成的 ticket 信息                       |
| **writing-great-skills**    | 教怎么写好的 skills                            |

---

## 🔄 主流流程（一次完整交付）

```
1. grill-with-docs ── 盘问打磨想法，有状态记录到 context.md + ADR
        ↓
2.（可选）prototype ── 问题需要可运行答案时，提保真度
        ↓
3. 小工作量 → 直接 /implement
   大工作量/跨多会话 → to-spec（46.1K 讨论 → 文档）
        ↓
4. to-tickets ── 每个 ticket = 一个上下文窗口大小
        ↓
5. implement ── 一次一个 ticket，之间清空上下文
        ↓
6. code-review ── 子 agent 对照 spec + 仓库标准
```

---

## 🔀 spec 对比：Matt 一次性 spec vs OpenSpec 长期 spec

> **一句话**：方向对——OpenSpec 的 spec 是**持续演进的活规范**，Matt 的 spec 是**用完即弃的交接文档**。但精确的差异比"长期 vs 不看"更本质：**一个把 spec 当"真相来源"维护，一个把 spec 当"临时导航"销毁**。

| 维度 | OpenSpec | Matt (to-spec) |
|:-----|:---------|:---------------|
| **spec 定位** | 长期真相来源（source of truth），系统的"规范" | 跨会话工作的临时终点文档 |
| **存放** | `openspec/specs/`（长期）+ `openspec/changes/`（进行中） | issue tracker 里的一个 issue / 交接文档 |
| **生命周期** | 持续演进——每个 change 完成后**新需求 merge 进主 specs** | 实现完就删——关掉 issue，spec 消失，极少回看 |
| **真相在哪** | spec 文件本身（可 review / diff / 版本控制） | **代码 + 原始决策 ticket**（Wayfinder 链回第一手决策） |
| **维护成本** | 主动投入，spec 是资产 | 拒绝维护，spec 是沉没成本 |

**OpenSpec 的机制**：`/opsx:explore → propose → apply → archive`。change 是临时的 in-flight 文件夹（proposal/specs/design/tasks），实现完归档到 `openspec/changes/archive/`，同时**新需求写回长期 specs 目录**——spec 随功能发布持续生长。

**Matt 的机制**：`grill-with-docs → to-spec（46.1K 讨论→文档）→ to-tickets → implement → code-review`。spec 是把讨论压缩成的"最终长什么样"的描述，作用是跨多次会话时让到达终点的人（或 AFK agent）知道原本要去哪；落地后就删。

**哲学分歧（本质）**：
- OpenSpec：需求在聊天记录里会丢失、AI 会跑偏 → 把规范**版本化、前置到 repo、长期持有**，人和 AI 写代码前对齐
- Matt：spec 只是「会议的总结」，不是「真理」；真相在代码和决策记录里 → 反复评审/维护 spec 是**经典错误**（"他们真正该做的是赶紧写代码"），且 spec 会随实现过时

**何时用哪个**：
- 需要**跨团队/跨仓库/长期演进**的系统规范（平台团队、多人协作、规范复用）→ **OpenSpec**（配套 Spec-Kit、gstack 等 SDD 全家桶）
- **个人/单次交付**的临时计划（项目实现完 spec 就没价值）→ **Matt 式一次性 spec**
- 中间态：项目有长期维护需求 → 把 spec 的关键决策沉淀成 **ADR + 领域词汇表**（Matt 推荐），而非维护整份 spec

> 📌 相关收藏：B站《[精讲OpenSpec，从操作到原理](https://www.bilibili.com/video/BV1eU5A6MERk)》《[OpenSpec、Superpowers、gstack三器合一](https://www.bilibili.com/video/BV1uDQacCE2a)》

---

## 🗂️ ADR：架构决策记录（Matt 的"长期记忆"答案）

> **ADR = Architecture Decision Record（架构决策记录）**——一页纸的"决策备忘录"：记录做过的某个重要决策、当时为什么这么选、代价是什么，让"为什么"不随对话消失。这是经典软件工程概念，但 Matt 把它用作 Agent 工作流里的**长期记忆方案**。

### 经典格式（Michael Nygard 三段式）

```markdown
# 决策：订单状态用 enum 而非 boolean
## 背景（Context）
  之前讨论时纠结：用 isPaid / isShipped 多个 boolean 还是 enum？
## 决策（Decision）
  用 enum：standard / short / future
## 后果（Consequences）
  扩展第三种状态只需加一个枚举值，不会像 boolean 组合那样爆炸
```

### 在 Matt 方法论中的角色（关键）

Matt 对 ADR 的推荐，和他反对"长期 Agent 记忆（RAG）"是**同一件事的两面**：

|          | 长期 Agent 记忆（RAG）      | ADR + 最小化 DDD                  |
| :------- | :-------------------- | :----------------------------- |
| Matt 的态度 | ❌ 坏主意                 | ✅ 推荐                           |
| 存储       | 向量数据库，模糊、不可见          | **纯文本文件，存 repo 里**             |
| 特性       | 不可观测、不可编辑、塞不进 context | **可观测、可编辑、能塞进 context window** |

> 原话（SlopWatch 直播）：*"想要「超级可观测、超级具体、能立即编辑、能塞进 context window」的东西。方案：ADR + 最小化 DDD，不用 RAG 式长期记忆。"*

**为什么关键**：Agent 每次会话都要读 context。ADR 是纯 markdown 文本，直接读进 context 窗口（token 占用少）；RAG 记忆是模糊检索，Agent 拿到的是片段、不可控。**ADR = 用"文件"当记忆，而不是用"数据库"当记忆。**

### 在 Matt 流程中出现的位置

1. **grill-with-docs**（有状态盘问）：边追问边**自动生成 ADR + 术语表**，存到 repo——盘问的副产品就是一套决策档案
2. **domain-modeling**：敲定领域语言时，**难以逆转的决策**落成 ADR
3. **to-spec 之后**：实现完 spec 删掉，但**关键决策沉淀成 ADR** 长期保留（spec 对比章节的"中间态"方案）

### 与 spec 的关系（第三条路）

```
OpenSpec spec   → 长期维护整份规范（重）
Matt spec       → 一次性交接文档（轻，用完即删）
ADR             → 只保留"决策+理由"，不保留整份规范（Matt 的"长期记忆"答案）
```

**规律**：Matt 不维护整份 spec，但他**维护决策记录（ADR）**。一个持续更新整份文档，一个只存档每个决策点——ADR 是他在"spec 长期持有"和"spec 用完即删"之间的第三条路：**要长期的是"为什么"，不是"长什么样"。**

---

## 🎯 实战案例汇总

### 案例 1：SlopWatch（监控平台）⭐ 完整实战实录

- **做什么**：自托管编程 Agent 监控平台，追踪 Claude Code/Codex/Pi/Open Code/Copilot CLI 的 token 消耗、会话质量、context window 占用
- **方法（1:47 直播全流程）**：
  1. 直播间收集项目点子 → 锁定「编程 Agent 监控平台」（78% 投票）
  2. **GrillMe** 诘问需求：主要用户？可见性模型？怎么接入？
  3. **sub-agent 并行研究 5 个编程 Agent** → 发现「每个 agent 有 hook + append-only JSONL，但 schema 全不同且演变，per-agent adapter 不可避免」
  4. **domain model skill（DDD）** 敲定统一语言：session / turn / model request / listener / server
  5. 架构定型：单二进制 + Postgres，自托管 on-prem
- **关键决策**：
  - **V1 认证**：admin 签发一次性 token（绑定 user record，可撤销），**OIDC 推 V2**——先验证想法不做重架构
  - **capture 无 daemon**：session start hook 拉起 listener 子进程（每会话一个），hook 配置 = 安装
  - **session 内部用 DAG 建模**：turns 是 DAG 节点（agent 会话可能分支）
  - live spectate 用 **5 秒轮询**，不做实时推送
- **亮点**：整场直播**没有写代码**，全靠诘问和领域建模把模糊想法推进到明确架构；名字 SlopWatch 来自观众 Eddie

### 案例 2：CVM 加 TikTok 创建器（Wayfinder 演示）

| 决策点 | 结论 |
|:------|:-----|
| TikTok 视频是什么 | 复用 videos 表，加 format 属性 |
| boolean vs enum | **enum**（standard/short）——3 boolean = 6 状态，enum 只有 3 状态 |
| 和 pitch 关系 | 独立，低摩擦不强制关联 |
| 渲染管线 | 抽独立仓库 + 本地 shell out |
| 发布方式 | Buffer 排队（绕过 TikTok 审核） |

### 案例 3：AI Hero CLI 精简（主流流程演示）

- **目标**：删掉大部分内部工具，只保留公开部分
- **流程**：grilling（6 问题）→ to-spec（46.1K→文档）→ to-tickets（3 个）→ implement → code-review
- **结果**：删 10 命令文件 + 3 测试，子 agent review 通过

### 案例 4：还原魔方（teach skill）

- **mission**：独立还原一次三阶魔方（目标=成就，不是速度）
- **教学**：第一课（结构/记号/白色十字）→ 进度追踪 → 角循环定制课（四步口诀 + 可点击交互）
- **关键**：最近发展区——每节课精准落在「正好被挑战、不被吓退」的区间

---

## 💡 深度洞察

1. **雾（fog）的比喻**：探索阶段到处都是不知道路怎么走的「雾」，Wayfinder 帮你找路，ticket 是在雾里跟踪进度的方式
2. **渐进式披露**：map → research ticket → thick docs，顺着链往下找，需要时才加载完整内容
3. **AI 的错位**：AI 很多错位在于「不问这些问题、不理解你的价值观、不理解你优先什么」——Wayfinder 解决的就是这个
4. **你（人类）是 lead，agent 是 junior**：规划阶段让模型保持专注，你得盯着、主导
5. **决策 ticket vs 实现 ticket**：决策 ticket 只能通过做出决策解决；实现 ticket 是决策在代码里落地
6. **Greenfield 项目在写代码前 = 研究任务**：直到想法被 reify 成代码，你本质上是在做研究（SlopWatch 直播）
7. **语言是你把程序带入生命的方式**：DDD 统一语言锁定心智模型；用具体场景 road test 语言（SlopWatch 直播）
8. **长期 Agent 记忆是坏主意**：想要可观测、可编辑、能塞进 context 的东西——ADR + 最小化 DDD 优于 RAG 记忆（SlopWatch 直播）
9. **别对 token 太抠门**：输入 token 极其便宜；人类 + AI 永远优于纯 AI
6. **Brownfield vs Greenfield 是胡扯**：Matt 认为没大家想的那么大差别
7. **开发者的时代机遇**：我们是第一批在 AI 极擅长的事情（写代码）上亲身体验 AI 的人——这个新空间的先行者

---

## 📖 推荐阅读

- **Ryan Singer《Shape Up》**（网上免费）——彻底改变了 Matt 做应用的方式，2019 年读后影响至今
- **Matt Pocock skills 仓库**：`npx skills@latest add mattpocock/skills`
- **SlopWatch 仓库**：https://github.com/mattpocock/slopwatch
- **Newsletter**：aihero.com（每次发布新 skill 第一时间知道）

---

## 🧩 你能直接带走的东西

### 如果你是独立开发者
1. 全局安装 Matt skills + symlink
2. 从 grill-with-docs 开始打磨想法
3. 遇到「它该长什么样」就用 prototype
4. 小活直接 implement，大活走 to-spec → to-tickets

### 如果你是团队
1. **项目级**安装 skills（大家共用一套）
2. issue 跟踪配 Jira/Linear（本就支持）
3. 每个 ticket 控制在聪明区内
4. code-review 用子 agent 独立上下文

### 无论你是谁
- **保真度思维**：别用文字硬聊「长什么样」的问题，上原型
- **聪明区纪律**：不贪多，保持 140K 内的清醒
- **有状态优先**：跨会话的事，让它记住
- **先跑起来再固化**：先用简单 harness 跑起来攒 trace，再把反复出现的流程固定成 Graph/Wayfinder map

---

## 关联笔记
- [[2026-08-02-Matt-Pocock-grill-skill的9个误区]] — grill 误区与参数化知识
- [[2026-08-01-Matt-Pocock-Wayfinder概念详解与FAQ]] — Wayfinder 概念与 FAQ
- [[2026-08-05-Matt-Pocock实战-用Claude-Code从零开发真实项目]] — 完整项目实战实录（ghost course）
- [[2026-08-06-Matt-Pocock-Agent-vs-Workflow科普]] — Agent vs Workflow 判定

- [[2026-07-31-10分钟讲透AI-Agent-8种主流架构]] — Agent 架构知识地图（AI架构师Leo）
- [[2026-07-31-Agent系统架构设计-Harness-Loop-Graph怎么选]] — Harness 工程化深入版
- [[2026-07-30-AI-Agent落地铁三角-Graph-Loop-Harness]] — 小红书版铁三角

> 💡 **一句总括**：Matt Pocock 的方法论本质是「**把 Agent 当成可编排的 junior 团队**」——用 GrillMe 诘问需求、Wayfinder 编排探索、prototype 提保真度、to-spec/to-tickets 拆解交付、teach 传承知识，而你自己始终是那个 lead，牢牢掌控着方向和边界。
