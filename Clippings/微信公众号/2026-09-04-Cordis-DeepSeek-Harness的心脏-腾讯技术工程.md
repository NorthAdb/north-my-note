---
created: 2026-09-04
updated: 2026-09-04
tags: [AI, Agent, DeepSeek, Harness, Cordis, 插件架构, 一切皆插件, Fiber, effect, inject, Schema, 元框架, 2026论文, 腾讯技术工程]
source:
  - "https://mp.weixin.qq.com/s/3vtCkp6EbA5MhRERD6f17A"
  - "公众号：腾讯技术工程"
publisher: 腾讯技术工程
author: lss233
article_date: 2026-09-04
evidence: article-summary
---

# DeepSeek Harness 背后的「心脏」：Cordis 到底是什么

> **作者**：lss233 | **来源**：微信公众号「腾讯技术工程」 | **发布**：2026-09-04
>
> 副题：从 Koishi 到 DeepSeek Harness——Cordis 的前世今生
>
> 本笔记基于原文重新整理；框架与论文表述均以原文、官方文档和预印本为准。

## 一句话结论

Cordis 是 Shigma 编写的一个**元框架（meta-framework）**：不管"插件是什么业务"，只管"插件何时加载、何时卸载、彼此如何依赖"。DeepSeek Harness（DSH）"一切皆插件"的底座就是它。它把插件系统最难的两件事——**卸载**（可逆副作用）和**协作**（响应式依赖）——从"插件作者的自觉"上升为**框架级保证**，并配有 88 页论文将其机制形式化（北大 + DeepSeek-AI）。

**与仓库内 DSH 笔记的关系**（本次另起一篇的原因）：
- [[Clippings/微信公众号/2026-08-14-DeepSeek-Harness拆解-一套能拼装的Agent架构]]（chino，2026-08-14）——已深挖 Cordis 的实现细节（fiber 状态、撤销栈、遮蔽算法）；
- [[Clippings/微信公众号/2026-08-18-一文读懂DeepSeek-Harness插件-Datawhale]]（Datawhale，2026-08-17）——概念入门视角；
- 本文是 **Cordis 本体的完整读本**：五个核心概念逐一拆解 + DSH 落地细节 + 插件生态 + 论文定理。三篇按"入门 → 实现 → 理论"递进，合起来覆盖完整。

## 为什么插件系统需要一个框架

模块化只解决"代码怎么组织"，解决不了另外四件事：

| 问题 | 内容 |
| --- | --- |
| 安装 | 新功能怎么被"接"进系统？ |
| 配置 | 同一功能在不同环境用不同配置，写在哪里？ |
| 卸载 | 功能下线时，定时器/监听器/连接由谁清理？清理不干净就是泄漏 |
| 协作 | A 依赖 B 的能力，但 B 可能没启动、之后可能被替换，A 怎么办？ |

插件系统就是这四问的回答。Cordis 的特别之处：**把"卸载"与"协作"从插件作者的自觉，变成框架级保证。**

### 作者 Shigma

第三方 QQ 机器人圈里人称"梦梦"，GitHub 约 130+ 公开仓库，npm 上一长串 maintainer 都是他：

| 包 | 一句话描述 |
| --- | --- |
| koishi | 跨平台聊天机器人框架（"Made with Love"） |
| **cordis** | 插件化应用框架，本文主角 |
| @satorijs/core | 跨平台聊天协议适配层 |
| schemastery | 类型驱动的 schema 校验器 |
| minato | 类型驱动的数据库框架 |
| cosmokit | 通用工具集 |

一套一个人撑起的技术栈：**Cordis 骨架（生命周期+依赖）、Schemastery 管配置校验、Minato 管数据、Satorijs 管协议、Koishi 集大成**，全部运行在 Cordis 之上。2023 年底接受腾讯媒体研究院《20 多岁做什么时间更有价值》专访（BV1AQ4y157S8），那时是研二学生；如今 Cordis 配套论文《A Programming Paradigm for Spatiotemporal Composability》以预印本发布，署名北大 + DeepSeek-AI，第一作者 Yifan Shi。

### 出身与名字

Koishi 2020 年 1 月发布首个正式版；2022 年 4 月 `cordis` 包登上 npm；2023 年底进入 3.x；2024 年 11 月起迭代 Cordis 4（彻底重构，引入基于 fiber 的生命周期体系）。**Cordis 是拉丁语"心"（cor）的所有格**——它是 Koishi 的心脏，如今也是 DeepSeek Harness 的心脏。

