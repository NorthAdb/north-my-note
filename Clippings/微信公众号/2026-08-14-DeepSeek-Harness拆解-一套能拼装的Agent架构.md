---
created: 2026-08-14
updated: 2026-08-14
article_date: 2026-08-14
tags: [AI, Agent, DeepSeek, Harness, Cordis, 插件架构, 可逆副作用, 运行时, worker_threads, 腾讯技术工程]
source:
  - "https://mp.weixin.qq.com/s/DeIty-Nn8tQvE4osy7_bpg"
  - "https://mp.weixin.qq.com/s/gip3nyB6mw48aTa-Q3gYjg"
publisher:
  - "腾讯技术工程"
  - "Datawhale"
author: chino
---

# DeepSeek Harness 拆解：一套能拼装的 Agent 架构

> **来源**：腾讯技术工程《DeepSeek Harness 拆解：一套能拼装的 Agent 架构》（作者：chino，腾讯 WXG 微信小店前端开发工程师，2026-08-14）
> **原文**：https://mp.weixin.qq.com/s/DeIty-Nn8tQvE4osy7_bpg
> **副题**：DeepSeek 发布了第一款 Agent 产品

> **一句话结论：DeepSeek Harness 的竞争力押在运行时这一层——"everything is a plugin"。** 模型接入、工具执行、会话记录、循环本身、界面全部走插件机制，而这一切建立在 Cordis 的「可逆副作用」哲学上：**卸载 = 完整逆转**，让 Agent 能做到热插拔、自进化、失败原子性。

![DeepSeek Harness 架构](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-01.png)

## 🎯 引子：跳过共识，只讲不一样的

文章跳过 agent 领域已成共识的部分（检测工具调用、执行、回填、循环——每个框架都长一个样），重点放在 DeepSeek Harness 真正与众不同的几处设计：
- **Cordis 插件运行时**怎么管理生命周期
- **preset 的 scope 继承链**为什么拆成两层
- **Code Mode** 为什么选了一个"看起来不太安全"的隔离方案
- **工具注册的遮蔽算法**怎么写

并顺带与 Codex 的工程决策做对比。

---

## 一、为什么是 Cordis

"everything is a plugin" 的基础是 Shigma 的开源项目 **Cordis**——定位 "Meta-Framework for Modern JavaScript Applications"，处理依赖注入、作用域服务、生命周期清理，与 agent、LLM 都无直接关系。

Cordis 在开源社区有实际使用者：**Koishi**（跨平台聊天机器人框架，QQ/Discord/Telegram/微信都接）。聊天机器人场景天然是"几十个插件拼在一起、热插拔、随时改配置"，与 agent 场景几乎相同，只是没有 LLM。DSH 把整个框架 vendor 进仓库，改名 `@deepseek-ai/cordis`，每个包都设成对它的 peer dependency——**整个产品都构建在 cordis 之上**。

### 三种插件形态对比

| 维度 | 传统 DI 容器 | Pi 的 Extension | Cordis |
| --- | --- | --- | --- |
| 清理/卸载 | 要么不管（泄漏）、要么手写 lifecycle hook | **不能卸载** | 可逆副作用，卸载完整逆转 |
| 依赖管理 | 无"依赖驱动的重载"（LLM provider 热替换，依赖它的 ToolRegistry 不会自动重启） | 无插件间依赖管理（加载顺序即优先级） | 插件间可声明依赖，依赖变化自动激活/停用 |
| 配置 | 启动时读一次 | — | `cordis.yml` 一行一个插件实例，改配置直接触发 HMR |
| 本质 | 容器 | 扩展点钩子 | 带依赖声明、生命周期、可逆副作用的"组件" |

### 时间可组合性 vs 空间可组合性（整篇文章的地基）

Cordis 设计论文把核心问题问得很直接：**有没有一种 programming model，能让动态本身具备类似进程那样的生命周期隔离？**

进程/容器的好处：kill 掉再启动，状态就清空了；代价：重启丢 cache、connection、partial computation。插件系统希望**不用重启**也能完成同样的清理。论文给了两个定义：

- **时间可组合性**：卸载组件时，它对共享环境所做的修改必须被完整、安全地逆转——要求追踪每次资源分配、事件注册和状态变更；
- **空间可组合性**：依赖变化时，相关组件自动激活或停用。

