---
created: 2026-08-05
updated: 2026-08-05
tags: [Matt Pocock, Claude Code, GrillMe, DDD, 统一语言, AFK Agent, Sandcastle, Ralph Loop, QA, 白班夜班, B站笔记]
source:
  - "https://www.bilibili.com/video/BV1GPuw6DEGN/"
  - "翻译：ChHsich | 原作者：Matt Pocock（@mattpocockuk）"
  - "原视频：https://www.youtube.com/watch?v=hX7yG1KVYhI"
evidence: bilibili-ai-subtitles
---

# Matt Pocock 实战：用 Claude Code 从零开发一个真实项目（ghost course 全流程）

> **UP 主**：ChHsich（搬运翻译） | **原作者**：Matt Pocock（Total TypeScript 创始人）
> **时长** 1:31:26 | **事实主干**：B站 AI 字幕（agent-reach / OpenCLI，734 段逐句）
> **证据等级**：`bilibili-ai-subtitles`
>
> **证据说明**：字幕为 AI 转录+人工校对的中英双语。视频为 Matt Pocock 在真实项目 course-video-manager 上开发 ghost course 功能的全程实录。字幕中部分数字（提交数、issue 数、token 数）为口述，未逐一核验仓库。

---

## 一句话总论

> 这是 Matt 的「**白班夜班**」模式的完整演示：人上白班——用 GrillMe 把模糊想法磨成 8 个明确要点、用 DDD 统一语言沉淀术语、产出 PRD 并拆成 GitHub issue；LLM 上夜班——AFK agent 在 Docker 沙箱里用 Ralph loop 逐个实现并关 issue；人醒来后走 QA 循环，发现问题再反馈成 issue，agent 在后台持续修复。

**关键结论**：难的部分（想清楚、人在回路的部分）做完后，剩下的实现工作基本可以放心交给 AFK agent；而 QA 反馈循环里发现的边界问题，是「从 spec 到代码」路线永远规划不到的。

---

## 一、项目与需求背景

### course-video-manager 是什么

Matt 日常内容创作（录视频、组织课程、写文章、封面编辑）的统一入口：

- 技术栈：React Router + TypeScript + Node、Drizzle ORM、Postgres、Vitest、Effect
- 规模：约 **1200 次提交、关闭 637 个 issue**
- 部署：不部署、本地 `npm start`（彩蛋：本视频就是用它制作的）

### 本次要做的功能（三条模糊需求）

1. **直接创建 real lesson**（不必先建 ghost 再 create on disk）
2. **直接删除 real lesson**（不必先转 ghost 再删）
3. **创建 ghost course**（文件系统上还不存在的课程，自由规划）

### 已有概念

- **ghost lesson**：只存在于数据库、磁盘上不存在的课程单元；右键 "create on disk" 才创建，可再转回 ghost（删除磁盘文件）
- **real lesson**：磁盘上真实存在的课程单元

---

## 二、GrillMe：22 分钟把想法磨成 8 个要点

### 为什么「为什么」很重要

> 向 LLM 解释**为什么**特别重要——知道做什么只能理解要建什么；知道**为什么**，它才能给你替代方案。

Matt 用语音输入口述想法直接丢给 `grill me`，进场第一件事是**验证想法、打磨想法**。

### GrillMe 的完整工作流程

1. **粗糙输入**：语音口述未打磨的模糊需求，不加整理直接丢给 LLM
2. **explore 阶段**：sub-agent 在自己的上下文窗口里大量读代码库文件，只回传总结（非常省 token）
3. **挑战痛点描述**：主动质疑你的问题描述，用代码证据纠正假设
   - 例：它读代码发现 `CourseWriteService.deleteLesson` 已能直接删 real lesson，问题其实是 **UI 缺口**
