---
created: 2026-07-31
updated: 2026-07-31
tags: [Matt Pocock, SlopWatch, GrillMe, DDD, 领域建模, 统一语言, Agent监控, Live直播, YouTube字幕]
source:
  - "https://www.youtube.com/watch?v=K-mA3MZ_EzU"
  - "作者：Matt Pocock | 直播：2026-04-18"
---

# Matt Pocock 直播实战：从零搭建一个全新项目（SlopWatch 完整实录）

> **原作者**：Matt Pocock（@mattpocockuk）| **直播** 2026-04-18 | **时长** ~1:47
> **事实主干**：YouTube 直播完整字幕（用户提供）
> **证据等级**：`youtube-transcript`

---

## 一句话总论

> **这是一场「从零到架构」的完整直播：Matt 从收集项目点子开始，用 GrillMe 反复诘问需求、用 sub-agent 研究 5 个编程 Agent 的数据接入方式、用 DDD 敲定统一语言，最终把「编程 Agent 监控平台」从模糊想法推进到明确架构。** 全程**没有写代码**——正如他所说，greenfield 项目在开始写代码之前，本质是一个研究任务。

---

## 项目诞生：SlopWatch

**做什么**：自托管的编程 Agent 监控平台，追踪团队使用 Claude Code / Codex / Pi / Open Code / Copilot CLI 时的：
- token 消耗
- 会话质量（成功/失败）
- context window 占用
- 使用什么模型

**名字来源**：直播间观众 **Eddie** 提议「Slop Watch」，击败了 Yardstick 等候选名。Matt：「这个名字把想法结晶了。」

**约束**：自托管、on-prem、数据不离开组织服务器；开源工具而非商业公司。

---

## 第一幕：收集项目点子（0:00 - 9:00）

Matt 直播征集项目，约束条件：
1. 日常工作中真正有用
2. 有前端 + 后端
3. 有一定复杂度
4. 最好对观众也有用

候选包括：to-do app、直播聊天管理器、coding agent observability platform……最终锁定**编程 Agent 监控平台**——因为这是他自己教学中的「missing link」，他也想观测自己的会话。

**关键数据点**：78% 观众投票支持。

---

## 第二幕：GrillMe 诘问需求（9:21 - 17:46）

Matt 打开终端，用 **Grill Me** skill 把想法「harden 成一个可构建的好想法」。GrillMe 问出了一系列关键问题：

### Q1：谁是最主要的用户？
> 个人开发者 / 工程经理 / 平台负责人 —— 它们指向**完全不同的 UX**

- A：个人时间线 + 单会话深挖
- B：团队 cohort 仪表盘 + 对比（会造成「被监视焦虑」）
- C：聚合指标（不关心个体）

**决策**：个体开发者优先，C 式聚合作为自然汇总。但**经理要能打开并审查某个工程师的具体会话**（团队需要 DRI 负责让 AI 变好）。

### Q2：同意与可见性模型
- 每个会话 opt-in / 永远开启 / 开发者控制+脱敏

**决策**：会话对组织**默认可见**（tacitly understood）。理由：编码会话常含机密、半成品想法，PII 很重要；没有清晰的同意模型，要么被法务/安全封杀，要么开发者自我审查导致数据质量下降。「开发者的会话对组织公开」应被默认理解。

### Q3：会话怎么进入系统（技术主干）
> 这是决定整个客户端形态的「technical spine」

- **Hooks**：Claude Code 的 post-tool-use / stop / session end hooks
- **JSONL 追踪**：agent 在磁盘上创建 transcript 文件，一个小 daemon 监视目录、上传增量

**决策**：先不急着定，派 sub-agent 做研究（见下）。

---

## 第三幕：研究 5 个编程 Agent（17:46 - 25:55）

Matt 用 **sub-agent 并行研究** 5 个主流编程 Agent：Claude Code、Codex、Pi、Open Code、GitHub Copilot CLI。

**研究发现**：
- 每个 agent 都有 **event hook** 和 **append-only JSONL on disk**
- **schema 全部不同且持续演变**（Pi 的 schema 在一次 patch 版本中就变了）
- Copilot 和 Pi 在 CLI 层面没有有用的 OpenTelemetry
- Copilot 的 proxy 路径被堵死
- **per-agent adapter 不可避免**

**关键结论（研究改变了想法）**：
- 单靠 hooks 不够：Claude Code hook payload **不含 message 内容**（还得读 JSONL）；Copilot hook payload 太薄；Codex hooks 需要 flag 开启且 **Windows 被排除**（Matt 就在 Windows 上）
- JSONL 单独对某些 agent 来说高效且足够
- **必须按 agent 切换接入方式**

