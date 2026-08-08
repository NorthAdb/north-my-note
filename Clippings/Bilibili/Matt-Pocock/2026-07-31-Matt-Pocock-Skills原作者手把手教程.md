---
created: 2026-07-31
updated: 2026-07-31
tags: [Matt Pocock, Skills, grill-with-docs, to-spec, to-tickets, implement, code-review, 教程, B站笔记, OpenCLI字幕]
source:
  - "https://www.bilibili.com/video/BV1SmNE6DE8v/"
  - "翻译：ChHsich | 原作者：Matt Pocock"
---

# Matt Pocock Skills 原作者手把手教程（官方主流流程）

> **UP 主**：ChHsich（搬运翻译） | **原作者**：Matt Pocock
> **时长** 17:17 | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI，302 段逐句）
> **证据等级**：`bilibili-ai-subtitles`

---

## 一句话总论

> **这是 Matt Pocock skills 仓库（16 万 star、750 万下载）的第一份官方教程。** 完整走一遍主流流程：`grill-with-docs →（可选 prototype）→ implement`，或跨多会话时 `→ to-spec → to-tickets → implement → code-review`。核心心法：**把每个 ticket 控制在「一个上下文窗口/聪明区」的粒度，用完即清空上下文。**

---

## 学完你应该获得什么

- 学会安装和配置 Matt Pocock skills（`npx skills@latest add mattpocock/skills`）
- 理解「主流流程」的完整链路和各环节的作用
- 掌握 spec 与 tickets 的区别（spec=最终长什么样，tickets=怎么走到那里）
- 知道「聪明区（smart zone）」≈ 140K 上下文上限，以及它的意义
- 明白为什么 code review 要用子 agent（主 agent 不擅长审自己写的代码）

---

## 安装与配置

### 安装：`npx skills@latest add mattpocock/skills`

| 步骤  | 说明                                                 |
| :-- | :------------------------------------------------- |
| 前提  | 已装 Node.js；npx 运行 Vercel 的 skills.sh CLI 安装器       |
| 发现  | 仓库有 **38 个 skills**，分两组：官方（质量够发布）+ 实验（可能删掉）        |
| 选择  | 空格勾选全部官方 skills → 回车                               |
| 配置  | 选择你的 agent（Claude Code / Cursor / Codex / Cline 等） |
| 范围  | 团队协作选**项目级**（大家共用一套）；独立开发者选**全局**                  |
| 链接  | 选 **symlink**（简洁清爽，不复制两份）                          |

**轻量性**：Matt 的 skills 大多需**手动调用**，描述简短精准，全部 skills 只占 **660 token** 上下文，非常轻量。

### 配置：`setup-mattpocock-skills`

| 配置项            | 选项                                                 |
| :------------- | :------------------------------------------------- |
| **Issue 跟踪工具** | GitHub / Jira / Linear / 本地 Markdown…（几乎无限，说配啥就配啥） |
| **Triage 标签**  | 用默认即可                                              |
| **领域文档**       | 单一上下文（99% 够用）vs 多上下文（大型 monorepo 用）                |

配置完成后写入 CLAUDE.md 链接，spec 和 issue 存到 scratch 文件。

### ask-matt 技能

Matt 把自己做成了 skill——知道关于 skills 仓库和「该先做什么」的一切。可以直接问「我该怎么开始」。

---

## 主流流程（默认链路）

```
grill-with-docs（盘问打磨想法，有状态：记录到 context.md + ADR）
      ↓
（可选）prototype —— 问题需要可运行的答案时
      ↓
小工作量 → 直接 /implement
      ↓
大工作量 / 跨多会话 → to-spec → to-tickets → 逐个 implement → code-review
```

---

## 各环节详解

### grill-with-docs — 把想法打磨成方案

- 通过一连串追问帮你想清楚（有状态，记录到 context.md 和 ADR）
- 把「我想改 X」变成一份清晰、站得住脚的方案
- 一次 grilling 一般 **~20 个问题**（看项目大小；演示里只问了 6 个）
- 用 **auto 模式**（不是 plan 模式）

### to-spec — 把讨论压缩成文档

- 把全部讨论（演示里 46.1K token）压缩成一份 spec
- 输出到 issue 跟踪工具（本地 Markdown）
- **spec = 对最终成品长什么样的完整描述**
- 内容：问题陈述、解决方案、用户故事、实现决策、测试决策

