---
created: 2026-08-11
updated: 2026-08-11
article_date: 2026-05-22
tags: [AI, Claude Code, Harness, 大代码库, Agent, Hook, CLAUDE.md, LSP, Subagent, 小林coding]
source: "https://mp.weixin.qq.com/s/YhiQGIMn7ewBTyrsBN56sw"
publisher: "小林coding"
---

# Claude Code 大代码库实战：模型是地板，harness 才是天花板

> **来源**：小林coding《面试官皱眉："公司项目几百万行代码，Claude Code 怎么扛得住？"我："换 Opus 4.7"，他叹气：模型是地板，harness 才是天花板》（2026-05-22）
> **原文**：https://mp.weixin.qq.com/s/YhiQGIMn7ewBTyrsBN56sw
> **一手信源**：Anthropic 官方博客《How Claude Code works in large codebases: Best practices and where to start》 + Claude Code 创始人 Boris Cherny 分享

> **一句话结论：大代码库下 context 爆不是模型小，是 harness（外围基建）没搭好。** 换更大模型没用，正确做法是把 harness 七层一层一层搭起来：CLAUDE.md → Hooks → Skills → Plugins → MCP + LSP + 子 agent。

## 🎯 为什么在百万行大代码库上会崩

平时在自己的小项目上跑 Claude Code 很舒坦，一旦放到公司百万行级别的大代码库，所有问题立刻浮出来。小林把 Anthropic 官方博客 + 创始人 Boris 的经验串起来，总结出大代码库里最容易踩的 **7 个坑**：

| 问题 | 一句话答案 |
| --- | --- |
| Q1 context 老爆，是模型太小吗？ | 换模型没用，是**没给 Claude 一个好的起点 context**（agentic search） |
| Q2 CLAUDE.md 写多长合适？ | 单文件 **200 行以内**，分层加载、持续狠删、每 3-6 个月审查 |
| Q3 找函数总找错文件？ | 装 **LSP**（按符号搜）+ **在子目录启动** |
| Q4 跨几十个文件改一半就崩？ | 拆成**多个会话 + subagent**，不要写更长的 prompt |
| Q5 只有我一个人会用，怎么推广？ | **skill → plugin → MCP** 打包分发 + 专人维护 |
| Q6 创始人平时怎么用？ | 并行跑多个 Claude，Plan Mode 起步，PostToolUse hook 兜底 |
| Q7 什么项目不适合用？ | 非「Git + 工程师 + 标准目录」场景会吃力 |

---

## 核心概念：harness 跟模型一样重要

很多人评估 Claude Code 时盯着模型选型（Sonnet 还是 Opus？benchmark 谁高？），但 Anthropic 抛了个反直觉论点：**「The harness matters as much as the model」**——在实际生产中，围绕模型搭的那套外壳对最终效果的影响，比模型本身还大。

> 打个比方：你请了个米其林三星大厨到家里做饭，他厉不厉害是**模型能力**；但家里有没有趁手的灶台、菜刀、调料架、抽油烟机，这才是 **harness**。灶台不行，再牛的厨师也炒不出锅气。

![harness 比喻：大厨 vs 灶台](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-04.png)

**Anthropic 的 harness 一共七层**，每层都建立在前一层基础上：

```
CLAUDE.md → Hooks → Skills → Plugins → MCP
         + 两个增强：LSP、子 agent
```

![harness 七层架构](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-05.png)

后面的 Q 按官方顺序逐层拆解：Q2 拆 CLAUDE.md（含 Hooks）、Q3 拆 LSP 和子目录、Q4 拆子 agent、Q5 拆 Skill/Plugin/MCP、Q6 看 Boris 怎么组合。

---

## Q1：大代码库下 context 老爆，是不是模型太小？

**Anthropic 官方答案：换模型没用，问题不在模型，在 Claude Code 怎么找代码。**

Opus 4.7 已支持 1M token（约两百多万字），但像样点的项目动辄几百万行代码，再大的窗口也塞不下整个代码库——**这是物理上的事**。

![大代码库与 context 的物理矛盾](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-01.png)

### agentic search：Claude Code 的反主流答案