4. **Socratic 式逼问**：把含糊的话一点一点展开，拆成 A/B 方案并列出 trade-off，甚至给推荐
5. **先看代码再问人**：skill 内置规则——"如果一个问题靠看代码就能回答，它就先去看代码"
6. **双向驱动**：有时 AI 驱动追问，有时开发者驱动（"你能把两种方案的取舍讲讲吗？"）
7. **产出清单并确认**：结尾列出"我理解的完整范围"逐条确认，等你点头或补漏
8. **不用内置提问工具**：刻意不用 Ask User / Question 工具——①不喜欢那个 UI；②有"调用工具 vs 不调用工具"选择时不调用永远更省 token（工具调用要用 JSON 包裹）
9. **产出即流水线素材**：问答形式信息密度极高（问题和答案放一起，对 LLM 注意力机制是热点）

### 22 分钟磨出的 8 个要点

1. courses 表的 **file path 改为可空**（唯一 schema 改动）
2. ghost course 的创建
3. 新的创建课程流程：**只问名字，不要文件路径**
4. ghost course 的 UI：隐藏 publish 和 export 操作
5. "创建 real lesson"按钮在 real 和 ghost course 里都可用；ghost course 里触发 modal 先问文件路径，然后 **cascade materialize**（课程变 real → section materialize → lesson 在磁盘创建，一气呵成）
6. ghost lesson 的创建保持不变，到处可用
7. **real lesson 的直接删除**：一步清掉磁盘和数据库；同时保留"转成 ghost"选项
8. ghost lesson 的删除保持不变

### 关键边界决策（GrillMe 逼出来的）

- real course 一旦有文件路径**永远是 real**（磁盘上的东西不会消失，不会变回 ghost course）
- section 状态由 lesson 推导：含任一 real lesson 即 real section（本质是磁盘目录）
- 拒绝"比 ghost 更丰满的中间状态"
- 应用不负责创建仓库，只指向已存在目录
- 打磨 22 分钟后上下文仅 **~40k token**（explore 总结机制的功劳）

---

## 三、DDD 统一语言：让 AI 协作更精确

### 理念

Matt 在读《领域驱动设计》（Eric Evans），把 DDD 的**术语表（glossary）**重新解释为弥合「领域专家（他）与开发者（LLM）」之间的鸿沟——人和 AI 需要共同语言才能精确交流。

### 落点

一份统一语言文档存在仓库里；"每当 LLM 搜索相关内容时就会碰到这份文档"，同时服务于人类阅读与 AI 检索。

### 本次新增的术语

- **ghost lesson / ghost section / ghost course** 三兄弟
- **materialize**（动词）：把 ghost entity 转成 real entity——通过创建其磁盘表示的动作
- **create on disk**（动作）+ 文档里列出"要避免的别名"
- **materialization cascade**：在 ghost course 里 materialize 一个 lesson 引发的连锁反应（分配文件路径 → materialize section → materialize lesson）

### 维护方式

grilling 产出新想法后，让 LLM 更新统一语言文档，**先接受、之后再 review**；废弃概念也在文档中标记（plan / plan section / plan lesson 已废弃，ghost course 是新的做法）。

### 收益

- 之后可以说"materialization cascade 里有个 bug"，LLM 完全听懂
- 统一语言让函数命名变得容易
- 接口设计（如 course write service 加新方法）更自然

---

## 四、PRD → issues：把范围拆成可执行单元

### write PRD skill

- 直接把 grilling 对话转成更精炼的总结文档（问题+答案在一起，信息密度高）
- **先勾勒主要模块**再写 PRD（"我不一定需要看进模块内部，我只想知道它们怎么变"）
- PRD = 用户故事 + 实现决策 + **测试决策**（测试决策让 LLM 更可能走 TDD、建反馈回路）
- Matt 凭信心接受 PRD 不 review（"LLM 做总结特别在行"）

### PRD to issues

