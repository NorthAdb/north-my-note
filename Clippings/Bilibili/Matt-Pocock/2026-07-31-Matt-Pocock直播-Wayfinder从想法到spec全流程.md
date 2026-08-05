---
created: 2026-07-31
updated: 2026-07-31
tags: [Matt Pocock, Wayfinder, GrillMe, Skills, Spec, Ticket, Agent工作流, DDD, SlopWatch, B站笔记, OpenCLI字幕]
source:
  - "https://www.bilibili.com/video/BV1q5KU6UEpW/"
  - "翻译：ChHsich | 原作者：Matt Pocock"
---

# Matt Pocock 直播实战：Wayfinder 从想法到 spec 全流程

> **UP 主**：ChHsich（搬运翻译） | **原作者**：Matt Pocock
> **时长** ~1 小时 | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI，777 段逐句）
> **证据等级**：`bilibili-ai-subtitles`

---

## 一句话总论

> **Wayfinder 是 GrillMe 的进化版——不是一次 grilling 会话，而是 grilling 会话的编排器。** 它把「一个宽泛需求」拆成多个聚焦的决策 ticket（grilling / research / prototype），并行推进、逐个收敛，最终产出一份链回所有决策的详尽 spec，直接交给实现 Agent。它解决的是 **spec 之前**的探索阶段。

---

## 学完你应该获得什么

- 理解 Wayfinder 在「想法 → spec → tickets → 实现」流程中的定位
- 掌握「决策 ticket」「research ticket」「prototype ticket」三种工作单元的区别
- 学会用「map」跟踪雾中进度、发现决策前沿、处理 blocking 关系
- 知道为什么 Matt 用 Opus 4.8 Medium 而不是更强的模型
- 理解「Agent = Model + Harness + 环境」，以及为什么优化 harness 收益更大

---

## Wayfinder 是什么

| 维度 | 说明 |
|:----|:-----|
| **定位** | GrillMe 技能的进化版，解决 spec 之前的探索阶段 |
| **本质** | grilling 会话的**编排器**（orchestrator），不是单个 grilling |
| **工作方式** | 口述需求 → 扫描代码库 → 绘制 map → 拆成聚焦的小会话 → 并行推进 |
| **产出** | 一份决策完备的 spec + 链回所有决策的 map |
| **自动降级** | 如果判断不需要 map，就用普通 grilling 会话（grill with docs）直接做 |

**Matt 的原话：**

> "Wayfinder 的思路是，我们用它生成一个 spec，然后再去实现。"
> "它试图找出决策的前沿，然后带你一路走过去。"
> "它不是一个 grilling 会话，而是 grilling 会话的编排器。"

---

## 完整工作流（从想法到 spec）

```
1. 口述需求（打开 Wayfinder，第一次口述）
      ↓
2. Wayfinder 扫描代码库，绘制 map（找到路径、解决悬而未决的决策）
      ↓
3. 生成决策 ticket（grilling / research / prototype 三类）
      ↓
4. 并行处理：research 派子 agent 自动跑，prototype 提保真度
      ↓
5. 逐个收敛决策，关掉 ticket，记录 resolution
      ↓
6. 生成 spec（to spec），链回所有决策和 research 文档
      ↓
7. to tickets → 拆实现子任务 → 单独 context 实现 → code review → 人工 review
```

---

## 三种 ticket 类型

| 类型                   | 含义         | 例子                                            |
| :------------------- | :--------- | :-------------------------------------------- |
| **Grilling ticket**  | 需要讨论决策的问题  | 「TikTok 创建流程怎么发起？」「format 是 enum 还是 boolean？」 |
| **Research ticket**  | 调查事实、研究方案  | 5 个编程 Agent 的 hook/JSONL；TikTok 上传 API        |
| **Prototype ticket** | 动手做原型、提保真度 | 竖屏创建视图 A/B 变体                                 |

> **关键区分**：Wayfinder 的「决策 ticket」只能通过做出决策来解决；ToTickets 创建的「实现 ticket」是决策在代码里的落地——两者不同。

---

## 案例：CVM 加 TikTok 创建器

Matt 用自己真实的 Course Video Manager（CVM）做演示，目标：加一个 TikTok 创建器。

### 关键决策过程

| 决策点 | 纠结 | 最终结论 |
|:------|:-----|:---------|
| TikTok 视频是什么？ | 独立视频 vs 衍生 | **复用 videos 表，加一个新属性** |
| 加什么属性？ | boolean vs enum | **format enum**（standard / short），不用 boolean |
| 和 pitch 的关系？ | 关联 vs 独立 | **独立**，低摩擦，不强制关联 pitch |
| 顶层 UI 放哪？ | 新侧边栏 vs 混入现有 | **独立 TikToks 侧边栏项** |
| 渲染管线放哪？ | 拉进 monorepo vs 独立仓库 | **抽成独立仓库，本地 shell out**，暂不合并 |
| 发布方式？ | 直接 TikTok API vs Buffer | **Buffer 排队**（绕过 TikTok 审核，付小钱省麻烦） |

### 为什么用 enum 而不是 boolean（工程经验）

Matt 的观点：新手容易用 `boolean isShort`，但万一有第三种类型，boolean 就爆炸了（三个 boolean = 6 种状态）。**enum 只有 3 种状态**（standard / short / future）。这是他工程经验在发挥作用——也是 AI 不问这些问题时容易踩的坑。

---

## Agent = Model + Harness + 环境

Matt 的核心框架：

```
        [ 环境 ]    ← 文件系统、代码库
    [   Harness  ]  ← Claude Code、Codex、Claude AI、ChatGPT
      [  Model ]    ← Opus、GPT 5.6
```

