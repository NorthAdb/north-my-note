---
created: 2026-08-01
updated: 2026-08-01
tags: [Matt Pocock, Wayfinder, Skill, Ticket, Fog of War, Frontier, Spec, 跨会话规划, 概念讲解, B站笔记, OpenCLI字幕]
source:
  - "https://www.bilibili.com/video/BV1VoGK6kEQj/"
  - "原作者：Matt Pocock"
---

# Matt Pocock 实战教程：Wayfinder 概念详解与 FAQ

> **原作者**：Matt Pocock | **时长** ~13 分钟 | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI，234 段逐句）
> **证据等级**：`bilibili-ai-subtitles`
> 🔗 **互补视频**：[[2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] —— 那篇是**直播实战演示**（跑一个真实需求），本篇是**概念讲解 + FAQ**（Wayfinder 是什么、怎么用、何时用）

---

## 一句话总论

> **Wayfinder 是一个把"超大块工作"建模成"地图"的 skill：起点是模糊想法，终点由你定，中间是战争迷雾，每个待决策点是一张 ticket，分布在 issue tracker 里由 skill 跨多次会话编排。** 它解决的问题是——单次会话的 context window（尤其 smart zone）塞不下大任务，而旧方法（Grill with Docs）把工作预先拆小块、在单次会话里推进，一遇迷雾就卡住。

---

## 为什么需要 Wayfinder（Motivation）

- 已有/自己做的规划工具都**太受限、太绑死单次会话**
- 有些工作大到塞不进 context window（尤其是 agent 的 **smart zone**），而且你在开始之前就知道这点
- 旧做法：会话前先手动拆小块（"我就先啃这一小口"），推进过程中撞上回答不了的问题，**迷失在迷雾里**，全程还要管理 smart zone、省 token
- 反过来的结果是"把要做的东西**削足适履去迁就 AI**"，这感觉不对
- Wayfinder 没有这个限制：**可以规划超大块工作，跨多种会话编排整个规划过程**，理解决策依赖，甚至允许并行规划

---

## 核心概念：地图（Map）隐喻

```
   起点（模糊想法） ── 迷雾 ── 迷雾 ── 终点（大概知道自己想去哪）
        │                                    ↑
        └── 每个决策点 = 一张 ticket（单独一次 agent 会话）
```

- 你有一个起点（模糊的想法，不清楚怎么到达）和一个终点（大概知道想结束在哪），**中间步骤超级模糊**
- 大地图上的每一样东西都是一张 **ticket**；每张 ticket 需要和 agent **单独开一次会话**（原型会话 / grilling 会话 / 调研会话）
- Wayfinder 只靠一个 skill 就编排所有这一切，**跟任何 coding agent 都能配合**
- 第一件事通常是先做一次 grilling 会话（让 AI 采访你，搞清楚基本前提）；有的工作到这就能直达终点，很多则仍身陷迷雾 → 继续加原型/调研/grilling 会话

### Frontier 与战争迷雾（Fog of War）

- **Frontier**：Wayfinder 掌握的一批"现在就能做"的决策集合
- **迷雾里**：还没法决定的事——因为还没做调研、还没原型可看、或 grilling 谈得不够
- 推进方式：解决一张 ticket → 看看**打开了哪些新 ticket** → frontier 移到哪
- 到某个时刻迷雾全部解决，做出足够多决策，最终到达终点

---

## 四种 Ticket 类型（比直播笔记多一种：task）

| 类型 | 用途 | 说明 |
|:-----|:-----|:-----|
| **Research** | agent 去找信息并带回来 | 通常在 **sub-agent** 里立刻开干，你不需要盯着，做完回报 |
| **Prototype** | 建原型获得**高保真反馈** | 规划时让东西真正活起来，无比珍贵；复用他的 prototype skill |
| **Grilling** | 讨论计划的某方面/实现细节 | 针对需要做决策的具体问题 |
| **Task** | 真实世界里要做的事 | agent 自己做不了的（联系某人/跑一趟办事），或 agent 能做但排在后面 |

> - ticket 类型会带进 issue tracker（如 "Wayfinder research" 标签），不只是口头概念
> - Wayfinder 在 ticket 之间建立 **blocking 关系**——有些决策只能在其他决策做出之后才能做

---

## 怎么用（Workflow）

1. **开新会话**：先决定**终点**（想要什么样的产出）。例：给 CVM 应用加 command palette，终点 = buildable spec
2. **初始 Wayfinder prompt**：描述你想要什么 → 它探索仓库、调用 grilling skill 问你要什么（会问"done 长什么样"、推荐用 spec）→ 创建第一批 ticket 和第一张 map（其余作为 sub-issue）
   - 真实例子：立刻就有 **7 张 ticket，但只有 3 张现在就能做**（icon 名字来源 / 组件存储 schema / palette 信息架构和 grid 键盘）
3. **逐张处理 ticket**：在**新会话**里对**具体的 ticket** 调用 Wayfinder（Wayfinder + ticket URL）
   - 进阶：用 **handoff skill** 自动写好 prompt 并 spawn 一个 Claude subagent
4. **解决后看 frontier 移动**：好，解决了那张 ticket，打开了哪些新 ticket？
5. **map 完成后**：toSpec 把 map 转成 spec → toTickets 拆实现 ticket → 实现每张 → code review

> 注意：map 太密时可能不好直接生成 spec——Matt 喜欢先 `toSpec` 把决策全部拉进一份长文档（他实际生成的 spec **初稿大到超出 GitHub 字符数限制**），然后再 `toTickets`。流程是 **toSpec → toTickets**，不是直接拿 map 开跑。

---

## Wayfinder 在 Matt 工作流中的位置

```
Grill with Docs → toSpec → toTickets   （旧，单次会话）
          ↓
Wayfinder map → toSpec → toTickets → 实现 → code review   （新，跨会话）
```

- Wayfinder **正好嵌在 Grill with Docs 所在的位置**：不是在单次会话里 grilling，而是花多得多的时间创建巨大的 map
- 生成的 spec **极其密，且全部链回原始决策 ticket**——你（或 agent）在搞不清任何东西时能看**第一手原始资料**
- 这正是 Grill with Docs 的弱点：你依赖 spec 作为真相来源，但 spec 只是会议内容的**总结**；Wayfinder 让你拿到原始决策

---

## 关键观点：spec 是"非持久化"的

- spec = **一份跨多次会话工作的终点文档**，让到达终点时能搞清楚本来要去哪
- 与 **spec-driven development（SDD）** 的本质区别：
  - SDD 的人会把 spec 当长期真理，回去改 spec、维护 spec
  - Matt：**spec 一旦体现在代码里就可以删掉**——关掉装 spec 的 issue，spec 就没了，极少甚至从不再翻它
- 每个实现会话都在一张实现 ticket 里完成（决策 ticket ≠ 实现 ticket）

---

## FAQ（什么时候用 / 是不是 SDD / 是不是瀑布流）

**Q：流程太多了，对我做的工作太重，什么时候该用它？**
- 工作能在**单次会话内完成和规划** → 就在单次会话里规划
- 大概**已经知道去终点的路** → 没必要用（单次会话自己摸过去搞定）
- **有战争迷雾、完全不知道往哪走** → 用 Wayfinder（先开始，看能走到哪）
- 非编码任务也在用：规划 garden office（花园办公室）——安排场地勘测、搞清该联系谁、找不同施工公司

**Q：这是 spec-driven development 吗？我不想搞 SDD**
- 不是：Matt 的 spec 是**非持久化**的终点文档，实现完就删；SDD 会回去维护 spec

**Q：这不是瀑布流吗？**
- **原型就是防止瀑布流的办法**——大量低保真的前期规划 + 高保真原型验证（Wayfinder 鼓励你建那么多原型，产出质量好得惊人）

---

## 关键洞察

1. **终点完全由你定**：可以让它生成 spec 交给 AFK agent 去跑（Matt 的默认做法）；也可以只让 task 直接替你实现工作
2. **issue tracker 中立**：GitHub / Linear / JIRA 都行，通过 `setup-matt-pocock-skills` 配置；GitHub issues 只是他个人的默认
3. **决策会写回父级 map**：sub-issue 解决后（如"发布竞态时关闭 clips"），解决方案会写回父级 map，Wayfinder 记录所有已做的决策/原型/task
4. **真实数据**：一个公开 map 下有 12 个子任务/sub-issue；某个 map "17 张里做完了 14 张"，但围绕的 skill 还没建——建完还得回头处理派生事项
5. **John 效应**：有用户因为太喜欢 Wayfinder 思路，自己搭了一套 harness 并做了"星图"可视化；有人"一次原型搞定，一连好几个月反复从头开始"

---

## 证据与原文位置

- 字幕来源：agent-reach / OpenCLI `opencli bilibili subtitle BV1VoGK6kEQj`（234 段逐句带时间轴）
- 字幕全文：`原始材料/BV1VoGK6kEQj_archive/subtitles/`

## 来源、覆盖与局限

- **URL**：https://www.bilibili.com/video/BV1VoGK6kEQj/
- **BVID**：BV1VoGK6kEQj | **原作者**：Matt Pocock
- **字幕**：agent-reach OpenCLI 提取 B站 AI 字幕（234 段，全程覆盖）
- ⚠️ B站 AI 字幕为英文音轨的翻译字幕，个别术语可能因音译有偏差（Wayfinder/spec/ticket 等已结合语境校正）
- ⚠️ 本篇为概念讲解，未演示完整实操流程；实操看互补的直播演示笔记

## 关联笔记

- [[2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] — Wayfinder **直播实战演示**（理念 vs 实操，两篇互补）
- [[2026-07-31-Matt-Pocock直播-从零搭建项目-SlopWatch]] — GrillMe + DDD 方法论（Wayfinder 前身）
- [[Matt-Pocock-Skills仓库全量分类汇总未命名]] — Matt Pocock skills 仓库全景