- PRD 刚写完还在上下文里，直接拆 issue
- issue 带**阻塞关系**（blocked by）、验收标准、链接父 PRD
- 任务粒度控制：太小不值得启动一个 agent（隐藏 publish/export 这种小任务合并掉）
- 最终 6 个合并成 **4 块**：ghost course 创建、UI 相关、直接删除操作、materialization cascade
- 粒度参考：一个 agent 启动一次有固定成本，任务要「不太大、也不太小」

---

## 五、AFK Agent：Sandcastle + Ralph loop

### 定位

AFK（Away From Keyboard）agent = 人在离开时后台运行的 Claude Code。Matt 过去 24 小时打磨的配置，暂定名 **Sandcastle**。

### Sandcastle：Docker 隔离执行环境

```text
Dockerfile
  → 起 Docker 容器
  → 工作目录挂载进容器
  → 容器内跑 Claude Code 做提交
  → 把提交作为 patch 拉出来
  → 应用到本地仓库
```

- 配置极其灵活：一份 Dockerfile + 一个 prompt，可以一遍又一遍跑 Ralph loop
- **隔离**：agent 的提交不直接污染本地仓库，作为 patch 审查后应用

### Ralph loop：实现 → 测试 → 关 issue

```text
pnpm ralph
  → 拉起 AFK agent，最多 100 次迭代
  → 拉取所有 GitHub issue，选一个干
  → 每次提交都跑测试和类型检查
  → 做出提交 → 关闭对应 issue
  → 循环直到 issue 用光
```

- 本次结果：5 次干净的 agent 运行 → **6 次提交**，留下漂亮详细的提交信息
- 关键：**每次提交都跑测试 + 类型检查**，否则 agent 会越走越偏；大多数 loop 里它也在补测试（更新 reducer 测试覆盖新 action）

---

## 六、QA 验收循环：人在回路 + 后台修复

### 流程

1. 开新 Claude session：拿最近 5 次提交 → 生成 **QA 计划**（逐步指南，怎么测试新实现的每一部分）→ 存成 GitHub issue
2. 人（Matt）自己走一遍 QA 计划
3. 发现问题的反馈路径：**反馈按钮** → 详细描述 → haiku 生成标题 → 自动创建 GitHub issue（带来源路由和反馈原文）
4. Ralph loop 在后台接走这些 issue 去修，人继续 QA
5. 需要人的任务：给 issue 加 label 或注明"人在回路"，agent 不会碰

### 本次 QA 发现的问题（值得注意）

- add course modal 出现两个 tab 建 real/ghost course → 希望**只静默创建 ghost course**（UI 里不该出现 ghost 这个词）
- 创建 ghost course 后没跳转新页面、modal 没关、出现 minified React error、按钮无 loading 状态
- 右键菜单"创建 Ghost Lesson / 创建 Real Lesson"两个按钮不如一个 modal + **"同时在文件系统上创建"复选框**（这类问题不实际看到 UI 无法判断——GrillMe 阶段规划不到）
- 删除 real lesson 要加**确认 modal**（防误删）
- **showstopper**：如果 course 仓库不是 Git 仓库，相关操作失败时应该**回滚已创建的目录**，否则目录/文件系统和数据库不一致

> 💡 **核心洞察**：有些问题在设计/原型阶段无法预见，只有在 QA 循环里撞上——"从 spec 到代码这条路永远走不通，因为当你朝某个东西迭代时，会撞上很难提前规划到的小而诡异的边界情况"。

### 数字

- QA 8 分钟创建 6-7 个 issue
- 整个功能构建约 **14 次提交**，约 30 分钟跑完一轮
- 灵活 backlog 的好处：任何时候都能往里塞一堆 bug 修复，agent 就会进去修

---

## 七、白班夜班模式（Day Shift / Night Shift）

> 朋友 **Jamin** 在 Twitter 上的比喻：**我上白班**——想点子、和 LLM grilling、把想法变成 PRD、PRD 变成 issue；**LLM 上夜班**——Claude 去 AFK 实现这些东西，我去歇一会儿、散个步。