### 定位：元框架

不是机器人框架（Koishi 才是），也不是传统 DI 容器。传统 DI 回答"谁创建谁"，Cordis 回答更难的："**谁在什么时候活着**"。论文定位为 meta-framework：规定"副作用如何组合、依赖如何解析"，但不预设业务领域。

![Shigma 的 GitHub 主页](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-01.png)

## 五个核心概念

### 1. 插件：三种形态

写插件完全不需要框架启动代码，`apply(ctx)` 就是全部：

```ts
export const name = 'hello'
export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

应用长什么样由 `cordis.yml` 决定 → **"配置即组合"**。三种形态：

| 形态 | 写法 | 用途 |
| --- | --- | --- |
| 函数 | `export function apply(ctx)` | 最常见 |
| 对象 | `{ name, apply(ctx) }` | 带元信息 |
| 类 | `class X extends Service { constructor(ctx) { super(ctx, 'myService') } }` | 对外提供服务 |

DSH 里每个工具插件、适配器插件、面板插件都是这三种之一。

### 2. 上下文 ctx：一切操作的入口、一棵树

```ts
ctx.on('some/event', handler)   // 监听事件（卸载自动移除）
ctx.effect(() => ...)           // 注册副作用（卸载自动回滚）
ctx.plugin(SomePlugin)          // 挂载子插件（随父插件卸载）
ctx.get('someService')          // 读服务（没有则 undefined）
ctx.provide('someValue', 42)    // 提供服务
```

`ctx.plugin(child)` 不是"注册"，而是**派生一个子上下文**。插件因此是一棵树：子上下文继承父上下文的一切，卸载按层级递归（父卸载 → 子全卸；子卸载 → 不动兄弟和父级）。DSH 里每个功能（工具、模型、会话、面板）都是树上的一次 `ctx.plugin()`。

![插件树：Context 派生结构](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-02.png)

### 3. fiber：插件的生命周期

Cordis 4 为每个已加载插件实例维护一个 fiber（纤维）：

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
              ↓
            FAILED（apply 抛异常 / 配置校验失败）
```

插件会因配置修改、热重载、显式 `dispose()` 或依赖服务消失而卸载，**无论何种原因，清理都是自动的**。DSH 里 `cordis_inspect` 巡检的就是每个 fiber 的状态；"插件加载了却没反应"，多半蹲在 PENDING。

![fiber 状态机](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-03.png)

### 4. effect：可逆的副作用（核心机制一）

```ts
ctx.effect(() => {
  const conn = createConnection()
  return () => conn.close()   // disposer：如何清理
})
```

主体加载时执行，返回的 disposer 卸载时执行——**你永远不需要自己调用清理函数**。更关键的是：`ctx.on()`、`ctx.plugin()`、服务注册，**全部都是 effect 的特例**。论文实现章节的结论：所有对上下文的变更最终都归结为 `ctx.effect` 一个原语。因此"任何通过上下文进行的操作自动可追踪、可恢复"不是口号，而是**结构事实**。

插件作者铁律：凡是自己创建、Cordis 不管的资源，都包进 `ctx.effect()`。副作用可逆 ⇒ 插件可安全卸载重装 ⇒ HMR、故障自动恢复、测试隔离全部成立。**DSH "改配置不用重启"的底层原理就是它。**

### 5. 服务与注入：响应式的依赖（核心机制二）

```ts
export const inject = ['greeter']   // 声明依赖
export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))  // 此时 greeter 必定就绪
}
```

`inject` 语义：插件保持 PENDING，直到所列服务**全部就绪**；配置顺序无关紧要，启动顺序由依赖关系决定。

与传统 DI 的本质区别：传统 DI 假设"一旦绑定，服务就一直在"；Cordis 假设"**服务可以随时出现，也可以随时消失**"。Agent 场景是常态（LLM 限流、MCP 崩溃、watcher 被杀）。Cordis 的处理：提供方卸载 → 依赖它的插件自动卸载（effect 回滚）；新提供方就绪 → 自动重载。**依赖方不需要写任何重连代码。**