> **时间可组合性是全文主线**——Fiber、effect、系统边界、preset、Code Mode 都在回答同一个问题：怎么在工程上实现「卸载 = 完整逆转」。

Cordis 生态配套包随项目一起 vendor：配置文件加载器（解析 cordis.yml）、Include 插件（把配置当子树挂载，preset 机制用的就是它）、HMR 模块、日志和定时器基础设施。

---

## 二、Fiber 与 effect：可逆副作用的地基

### Fiber：插件实例的生命周期状态机

插件的类型定义是联合类型：`Plugin.Function | Plugin.Constructor | Plugin.Object`——裸函数、类、带 apply 方法的对象，统一解析成一个回调。挂载调 `ctx.plugin(plugin, config)`，按回调函数身份做 key（同一插件多次挂载共享 Runtime），每个实例建一个 **Fiber**。

Fiber 六个状态：**PENDING、LOADING、ACTIVE、FAILED、DISPOSED、UNLOADING**。插件声明 `inject: ['tools', 'shell']`，对应 Fiber 停在 PENDING 直到两个服务都在依赖链上出现，才真正跑插件代码进入 ACTIVE——这是 Cordis 自己实现的依赖解析，插件作者不用手写"等对方准备好"的轮询。

### effect：副作用与逆写在一起

插件注册的任何东西（事件监听、服务、定时器）都通过 `ctx.effect()` 登记：

```ts
ctx.effect(() => {
  // 做副作用：注册监听、开定时器、提供服务……
  return () => { /* 逆：撤销上面的操作 */ }   // 返回 disposer
})
```

- 登记时**立刻执行一次**，返回的撤销函数被推进 disposables 列表；
- callback 可以返回 disposer、Promise、或（异步）迭代器（逐段 yield disposer）；
- 卸载时按 **LIFO** 逆序执行：`disposables.splice(0).reverse().forEach(dispose => dispose())`。

**「副作用」的定义**：任何通过 Context 对共享环境进行的修改：

| 类别 | 例子 |
| --- | --- |
| 事件注册 | `ctx.on("foo", handler)` |
| 提供服务 | `ctx.provide(name, service)` |
| 挂载子组件 | `ctx.plugin(plugin)`（子插件本身就是父级上的副作用） |
| 上下文扩展 | `ctx.extend()` / `ctx.intercept()` / `ctx.isolate()` |
| 外部资源 | 定时器、文件监听、HTTP server、子进程、DB 连接 |
| 状态变更 | 改 config、改注册表、改 store |

**插件本身 = 父 Fiber 上的一个 effect**：子组件的逆会 prepend 到父上下文的累加器上，形成递归结构（论文叫 twisted composition，𝜕²Γ）。所以卸载一个插件会**连带把子插件一起按栈顺序拆干净**，不需要单独维护"谁依赖谁"的清理表。

**两个可靠性细节**：
- **幂等**：`dispose()` 首次调用把 armed 置 false，之后直接返回——每个逆最多执行一次；
- **中断**：callback 返回迭代器时，每一步之前查 guard，失效立即停止，只执行已累积的逆——热重载/卸载中途失败能安全停住。

`_unload()` 跑完撤销后若发现依赖又重新满足，会立刻 `_reload()` 重新拉起来——**"换掉某个服务的实现"在运行时层面就是一次自动的卸载再加载**。

### 写法示例（会话日志插件）

```ts
export const name = 'session-logger'

export function apply(ctx: Context) {
  ctx.effect(() => {
    // 副作用 1：订阅事件总线
    const off = ctx.on('tool/result', (event) => appendToLog(event))
    // 副作用 2：定时器刷缓冲
    const timer = setInterval(flushBuffer, 1000)
    // 返回撤销函数：卸载时按注册的相反顺序执行
    return () => {
      clearInterval(timer) // 后注册，先撤销
      off()                // 再退订监听
    }
  })
}
```

![插件写法示例](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-02.png)

> **真的很像 React**：`useEffect(() => { subscribe(); return () => unsubscribe() }, deps)`——副作用和它的逆写在一起，销毁由框架触发。差异：React 在渲染后执行、依赖变化先清理再重建；Cordis 在注册时立即执行、LIFO 保证清理顺序。

### 为什么"逆转"值得这么大篇幅

论文把「**自进化 Agent Harness**」列为两大动机之一：