**为什么这不慢**：

> 它慢，是因为你在努力把想法从人脑里榨出来。而在这发生的同时，AFK agent 在后台实现你之前 grilling session 的产出。**一旦想法理清楚，我们的工作其实就差不多做完了，直到要去 QA 产出。**

**白班内容**（人的时间线）：

1. 日历里留半小时专门想清楚要做的功能
2. GrillMe grilling（22 分钟）→ 8 个要点
3. 更新统一语言文档（接受后 review）
4. write PRD（不 review）
5. PRD → issues（不 review，已预 review 过）
6. 启动 pnpm ralph，走开

**夜班内容**（agent 的时间线）：Sandcastle 容器内逐个实现 issue → 测试 + 类型检查 → 提交并关 issue

**下一步并行化**：Matt 想做的优化是把多个 Ralph loop 并行化，搞一支 agent 团队一起干。

---

## 八、核心方法论洞察

1. **你很少需要看代码**：Matt 在整场几乎不看代码，他做的是 **review 输入和输出**——接口怎么变、模块长什么样，偶尔翻进去确认方向
2. **接口思维优先于实现**："我更多在想接口而不是实现；实现我不太在乎。但要确保可测，且仓库其余部分以及未来的 AI agent 都能看懂。"
3. **人在回路边界**：写代码可以放手，但**提交代码、review 统一语言文档**等仍自己控制
4. **紧凑循环**：review AI 产出 → 往里喂更多信息 → 形成紧凑反馈回路
5. **并行红利**：Claude 能 AFK 跑，就能把自己的 QA 和 bug 修复并行起来
6. **测试基建是底座**：Effect + E2E 测试（建测试数据库 + 临时 git 仓库）让"可测试单元"成为一等公民，降低 AI 改动风险
7. **20 年老办法依然好用**：关注架构、反馈回路、不过度前期规划——把 Claude Code 当"团队里可委派任务的人"

---

## 九、时间轴索引

| 时间 | 内容 |
|---|---|
| 00:00 | 开场：老办法在 Claude Code 上依然好用；回应"来点实际的" |
| 00:50 | 项目介绍：course-video-manager（1200 提交 / 637 issue） |
| 01:20 | 技术栈：React Router + TS + Drizzle + Postgres + Vitest + Effect |
| 02:30 | 现状：ghost lesson / real lesson 概念；"同一个界面既规划又创建" |
| 03:20 | 三条模糊需求：直接建 real lesson、直接删 real lesson、建 ghost course |
| 04:10 | 启动 grill me；为什么"为什么"重要 |
| 05:00 | explore 机制解释（sub-agent 隔离上下文读文件，回传总结） |
| 06:20 | GrillMe 第一轮反诘：delete lesson 已支持，问题在 UI |
| 07:30 | DDD 统一语言引入：术语表弥合领域专家与 LLM 的鸿沟 |
| 09:00 | CourseWriteService 讲解（写入侧、接口清晰、E2E 测试） |
| 11:30 | ghost course 拆解：无文件路径 → 无 git → 全 ghost |
| 13:30 | "materialize" 动词让按钮命名变干净；隐藏 publish/export |
| 15:00 | "这里花得越多，后面引导 LLM 要做的就越少" |
| 16:30 | A/B 方案权衡：ghost course 里建 real lesson 触发 materialization cascade |
| ~22:00 | GrillMe 列出完整范围 = 8 个要点（22 分钟视频产出） |
| ~23:00 | 更新统一语言文档（ghost 三兄弟 / materialize / cascade） |
| ~24:00 | 进入 write PRD：先勾勒模块，加测试决策 |
| ~26:00 | PRD 拆 issue：阻塞关系、合并小任务、6→4 块 |
| ~28:00 | 回顾 GrillMe 为什么不带 Ask User（UI 不喜欢 + JSON 包裹成本） |
| ~30:00 | 打磨 22 分钟后上下文仅 40k token |
| ~31:00 | 启动 AFK：Sandcastle（Docker 容器 + patch） |
| ~32:00 | 跑 pnpm ralph（最多 100 次迭代，提交关 issue） |
| ~34:00 | 白班夜班：Jamin 的比喻；人散个步、agent 实现 |
| ~40:00 | 回来：5 次 agent 运行 → 6 次提交；生成 QA 计划 |
| ~42:00 | QA：add course modal 问题 → 反馈按钮 → haiku 标题 → issue |
| ~44:00 | ghost course 创建 bug（不跳转 + React error + 无 loading） |
| ~46:00 | 右键双按钮 vs modal + "文件系统上创建"复选框 |
| ~48:00 | QA 8 分钟建 6-7 个 issue；Ralph 后台修；showstopper（非 git 仓库回滚） |
| ~50:00 | 第 8 次迭代：删除确认 modal；14 次提交收尾 |
| ~52:00 | 检查：add lesson 不再分 ghost/real，只有一个复选框 |
| 结尾 | 复盘：很少看代码、review 输入输出、并行红利；训练营广告 |

