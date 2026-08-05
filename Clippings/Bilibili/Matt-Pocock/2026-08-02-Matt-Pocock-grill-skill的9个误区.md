---
created: 2026-08-02
updated: 2026-08-02
tags: [Matt Pocock, GrillMe, GrillWithDocs, Skill, PlanMode, 高保真, 低保真, 变笨区, Handoff, PRD, 追问式开发, B站笔记, YouTube字幕]
source:
  - "https://www.bilibili.com/video/BV1zn396mEfz/"
  - "原视频：https://www.youtube.com/watch?v=UzMNBN6xLLA"
  - "原作者：Matt Pocock | 搬运翻译：ChHsich"
---

# Matt Pocock：别让你的 Agent 问你 200 个问题 —— grill skill 的 9 个误区

> **原作者**：Matt Pocock | **时长** 24:58 | **事实主干**：YouTube 英文字幕（yt-dlp 提取，1228 段）
> **证据等级**：`youtube-transcript`
> **主题**：Grill Me / Grill with Docs 两个 skill 使用中的 9 个常见误区——追问式开发能不能发挥价值，取决于**回答问题的你**是否擅长规划。

---

## 一句话总论

> **Grill 类 skill 不是让你被动挨问的工具——它用不断追问替代 Plan Mode，但效果好坏 100% 取决于你（工程师）的规划能力：判断问题保真度、把握范围、主动掌控节奏。** 九个误区大多源于把「对话」当成了「采访」。

---

## 背景

- Grill Me / Grill with Docs 已发布一段时间，全球用户在用它**替代 agent 的 Plan Mode**
- 常见抱怨："Codex 问我 200 个问题"——Matt 听到会皱眉：这不是 skill 的问题，是用法的问题
- skill 本质：**毫不留情地追问你，直到双方对要构建的东西达成共识**
- skill 是"辅助你做工程师"，不是"替代你做工程师"——它的定位就是帮你，不是限制你

---

## 三个理解视角（Lens）

> 不懂这三个视角，就改不掉失败模式。

### 1. 高保真 vs 低保真问题（借自 Ryan Singer《Shape Up》）

| 类型      | 定义                 | 例子                             |
| ------- | ------------------ | ------------------------------ |
| **高保真** | 需要放大镜看细节/高保真原型才能回答 | 这个 UI 用起来什么手感？表单字段拆多页还是一张巨大表单？ |
| **低保真** | 不需要原型、直接回答         | 这个 URL 该放哪个路由？                 |

- **grillable**（可在追问会话回答）= 低保真问题
- **ungrillable**（不可在追问会话回答）= 高保真问题

### 2. 范围（Scope）与"变笨区"（Dumb Zone）

- grill 的东西太大 → 两个问题：
  1. 高保真问题藏在里面，不看到全貌答不了
  2. **撞上模型的变笨区**：约 **120k token** 起，SOTA 模型的注意力关系开始吃力、决策变蠢
- 对策：**提前让 agent 把大范围拆成小范围**，逐个 grill、回答所有问题
- 教训：不要试图"把未来几天的任务全部规划完"——没建在坚实基础上，排出来的是屎（crap）

### 3. 主动 vs 被动

- **太被动**：坐着让 agent 问 540 个问题、爆炸范围、问一堆太低保真的问题
- **太主动/太固执**：对低保真问题死磕到底，该动手建东西看效果时还不去写代码
- 原则：**这是对话，不是采访**——agent 问你，但由你决定走向、把握范围、保持轨道

---

## 九个误区（实践清单）

|  #  | 误区                                     | 正确做法                                                                 |
| :-: | -------------------------------------- | -------------------------------------------------------------------- |
|  1  | 在 grilling 会话里**硬答高保真问题**（ungrillable） | 用 **handoff skill** 交接给一个**原型会话**，高保真看完，再把学到的 handoff 回原 grilling 会话 |
|  2  | **范围定得太大**（scope too big）              | 先让 agent 拆成小范围，逐个 grill；在坚实基础上增量构建                                   |
|  3  | **无视变笨区**，让上下文涨过 ~120k token           | 盯紧 context window，避免模型在注意力压力下做蠢决策；必要时 handoff/compact                |
|  4  | **太被动**（让 agent 问 540 个问题、爆炸范围）        | 主动主导对话：你是对话的 leader，不是受访者                                            |
|  5  | **太主动/太固执**（对低保真问题无限 grilling）         | 该动手时就动手——需要看到东西动起来时，就去做原型/写代码                                        |
|  6  | **清空上下文再跑 to PRD**（把 grilling 的价值扔掉）   | grilling 里每个设计决策都价值连城（~10 万 token 好决策）；应**保留上下文**直接实现，或做成 handoff 文档 |
|  7  | **不记录决策 / 不做交接产物**                     | 决策要落成代码，或写进 handoff 文档（如 **to PRD** 的工程向交接文档），别丢                     |
|  8  | **用太笨的模型做 grilling**                   | grilling 依赖**参数化知识**（模型天生对系统的理解）→ 需要大模型；实现阶段反而多为上下文知识，可用笨一点的模型       |
|  9  | **一次只 grill 一个会话**                     | **并行 grill 2 个会话**，吞吐量直接翻倍（来回切换 = 同时管理两条 Slack 线程；熟练后可到 3–4 个）       |