业内的主流答案是 **RAG**：代码切片 → embedding → 向量数据库 → 相似度召回（Cursor、Copilot、Windsurf 都走这条路）。但 Claude Code **偏偏不走**——连 embedding 和向量数据库的影子都没有，就靠 grep、读文件、看目录这种最朴素的方式，Anthropic 叫它 **agentic search**。

![RAG vs agentic search](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-02.png)

Claude 像一个真人工程师：先 `ls` 看根目录 → 进 `auth/` 看里面 → `grep "login"` 找函数 → 读 `middleware.ts` 和 `session.ts`，读一个文件决定下一步读什么，循环往复。

![agentic search 工作示意](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-03.png)

### 为什么选 agentic search（三个理由）

| 理由 | 说明 |
| --- | --- |
| **索引会过期** | 千人团队每天几百个 commit，embedding pipeline 跟不上；等查的时候索引里可能是两周前已被重命名的函数。agentic search 每次基于当下代码，没有这个问题 |
| **冷启动几乎为零** | RAG 在百万行代码库建一次索引要十几分钟，Claude Code 打开就能用 |
| **精确匹配向量干不了** | 查 `getUserById`，向量召回会返回 `getUserByName`、`getUserByEmail` 一堆"相关"函数。代码很多时候要的是精确，不是相似 |

### agentic search 的代价

> 官方原话：**它严重依赖一个好的起点 context。** 不给清晰起点，Claude 就会乱翻，等摸清结构 context 已被烧得差不多。

所以 context 爆不是模型小，是**没给 Claude 一个好的起点**。Q2-Q7 都在解决这件事。

---

## Q2：CLAUDE.md 到底写多长合适？

**官方答案：单文件控制在 200 行以内。** 超过 200 行后，Claude 开始忽略指令的概率会肉眼可见上升（CLAUDE.md 每次启动都被整个塞进 context，写太长等于跟自己抢空间）。

### 分层加载

> 官方原话：「根目录的 CLAUDE.md 应该只放**指针和关键的坑**，其他细节都会变成噪音。」

- **root 文件**：只放跨包通用约定（"生产数据库千万别动""提 PR 前要跑 lint"）；
- **每个子目录**：放自己的 CLAUDE.md 写模块细节。Claude 会自动从当前目录往上走树，沿途把每个 CLAUDE.md 都加载进来。

![CLAUDE.md 分层加载](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-06.png)

### 持续狠删（Boris 的 slogan）

**「Ruthlessly edit your CLAUDE.md over time」**——对你的 CLAUDE.md 下狠手，毫不留情地删。检查法：对每一行问自己「如果删掉这行，Claude 还会按这条规则做事吗？」答案是「会」（常识或代码已体现）就该删，答案是「不会」才值得留。

![删行检查法](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-07.png)

发现 Claude 反复犯某个错，先别急着加新规则，先看 CLAUDE.md 是不是已经太长把规则淹没了。

![CLAUDE.md 过长淹没规则](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-08.png)

### 每 3-6 个月完整审查一次

**因为模型在进化。** 三个月前为了约束 Claude 写的"每次重构只改一个文件"，可能在新模型上反而变成枷锁——新模型已能做跨文件协调编辑。为弥补旧模型弱点写的 Hook、Skill，模型升级后可能变成多余负担。感觉 Claude Code 用得上不去一个台阶，先回去看 CLAUDE.md 是不是过期了。

### 🔗 Hooks：让 CLAUDE.md 自我进化（本文的 Hook 重点）

Hooks 是 Claude Code 的**事件钩子机制**，在「编辑完文件之后」「会话开始之前」「工具调用之前」这些时间点上挂脚本做事。

大多数人对 hook 的认知停留在「防止 Claude 做错事」（自动 lint、自动 format）。但官方点出一个反直觉的洞察：

> **hook 真正的价值不是阻止 Claude 做错事，而是让你的整套设置自我进化。**

- 挂 **Stop hook**：每次会话结束时自动反思「这次 Claude 有没有常犯的错误？要不要写进 CLAUDE.md？」然后 hook 自己改 CLAUDE.md；
- 挂 **Start hook**：根据当前所在子目录动态加载模块特有 context——今天在 `payments/` 下就自动拉支付 skill，换到 `auth/` 下就换认证相关；
- **Boris 自己挂了一个 PostToolUse hook**：给 Claude 写完的代码自动跑格式化，把偶尔遗漏的 10% 格式问题直接抹平。