> 未来的 harness 会在持续服务请求的同时生成并部署对自己组件的修改……没有时间可组合性，每次自我修改都迫使完整重启，丢弃所有进程内累积状态；更糟的是，一个有缺陷的自我修改可能废掉唯一能用来恢复的那个进程。

对照现有生态：**VSCode 无法卸载单个扩展**（禁用/卸载必须重启整个宿主）；Koishi 社区改配置 reload daemon 是常态。Cordis 的逆转让"卸载"成为与"加载"对称的一等操作——服务不中断，其他插件不受影响。

**顺带解决的两个工程问题**：
- **失败原子性**：插件初始化中途抛错 → fiber 进 FAILED 并 dispose，已应用的一半效应被逆序清掉，不残留部分初始化状态；
- **级联清理**：作者不用写卸载代码——"正确性从依赖每个作者的自觉，变成由抽象层一次性承担"。

> 逆转机制四个关键作用：① 开发期 HMR（改代码不重启、出错可事务回滚）；② 生产期运行时插件装卸（服务零中断）；③ 自进化 Agent（改自己、坏了能自救）；④ 失败原子性与级联清理。

---

## 三、四条约束与系统边界：框架怎么保证 effect 被执行

如果作者把副作用写在 effect 外面（直接给 `globalThis.foo` 赋值），机制还有效吗？论文正面回答：框架分两边处理——**API 层面约束** + **明确声明系统边界**。

### 四条约束

| 层次 | 谁保证 | 内容 |
| --- | --- | --- |
| **路径强制** | 框架（API 设计） | 用框架功能只能走 `ctx`，ctx 内操作全是 effect 封装 → 必然被追踪 |
| **时间强制** | 框架（状态机） | 已卸载 fiber 上建 effect 直接抛 `INACTIVE_EFFECT` |
| **属性拦截** | 框架（Proxy） | ctx 属性读写被代理拦截、可记录（`ctx.foo = x` 也会被拦截） |
| **模块级事务** | 框架（HMR backup/restore） | import 失败整体回滚，不进半重载态 |

### 边界内 vs 边界外

论文把环境分成两半：
- **边界内（inside）**：系统能独占修改、且能恢复原状的位置，操作记录在 Γ，可以 recover；
- **边界外（outside）**：直接改全局变量、monkey-patch、写公共文件——既不追踪也不恢复（idΓ）。

> 边界按**位置**划分，与介质无关：私有路径的 scratch 文件、只有本系统写的内存属于边界内；公共文件、别的进程也在写的东西属于边界外。

**获取/发射两阶段**：获取阶段（open/malloc/fork）在边界内可逆；发射阶段（write/send 数据离开系统）在边界外不可逆——恢复只能靠 **withholding**（暂缓发射）或 **compensation**（补偿动作：删创建的文件、退已收的款）。

**边界移动 = 服务物化（coeffect）**：把所有对某外部位置的访问限制在一组操作内、每个操作都提供逆，于是原本 idΓ 的操作变得可追踪可恢复。工程上就是 service 抽象——`ctx.database`、`ctx.assets` 由服务提供者负责写对的逆，消费者只面对高层接口。副作用从各处集中到服务实现内部。

**逆的正确性标准**：不要求"物理复原"，读作**观察等价**——"两个状态相关，当没有任何观察者能区分它们"。恢复前后的状态只要没有任何 coeffect 操作能察觉差异，就算恢复成功。

> 论文很诚实：「回调提供了逆，但该逆是否真的恢复了伴随它的效应，是**组件作者的责任（obligation）**，而不是运行时验证的性质（property）」。运行时保证"逆会被调用"（结构性），不保证"逆写得对"（语义性）。

### 为什么在 TS/JS 里落地最直接

大多数编译语言的类型系统基于类、类型布局编译期固定，无法动态加载新编译结果覆盖，加载进内存的类型信息通常也无法卸载。TS/JS 是原型系统：Agent 造新工具直接挂原型、所有子组件立即感知，撤销时从原型链切断立即恢复；Proxy 提供"逻辑上极度解耦、语法上完全透明"的动态路由；Module Augmentation 让 agent 生成的新工具类型提示瞬时同步整个系统；JS 有程序化的模块注册表可卸载模块让 GC 回收。像 Swift 这种语言彻底清理已加载的类型元数据几乎不可能。

---

## 四、ctx.`<service>` 怎么解析：一个 Proxy 加一条 Fiber 链

