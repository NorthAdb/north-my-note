---
created: 2026-08-09
updated: 2026-08-10
article_date: 2026-07-28
tags: [AI, Agent, Graph Engineering, Multi-Agent, 工作流编排, 实操手册, Loop Engineering, Datawhale]
source:
  - "https://mp.weixin.qq.com/s/LzpfUsJRMcpPHzDovo5IrA"
  - "https://mp.weixin.qq.com/s/GZrKCHJaG5g7Ua-ZiEP-fQ"
publisher:
  - "腾讯技术工程"
  - "Datawhale"
---

# Graph Engineering：从 Loop 到 Graph 的工程化学习笔记

> **主要文章**：腾讯技术工程《Loop Engineering 已死？一文带你了解 Graph Engineering》（作者：lukiexing，2026-07-28）  
> **补充材料**：Datawhale 整理的 Codez《Graph Engineering 实操手册》（2026-08-09）

![Graph Engineering 文章封面](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-cover.jpeg)

---

## 🧭 阅读地图：三份材料各自解决什么问题

| 材料                           | 重点问题                          | 学习定位        |
| ---------------------------- | ----------------------------- | ----------- |
| 腾讯技术工程文章                     | 为什么需要 Graph、Graph 是什么、什么时候值得用 | 建立全局认知与选型判断 |
| Datawhale 14 步手册             | 如何拆节点、定契约、并行、路由、验证、收敛         | 把认知落成工程实践   |
| [[领域/AI Agent 智能体学习路线 2026]] | 应该先学什么、后学什么                   | 规划学习顺序      |

> **一句话结论：Graph 不是 Loop 的升级版，而是 Loop 的组织方式。** 先把单体 loop 跑稳，再用 graph 管理多个执行节点之间的状态、路由、验证和恢复。

---

## 一、腾讯技术工程文章：Graph Engineering 的核心认知

### 1. 五层演进：Graph 建立在前四层之上

```text
Prompt Engineering  →  一次对话怎么说
Context Engineering →  给模型哪些上下文
Harness Engineering →  工具、护栏、状态、重试如何组织
Loop Engineering    →  单个 Agent 如何持续发现、执行、验证
Graph Engineering   →  多个节点如何协作、路由、恢复、审计
```

![Prompt、Context、Harness、Loop 到 Graph 的五层演进](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-02.png)

这五层不是互相淘汰，而是逐层叠加。**Loop 解决“一个 Agent 如何持续工作”，Graph 解决“多个 Agent、工具和人如何组成可观测、可恢复、可扩展的系统”。**

### 2. Loop 的结构性缺陷

当任务变成长时、多角色、需要独立复核时，单循环会暴露出：

- **上下文腐烂**：思考、工具调用和观察结果不断堆入同一窗口，原始目标被淹没。
- **错误级联**：工具失败后，模型可能在同一条链上反复试错，直到预算耗尽。
- **工具过载**：工具过多或功能相近时，选择准确率下降。
- **控制粒度不足**：难以暂停子任务审批、给不同步骤配不同模型或插入独立质检。
- **可观测性差**：只能从长对话里反推分支原因，难以复现和审计。
- **目标失明**：单一指标被过度优化后，可能偏离真实目标，符合“古德哈特定律”。

![单个 Agent 的自主循环：观察、行动、检查、修正](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-03.png)

![指标上升但客户续费率下降：目标失明与古德哈特定律](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-04.png)

### 3. Graph 的最小模型：从四构件到生产级五构件

腾讯文章用下面的形式描述 Graph：

> **G = (V 节点, E 边, S 状态, P 策略)**

| 构件 | 工程含义 |
|---|---|
| **V — Nodes** | 一个 Agent、函数或工具步骤；输入输出清晰，只负责一件事 |
| **E — Edges** | 数据传递与路由；包括顺序、条件、扇出、扇入和回环 |
| **S — Shared State** | 任务、证据、预算、产物、检查点等沿图流动的状态 |
| **P — Policy** | 权限、工具范围、预算、副作用和人工审批等治理约束 |

Datawhale 14 步手册进一步强调 **Failure Routing（失败路由）**。因此可以这样理解：