![Hooks 让 CLAUDE.md 自我进化](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-09.png)

> **CLAUDE.md 不是写一次的文档，是一份持续打磨的活文件。** Anthropic 内部：整个 Claude Code 团队共享一份 CLAUDE.md 提交到 git，发现 Claude 做错了就立刻加进去。

**Q2 总结**：单文件 200 行以内、分层加载、持续狠删、每 3-6 个月审查一次 + 用 Hooks 让维护自动化。

---

## Q3：大代码库里让 Claude 找一个函数，总找错文件？

多语言大代码库（C/C++/Java/PHP 这种符号歧义高的语言）特别突出。官方答案：**装 LSP + 在子目录里启动 Claude**。

### LSP：从按字符串 grep 到按符号搜

LSP（Language Server Protocol）就是 VS Code 里「go to definition」「find references」背后跑的东西。Claude Code 接上 LSP 后，搜代码不再按字符串 grep，而是**按符号搜**：

- 没有 LSP：grep 一个 `getUser` 返回三千个匹配（前端、后端、测试都有），Claude 一个个读文件判断，context 直接被烧光；
- 有 LSP：Claude 直接问「找 `auth/login.ts` 那个 getUser 同源的所有引用」，LSP 一口气返回精确的三个——**过滤工作在 Claude 读文件之前就完成了**。

![LSP 符号检索](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-10.png)

官方把 LSP 称作多语言大代码库下 **「one of the highest-value investments」**。安装方式：`/plugin` 里搜「lsp」装对应语言的 code intelligence plugin（typescript-lsp / pyright-lsp / rust-analyzer-lsp 等），再装对应的语言服务器二进制（pip 装 pyright、npm 装 typescript-language-server），全程不超过两分钟。

![LSP 安装](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-11.png)

### 子目录启动：反直觉但官方反复强调

大多数人习惯 `cd` 到根目录再 `claude`。小项目没毛病，但大代码库里这会让 Claude 一上来就把根目录那个超大的 CLAUDE.md 全部加载，前端后端 infra 所有微服务的规则全来一遍。

官方原话：**「Initializing in subdirectories, not at the repo root」**。正确做法是直接在要改的子目录启动（如 `cd services/payments && claude`）——Claude 会自动往上走树加载根目录 CLAUDE.md（通用规则不丢），但优先加载 `payments/` 子目录的，context 立刻聚焦到一个领域。

![子目录启动](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-12.png)

### 三个配套小细节

1. **测试/lint 命令按子目录写进 CLAUDE.md**：Claude 改支付服务一个文件，却跑整个项目的测试套件，几十分钟出结果 + context 烧光。每个子目录应写明"这块用什么命令测、怎么 lint"；
2. **用 `.ignore` 排除生成文件/构建产物/第三方代码**：把 `permissions.deny` 规则提交到 `.claude/settings.json`，全团队自动共享；
3. **目录结构不直观时放一张「代码库地图」**：一个 markdown 文件，列出每个顶层文件夹的一句话说明，Claude 动手前先扫一眼。

---

## Q4：跨几十个文件的改动，改一半就崩？

**官方答案：跨大量文件的改动，正确解法是把任务拆成多个会话 + 用 subagent，不是写更长的 prompt。**

Boris 的原话：「Pour your effort into the plan so Claude can one-shot the implementation」——与其用超长 prompt 让 Claude 一次搞定所有事，不如先单独花一轮把方案敲定，再分多个会话实现。

### 第一步：派 subagent 出去探索，主 agent 留着干净的 context

大代码库下「读懂系统怎么工作」本身就要烧掉好几万 token。让 Claude 一边读代码一边改代码 = 让人一边查资料一边写论文。

**Subagent 思路**：派一个小弟去探索，写一份 findings 报告回来，主 agent 看完报告再动手。小弟在独立 context 窗口里跑，读几十个文件烧的是自己的 context，最后只把几百字摘要给主 agent。

![subagent 探索 1](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-13.png)

![subagent 探索 2](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-14.png)

最简单的操作：直接跟 Claude 说「先用 subagent 调查一下 X 是怎么实现的，写成 findings 文件，再回来动手改」。

### 第二步：会话拆分

- 会话 1：只做探索、写 plan，不动代码；
- 会话 2：加载 plan，实现一个模块，跑通测试；
- 会话 3：实现下一个模块。