**研究产出**：约 45K token 的研究被 compact 成一份 research document 存入新 repo。

---

## 第四幕：领域建模与统一语言（42:31 - 之后）

Matt 用新的 **domain model skill**（GrillMe 的进化版）做 DDD 领域建模。他的理念：

> 「语言就是你如何把程序带入生命。如果 session 这个词是模糊的，整个数据模型都是模糊的。」

**先定架构形态**：
- 单二进制（TypeScript/Bun 或 Rust）+ Postgres
- Postgres 需要独立于应用的可部署单元（数据要活得比应用久）
- 前端：React（或 Svelte）
- live spectate 用 **轮询（5 秒）** 而不是实时推送（不需要那么 live）

**核心术语表（ubiquitous language）**：

| 术语 | 定义 |
|:----|:-----|
| **coding agent** | 编程代理（如 Claude Code / Codex） |
| **session** | 一个 coding agent 的一次逻辑运行（一个开发者、一个工作目录、一个 agent 版本） |
| **turn** | 一条用户消息 + 完整的 assistant 响应 |
| **model request** | turn 中 agent 向模型供应商发起的一次 HTTP 调用 |
| **listener** | 运行在 agent 旁边、捕获数据并上报的进程 |
| **server** | 自托管进程：接收 listener 事件、存 Postgres、提供 dashboard 和 admin 面板（每组织一个） |

**关键建模决策**：
- session 包含一个 **DAG（有向无环图）** 的 turns（turn 是 DAG 节点，有 parent turn ID）——因为 agent 会话可能分支
- **sub-agent 是 child session**（带 parent session ID）
- resume/compact 的建模**保持开放**，等看到真实数据再定

**命名过程（bike-shedding）**：
- 捕获进程：sidecar → capture → capturer → listener → watcher……最终 **listener**（虽然和引擎内部术语有碰撞风险，但「good enough」）
- 服务端：binary → server（server 胜出）
- 关键教训：**别在语言上 bike-shed 一整天**，先到 good enough，之后可以修

---

## 第五幕：认证方案（V1 vs V2）

GrillMe 问：session 怎么绑定到开发者的身份？sidecar 怎么向后端认证？

**完整 OIDC 方案**（重）：
- CLI 登录走 device flow（用户浏览器完成登录）
- IDP 是组织的身份源（Okta / Auth0 之类）
- OIDC 构建在 OAuth 2.0 之上

**V1 轻量方案（采纳）**：
- **admin 签发的一次性 token**：自托管后端的管理员打开 admin 页面 → 添加用户（姓名+邮箱）→ 生成一次性 token → 发给开发者（Slack 等）
- 开发者运行 `stopwatch login` 粘贴 token，sidecar 存储，作为 bearer token 随请求发送
- 身份可信（token 绑定到 user record）、**撤销即 de-provisioning**
- 没有 IDP 集成，一天就能做完

> **OIDC 推到 V2。** 核心原则：先验证想法，别在抽象阶段做重架构。

---

## 关键哲学与方法论

### 1. Greenfield 项目在写代码前 = 研究任务
> 「直到你开始写代码、开始写 PRD、开始看到想法被 reify 成代码，你本质上就是在做研究。」

### 2. DDD 是 AI 编程的完美匹配
> 「DDD 对 AI coding 是 fantastic match 100%。」 统一语言让 AI 和人在「怎么谈论应用」上同步。

### 3. 语言是你把程序带入生命的方式
- 术语一旦结晶（calcify），就锁定了心智模型
- 用具体场景「road test」语言（DDD 的验证方法）——如果语言能顺畅描述场景，通常也更容易实现

### 4. 长期 Agent 记忆是坏主意
> 想要「超级可观测、超级具体、能立即编辑、能塞进 context window」的东西。方案：**ADR + 最小化 DDD**，不用 RAG 式长期记忆。

### 5. 复杂 vs 简单
> 「复杂应用难改，简单应用易改。」用 improve-codebase-architecture skill 把复杂应用变简单。

### 6. 别对 token 太抠门
> 「人们太在意 token 用量了。输入 token 极其便宜，输出 token 更贵。」（用 WhisperFlow 口述，不清理 prompt 省 token）

### 7. 人类 + AI 永远优于纯 AI
> 不管 AI 多强，「human plus AI is always going to outperform AI」。