---

## 十、你能直接带走的东西

### 工作流模板（Matt 完整链路）

```text
日历里留半小时 → 语音口述想法
  → grill me（22 分钟磨出 8 要点，~40k token）
  → 更新统一语言文档（先接受后 review）
  → write PRD（模块勾勒 + 测试决策，不 review）
  → PRD to issues（带阻塞关系，任务粒度适中）
  → pnpm ralph（Sandcastle Docker 隔离 + 每次提交测试）
  → 人走开 / 睡觉 / 散步
  → 醒来生成 QA 计划 → 走 QA → 反馈按钮建 issue
  → Ralph 后台修 → 循环直到验收通过
```

### 三个可立即复用的习惯

1. **打磨想法花时间值得**："这里花得越多，后面引导 LLM 要做的就越少"
2. **explore 是省 token 杠杆**：sub-agent 隔离上下文读文件回传总结，一个 session 用好几次
3. **每次提交跑测试+类型检查**：Ralph loop 成功的基石

### 对「spec 先行」的修正

- GrillMe/原型阶段规划不到的边界问题（非 git 仓库回滚、删除确认、UI 措辞）**只能靠 QA 循环发现**
- 所以闭环必须是：**打磨 → 实现 → QA 反馈 → 再实现**，而不是 spec → 代码 一次性完成

---

## 关联笔记

- [[Matt-Pocock/2026-07-31-Matt-Pocock直播-SlopWatch从零搭建完整实录]] — GrillMe + DDD 完整实战（首次亮相 domain model skill）
- [[Matt-Pocock/2026-07-31-Matt-Pocock-Skills原作者手把手教程]] — grill-with-docs → to-spec → to-tickets → implement → code-review 主流流程
- [[Matt-Pocock/2026-07-31-Matt-Pocock实战-从原型到spec-driven开发]] — prototype + 保真度
- [[Matt-Pocock/2026-08-02-Matt-Pocock-grill-skill的9个误区]] — GrillMe 使用误区
- [[Matt-Pocock/Matt-Pocock方法论精华-完整工作流与技能体系]] — 完整方法论地图与技能体系汇总
- [[Clippings/Bilibili/2026-08-03-Claude-Code上下文压缩与会话恢复机制]] — AFK 长任务运行背后的上下文机制

> 💡 **一句总括**：这期视频是 Matt 方法论的"交付端"闭环——前几期讲探索（GrillMe/Wayfinder/prototype）和设计（DDD），这一期完整展示**探索 → 统一语言 → PRD → issue → AFK 实现 → QA 反馈**的整条流水线如何在一个真实项目上运转，以及为什么"白班夜班"让一个人的产能可以并行。