配套机制：
- `ctx.isolate(key, realm)`：隔离，让某作用域内服务解析到独立实例（多租户、测试隔离）；
- `ctx.intercept(key, meta)`：拦截，给依赖访问附加元数据，外层约束内层如何使用依赖，不改组件本身（沙箱策略等）。

DSH 中 `ctx.tools`、`ctx.llm`、`ctx.shell`、`ctx.sessions`、`ctx.skills` 全都是这样的服务。

### 事件：喊话与决策链

类型化事件（TypeScript 声明合并获得全链路类型安全）。分发模式是公开契约：

| 模式 | 语义 |
| --- | --- |
| emit | 同步广播，不等待、不收集返回值 |
| parallel | 所有监听器并发执行并等待 |
| serial | 按序执行，第一个非空返回值胜出并短路 |
| bail | serial 的同步版本 |
| **waterfall** | 环绕中间件：每个监听器拿到参数 + `next()` continuation |

waterfall 本质是把 Koa/Express 中间件搬进事件系统：不调 `next()` = 否决短路；调 `next()` = 放行。**多个互不相识的插件由此组成一条决策链。** DSH 明文纪律：只负责观察记录的 waterfall 监听器必须调用 `next()`，否则会无声吞掉下游所有默认行为。DSH 的工具执行管道 `tools/pre-execute → tools/execute → tools/post-execute`、`approval/request`、`agent/request` 都是 waterfall 链。

![waterfall 决策链](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-04.png)

### Schema 与声明式组合：配置即程序

`cordis.yml` 每条 entry 可带元数据：`id`（稳定身份，loader 靠它区分"修改"与"删了重加"）、`group`（整组装卸）、`disabled`（保留但不挂载）、Schema 校验（schemastery，apply 前校验，配置非法则精确报错，插件绝不会半启动）、`!!js` 表达式（DSH Loader 扩展，运行时求值，如 `!!js process.env.X ?? 'default'`）。

## 在 DeepSeek Harness 中：一切皆插件

以下基于 DSH 0.1.0-rc.6 实际源码（`@deepseek-ai/cordis` 4.0.1）。

### 启动：约二十行代码搭起整个应用

```ts
const ctx = new Context()
ctx.provide('dshHomePath', dshHomePath)      // 引导值
await ctx.plugin(Loader)                     // 挂载 Loader
await mountRootInclude(ctx, configPath, patches, baseUrl)  // Include 挂载配置树
await ctx.get('loader')?.await()             // 等整棵树稳定
await assertEntriesActivated(ctx, binName)   // 审计：不允许条目半死不活
```

根 Context 只做三件小事，**其余一切来自配置树**。

### Profile：应用被拆成可叠加的层