### 8. 别过度评审计划
> 「人们反复评审他们要创建的 spec，而他们真正该做的是赶紧写代码。」经典 web 开发错误。

---

## 关键工具

| 工具                                | 用途                                  |
| :-------------------------------- | :---------------------------------- |
| **Grill Me**                      | 诘问需求，把想法 hardening                  |
| **domain model**（GrillMe 进化版）     | DDD 领域建模，敲定统一语言                     |
| **zoom-out**                      | 遇到不懂的概念（如 OIDC）时 explain-like-I'm-5 |
| **improve-codebase-architecture** | 重构代码库、提升可测试性                        |
| **Sandcastle**                    | coding agent 编排器（沙箱跑 agent），后续构建会用  |
| **WhisperFlow**                   | 语音输入转写                              |
| **sub-agents**                    | 并行研究多个编程 Agent 的能力                  |

---

## 对 Agent 监控平台的完整理解

**架构形态**：
```
开发者机器上：coding agent + listener（每会话由 hook 拉起的 subprocess）
                    ↓ 上报事件（bearer token 认证）
自托管服务器：server（单二进制）→ Postgres → Dashboard + Admin 面板
```

- **capture 无 daemon**：hook 配置 = 安装，session start hook 拉起捕获进程，进程只活一个 session（避免「忘了跑 daemon」问题）
- **每个 agent 一个 adapter**：hook（实时事件）+ JSONL（兜底），归一化成内部事件 schema
- **身份**：V1 admin 签发 token 绑定 user record
- **DRI 视角**：经理/负责人能打开单个工程师会话审查、live spectate（轮询）、debug

---

## 自测题

1. SlopWatch 是什么？核心解决什么问题？
2. 「谁是主要用户」这个问题为什么关键？
3. 为什么单靠 hooks 不够？研究发现的关键点是什么？
4. 5 个编程 Agent 的数据接入有什么共同点？难点在哪？
5. session / turn / model request / listener / server 分别是什么？
6. 为什么 session 内部用 DAG 建模？
7. V1 认证方案是什么？为什么不做 OIDC？
8. 「greenfield 项目在写代码前本质是什么」？
9. Matt 为什么说「长期 Agent 记忆是坏主意」？
10. live spectate 用什么技术？为什么不用实时推送？

<details>
<summary>参考答案</summary>

1. 自托管编程 Agent 监控平台，追踪 token 消耗/会话质量/context 占用/模型使用
2. 个人 vs 团队 vs 平台负责人指向完全不同的 UX 和数据模型
3. Claude Code hook 无 message 内容；Copilot hook 太薄；Codex hooks 需 flag 且排除 Windows——所以 JSONL 兜底，per-agent adapter
4. 共同点：都有 event hook + append-only JSONL；难点：schema 各不相同且持续演变
5. 见术语表
6. 因为 agent 会话可能分支（Pi 会产生真实分支），需要 DAG 记录分支关系
7. V1=admin 签发一次性 token 绑定 user；OIDC 太重，推到 V2，先验证想法
8. 研究任务（research task）
9. 想要超级可观测、可编辑、能塞进 context 的东西；用 ADR+最小 DDD 而非 RAG 记忆
10. 轮询（5 秒）；不需要那么实时，轮询足够且简单

</details>

---

## 证据与原文位置

- 字幕来源：YouTube 直播字幕 `LIVE Watch me build a brand-new project from scratch.md`（用户提供）
- 原文存档：`原始材料/BV1FENq6WEdZ_archive/subtitles/transcript/`

## 来源、覆盖与局限

- **URL**：https://www.youtube.com/watch?v=K-mA3MZ_EzU
- **作者**：Matt Pocock（@mattpocockuk）| **直播**：2026-04-18 | **时长** ~1:47
- **字幕**：YouTube 完整转录，全程覆盖
- ⚠️ 直播节奏快、有多人发言，部分对话段落需要结合画面理解
- ⚠️ 英文原文转录，已翻译为中文并提炼；术语按 Matt 语境校正

## 关联笔记

- [[Matt-Pocock/Matt-Pocock方法论精华-完整工作流与技能体系]] — 本视频是方法论精华中「探索+设计」环节的完整实战
- [[Matt-Pocock/2026-07-31-Matt-Pocock直播-Wayfinder从想法到spec全流程]] — Wayfinder（domain model 的进一步演化）
- [[Matt-Pocock/2026-07-31-Matt-Pocock-Skills原作者手把手教程]] — GrillMe 等 skills 的主流流程