插件之间怎么拿到对方提供的能力，是"everything is a plugin"能落地的另一半。

根上下文包一层 Proxy：`new Proxy(this, ReflectService.handler)`。访问 `ctx.tools`、`ctx.llm` 这类属性走 get 陷阱，自身已有属性直接 `Reflect.get`，没见过的属性名沿当前 Fiber 往上找：

```ts
let fiber = ctx.fiber
while (true) {
  const impl = fiber.store?.[prop]
  if (impl) return impl.value
  if (prop in fiber.inject) throw new Error('service not active')
  if (!fiber.runtime) throw new Error('unknown service')
  fiber = fiber.parent.fiber
}
```

发布服务靠 `Service` 基类，`provide()` 包在 `ctx.fiber.effect(...)` 里，往当前 Fiber 的 store 塞记录 + 唤醒等待者；撤销函数把记录摘掉再通知一遍。**一个服务被移除走的是与插件卸载完全相同的 effect 撤销机制**——系统里没有一个专门的"服务注册中心"类。

**直接后果：作用域隔离是天然的**。子 Fiber 能看到父 Fiber 注册的服务，反过来不行。想让某插件的服务只在特定范围生效，把这个插件挂在对应层级 Fiber 下即可。

**设计验证双胞胎**：DSH 定义了 `llm` 服务接口，只规定"怎么发一次流式调用、怎么处理重试"，系统同时挂两个实现：`dsh-llm-deepseek`（自家模型）+ `dsh-llm-pi-ai`（包第三方库 `@earendil-works/pi-ai` 做多家模型协议转换，近 40 家厂商大部分由它支持）。两插件互不知道对方存在，换哪个上层 loop 调用不用改一行——两套完全独立的实现都满足同一份接口定义，从侧面证明接口足够通用。

---

## 五、Agent Loop 里真正值得说的几处设计

Loop 整体形态与大多数框架类似，挑几处具体做法：

1. **轮次结束条件有多种来源**：除了"没有新的工具调用"，工具执行结果本身可带 `concludesTurn: true` 标记，不需要等模型再说一句"好了"；
2. **max-tokens 状态是粘性的**：一旦某 step 因输出上限被截断，即使后续 step 正常完成，轮次最终报告的结束原因也不会被覆盖为"正常完成"——上层统计/日志看到的"这轮被截断过"的信号不会丢；
3. **`agent/turn-stopping` 事件**：轮次真正关闭前广播，是插件可截住的钩子。插件调 `agent.steer(...)` 塞新消息轮次继续跑，什么都不做就默认结束——"什么时候真正收尾"从 loop 内部移到外部，任何插件都能在最后一刻决定"还没完"；
4. **工具调度的并发判断不依赖静态白名单**：每次实际调用前向工具询问能否并行（`executionMode(exec)`）——读文件永远并行安全，写文件只有目标路径不冲突才安全。并行组里混进当下判定为互斥的调用会截断该组，不硬塞进并行批次；结果按模型原始顺序提交，与调度完成顺序无关；
5. **执行前后检查是独立、可插拔的阶段**：`tools/pre-execute`（waterfall 事件，可拦截/审批）→ 内置防护检查（如同一工具同参数连续调用太多次会被提醒）→ 真正调工具 → `tools/post-execute`（二次处理）。四段固定顺序、彼此独立注册，新增审批策略/防护规则不改执行链路。

---

## 六、Preset 的两层 Scope 链：真实继承与影子路由表

Preset 解决：同一份配置被多个会话复用，又不用每次重新解析。

一份预设是一个目录，核心是 `agent.cordis.yml`（按顺序列要挂载的插件和参数）。挂载用复用自 cordis 生态的 **Include** 插件，改名 `PresetTree`：解析裸模块名基于系统 baseUrl（预设目录不参与）；写回配置被禁用（源文件不被意外修改）；挂载后核对"有无插件停在不可用状态""有无服务泄漏到预设作用域外"，不满足整体回滚。

**scope 链构造——两段不同机制**：
- **全局 → 预设**：真实的上下文派生。`createScope()` 内部调 `ctx.plugin(scope)` 拿到 Fiber，再 `fiber.ctx.extend({[kScope]: key})`——在 Cordis 上下文树里真实存在一个节点；
- **预设 → 会话**：`bindScopeParent(agentKey, standing.key)` 往 WeakMap 记一条"逻辑上级"关系，**不在 Fiber 树里新开任何节点**。