`$DSH_HOME/profiles/<名字>/` 含 manifest（`dsh.profile.bundles` 按顺序应用的组合包）+ 用户 `cordis.patch.yml`。Bundle 是一个 npm 包，`package.json` 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`。核心组合包 `@deepseek-ai/dsh-base` 的 patch 就是一次 insert 几十个插件（timer、hmr、llm、session、agent、jobs…）。部署方改默认行为无需改源码，在自己的 patch 层按 `id` 覆盖一行即可——**"应用由哪些插件组成、各是什么配置"本身，就是可叠加、可覆盖、可审计的声明**。

![Profile / Bundle / Patch 叠加组装](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-05.png)

### 工具流水线：Agent 的每一项能力

`ctx.tools` 是 Agent 能力边界，注册工具本质就是一个 Cordis 插件：`inject: ['tools']` 等注册表就绪 → `ctx.tools.register(...)` 注册**即 effect**（插件卸载工具自动注销）→ `tools/result` 事件让任何插件观察每次调用。本文写作用到的 bash、read、grep、subagent 全部挂在 `ctx.tools` 上。

### 自指的设计：Agent 检查并改装自己的运行时

`@deepseek-ai/dsh-tool-cordis`，官方称"自指的 Cordis 工具集"（self-referential Cordis toolset），给 Agent 五个工具：

- `cordis_inspect`：只读巡检（哪些服务在运行、各 fiber 状态、注册了哪些工具、每个 `ctx.<key>` 的契约）；
- `cordis_define`：现场定义小插件包（可带"宿主半 + 浏览器半"），只记录不执行；
- `cordis_run`：宿主半进 `node:vm` 沙箱执行，浏览器半推送到打开的网页；
- `cordis_stop` / `cordis_undefine`：卸载动态包。

Agent 可以检查自己运行的框架、现场编写运行动态插件、用完卸载——**全程不动 cordis.yml、不装 npm 包、不重启进程**。信任立场：动态包与 bash 同权，沙箱隔离全局但不构成安全边界。配套"双半插件"设计：宿主半跑服务端管逻辑，浏览器半跑网页管 UI，`host.call` 做 RPC——前端 UI 组件也是插件（`dsh-client-ui-*` 几十个包），浏览器里跑着独立的 Cordis 客户端运行时。**自省 + 现场改装 = "可进化 Agent" 的雏形**（论文结论把它列为未来验证方向）。

## 在 DSH 里能开发什么：槽位与生态

### 槽位：扩展点全部是服务

DSH 的扩展点不是 API 列表，而是一张 Cordis 服务注册表：任何插件可注册新服务，也可替换已有提供方（§5 响应式依赖的直接应用）：

| 类别 | 槽位（`ctx.<key>`） | 现有提供方（可替换） |
| --- | --- | --- |
| 执行 | shell / codeRuntime / subprocess / terminals / lsp | bash-local、bash-sandbox、pwsh-local、E2B、code-runtime-worker、lsp-local |
| 模型 | llm | llm-deepseek、llm-pi-ai、llm-replay |
| 智能 | agents、agentLoop、agentDefaultModel、agentPresets、subagents | agent-loop、subagent-spawn/fork-in-process、ACP、Codex、Claude Code |
| 数据 | sessions、sessionPersistence、sessionQuery、storage、attachments、spillStore、sessionTitle、sessionProjections | session-persistence-jsonl/-sqlite、storage-json/sqlite |
| 环境 | fs、web、credentials、settings、webServer、directoryPicker | fs-local/fs-sandbox/fs-e2b、web-fetch-http、web-search-deepseek/exa/perplexity |
| 治理 | tools、approval、permissionPresets、sandboxPolicy | 所有 dsh-tool-\*、approval、permission-presets |
| 编排 | goals、jobs、workflowEngine、planMode | dsh-goal、dsh-jobs-local、workflow-worker-thread |
| 自指 | dynamicCordisRunner、cordisInspect | cordis-host-runner |
| 前端 | slots、clientModules、theme、locale | dsh-client-ui-\* 系列 |

![DSH 设置页插件清单](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-06.png)

### 槽位背后的设计：三种角色

以 `shell` 为例：**Service Definition**（只声明服务与类型，几乎不变）/ **Service Provider**（可独立替换，改一行配置，Definition 和所有 Consumer 不变）/ **Consumer**（与 Provider 互不依赖）。每一项能力都是一个 seam（接缝），两侧独立演进。

![Definition / Provider / Consumer 三种角色](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-07.png)

### 三种开发入口

1. **patch 覆盖层（最快）**：`dsh web --patch ./my-plugins.yml` 启动时插入条目；
2. **profile 的 cordis.patch.yml**：常驻生效的用户层；
3. **bundle 包**：打包成 npm 包，声明 `dsh.bundle.patch`，成为可复用"组合包"。

### 已存在的插件（举例）

- **工具类**：tool-bash、tool-fs、tool-fs-search、tool-web、tool-subagent、tool-subagent-control、tool-workflow、tool-goal、tool-jobs、tool-todo、tool-skill、tool-ralph、tool-ask-user、tool-lsp、tool-cordis；
- **提供方类**：模型适配器（DeepSeek、pi-ai、replay）、shell/fs/web 搜索/持久化/存储/技能/会话标题等各提供方；
- **系统类**：compaction-basic（上下文压缩）、token 计量、消息反馈、权限预设、计划模式、长期目标、审批管道；
- **前端类**：dsh-client-ui-\* 数十个包（设置页、模型选择、插件管理、任务面板、子 Agent 面板、主题）。

**社区生态**：GitHub 精选列表 awesome-dsh-plugin（搜 `dsh-plugin` 话题），截至 2026 年 8 月收录 **174 个**可通过 `dsh plugin add` 安装的插件。代表例子：

- 改界面：dsh-visualize（模型画交互式 HTML 卡片）、dsh-TUI（像素鲸鱼全屏终端 UI）、dsh-deep-whale（Web 皮肤）、dsh-balance-meter；
- 改记忆：dsh-memento（有界分层带审批门可审计的跨会话记忆）、dsh-mneme（SQLite + Markdown 镜像）；
- 改能力：dsh-computer-use（控制 macOS）、dsh-data-agent（数据库写 SQL）、dsh-docker（带护栏容器控制）、dsh-toolkit；
- 改协作：dsh-agent-teams（多智能体团队）、dsh-crosstalk（跨会话互发消息）、dsh-chat-import；
- 自进化：dsh-evolve（会话内热挂载/卸载持久化插件）、dsh-continual-evolve（从轨迹沉淀可审计可回滚的 harness 状态）；
- 插件基建：dsh-find-plugin（会话内搜索并安装插件）、dsh-plugin-manager（`dsh pm` 多源管理）。

![社区插件：dsh-agent-teams 截图](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-08.png)

![DeepSeek Harness 工作区（原文配图）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-09.jpg)

### 可能的插件：常规与"惊艳"

**常规**：新工具（数据库查询、浏览器自动化、Docker/K8s、MCP 客户端、Git、部署测试）、新提供方（任意 LLM 适配器、远程/容器 shell、新搜索源、持久化后端）、治理与可观测（审批策略、权限预设、审计日志、token 预算、OTel）、UI（Web 面板、命令面板、HTTP API）。

**惊艳**（每一步都落在现成机制上）：

- **会写插件的插件**：`ctx.loader.create()` 动态挂载/卸载配置条目 + `cordis_define`/`cordis_run` → 现场生成并运行自己的工具，自进化闭环"生成 → 沙箱运行 → 观测 → 保留或卸载"；
- **会话级隔离"分身"**：`ctx.isolate` + `agentPresets`，一个进程里跑多个"虚拟 Harness"；
- **不停机换引擎**：patch 层替换 llm/shell 提供方，依赖方自动卸载重载；按论文 service broker 模式可实现灰度与滚动切换；
- **策略网关**：waterfall 把"审批 → 权限 → 限流 → 审计"串成一条链，加策略 = 加一个监听器插件；
- **专家 Agent 池**：基于 `ctx.subagents` 常驻子 Agent（审查员/测试员/文档员），子 Agent 从"一次性工具"升级为"常驻服务"；
- **分层记忆系统**：在 `sessionProjections` 上实现工作记忆/长期记忆/摘要折叠；
- **浏览器双半插件**：一个动态包同时带宿主半 + 浏览器半，现场注入 UI 面板用完即卸；
- **工作流即插件**：在 `workflowEngine` 上注册新的编排范式。

共同点：都靠"插一个插件"实现，而不是"改框架"。

## 论文说了什么：从"好用的框架"到"被证明的范式"

《A Programming Paradigm for Spatiotemporal Composability》（cordiverse/paper，88 页预印本，北大 + DeepSeek-AI）。主张：**动态组合（运行时装卸组件）缺乏形式基础；Cordis 用两个运行时机制补上，并证明了由此得到的性质。**

### 要解决的问题：动态组合为什么难

动态组合拆成两个正交维度：

- **时间维度（temporal composability）**：组件被移除时，它对共享环境做的修改必须被完整、安全逆转。静态程序里对应词法作用域/RAII；动态场景下副作用是长生命周期、没有词法边界的。
- **空间维度（spatial composability）**：组件必须能声明、发现、解析相互依赖。

长期被回避的原因：OS 和容器提供了粗粒度替代——**进程粒度的时间可组合性 + 服务粒度的空间可组合性**，但代价沉重（重启丢全部进程内状态、跨地址空间只能网络表达）。VS Code 是反例：前 100 个热门扩展 87 个含可执行代码，但扩展宿主无法运行时卸载单个扩展（必须重启）；`deactivate` 只在进程退出时调用，且"创建效果"与"清理效果"分在两处；`extensionDependencies` 只有 7 个扩展使用（类型是 any）。

### 理论支柱：effect 与 coeffect

- **effect（效应）**：计算对环境做了什么（写文件、发消息）；源自 Moggi 的 monad 理论、Haskell IO、代数效应。
- **coeffect（余效应）**：计算对环境要求什么（读哪些配置、依赖哪些服务）；源自 Petricek 等人 2013 年的工作，是 effect 的对偶。

关键观察：这两个概念传统上都是**编译期静态分析工具**；Cordis 论文把它们提升为**运行时机制**，让它们在组件随时到达离开的动态场景下工作。

### 可逆副作用：把"撤销"做成结构保证（时间维）

- **建模**：`e : Γ → Γ × (Γ → Γ)`——effect 作用于当前上下文 Γ，返回新上下文 + 逆函数。把逆交给运行时，effect 就可追踪。
- **track / recover**：执行时把逆复合进累加器 φ；卸载时对当前状态应用 φ，`φ(γ) = γ₀` 即健全性不变量（soundness invariant）。逆按相反顺序累积（twisted composition），天然 LIFO。
- **性质 1 精确恢复**（Theorem 7、16）：每个逆在它自己应用时的状态上执行，一切状态被还原。
- **性质 2 独立性**（Corollary 21）：两两独立的 effect 可按任意顺序撤销——"从交错运行的多个组件中单独撤掉一个"的理论基础。不同 key 上的操作天然独立（Theorem 40），这正是事件监听器、路由注册可任意增删的数学解释；有序链（中间件顺序）不独立，所以需要 `{ prepend }` 和 waterfall 显式顺序语义。
- **性质 3 观察等价 ≃**：恢复不是字面相等（free 不恢复堆布局），而是"任何观察者都看不出区别"（Definition 33）。观察手段即 coeffect，每个依赖键自带等价关系。

![论文：effect 与逆的累加（LIFO）](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-10.png)

### 响应式余效应：把"依赖"做成结构保证（空间维）

- **依赖表 Σ**：类型化部分函数，依赖键 k → 类型 V_k 的值；`set(k, v)` 的类型恰好是 effect 函数——**注册依赖本身是可逆 effect**，撤销提供方时依赖自动消失。
- **规范与通知**：组件用规范 `d ⊆ K` 声明所需依赖；每个上下文变化对照规范分类为 **activating**（之前不满足现在满足）/ **deactivating**（反）/ **neutral**。反应式不变量：activating 触发组件执行（带完整 effect 追踪），deactivating 触发组件恢复。所有变更都经 effect 函数，满足性变化在每个 effect 边界可检测——**"每个依赖变化都会被观察到"是代数层面的保证**，不是轮询。
- `inject: ['greeter']` 的数学形式：声明 `d = {greeter}`，fiber 保持 PENDING 直到 `σ ⊨ d`。
- **隔离**：realm 表，`get(k)` 先解析 realm；同键不同上下文解析到不同值（运行时 ad-hoc 多态）；派生实现，回收时直接丢弃无需逆。
- **拦截**：`get(k, μ)` 实际调用 `σ(k)(μ ⊕ ι(k))`，外层可约束内层如何使用依赖——沙箱策略给 shell 键附加元数据，约束所有使用方。

![论文：依赖变化的 activating / deactivating](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-11.png)

### 统一上下文：Context 即范式

**一个自相似的递归类型**：`Γ∞ = μΓ. Γ × (Γ→Γ) × Σ`（当前状态 / 累加器 / 依赖表）。三个投影把"效果"与"余效应"收进同一个 ctx，每个操作可归因到具体组件；Σ 值类型无约束 → **任何需要共享的状态都能编码成依赖**。层次组合 = 递归结构直接推论：加载=插电，卸载=拔电，父聚合子 effect，任意嵌套——这就是插件树。

**观察等价供给独立性**：在 coeffect 上定义观察等价并商掉，独立性变得可达（Theorem 42）——"把计算的可交换部分与对顺序敏感的部分分开"：可交换的交给 effect（任意顺序），顺序敏感的交给 coeffect（依赖强制顺序）。

**范式定位：两端之间的第三极**：

| 传统 | 代表 | 优点 | 代价 |
| --- | --- | --- | --- |
| 显式状态传递（函数式） | State monad、代数效应 | 效果类型可见、可等式推理 | 状态参数传递、monad 样板爆炸 |
| 隐式变更（命令式/OOP） | React useEffect、Spring getBean | 易用无样板 | effect 靠调用顺序标识；依赖是全局注册表 + null 检查 + 类型转换 |
| **Context 范式** | Cordis | 函数式的可追踪性 + 命令式的易用性 | — |

"对于可逆副作用，开发者提供每个原子操作的逆，复合的逆由组合自动得出，teardown 从 loading 推导，而不是另写；对于响应式余效应，组件只声明所需依赖，运行时自动解析与重接线。"——**原本依赖开发者纪律的正确性，变成了范式的结构性质。**

![论文：Γ∞ 递归类型与统一上下文](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-12.png)

### 动态组合演算与元理论：从局部保证到全局保证

组件 = 三元组 (d, p, e)：声明依赖、声明提供、带见证的 effect 函数；fiber 是一次实例化（携带生命周期状态、累加器、committed view）。系统状态是 fiber 树；coeffect 上下文不是存储的，而是从所有活跃 fiber 的提供表**联合推导**（每个键有且仅有一个提供方，由声明决定）。

五条元理论性质：

| 性质 | 直觉 | 工程意义 |
| --- | --- | --- |
| Preservation（保型） | 每个迁移保持系统良构 | 状态机不会走到非法状态 |
| Temporal composability（全局） | 撤掉一个 fiber，贡献归零，周围原样（Corollary 62） | 单个插件可热卸载，不影响邻居 |
| Spatial composability（全局） | 提供方离开前，依赖它的 fiber 先被停用（Theorem 63） | 不需要手动编排卸载顺序 |
| Progress（进展） | 未达 quiescent 必有规则可应用；步骤数有上界，必然终止（Theorem 66） | 配置更新一定收敛，不会卡死 |
| Confluence（汇合） | 最终状态只由配置决定，与中间顺序无关（Theorem 73） | **对账可增量、任意顺序执行** |

Progress 的前提：precedence 关系（我提供你声明的键）必须**无环**——依赖图不能成环的数学表达。Confluence 是最大工程收益：Loader 可以对配置做增量对账而不担心顺序。

### 理论如何变成代码（摘录）

| 理论 | 实现 |
| --- | --- |
| 上下文 Γ∞ | `ctx`（一等上下文） |
| 效应上下文 ∂Γ、累加器 | `ctx.effect`、fiber 的 accumulator |
| get/set | `ctx.get(key)` / `ctx.set(key, value)` |
| isolate / intercept | `ctx.isolate(key, realm)` / `ctx.intercept(key, meta)` |
| 组件实例（fiber） | fiber：uid、inject、provide、apply、parent、state、committed、inertia |
| 注册表 | `ctx.registry` |
| 恢复 | `fiber.dispose()` |
| entry | id（对账键）、url（cordis.yml 里写作 name）、isolate、intercept、config、disabled——与 DSH 条目一一对应 |
| 配置对账 | 按字段差异选最小操作：id/url 变 → 重建；isolate 变 → 重分配 realm；intercept → 原地更新；config → 组件自行 diff；disabled → 卸载/重载；group 子列表按 id 键控 diff 递归下探 |
| HMR 三阶段 | ① 依赖图不动点分类 accept/decline → ② 按 entry 传递依赖树检测过期 → ③ 事务性重载（备份模块缓存，逐 dispose 旧 fiber + 挂载新 fiber，失败回滚）。**无需开发者标注 accept 边界**（对比 Webpack/Vite）——fiber 本身就界定了组件全部效果的边界 |

**工程延伸**：系统边界（可独占修改、可恢复的位置"在边界内"；`ctx.intercept` 可把外部位置 reify 成可逆操作，边界向内移动——换取可恢复性，付出开销）；补偿（无法逆转的"发射"用应用级补偿，按 LIFO 组合）；服务多路复用（独占绑定会扰动所有消费方 vs **service broker**：常驻 broker 注入提供方和消费方，多提供方共存、热更新不触发重载 → 负载均衡、滚动更新、跨进程调用——DSH 的 llm 适配器注册表与 pi-ai 多提供方的论文原型）。

### 与既有系统的对比：特点的证明

| 系统 | 时间维（卸载） | 空间维（依赖） | 差距 |
| --- | --- | --- | --- |
| 传统 DI（Spring/NestJS） | 无：getBean 拿到的服务没有生命周期管理 | 全局注册表，null 检查 + 类型转换 | 绑定即永久；依赖关系隐式散落 |
| React useEffect | effect 可清理，但靠调用顺序标识，目标隐式 | hooks 依赖数组与 effect 分离 | 卸载语义不随组件树自动成立 |
| OSGi | 服务可卸载，但消费方不自动重连 | 服务模型（provider/consumer） | 有动态机制，无响应式语义 |
| VS Code 扩展宿主 | 无法热卸载：87% 热门扩展需重启宿主 | extensionDependencies 几乎没人用，exports 是 any | 只能整宿主重启 |
| 微服务、容器 | 进程粒度重启，丢弃全部状态 | 编排器粒度，无法表达共享地址空间的依赖 | 粒度错配 |
| monadic effect（ZIO 等） | 追踪靠 monad 嵌入，程序必须写进 effect 类型 | 需求靠解释器；服务撤走后已执行操作留在原地 | 不能对普通宿主代码做覆盖式追踪 |
| 代数效应（Effekt、Koka） | 静态类型层纪律，能力限于词法作用域 | 静态解析 | 目的不同：Effekt 为解释，Cordis 为追踪与逆转 |
| Webpack / Vite HMR | 需要开发者标注 accept 边界 | — | 替换边界靠人工声明 |
| **Cordis** | 每个变换携带逆，恢复是定理；HMR 不需要 accept 边界 | 依赖是类型化表，变化被分类通知，自动重连是定理 | 时间与空间两维同时由运行时承担 |

Agent 视角下更直观：

| 需求 | 传统 DI 容器 | Cordis |
| --- | --- | --- |
| 服务被替换/卸载后自动清理 | 不负责，需手写或泄漏 | effect 自动回滚 |
| 依赖提供方热替换后自动重建 | 做不到 | 依赖方自动卸载重载 |
| 配置决定组合，改配置即热重载 | 配置通常启动时一次性 | cordis.yml 即插件树 |
| 服务运行时消失 | 拿到失效引用 | 自动停用/重载 + 补偿 |

![Koishi 插件市场：生态起源](https://gitee.com/cheng-jiaqing/images/raw/master/cordis-20260904-13.png)

## 记忆卡

```text
Cordis = 元框架：管"谁在什么时候活着"，不预设业务
五个概念：插件（3 形态）/ ctx（树）/ fiber（状态机）/ effect（可逆副作用）/ Service+inject（响应式依赖）
两个核心机制：可逆副作用（卸载自动回滚）+ 响应式余效应（依赖变化自动激活/停用）
事件 5 模式：emit / parallel / serial / bail / waterfall（= 中间件，决策链）
配置即程序：cordis.yml + id + patch 覆盖 + !!js，改配置不用重启
论文两点论：动态组合缺形式基础；effect/coeffect 从编译期静态分析 → 运行时机制；恢复是定理不是纪律
五条元定理：保型 / 时序 / 空间 / 进展 / 汇合 → 增量对账任意顺序
HMR 不需要 accept：fiber 就是效果边界，替换 = 一次 fiber 操作
```

> 记住一句话：**Cordis 把"正确卸载、正确接线"从开发者要守的纪律，变成了定理。** 这正是 DeepSeek Harness 敢说"一切皆插件"的底气。

## 关联笔记

- [[Clippings/微信公众号/2026-08-14-DeepSeek-Harness拆解-一套能拼装的Agent架构]] — Cordis 实现机制（fiber 状态、撤销栈、遮蔽算法）与 Codex 对比
- [[Clippings/微信公众号/2026-08-18-一文读懂DeepSeek-Harness插件-Datawhale]] — DSH 插件体系概念入门（Profile/Bundle/Patch、四种运行模式）
- [[Clippings/微信公众号/2026-08-13-多Agent通信实践全景-从编排到终端直连]] — agent 间通信 vs Cordis 组件间通信（ctx 服务 + waterfall）
- [[Clippings/微信公众号/2026-08-11-Agent治理-用Hook堵住LLM的偷懒越权与失忆]] — 工具执行前后挂护栏 vs DSH 的 waterfall pre/post-execute 决策链

## 来源

- 原文：<https://mp.weixin.qq.com/s/3vtCkp6EbA5MhRERD6f17A>（公众号「腾讯技术工程」，作者 lss233，2026-09-04）
- 论文：《A Programming Paradigm for Spatiotemporal Composability》（cordiverse/paper，北大 + DeepSeek-AI）
- 生态：awesome-dsh-plugin（GitHub `#dsh-plugin`，2026-08 收录 174 个插件）
- 图片图床：Gitee（通过 PicGo 上传，13 张）
