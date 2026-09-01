---
created: 2026-07-30
updated: 2026-07-30
tags: [Agent, Graph, Loop, Harness, 架构, Agent落地, 大模型, 小红书笔记]
source:
  - "https://www.xiaohongshu.com/explore/6a65ac06000000000503b864"
  - "小红书：咔嗒库栗"
---

# AI Agent 落地铁三角：Graph、Loop、Harness

> **作者**：咔嗒库栗 | **平台**：小红书
> **互动**：❤️ 302 · ⭐ 443 · 💬 16

---

## 全文

> 企业级 AI Agent 规模化落地的标准安全架构：**Graph 把任务怎么分块、整体怎么走锁死在代码里，Loop 只给模型留下块内事情怎么做完美的自由度，Harness 锁住底线。**

### 笔记配图（共 23 张翻页图）

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-01.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-02.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-03.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-04.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-05.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-06.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-07.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-08.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-09.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-10.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-11.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-12.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-13.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-14.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-15.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-16.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-17.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-18.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-19.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-20.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-21.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-22.png)

![](https://gitee.com/cheng-jiaqing/images/raw/master/xhs-carousel-23.png)

---

### ✅ Graph = 顶层业务总流程

审批流/工单流/系统业务链路。由一个个**业务节点 + 分支走向、回退、并行、汇聚规则**组成，拓扑结构是**开发人员固定好的大框架**。

> 📌 模型没有修改 Graph 的自由度

### ✅ Loop = Graph 里单个节点内部的执行逻辑

大流程不会管某个节点内部怎么干活。节点内部依靠**循环闭环**，反复试错、补齐信息、自检修正，把当前这个节点的任务做合格再交给 Graph 流转到下一站。

> 📌 模型在这里有「怎么做」的自由度

### ✅ Harness = 横贯整个 Graph 所有节点、所有 Loop 的底层运行围栏

不管是 Graph 流转跳转，还是节点内部 Loop 反复执行，**所有动作都会被 Harness 统一管控**：

- 字段强校验
- 接口权限
- 调用频次限流
- 异常熔断
- 全量日志审计
- 高危操作拦截

---

## 公式

```
Agent 系统 =
    Graph（业务主流程编排）
    + 每个 Node 内置 Loop（节点内自主执行闭环）
    + 全局 Harness（安全与稳定性底座）
```

---

## 评论区

共 16 条评论，以下为公开可见的精选：

### 用户互动

| 用户                   | 内容                                                                                                                      |  时间   |
| :------------------- | :---------------------------------------------------------------------------------------------------------------------- | :---: |
| 🗣 **@摆动的记忆力**       | 怎么感觉就是 LangGraph 呢                                                                                                      |  4天前  |
| 💬 **作者回复**          | **兄台好眼力 👀**                                                                                                            |       |
| 🗣 **@Wayne Chueng** | 很少讲的这么细的                                                                                                                |  1天前  |
| 💬 **作者回复**          | 我这有真东西                                                                                                                  |       |
| 🗣 **@风飞**           | 通用 Agent 系统没法预先编排 Graph                                                                                                 | 12小时前 |
| 💬 **作者回复**          | **应用场景**，现在 AI 削尖脑袋想进企业                                                                                                 |       |
| 🗣 **@随机比特**         | **补一个边界**：Harness 不只是安全围栏，还要负责**上下文装配、工具发现、状态恢复和验收记录**；否则这些能力落进每个 Node，Graph 很快又会被运行时细节污染。我主页写过《我打赌，一半人没看到 harness 的精髓》 |  3天前  |
| 💬 **作者回复**          | 是的                                                                                                                      |       |

### 作者置顶

| 内容 |
|:-----|
| 👈 **强烈推荐大伙去我主页瞅瞅**，超多 Agent 落地硬核、稀缺干货 |
| ⚠️ Graph 并不是一个很新的东西，只是最近才大火起来，也算标志着行业从 **demo 迈入生产级落地** |
| 需要原文的朋友，可以留言，我看到就**免费分享**给你 |
| 之前讲解人任务编排的笔记 👉 AI Agent 核心：规划-... |

---

## 关键洞察

这套三层架构最精彩的地方在于**职责边界的划分**：

| 层 | 控制什么 | 谁决定的 | 模型自由度 |
|:--:|:---------|:--------:|:---------:|
| **Graph** | 做什么（业务流程） | 开发人员锁死 | ❌ 无 |
| **Loop** | 怎么做（执行策略） | Agent 自主决策 | ✅ 高 |
| **Harness** | 不能做什么（安全底线） | 平台/运维强制 | ❌ 无 |

> **Graph 锁死「做什么」的大框架，Loop 交出「怎么做」的自由度，Harness 守住「不能做什么」的底线。**

这也是 LangGraph、Dify、Coze 这类平台的底层设计逻辑都在复现的铁三角。

---

## 关联笔记
- [[2026-07-31-Agent系统架构设计-Harness-Loop-Graph怎么选]] — 同主题视频深度版（互链）
- [[2026-08-04-从Context到Graph-Agent工程的四个层次-Qoder]] — 工程演进视角

- [[2026-07-30-Agent架构Skill和Tool本质区别-动画详解]] — Skill vs Tool，架构层面的另一个拆解
- [[2026-07-28-Claude-Code工程化六件套详解-小林coding]] — Hooks 就是 Harness 在 Claude Code 里的具体实现
- [[2026-09-01-RAG核心知识全解析（范式演进）]] — RAG 的索引/检索/生成也有类似的分层逻辑