这样拆的动机：一份预设的"标准挂载实例"只建一次（`ensureStanding()`），后续任意多个会话（写作/编码/调研）复用同一份实例，靠往 WeakMap 追加绑定即可，不用每个新会话重新跑插件加载。子 agent 继承父 agent 的预设是同步查找+绑定，拿到的是**完全同一份插件实例**（同一批工具和提示词）。

**会话当前预设的判定**：倒着扫事件日志找最新一条 `agent-preset/selected`，没有则回退会话创建时的默认值——理论上会话中途换预设是架构允许的。

**预设发现机制**：只认目录名匹配命名规则、且内部有 `agent.cordis.yml` 的子目录；坏条目标记出来不中断扫描；多根目录合并按顺序先出现的 id 优先——给了"用户自定义覆盖官方预设"的空间。

---

## 七、Code Mode 的隔离方案：worker_threads

让模型写代码去编排多个工具调用，这个思路 Codex 也做了。值得看的是隔离选型。

DSH 用 **`node:worker_threads`**（普通 Node 工作线程），而不是 V8 隔离环境或 vm2。选型考量：`node:vm` 那类同进程沙箱**原型链可以逃逸到主环境**、热循环无法从外部真正打断，被认为不可靠；独立工作线程换来**真正独立的 V8 堆** + 可从外部强制终止的执行环境，代价是每次跑代码要开一个新线程。

**执行路径**：
- 模型写的代码先 `stripTypeScriptTypes` 剥类型标注，剥的时候用包装前后缀撑住位置，保证**剥完行列号不变**（报错定位对得上）；
- 塞进动态构造的异步函数：`new AsyncFunction(...bindingNames, ...errorClassNames, 'console', code)`——每个工具命名空间、约定的错误类、替换 console 都变成形参实参，模型代码可直接顶层 `await`、直接 `return`；
- 工作线程有 `resourceLimits.maxOldGenerationSizeMb` 堆限制 + 两道独立超时（事件循环利用率轮询 CPU 时间 + `setTimeout` 兜底总耗时），任意触发强制中断；
- 工具调用走消息通道：`{ type: 'call', id, global: 'tools', name, args }`（worker → host）→ `{ type: 'reply', id, ok, true, value }`（host → worker）。宿主把它当正常工具调用推进 pre-execute/防护/dispatch/post-execute 流水线，与模型直接发起的结构化调用**共用同一套执行内核**；
- 两端手动校验每条消息字段；宿主用 `Object.hasOwn()` 精确判断自有属性，`tools` 命名空间用 `Object.create(null)` 起手 + 逐个 `Object.defineProperty` 挂载——防 `__proto__`/`constructor` 绕过原型链。代码注释里直接写：**"把 worker 当成敌对的另一方来处理"**，信任假设明确声明。

**与 Codex 的差异**：Codex 用独立 **V8 isolate**，工具挂全局 `tools` 对象上。解决同一个问题，背后权衡的是"每次新建 isolate 的开销" vs "每次开工作线程的开销"哪个划算，以及各自生态里现成的隔离原语。

---

## 八、工具注册的分层遮蔽算法

每个作用域维护一份自己的 **ToolLayer**（按名字索引的表）。决定"当前 agent 能看到哪些工具"的是 `view()`：

1. 先加载**全局层**工具作为基础；
2. 每层祖先 scope 依次覆盖同名条目（越靠近当前 scope 优先级越高）——只处理继承来的部分；
3. **限制规则**（如某 preset 禁掉某工具）在这之后筛一遍；
4. 当前 scope **自己直接注册的工具**覆盖前面所有继承来的同名条目，即使限制规则禁用了这个名字。

> 顺序很关键：**自己注册的 > 继承和限制规则**。

注册本身也有硬校验：
- `run_code` 名字**硬编码保留**——任何插件不能注册或覆盖它（一旦某 preset 挂载进来可能跟 Code Mode 的调用入口冲突，无条件预留）；
- `assertSupportedJsonSchema` 校验输出 schema；
- 注册动作走 `effect()`，撤销函数插进当前 Fiber 的 disposables——**插件卸载时注册的工具自动从层里摘掉**，工具注册只是"注册即副作用"的一种。