每个会话从干净 context 开始，**plan 文件做桥梁串联**。

![会话拆分](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-15.png)

### 第三步：跑大型迁移用 /batch

「整个项目从框架 A 迁到 B」「几十个文件的某种调用全部替换」这种大规模迁移，Claude Code 内置了 `/batch`：先用对话敲定迁移方案，它一次性派出**几十个并行 subagent**，每个在独立 git worktree 里跑、自测、开 PR。不用守屏幕，跑完直接给你一堆 PR 等 review。

![/batch 并行 subagent](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-16.png)

> 这就是 Boris 本人正在用的工作流——以前要自己手撸的多 agent 编排，现在一行命令搞定。

**跨大文件改动救不回来的不是 prompt，是会话边界。**

---

## Q5：团队里只有我一个人会用 Claude Code，怎么推广？

官方答案四步：**先把好实践做成 skill → 用 plugin 打包分发 → 用 MCP 接内部系统 → 最后得有人维护。**

### 第一步：把高频操作做成 skill

Skill 可以理解成「针对某个具体任务的 SOP」（如"数据库迁移怎么做""微服务上线标准流程"）。它跟 CLAUDE.md 最大的区别在**按需加载**：CLAUDE.md 每次会话都全文加载，skill 只在 Claude 判断"当前任务需要"时才加载——官方术语叫 **progressive disclosure（渐进式披露）**。

> Boris 名言：「如果一件事你一天做超过一次，就把它做成 skill。」

Skill 还可以绑定到特定路径（如"支付服务部署 skill"绑定到 `services/payments/`），避免改前端代码时支付 skill 也来凑热闹。

![skill 按需加载](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-17.png)

### 第二步：用 plugin 打包分发

大公司经典问题：好的工具配置只在小圈子流传——某个高级工程师本机配了三十个 skill、十几个 hook、五个 MCP server，实习生啥都没配，体验像 demo 版。

**Plugin 本质是一个安装包**，把 skill、hook、MCP、LSP 配置打包在一起。新人入职第一天 install 一下，立刻和团队所有人能力一致。公司还可以建自己的 **plugin marketplace**，谁有好实践就更新进去。

![plugin 打包分发](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-18.png)

### 第三步：用 MCP 把团队内部系统接进来

装 Slack MCP 就能搜公司 Slack 消息，装 BigQuery MCP 就能跑数据查询，装 Sentry MCP 就能拉线上错误日志。

⚠️ **但官方特别提醒：别太早上 MCP。** 很多团队 CLAUDE.md 还没写好、hook 也没挂，就着急接各种 MCP，反而把 context 搞得更乱。**正确顺序：先把 CLAUDE.md 和 skill 打磨好 → 再用 plugin 打包分发 → 最后才上 MCP。**

![MCP 接入顺序](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-19.png)

### 第四步：得有人负责维护

推广最顺的组织有个共同点：大面积铺开前，先安排一小队人把整套基础设施搭好，然后才放开访问。开发者第一次用就能跑通——**第一印象如果是"这东西不好使"，后面要翻盘就太难了**。

官方点出一个正在浮现的新角色 **Agent Manager**（半 PM 半工程师，负责 plugin 分发、CLAUDE.md 规范、skill 审批）。小团队没条件设岗位，也至少要有一个 **DRI（直接责任人）** 维护配置、有拍板权。没有人盯着，再好的 plugin 也会变成「张三两年前搭的，没人会改」的部落知识。

---

## Q6：Boris 自己平时怎么用 Claude Code？

创始人分享过一段让人破防的话：**「我同时在终端里跑 5 个 Claude，再加 5 到 10 个跑在 claude.ai/code 上，并行处理不同任务。」**

![Boris 的 setup](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-20.png)

他的 setup 拆解（藏着创始人对 Claude Code 用法的全部理解）：