**观点**：
> "大家都在讲换 model、提升 model，但这只是图中的一小块。还有一种免费的做法——把 harness 做得更好，或者把 agent 所处的环境做得更好。所有这种'换个 model 就好了'的说法，说实话，在环境和 harness 上下功夫，能拿到的收益多得多。"

**模型选型**：全程用 **Opus 4.8 Medium**（grilling 完全够用，不需要 Fable/更聪明的模型）。effort 权衡：medium 烧 token 少、延迟低，质量收益递减。

---

## 实战细节与工具

| 工具/技巧 | 说明 |
|:---------|:-----|
| **prototype 技能** | 提保真度，做出可看的 UI 变体，用悬浮条切换 A/B |
| **research 子 agent** | research 任务直接派子 agent 并行跑，自己汇报 |
| **Claude Code Agents 视图** | 顶层查看多 worker，管理并行会话 |
| **worktree** | 每个 prototype 在独立 worktree，但跨 worktree 状态同步很痛苦（想要沙箱/云主机） |
| **渐进式披露** | map → research ticket → thick docs，顺着链往下找 |
| **Smrt zone** | 控制日常任务在 150K token 以内 |
| **vibe coding** | 原型阶段可以 vibe coding，spec 才是要发布的东西 |

---

## 关键洞察

1. **你（人类）是 lead，agent 是 junior**：规划阶段要让模型保持专注，你得盯着、主导——「很多人在这个阶段对 agent 太被动了」
2. **Wayfinder 不依赖 GitHub**：可以用 Linear、Todoist、Jira 任意工具，GitHub issues 只是默认
3. **决策 ticket 之间冲突的处理**：让两个会话对着干，看哪个先做，做 agent 间通信；通常 blocking 关系能处理好
4. **「雾」的比喻**：探索阶段到处都是不知道路怎么走的「雾」，Wayfinder 帮你找路，ticket 是在雾里跟踪进度的方式
5. **AI 的错位**：AI 的很多错位在于「不问这些问题、不理解你的价值观、不理解你优先什么」——Wayfinder 解决的就是这个

---

## 评论与补充

- 完整流程链：**Wayfinder（探索）→ to spec（to spec 是 2PRD 的新名字）→ to tickets → 实现 → code review → 人工 review**
- spec 产物：极其详尽的 spec，链回所有决策 + research 文档，还有 spec diagrams 页面
- 一个 spec 的落地示例：6 个子任务 + GitHub action 在不同 session 跑完，产出 ~5000 行代码的 PR
- **Brownfield vs Greenfield 是胡扯**：Matt 认为没大家想得那么大差别
- 团队协作：不推荐本地 markdown 做 .scratch 协作

---

## 自测题

1. Wayfinder 和 GrillMe 的关系是什么？
2. Wayfinder 的三种 ticket 类型分别解决什么？
3. 为什么 format 用 enum 而不是 boolean？
4. Agent = Model + ? + ? 的完整公式是什么？
5. Matt 为什么用 Opus 4.8 Medium 而不是更强的模型？
6. 「决策 ticket」和「实现 ticket」的区别是什么？
7. Wayfinder 的完整工作流链路是什么？
8. 两个 grilling 会话决策冲突时怎么办？
9. 「雾」在 Wayfinder 语境下指什么？
10. 为什么说「换 model」不如「优化 harness 和环境」？

<details>
<summary>参考答案</summary>

1. Wayfinder 是 GrillMe 的进化版——grilling 会话的编排器，不是单个 grilling
2. grilling=讨论决策；research=调查事实；prototype=做原型提保真度
3. boolean 扩展性差（3 个 boolean=6 状态），enum 只有 N 种状态
4. Agent = Model + Harness + 环境
5. effort 权衡：medium 质量足够、token 少、延迟低，质量收益递减
6. 决策 ticket 靠做出决策解决；实现 ticket 是决策在代码里的落地
7. 口述→map→决策 ticket→并行处理→收敛→to spec→to tickets→实现→review
8. 让两个会话对着干，先做的先完成，做 agent 间通信
9. 探索阶段不知道路怎么走的状态；ticket 是雾里跟踪进度的方式
10. model 只是图中一小块，harness 和环境是「免费的改进」，收益更大

</details>

---

## 证据与原文位置

- 字幕来源：agent-reach / OpenCLI `opencli bilibili subtitle BV1q5KU6UEpW`（777 段逐句带时间轴）
- 字幕全文：`原始材料/BV1q5KU6UEpW_archive/subtitles/`

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1q5KU6UEpW/
- **BVID**：BV1q5KU6UEpW | **翻译**：ChHsich | **原作者**：Matt Pocock
- **字幕**：agent-reach OpenCLI 提取 B站 AI 字幕（777 段，全程覆盖）
- ⚠️ B站 AI 字幕为英文音轨的翻译字幕，个别术语可能因音译有偏差（Wayfinder/GrillMe/Remotion 等已结合语境校正）
- ⚠️ 视频直播节奏快、有多人发言，部分对话段落上下文需要结合画面理解

## 关联笔记

- [[2026-08-01-Matt-Pocock-Wayfinder概念详解与FAQ]] — Wayfinder **概念讲解 + FAQ**（本篇是直播实战演示，两篇互补）
- [[2026-07-31-Matt-Pocock直播-从零搭建项目-SlopWatch]] — Matt 的另一场直播，讲了 GrillMe + DDD 方法论（Wayfinder 的前身）
- [[2026-07-31-10分钟讲透AI-Agent-8种主流架构]] — Agent 架构知识地图
- [[2026-07-31-Agent系统架构设计-Harness-Loop-Graph怎么选]] — Harness 概念深入版（与「Agent=Model+Harness+环境」呼应）