工具执行也复用同一调度器的并发契约：Code Mode 里模型代码调工具会进入宿主侧同一个 `TOOL_RUNTIME_SCHEDULER` 队列，共享并发控制与前置检查。

---

## 九、跟 Codex 比，几个工程决策的差异

| 维度 | Codex | DeepSeek Harness |
| --- | --- | --- |
| **核心循环可替换性** | 驱动"采样-执行-反馈"的代码固定在核心，无法作为独立单元替换 | `ReactLoopAgent` 注册为工厂，理论上可被另一套循环实现替换（目前只有一个实现，工厂只允许注册一次） |
| **沙箱位置** | bwrap/Landlock/Seatbelt 直接编译进核心执行路径，属基础设施 | 沙箱是一层能力插件（Linux 上独立原生 Node 扩展调 Landlock），走同一条服务注册路径，理论上可换隔离实现 |
| **代码规模拆分** | Rust，约上百个 crate，但最终都服务于不可替换的核心循环 | TS，两百多个包，模块（含循环本身）理论上都在同一可替换层级，没有哪个地位特殊 |
| **代码编排隔离** | 独立 V8 isolate | worker_threads |
| **工具暴露粒度** | DIRECT/DEFERRED/CODE_MODE/Hidden 是工具自己声明的静态标记 | 运行时按 scope 链实时计算可见性/覆盖，同一工具在不同会话可见性可以完全不同 |

---

## 附：从进程启动到一次工具调用，整条链路

进程启动 → Profile 决定装哪些插件包 → Cordis 加载器逐个挂载（每插件一个 Fiber）→ host 层准备模型接入、凭据管理、沙箱 → 会话开始，agent 注册表分配实例挂在"全局-预设-会话"scope 链下 → 可见工具/提示词取决于链上位置 + 遮蔽规则 → Agent Loop 接管：读上下文、请求模型、检测工具调用、走统一工具流水线 → 每步发生的事作为事件追加进会话日志，**不删除不覆盖**。这份日志同时是：持久化来源 + 下一次请求的上下文来源 + 网页界面的展示来源（网页界面本身也是插件）。

![进程启动到工具调用链路](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-03.png)

**这套设计解决的四个具体问题**：
1. **加/改能力的风险** → 风险控制粒度做到插件级，影响范围框定在插件自己的边界里；
2. **同一套 agent 服务不同场景** → preset 两层 scope 链，一份插件树只挂载一次，多会话靠逻辑父子表复用；
3. **多步骤任务一次一个工具调用的来回成本** → Code Mode 让模型自己写代码编排，减少往返（与 Codex 几乎同时给出同一判断，隔离方案不同）；
4. **工具可见性/权限只有"全局/全局不可用"两种状态** → 分层遮蔽算法把可见性变成运行时按 scope 链实时计算。

> **更大的判断**：agent 产品的竞争力，除了模型效果，还取决于运行时这一层——怎么组织、扩展成本多高、换组件多大代价。DSH 用 Cordis 现成框架 + "可逆副作用"设计哲学，把这一判断落实到具体代码结构上。

![HMR 事故：作者 P.S. 示例插件把 UI 搞崩了](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-04.png)

---

## 🛠️ 实战：插件开发与安装（Datawhale 保姆教程）

> **补充来源**：Datawhale《最新！DeepSeek Harness 插件的安装教程来了！》（2026-08-14）
> 既然 DSH 最重要的设计原则是"一切皆插件"，本篇教两部分：**自己写一个最小插件**（理解 Harness 怎样注册工具）+ **安装别人写好的插件**（直接获得完整能力）。前者是学习机制，后者是日常使用。

### Part 1：写一个最小插件（greet 工具）

目标：Agent 调用 `greet` 工具并传入名字，返回 `你好，Datawhale！你的第一个 Harness 插件已经运行。`

**1. 准备源码环境**

```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
corepack enable
pnpm install
pnpm run build
```

- 实测 Node.js v24.19.0；Harness 声明的范围是 `^22.19.0 || >=24.0.0`，不确定时直接用 Node 24；
- ⚠️ `pnpm run build` 不要省——只装依赖时插件日志虽出现、Web 页面却缺少构建产物。