| # | 做法 | 要点 |
| --- | --- | --- |
| 1 | **不用 `--dangerously-skip-permissions`** | 用 `/permissions` 把常用安全命令预先加白名单，避免反复点确认，但不放弃权限审计 |
| 2 | **几乎所有复杂任务都从 Plan Mode 开始** | 先敲定方案，再切 auto-accept 模式让代码一发命中 |
| 3 | **挂 PostToolUse hook** | 自动格式化，抹平 Claude 偶尔遗漏的 10% 格式问题，避免 CI 挂掉 |
| 4 | **每天做超过一次的事都做成 slash command / skill** | 他自己有个 `/commit-push-pr` 命令，一天用几十次 |
| 5 | **给整个团队共享一份 CLAUDE.md 提交到 git** | 发现 Claude 做错就立刻加进去，持续打磨的活文件 |

> 创始人对 Claude Code 的态度不是「装上就用」，而是**把它当成一个会进化的工作伙伴**，每天都在喂它新规则、新工具、新工作流。这才是大代码库下用好 Claude Code 的底层心态。

---

## Q7：什么样的项目其实不适合用 Claude Code？

官方原话：「Claude Code 是围绕**传统软件工程环境**设计的，假设工程师是代码库的主要贡献者，仓库用 Git，代码遵循标准目录结构。」

以下几类场景会比较吃力：

| 场景 | 原因 |
| --- | --- |
| 游戏引擎等大量**二进制资源**的项目 | Claude 没法读 3D 模型、贴图、音频 |
| **非常规版本控制系统**（Perforce / Subversion / 自研 VCS） | 需要额外配置才能跑顺 |
| **非工程师为主贡献**的代码库（产品文档、Figma 配置文件） | harness 不太对得上 |

![不适合的场景](https://gitee.com/cheng-jiaqing/images/raw/master/claude-code-large-codebase-20260522-21.png)

> **Claude Code 不是万能药，最擅长的是「Git + 工程师 + 标准目录」这个最大公约数。**

---

## 最后：浓缩成 3 句话

1. **Claude Code 在大代码库不是"装上就能用"**，是要在 harness（外围基建）上花一次性功夫的；
2. **最高 ROI 的三个动作**：CLAUDE.md 砍到 200 行以内 + 在子目录启动 Claude + 装 LSP——做完体验立刻不一样；
3. **跨大文件改动、团队推广、CLAUDE.md 维护这些硬骨头**，官方都给了具体答案，Boris 自己也在用，照抄就行。

---

## 🔗 Hook 主题联动（跨笔记整理）

这篇和 [[Clippings/微信公众号/2026-08-11-Agent治理-用Hook堵住LLM的偷懒越权与失忆]] 都深度讲了 Hook，但角度不同，正好互补：

| 维度 | 本文（Claude Code 官方视角） | Agent 治理（腾讯 DECO 生产实践） |
| --- | --- | --- |
| Hook 用途 | **让 CLAUDE.md 自我进化** + 自动格式化/加载 context | **三类确定性兜底**：长文本 offload、危险操作 HITL、上下文联动 |
| 典型切面 | Stop / Start / PostToolUse（Claude Code 的事件钩子） | beforeTool / afterTool / beforeModel / afterModel / before&afterAgent（Agent 框架通用） |
| 心智模型 | harness 七层之一：CLAUDE.md → **Hooks** → Skills → Plugins → MCP | 横切关注点：不改业务循环，把护栏逻辑挂在切面上 |
| 共同结论 | **「The harness matters as much as the model」** | **「prompt 定意图，Skill 定规矩，框架 Hook 定边界」** |

**两者的本质是同一个思想**：把不能交给模型的确定性逻辑，下沉到框架/工具层用代码强制实现。本文教你**怎么用 Hook 让配置自我进化**（维护侧），Agent 治理教你**怎么用 Hook 兜住生产事故**（安全侧），合起来就是生产级 Agent 工程化的完整闭环。

---

## 📌 延伸思考

- **官方原文值得精读**：https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start
- **Agent Memory 笔记呼应**：[[Clippings/微信公众号/2026-08-11-TencentDB-Agent-Memory-实测笔记]] 讲的记忆金字塔，与本文的 CLAUDE.md 分层加载、skill 渐进式披露是同一类"按需加载、稳定前缀优先"的 context 管理思想。
- **你的库里有大量相关素材**：`项目/claude-code-best-practice/` 目录下已有 Boris 技巧、Skill 教程、Harness 文档等，可与本文交叉验证。
- **可立即落地**：按 Q2-Q3 的 ROI 三件套检查你手上的项目——CLAUDE.md 是否超过 200 行、是否在子目录启动、是否装了 LSP。