### to-tickets — 把 spec 拆成实现计划

- **每个 ticket 的粒度 = 一个上下文窗口 / 一个聪明区的大小**
- 真实例子：一份 spec 挂了 11 个子 issue（11 个 ticket），每个 ticket 很简短（验收标准在主 spec 里）

### implement — 逐个实现

- **一次只实现一个 ticket**，两个 ticket 之间清空上下文
- 手动操作：别说「把每个 ticket 都做了」，而是「去实现」，看是否到聪明区上限

### code-review — 最后的核对

implement 脚本自带 code review，基于**两个维度**：
1. 把已完成工作与原始 spec 对比（防漏做 ticket）
2. 对照仓库标准文档检查（没有就用经典规范，如 Martin Fowler 的代码异味）

**为什么用子 agent**：主 agent 自己刚写过这段代码，通常不擅长修改/改进自己写的东西；另起子 agent 用干净上下文窗口 review，效果好得多。

---

## 关键洞察：聪明区（Smart Zone）

> Matt 把 **140K 上下文** 当作「聪明区」上限。超过 140K，注意力退化，开始变笨、产生幻觉。

- 在这个区间内工作 = 模型表现最好
- 这就是为什么 ticket 要控制在「一个上下文窗口」大小
- 也是为什么要在 ticket 之间清空上下文

---

## 真实演示：AI Hero CLI 精简

- **目标**：删掉 CLI 里大部分内部工具，只保留对外公开部分
- **grilling**：约 6 个问题 → 方案：删 10 个命令文件、3 个测试、重新接线共享模块
- **to-spec**：46.1K 讨论 → spec 文档（含用户故事/实现决策/测试决策）
- **to-tickets**：3 个 ticket（演示说 3 个有点多，1 个就够）
- **implement + code-review**：42.7K 小工作，跑类型检查/构建/内部帮助检查，子 agent review 通过，提交到分支

---

## 自测题

1. 安装命令是什么？前提条件？
2. 团队协作该选项目级还是全局安装？为什么？
3. spec 和 tickets 的区别是什么？
4. 「聪明区」大约是多少上下文？超过会怎样？
5. 为什么 code review 要用子 agent？
6. 完整主流流程的链路是什么？
7. 什么时候可以跳过 to-spec 直接 implement？
8. 一次典型 grilling 大约多少问题？
9. Matt 的 skills 为什么上下文占用那么轻？
10. setup-mattpocock-skills 主要配置哪几项？

<details>
<summary>参考答案</summary>

1. `npx skills@latest add mattpocock/skills`；需已装 Node.js
2. 项目级——团队共用同一套 skills，一起维护决策
3. spec=最终长什么样；tickets=怎么走到那里
4. ~140K；注意力退化、变笨、产生幻觉
5. 主 agent 不擅长审自己刚写的代码；子 agent 干净上下文 review 更好
6. grill-with-docs →（prototype）→ implement，或 to-spec → to-tickets → implement → code-review
7. 工作小到单会话能完成时
8. 约 20 个（看项目大小）
9. 需手动调用 + 描述简短精准，全部只占 660 token
10. issue 跟踪工具、triage 标签、领域文档（单/多上下文）

</details>

---

## 证据与原文位置

- 字幕来源：agent-reach / OpenCLI `opencli bilibili subtitle BV1SmNE6DE8v`（302 段逐句带时间轴）

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1SmNE6DE8v/
- **BVID**：BV1SmNE6DE8v | **翻译**：ChHsich | **原作者**：Matt Pocock
- **原帖**：https://x.com/mattpocockuk/status/2075218406266036236
- **仓库**：`npx skills@latest add mattpocock/skills`
- **字幕**：agent-reach OpenCLI 提取 B站 AI 字幕（302 段，全程覆盖）
- ⚠️ B站 AI 字幕为英文音轨的翻译字幕，个别术语可能因音译有偏差，已结合语境校正

## 关联笔记

- [[2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] — Wayfinder 解决 spec 之前的探索阶段（本篇主流流程的前置）
- [[2026-07-31-Matt-Pocock实战-从原型到spec-driven开发]] — prototype skill 与保真度
- [[Matt-Pocock/2026-07-31-Matt-Pocock直播-SlopWatch从零搭建完整实录]] — Matt 的 GrillMe + DDD 方法论