**2. 创建插件文件**（`scratch-plugin/src/greet-tool.ts`）：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: {
        type: 'string',
        required: true,
        description: 'The name to greet',
      },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `你好，${args.name}！你的第一个 Harness 插件已经运行。`
    },
  }))
  console.log('[greet-tool] loaded; tool name: greet')
}
```

插件只有四个部分：`name`（插件名）、`inject`（声明需要 Harness 的工具服务）、`apply(ctx)`（加载入口）、`ctx.tools.register(...)`（注册模型可调用的工具）。`parameters` 告诉模型该传什么；`execute` 真正执行代码；`output` 约定结果的类型和显示方式。

**3. 把插件插入 Harness**（新建 `scratch-plugin/cordis.yml`）：

```yaml
- insert:
    - id: greet-tool
      name: '/Users/yourname/deepseek-harness/scratch-plugin/src/greet-tool.ts'
```

`name` 换成机器上的绝对路径。插件最好放在 Harness 源码仓库内——示例依赖仓库里的 `@deepseek-ai/cordis` 和 `@deepseek-ai/dsh-tools`，放别处可能 `Cannot find module`。

**4. 启动并检查**

```bash
pnpm dsh web --patch ./scratch-plugin/cordis.yml
# 端口被占用时：pnpm dsh web --patch ./scratch-plugin/cordis.yml --port 3082
```

看到 `[greet-tool] loaded; tool name: greet` + `dsh web: http://127.0.0.1:3082` 即就绪。进"设置 → 插件 → 插件列表"，`greet-tool.ts` 状态应为"已启用"。

![插件已挂载](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-05.png)

**5. 让 Agent 调用它**：新建标准模式会话，输入「请调用 greet 工具问候 Datawhale。」展开工具调用可见：

```
IN   { "name": "Datawhale" }
OUT  你好，Datawhale！你的第一个 Harness 插件已经运行。
```

![greet 工具调用成功](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-06.png)

**至此最小闭环跑通**：加载插件 → 注册工具 → 模型调用 → 返回结果。

### Part 2：直接安装生产级插件（Vision Toolkit 为例）

写一个完整插件难度较高，日常使用直接装大佬们做好的插件。以 **DSH Vision Toolkit** 为例——DeepSeek 当前 Chat Completions 路由是纯文本模型，不能直接理解图片；Vision Toolkit 把图片交给独立视觉模型，再把文字/坐标/文件产物送回 Harness（提供图片问答、OCR、元素定位、图片裁剪、像素对比、HTML 截图等工具）。