---

## 关键概念展开

### 模型知识的两个来源（误区 8 的底层逻辑）

```
上下文知识（contextual）   ← 你传进 context 的：读文件、prompt、工具结果
参数化知识（parametric）   ← 模型训练时学到的对系统/应用的先天理解
```

- **grilling 靠参数化知识**：让模型用"先天理解"给你**天马行空的建议**、问出你没想到的问题——所以要用参数多、训练强的**前沿大模型**
- **实现阶段靠上下文知识**：到实现时通常已有详细实现计划、传入了相关代码文件，大多是"照着抄/照着改"→ 笨一点模型也够用

### Handoff 工作流（误区 1 的解法）

```
grill with docs 会话（蓝）
        │  遇到 ungrillable 高保真问题
        ▼
  handoff → 原型会话（新 context，做高保真原型）
        │  学到的东西
        ▼
  handoff 回原 grilling 会话 → 继续 grillable 问题
```

- 大多数会话长这样：grill with docs → handoff 到原型 → handoff 回原会话
- **to PRD**（原 2PRD）：把追问成果沉淀成**工程向交接文档**，多会话/单会话都可用

### 并行的具体节奏（误区 9）

- 交替模式：在会话 A 回答问题 → 切到会话 B（通常已轮到它）→ 回答 B → 切回 A
- 上限：一般 2 个；除非某个在做长任务（如 research）会尝试 3 个
- 这是 Matt 找到的**唯一能提升吞吐量、用更少时间做更多规划**的方法
- 预判：grilling 是**越练越强**的技能，熟练后可加吞吐和并行度

---

## 总结（Matt 的收尾）

1. grilling 主要是关于**问题**：低保真问题可答（grillable）；高保真问题不可答（ungrillable）→ handoff 到原型
2. 把握**正确范围**：grill 太多 = 烧穿 context、耗干耐力、一无所获
3. **被动** = 被问题淹没；**太主动** = 在低保真问题上无限内耗、不写代码
4. 用**聪明模型**做 grilling（依赖参数化知识）
5. **并行 grill 两个会话**，掌握后可到 3–4 个

---

## 证据与原文位置

- 字幕来源：yt-dlp 从 YouTube 原视频提取英文原声字幕（`youtube.en.vtt`，1228 段）
- 原视频：https://www.youtube.com/watch?v=UzMNBN6xLLA
- 字幕存档：`原始材料/BV1zn396mEfz_archive/subtitles/transcript/youtube.en.vtt`

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1zn396mEfz/
- **BVID**：BV1zn396mEfz | **原作者**：Matt Pocock | **搬运翻译**：ChHsich
- **字幕**：B站侧无可提取 AI 字幕（无外挂/智能字幕），故用 yt-dlp 提取 YouTube 原版英文字幕，内容完整
- ⚠️ 视频标题称"9 个误区"，正文按三个视角展开 + 实践清单，个别条目在原文中为两两组合（如被动/主动、清空上下文/不记录决策），此处按九条整理
- ⚠️ 英文原声字幕含 YouTube 自动生成的重复行，笔记已去重整理

## 关联笔记

- [[2026-08-01-Matt-Pocock-Wayfinder概念详解与FAQ]] — Wayfinder 是 GrillMe 的**编排器**进化版（grilling 会话的调度）
- [[2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] — Wayfinder 直播实战演示
- [[2026-07-31-Matt-Pocock直播-从零搭建项目-SlopWatch]] — GrillMe + DDD 方法论（Wayfinder 前身）
- [[Matt-Pocock-Skills仓库全量分类汇总未命名]] — Matt Pocock skills 仓库全景