> **学习版 Graph = Nodes + Edges + State；生产版 Graph = 上述三者 + Policy + Failure Routing。**

没有失败边的图只是流程图；没有策略约束的动态图则可能把“谁能改数据库、谁能绕过审批”交给模型临场发挥。

![研究、事实核查、写作与人工审批组成的可恢复图](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-05.png)

### 4. 三种高频拓扑与 Anthropic 五种模式

| 拓扑/模式 | 结构 | 适用场景 |
|---|---|---|
| **菱形 / Parallelization** | 拆分 → 并行 → 合并 | 多信源研究、多文件审查、多视角验证 |
| **Supervisor / Orchestrator-Workers** | 主管动态分派 → 专业节点 → 汇总 | 子任务无法提前确定，需要运行时规划 |
| **Pipeline / Prompt Chaining** | A → B → C，阶段间传递结构化结果 | 固定、可验证的多步任务 |
| **Routing** | 分类 → 专用路径 | 不同输入需要不同工具或提示 |
| **Evaluator-Optimizer** | 生成 → 评估 → 修订，直到达标 | 有明确验收标准且迭代确实有收益 |

Anthropic 的核心建议是：**先用最简单的模式，只有在上下文保护、并行探索或专业化确实带来收益时才增加 Graph 复杂度。**

![Graph 的扇出、扇入：多个来源并行研究后交给写稿人](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-06.png)

![流水线中的检查点 Gate：不合格的中间结果不能进入下一步](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-07.png)

### 5. Graph 的真正杠杆：确定性，而不是 Agent 数量

- **模型负责判断**：分类、规划、写作、风险判断、候选方案生成。
- **代码负责可靠性**：schema 校验、去重、排序、预算计算、状态更新、边的选择。
- **独立验证器负责推翻结论**：验证节点使用干净上下文，只看结果和验收标准。
- **现实负责最终锚定**：测试真的跑过、数据真的对上、付款真的完成、线上指标真的恢复。

> 如果所有节点都只互相引用模型生成的结论，而没有测试、数据库、用户行为或其他现实证据，Graph 只是“更有组织的幻觉”。

![单个 Agent 同时规划、执行、裁判，容易形成自我确认](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-08.png)

![独立验证器：通过则放行，失败则打回重做](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-09.png)

### 6. Loop 与 Graph：同一个每日研究简报任务

| 维度 | 臃肿 Loop | 三节点 Graph |
|---|---|---|
| 上下文 | 搜索、写作、审查混在一起 | 每个节点上下文干净隔离 |
| 搜集 | 顺序读取信源 | 多信源并行扇出 |
| 审查 | 作者自己审自己 | 独立审稿节点挑错 |
| 控制 | 全有或全无 | 可暂停、重试、回滚和人工介入 |
| 成本 | 起步便宜 | 多提示词、状态和编排成本更高 |
| 选择 | 一次性、目标单一的任务 | 高频运行、需要质量和可恢复性的任务 |

**决策原则**：如果任务一次性、目标单一、停止条件清晰，优先保留 Loop；如果需要并行、角色分工、独立验证、持久化或人工审批，Graph 才值得承担额外成本。

![是否需要 Graph：能否分叉、并行、汇合、兜底和治理](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-11.png)

![每日研究简报：臃肿 Loop 与三节点 Graph 的对比](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-10.png)

### 7. 老工作流、ReAct 与 Graph 的关系

三者不是简单的替代关系：

- **老工作流**：路径稳定但固定，遇到意外情况不会转弯。
- **ReAct Loop**：节点内部足够灵活，但控制流泡在模型对话中，难复现、难审计。
- **Graph**：用固定的边和策略框住动态节点，让结构可治理、节点内部仍保留自主推理。

![老工作流、ReAct 与 Graph：结构确定性和节点自主性的折中](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-12.png)

### 8. 生产化信号

文章给出的工程判断可以归纳为：

1. 多 Agent 的质量提升往往以更高 token 消耗为代价，不能只看成功率。
2. Graph 的价值在于上下文隔离、并行搜索、专业化和可恢复执行。
3. LangGraph、Google ADK、AutoGen 等框架早已在实践节点、边和状态；Graph Engineering 更多是一个新的观察视角。
4. **工作图可以动态变化，权限图必须稳定、可审计。**