![Vision Toolkit 示意](https://gitee.com/cheng-jiaqing/images/raw/master/deepseek-harness-20260814-07.png)

**1. 安装插件**

```bash
dsh plugin --profile web add @dsh-external/dsh-vision-toolkit
# 或 npx @deepseek-ai/dsh@0.1.0-rc.6 plugin --profile web add @dsh-external/dsh-vision-toolkit
dsh --profile web --dump-config | grep vision-toolkit   # 确认装进配置
```

装完**重启 Harness Web 服务**（插件宿主代码和浏览器代码都在启动时加载，只刷新页面不够）。

**2. 配置视觉模型**：要求 Python 3.11+。在"设置 → 视觉工具"配置：OpenAI 兼容的视觉模型地址 + 视觉模型名称 + 一个 DSH Credential 引用（如 `VISION_API_KEY`）。密钥写入：

```bash
dsh credentials set VISION_API_KEY
```

> 配的是**视觉服务 Key**，不是默认 DeepSeek 文本模型 Key（除非使用的网关明确同时提供视觉模型）。远程图片问答/定位/OCR 需要视觉服务；裁剪、颜色分析、像素对比等本地工具不需要 Key。

**3. 会话中使用**：把图片复制进工作区（如 `./screenshot.png`），先加载插件附带的 Skill `/vision-tools`，再描述任务：

```
请用 vision_glance 分析 ./screenshot.png，告诉我页面上出现了什么错误。
请用 vision_ground 定位截图里的发送按钮，并生成带标注的预览图。
请比较 reference.png 和 actual.png，告诉我差异最大的区域。
```

插件按需向当前 Agent 暴露对应的 `vision_*` 工具；产物（裁剪图/热力图/报告）保存在工作区 `.dsh-vision-toolkit/artifacts/`。

**4. 更新或卸载**

```bash
dsh plugin --profile web update
dsh plugin --profile web remove @dsh-external/dsh-vision-toolkit
```

### 安装第三方插件前的注意事项

- 仓库是否公开、许可证和维护者是否清楚；
- 安装脚本会下载什么、是否会运行额外程序；
- 插件需要哪些目录/网络/凭据权限；
- 是否说明支持的 Harness 版本、卸载方式和测试方法。

> ⚠️ **Harness 插件运行在宿主进程里，属于可信代码。** 不要因为安装命令只有一行，就跳过源码和权限检查。

### 小结

- **自己写插件的最小结构**：`apply(ctx) → 注册工具 → execute(args) → 返回结构化结果`
- **使用现成插件的流程**：`plugin add → 重启 Profile → 配置凭据 → 加载 Skill → 调用工具`
- 前者让你理解 Harness，后者让 Harness 真正变得有用。

---

## 展望：一个能对自己热更新的运行时

- 社区已有人做连接通讯工具的插件（Telegram/Discord 消息接入）——当年 Koishi 靠 Cordis 把 QQ/Discord/Telegram/微信几十个插件拼成现象级项目，如今同一运行时内核搬进 agent 产品，那条路正在 DSH 上重演；
- Hermes、OpenClaw 的成长方式是往技能库不断加 skill（能力变多变、运行时结构基本不动）；**DSH 的成长方式在另一个维度**：插件可热装卸，界面/会话记录/循环都是插件，连 webui 都能在持续服务的同时被换掉；
- 对个人开发者：以前做 AI 产品要写前端、配后端、搭 agent 框架，三块互不相通；DSH 上三层都是插件，挂进同一运行时，界面还能热更新——"一个想法到可用的原型，甚至到能交付的成品"的距离正在被压缩；
- 当前仍是偏开发者的产品（需装 Node、开终端敲命令），但"宿主怎么启动、界面怎么呈现"都是可替换插件，复杂度包进安装包属于产品化问题，架构上已有位置。

**生态现状**：GitHub 上 `#dsh-plugin` topic 已 1000+ 仓库（检索时 1008 个）。精选索引：`awesome-dsh-plugin`（256⭐）、`AdamPlatin123/awesome-dsh-plugins`（507⭐）、`0xsline/awesome-deepseek-harness`（227⭐）、`Electricitysheep/dsh-handbook`（74⭐）。热门插件覆盖 Web UI 增强、视觉/多模态、终端与桌面端、记忆/上下文、多 Agent/工作流、通讯与通知、开发配套等。

---

## 📎 参考来源

- 架构拆解原文：https://mp.weixin.qq.com/s/DeIty-Nn8tQvE4osy7_bpg
- 插件教程（Datawhale）：https://mp.weixin.qq.com/s/gip3nyB6mw48aTa-Q3gYjg
- DeepSeek Harness 官方本体：https://github.com/deepseek-ai/deepseek-harness
- Cordis 88 页设计论文《Cordis: 一个运行中的 Agent，能不能"换零件"而不重启？》
- 为什么用 Cordis 做 AI Agent 运行时：从 QQ 机器人框架到 DeepSeek Harness（CSDN）
- DeepSeek Harness 背后的秘密，Cordis 的设计哲学深度解读（知乎）

---

## 📌 延伸思考

- **与 Claude Code harness 概念呼应**：[[Clippings/微信公众号/2026-08-11-Claude-Code大代码库harness-小林coding]] 讲"harness 跟模型一样重要"，本篇是这一判断的极致化落地——把整个运行时做成可热插拔的插件系统。
- **与腾讯 DECO 的 Hook 护栏对比**：[[Clippings/微信公众号/2026-08-11-Agent治理-用Hook堵住LLM的偷懒越权与失忆]] 用 beforeTool/afterTool 切面挂护栏；DSH 更进一步，把"工具执行前后检查"做成独立可插拔的 pre-execute/post-execute 阶段——同一思想（基础设施与推理解耦），不同抽象粒度。
- **与多 Agent 通信**：[[Clippings/微信公众号/2026-08-13-多Agent通信实践全景-从编排到终端直连]] 讲 agent 之间怎么通信；DSH 的 Cordis 提供了"agent 内部组件如何通信"的答案（ctx 服务 + Fiber 链作用域），生态里也已有 `dsh-agent-teams`、`dsh_workflow` 等把 agent 编排成团队的多 Agent 插件。
- **可迁移设计**：「时间可组合性」（卸载 = 完整逆转）是通用的插件系统设计原则——如果做 agent 产品，这套 effect/disposer 模式值得直接借鉴。