### 原文配图补全

以下是原文中的装饰图和公众号尾图，保留在图床中但不参与核心论证：

![文章装饰图](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-01.gif)

![腾讯技术工程公众号](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-13.gif)

![公众号尾图](https://gitee.com/cheng-jiaqing/images/raw/master/graph-engineering-20260728-14.jpeg)

---

## 二、Datawhale 14 步手册：如何把认知落成代码

### 核心观点

- **Graph = Nodes + Edges + Shared State + Failure Routing**（四个可单独验证的构件）
- 拆图是为了**腾出上下文、换取并行度**，不是为了好看
- 图里「用代码能做的」尽量用代码做——**边是免费的**，编排这一层不花 token
- 关键公式：**确定性（代码）+ 判断力（agent）** 分置不同层

---

### 1. 背景：从 Loop Engineering 到 Graph Engineering

上个月（2026-07）刚流行过 Loop Engineering 实操手册，新词就来了：

- **7 月初**，OpenClaw 创始人在 X 上发问：我们还在聊 loop，还是已经切到 graph 了？
- 随即有人跟帖：**「Loop Engineering 已死，Graph Engineering 永生」**

![Loop Engineering 已死，Graph Engineering 永生——X 上的讨论截图](https://gitee.com/cheng-jiaqing/images/raw/master/img1.png)

这篇文章整理了 Codez（X 博主）总结的 **14 步** 方法论，全网 570w+ 人看过，讲的就是怎么从一个人的 loop，走到一张能自己路由的 graph。

---

### 2. 动手前：四个问题 + 一个附加题

> Graph 烧的 token 比单个 loop 更多，协调开销更高，出了 bug 你要 debug 的是一整张你没亲眼看着跑的路由图。**先答完四个问题，再决定要不要建图。**

1. **这个任务真的能拆成不同角色吗？**
   拆不出清晰的「谁负责什么」→ 那就还是一个 loop，加节点只是加成本。

2. **有没有真正能并行的子任务？**
   没有独立并行的活，图比循环贵，却不比循环快。

3. **单个 agent 的上下文装得下全部背景吗？**
   装得下，就别急着拆。拆分是为了腾出上下文，不是为了好看。

4. **失败之后，你负担得起跳转分支的成本吗？**
   没想清楚重试耗尽后去哪，图会在你看不见的地方卡死或者乱跑。

![So do you actually need a graph?——四问决策图](https://gitee.com/cheng-jiaqing/images/raw/master/img2.png)

**附加题（比上面四个都重要）**：你已经有一个跑得稳的单体 loop 了吗？
> 没有，先别建图。**图是循环的组织方式，不是循环的替代品。**

### 谁适合 / 谁不适合

| | 适合 | 不适合 |
|---|---|---|
| **人** | 已经把至少一个 loop 跑稳的团队 | 还没让一个 loop 稳定跑起来的个人开发者 |
| **任务** | 能明确拆开角色（调研、撰写、复核），接受更高 token 成本换质量和并行度 | 线性依赖强、拆不开的任务 |
| **团队** | 瓶颈在单节点能力之外 | 瓶颈在协调开销而不在单节点能力的团队 |

> 结论：**Graph Engineering 真有用，但大部分人现在还用不上**——因为大部分人的 loop 都还没跑稳，更别提图。

---

### 3. Graph Engineering 的四个核心构件

一张能跑的图，拆开看就是四个能各自单独验证的部分：

| 构件 | 说明 |
|------|------|
| **Nodes（节点）** | 图的最小单位。一个节点 = 一个跑着自己 loop 的 agent 或确定性步骤，只认一件事，只对一件事负责。**节点判断不出「完了」，就不是节点，是隐藏依赖。** |
| **Edges（边）** | 决定谁接下一棒。顺序边永远触发；条件边看检查结果；并行边一次分给多个节点再汇合。**边留给模型运行时判断得越多，图越灵活，但也越难预判。** |
| **Shared State（共享状态）** | 大家都要读写的那份数据。下游节点要用的字段，必须有上游节点写进去。这个对象会逼你承认：这活儿里到底还有多少环节没被真正想清楚。 |
| **Failure Routing（失败路由）** | 失败之后的退路。重试耗尽后控制权去哪：退回上一步、转备用节点、还是转人工。**没有失败边的图只是一张流程图，不是一个能跑的系统。** |

---

### 4. 14 步实操手册

#### 第一步：先分清节点和边

**01 | 节点是任务，边负责传数据**

一张图其实只有两样东西：节点是工作单元（一个 agent / 一份边界明确的活 / 一个输入 / 一个输出），边是依赖关系（输出喂给下游当输入）。

> ⚠️ 最容易犯的错：把「然后」当成边。**「总结这个文件，然后告诉我天气」——这两步之间没有边**，天气根本用不到那份总结。这其实是两个互不相干的节点被线性脚本硬凑成先后顺序。**没真用上数据，就没有边。**

![节点是任务，边负责传数据](https://gitee.com/cheng-jiaqing/images/raw/master/img3.png)

**02 | 你的线性脚本，其实是一张退化的图**

把 agent 写成「先做 A、再做 B、再做 C」——你其实已经画出了一张图，只不过是一条不分岔的单链：每个节点只有一条边进、一条边出。能跑对，但慢、也脆：**C 卡住了 D 就永远轮不到，A 的产出也被困在上游。**

> 图工程的第一项真本事，是**重新画这条链**：对每一根箭头问「有数据真的传过去吗」。大多数链里都有两三根箭头没携带数据，只是当初顺手打的顺序。剪掉这些箭头，链就会塌缩成更宽的东西——几个可同时跑的独立节点，喂给一个需要它们全部到齐的节点。

![你的线性脚本，其实是一张退化的图](https://gitee.com/cheng-jiaqing/images/raw/master/img4.png)

#### 第二步：给节点和边定契约

**03 | 给每个节点定一份契约**

一个没法推理的节点，就没法拿去并行。契约三要素：**输入有边界**（必须显式传进去，不能指望蹭共享窗口）、**输出有边界**（定义好的形状，最好能校验）、**只干一件事**。

在 workflow 里，契约靠 schema 强制执行——校验发生在工具调用这一层，格式不对 Claude 会自己重试，不会甩给你一堆自由文本自己解析。

![给每个节点定一份契约](https://gitee.com/cheng-jiaqing/images/raw/master/img5.png)

```js
// 一个有真契约的节点：输入有边界，输出经过校验，只干一件事
const ITEM = {
  type: 'object',
  additionalProperties: false,
  properties: {
    title:  { type: 'string' },
    url:    { type: 'string' },
    impact: { type: 'string', enum: ['high', 'medium', 'low'] },
  },
  required: ['title', 'url', 'impact'],
};

const result = await agent(source.prompt, {
  label: `research:${source.key}`,
  schema: ITEM,             // 强制返回校验过的结构化数据
  agentType: 'general-purpose',
});
// result 现在是下一个节点能信任的形状，不用再靠人工解析
```

**04 | 把边也当成一份数据契约**

边不只是「B 排在 A 后面」，而是一个**关于「传的是什么」的承诺**：A 产出这个形状，B 就按这个形状设计来消费它。

> 按**数据**给边命名（而不是按顺序命名），两件事立刻变简单：① 一眼看出这条边是不是真的存在；② 形状不变的前提下，换掉边两端的节点不会弄坏整张图。

![把边也当成一份数据契约](https://gitee.com/cheng-jiaqing/images/raw/master/img6.png)

实际写的时候，边就活在普通 JS 里——派活和合成之间的归约（压平、去重、过滤）就是代码在处理节点返回的形状，不需要 agent。**很多人花模型 token 去做的事，其实就是一条边，而边是免费的。**

#### 第三步：构建扇出、扇入与菱形（最常用的拓扑）

**05 | 用 parallel() 扇出：把活儿一次性派出去**

N 个独立节点（N 个要核实的信源、N 个要审的文件、N 条要查的路由），不要串起来跑，让 Claude 一次性派出去并发执行。

两个决定稳定性的细节：
1. **parallel() 是一道屏障**——等所有函数跑完才返回，下一阶段看到完整结果集合
2. **一个抛错的函数被解析成 null**，而不是拖垮整个批次 → 记得 `.filter(Boolean)`

![用 parallel() 扇出：把活儿一次性派出去](https://gitee.com/cheng-jiaqing/images/raw/master/img7.jpg)

```js
phase('Research');
// 九个信源，九个 agent，同时开工
const raw = await parallel(
  SOURCES.map((s) => () =>
    agent(s.prompt, {
      label: `research:${s.key}`,
      phase: 'Research',
      schema: ITEM_SCHEMA,      // 每个节点都返回校验过的 JSON
      agentType: 'general-purpose',
    }),
  ),
);
const collected = raw.filter(Boolean);  // 把失败 agent 留下的 null 过滤掉
```

> 派活这一步活在**代码**里，不活在一轮模型对话里。Claude 的上下文从不同时装着九个信源，每个 subagent 带自己的一份，只有最终答案传回来。**编排这一层不花 token，因为它不是 Claude 又想了一轮。** 这就是一次 workflow 能扩展到几十上百个 subagent、却不会淹没会话的原因。

**06 | 在屏障处做汇入**

拢回来的节点 = 边汇聚的地方：一个 agent（或代码）一次性看到全部上游结果，做必须看到全集才能做的事——跨信源去重、按影响力排序、**总数为零就提前退出**。

```js
// 这条边就是普通 JS，没有 agent，零 token
const flat = collected.flatMap((c) => c.items);
log(`Collected ${flat.length} items`);

phase('Curate');
// 这个屏障节点需要全部结果凑齐才能去重排序
const curated = await agent(
  `Dedupe and rank these by impact:\n${JSON.stringify(flat)}`,
  { phase: 'Curate', schema: CURATED_SCHEMA },
);
```

> 判断方法：如果写成了 `parallel → transform → parallel`，中间那个 transform 又没有跨条目的依赖 → 该用**流水线**，完全不需要屏障。

**07 | 菱形：拆分 → 工作 → 合并**

把「派出去」和「拢回来」拼在一起，就是几乎每张正经 agent 图的主力拓扑——**菱形**。一个节点拆任务，多个节点并行干活，一个节点合并。

![菱形：拆分 → 工作 → 合并](https://gitee.com/cheng-jiaqing/images/raw/master/img8.png)

> 市场扫描、依赖审计、代码评审、研究报告，背后都是这个形状。标准写法有个值得记住的名字：**派发 → 归约 → 合成**（先派出去收集广度，用普通代码归约压缩，再用最后一个 agent 合成写出答案）。
>
> 看懂菱形之后，你就不再问「怎么让 agent 多做几步」，而是问**「拆分点在哪，合并点在哪」**——这才是真正能扩展的问题。

#### 第四步：路由、验证与隔离

**08 | 用条件语句在运行时给边选路**

不是每张图都是固定的。路由节点检查结果，决定走哪条下游路径——工单分类后分流、看 diff 大小决定快速评审还是完整审计。在 workflow 里就是一个普通的 `if` / `switch`。

![用条件语句在运行时给边选路](https://gitee.com/cheng-jiaqing/images/raw/master/img9.png)

> **节点上拿到 Claude 的判断力，边上拿到脚本的可靠性。** 路由判断可以由 Claude 完成，但路由本身是 Claude 写的代码，同样的分类结果每次都走同一条路。不会出现「Claude 自己决定跳过审计」——因为跳过这件事必须写进图里才会发生，而它没被写进去。

```js
// 路由节点：agent 负责分类，代码负责选边
const { severity } = await agent(
  `Classify this diff's risk:\n${diff}`,
  {
    schema: {
      type: 'object',
      properties: { severity: { enum: ['low', 'high'] } },
      required: ['severity'],
    },
  },
);

let review;
if (severity === 'high') {
  // 高风险路径：完整并行审计
  review = await parallel(FILES.map((f) => () => agent(`Audit ${f}`)));
} else {
  // 低风险路径：一次快速评审
  review = await agent(`Quick review of ${diff}`);
}
```

**09 | 在边上放一个验证器**

一张图真正的杠杆不是塞了更多 agent，而是**能围绕结果搭起多少确定性**。验证器节点蹲在结果被放行到下游之前，唯一的工作就是**试图推翻这个发现**——扛住了就放行，扛不住就到不了最终答案。

![在边上放一个验证器](https://gitee.com/cheng-jiaqing/images/raw/master/img10.png)

三种模式值得掌握：

- **对抗式验证**：给每个发现派 N 个独立怀疑者专门反驳它，多数没被驳倒才算站得住
- **多视角验证**：每个验证者盯不同方面（正确性 / 安全性 / 能不能复现），角度越分散越能揪出问题
- **评委制**：从不同角度生成 N 个方案，并行评委打分，取最好的一版作主线，再揉进其他版本的亮点

> 真实案例：某团队移植 Bun 运行时，就是靠对抗式代码评审焊进循环才做成的。

**10 | 把节点隔离开，别让一个失败污染整张图**

- 链式结构里失败会**级联**：C 死了 D 就跑不起来，整条链停摆
- 图里失败该被限制在节点内部：`parallel()` 抛错的函数解析为 `null`，`.filter(Boolean)` 就是那道防线
- **把每一次汇入设计成能容忍缺失的输入**，而不是假设总能凑齐全集

![把节点隔离开，别让一个失败污染整张图](https://gitee.com/cheng-jiaqing/images/raw/master/img11.png)

更隐蔽的失败：节点之间互相踩到对方（多个 agent 并行写文件会撞车）。解法是**隔离**——用 git worktree 让每个 agent 在自己的一份工作区里干活，在沙盒里完成再干净地合并回去。只在节点真的会并行写入时才用（它是拓扑真正需要的安全带，不是每次运行都要交的税）。

#### 第五步：循环、模型分层与拓扑

**11 | 可以加一个循环，但一定要让它收敛**

规模未知的探索（一次漏洞排查发现一个 bug 又带出三个新的）需要一条指回更早节点的受控边。危险：**不收敛的循环 = 不停派 agent 出去、直到预算耗尽才停的死循环机。**

能收敛的写法叫**「跑到干为止」**：持续派出发现者，直到连续 K 轮都没发现新东西才停下。

![可以加一个循环，但一定要让它收敛](https://gitee.com/cheng-jiaqing/images/raw/master/img12.png)

> ⚠️ 几乎每个人第一次都会踩的坑：**拿什么去重比对。** 要对着**「见过的一切」**去重，而不是只对着「已确认的结果」去重——不然被否掉的发现每轮都会重新冒出来，循环永远跑不干，最后搭出一台专门花钱反复发现同一批死胡同的机器。

```js
const seen = new Set();
const confirmed = [];
let dry = 0;

while (dry < 2) {                       // 连续两轮空手而归就停下
  const found = (await parallel(
    FINDERS.map((f) => () => agent(f.prompt, { schema: BUGS }))
  )).filter(Boolean).flatMap((r) => r.bugs);

  const fresh = found.filter((b) => !seen.has(key(b)));
  if (!fresh.length) { dry++; continue; }
  dry = 0;
  fresh.forEach((b) => seen.add(key(b)));  // 对"见过的一切"去重，不是只对已确认的

  // 每个新发现都要过一轮多视角验证才算数
  const judged = await parallel(fresh.map((b) => () =>
    parallel(['correctness', 'security', 'repro'].map((lens) => () =>
      agent(`Judge "${b.desc}" via ${lens} — real?`, { schema: VERDICT })))
      .then((v) => ({ b, real: v.filter(Boolean).filter((x) => x.real).length >= 2 }))
  ));

  confirmed.push(...judged.filter((v) => v.real).map((v) => v.b));
}
```

**12 | 给不同节点分配不同档位的模型**

不是每个节点都需要最好的模型。图的形状会把这件事摆明白：**有边界、会重复的活**（抽字段、工单分类）放便宜模型；**承载真正判断力的活**（合成报告、裁定发现是否成立）留在高档位。token 花在真正需要判断力的地方。

![给不同节点分配不同档位的模型](https://gitee.com/cheng-jiaqing/images/raw/master/img13.jpg)

> 注意：workflow 里 Claude 派出的每个 subagent **默认继承本次会话的模型**，除非脚本里显式覆盖——所以大规模运行前先看一眼 `/model`，把重复性节点降到便宜模型、合并节点留在高档位。**能把一张烧 token 的图从贵变便宜，还完全不用动它的形状。**

**13 | 拓扑结构，就是你的成本和延迟**

图的形状不是装饰，它是决定运行时间的最大杠杆。最容易踩坑的选择是 `parallel()` 还是 `pipeline()`：

| | parallel() | pipeline() |
|---|---|---|
| 行为 | 屏障：等最慢的节点才进下一阶段 | 无屏障：每条数据独立依次过所有阶段 |
| 场景 | 真需要全部结果到齐：跨集合去重、按总数提前退出、需要对照「其他发现」的 prompt | 默认选择 |
| 特征 | 慢的拖累快的 | 条目 A 可能在第三阶段，B 还在第一阶段 |

![拓扑结构，就是你的成本和延迟](https://gitee.com/cheng-jiaqing/images/raw/master/img14.png)

> **默认用 pipeline()。** 「代码更干净」「这些阶段感觉是分开的」都不是用屏障的理由——屏障带来的延迟是真实的、可测量的、被浪费掉的时间。**分开不代表必须同步。**

#### 最后一步：让 Claude 自己画图

**14 | 让 Claude 自己画图，自我路由**

对没法提前规划的活儿，不再自己动手画图——用 **dynamic workflows**：只要描述目标，Claude 会自己写编排脚本：拆解任务、决定怎么派活、派出一队 subagent、再合成结果。拿到的是**为这次运行量身定做的图**，而不是一张你希望它恰好合适的固定图。

三种用法：

1. **在 prompt 里说出「workflow」** → Claude 为这个任务写一份编排脚本
2. **跑存好/内置的 workflow**（如 `/deep-research`）→ 一张已在生产里跑着的真实图：定范围 → 并行搜索 → 抓取 → 对抗式验证 → 合成（正是整篇文章讲的那副骨架）
3. **打开 ultracode** → Claude 给会话里每个像样的任务都规划一次 workflow

跑得好的时候按 `s` 把脚本存进 `.claude/workflows/`，从此可以版本控制、按名字重新运行，谁 clone 了这个仓库都能直接跑起来。

![让 Claude 自己画图，自我路由](https://gitee.com/cheng-jiaqing/images/raw/master/img15.jpg)

```
› Run a workflow to audit every route under src/routes/ for missing auth.
  Spawn one agent per route file, then verify each finding before reporting.
● Claude wrote an orchestration script · launching in background…
/workflows — auth-audit · running
✓  Scope      1/1     2.1k tok
   Fan-out    18/18   one agent per route file
   Verify     11/18   3-vote skeptics per finding…
   Synthesize 0/1    waiting on verify
// 会话保持响应，队伍在后台继续跑
```

---

### 5. 结尾点睛

> 两年来，多 agent 协作的杠杆一直在单个 loop 上：更好的 verifier，更稳的退出条件，更干净的状态文件。
>
> 而现在，**把这些 loop 怎么连起来，成了新的护城河。**

---

## 三、与 AI Agent 学习路线对照学习

[[领域/AI Agent 智能体学习路线 2026]] 是“从基础到生产”的纵向路线；本笔记把其中的 **Loop、Harness、Multi-Agent、Eval、部署** 横向串成了一条 Graph Engineering 主线。

| 学习路线阶段 | Graph Engineering 对应知识 | 学习重点 / 验收标准 |
|---|---|---|
| **Stage 0：LLM 基础** | JSON Schema、Pydantic、API、Git | 能定义节点输入输出契约，不依赖自由文本解析 |
| **Stage 1：认识 Agent** | Loop 是 Graph 的前置条件 | 能解释 LLM、Agent、Workflow、Graph 的边界 |
| **Stage 2：Agent 循环** | max steps、timeout、retry、error handling | 先做出一个有停止条件、可重试、可观测的单体 loop |
| **Stage 3：Tool / RAG / Memory** | 工具节点、上下文隔离、共享状态 | 能把检索结果、工具结果和记忆字段显式放入 state |
| **Stage 4：Harness 工程化** | 节点抽象、条件边、checkpoint、HITL、状态管理 | 理解 Graph 本质上是更高一层的 Harness，而不是新模型 |
| **Stage 5：Multi-Agent** | 菱形、Supervisor、Pipeline、Evaluator-Optimizer | 能根据依赖关系选择并行、流水线或动态路由 |
| **Stage 6：MCP / A2A / Skills** | MCP/Skills 作为节点能力，A2A 作为跨 Agent 边 | 先区分“能力接入协议”和“任务编排拓扑”，不要混为一谈 |
| **Stage 7：Browser / Computer Use** | 浏览器操作、DOM 解析、人工审批节点 | 高风险副作用必须有 policy、验证器和人工确认 |
| **Stage 8：Eval / Observability / Safety** | 独立 verifier、trace、replay、权限、失败路由 | 每个关键结论都有验证路径，失败能重试、降级或转人工 |
| **Stage 9：生产部署** | durable execution、预算、日志、告警、灰度 | 能从 checkpoint 恢复，控制 token 成本并定位具体节点故障 |

### 对比结论

1. **学习路线回答“先学什么”**：从 LLM、Tool Calling 和单体 Loop 开始，逐步进入 Harness、Multi-Agent 和生产部署。
2. **Graph 文章回答“什么时候升级”**：只有任务存在明确角色、可并行子任务、上下文压力或独立验证需求时，才值得承担 Graph 的复杂度。
3. **14 步手册回答“怎么落地”**：节点契约、数据边、扇出/扇入、验证器、隔离、收敛循环、模型分层和拓扑优化。
4. **三者的共同原则**：代码负责确定性，模型负责判断；先用最简单的结构；所有关键路径都要有停止条件、失败路由和现实锚点。

### 建议的 Graph 专项练习阶梯

- [ ] **练习 1：稳定 Loop** — Calculator / Web Research Agent；记录成功率、步数、失败类型和 token 成本。
- [ ] **练习 2：RAG Loop** — PDF QA；把检索、引用和答案校验设计成显式状态。
- [ ] **练习 3：最小 Graph** — `Research（并行） → Write → Verify → Revise`，至少加入一次失败回路。
- [ ] **练习 4：LangGraph 生产能力** — 加入 State、Checkpoint、Streaming 和 Human-in-the-loop，验证中断后可以恢复。
- [ ] **练习 5：安全与评估** — 给高风险路径加 Router、独立 Verifier、权限策略、trace/replay 和预算上限。

> **最终学习顺序：先 Loop，后 Graph；先固定拓扑，后动态工作流；先验证质量，再扩展 Agent 数量。**

## 🔗 关联笔记

- [[领域/AI Agent 智能体学习路线 2026]] — 学习路线总纲（Stage 4-5：LangGraph 框架 / 多 Agent 系统）
- [[2026-07-31-Agent系统架构设计-Harness-Loop-Graph怎么选]] — Loop vs Graph 选型
- [[2026-07-31-10分钟讲透AI-Agent-8种主流架构]] — 8 种主流 Agent 架构
- [[2026-08-06-AI-Agent开发框架选型-LangChain-LangGraph-LlamaIndex-小林面试笔记]] — 框架选型

## 来源、覆盖与局限

- **主要文章**：<https://mp.weixin.qq.com/s/LzpfUsJRMcpPHzDovo5IrA>（腾讯技术工程，作者：lukiexing，2026-07-28）
- **补充文章**：<https://mp.weixin.qq.com/s/GZrKCHJaG5g7Ua-ZiEP-fQ>（Datawhale 整理，2026-08-09）
- **获取方式**：主要文章通过浏览器会话读取；Jina Reader 访问微信公众号时被验证码拦截。
- **图片处理**：主要文章封面与 14 张正文图片已上传至 Gitee 图床（`gitee.com/cheng-jiaqing/images`），并以 Markdown 图片形式嵌入；补充文章原有 15 张示意图继续使用原 Gitee 图床链接。
- ⚠️ 文章中的 Graph、workflow、`parallel()`、`pipeline()`、schema 等概念有重叠，但 Graph Engineering 不是一个全新的框架或协议；它更像是对多 Agent 编排工程重心的命名和总结。
