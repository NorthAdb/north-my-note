---
created: 2026-08-18
updated: 2026-08-18
article_date: 2026-08-17
tags: [AI, Agent, DeepSeek, Harness, Cordis, 插件架构, 一切皆插件, Fiber, inject, effect, Datawhale]
source: "https://mp.weixin.qq.com/s/r4C-RR1CthFooA7iZl6hzg"
publisher: "Datawhale"
---

# 重磅！一文读懂 DeepSeek Harness 插件！

> **来源**：Datawhale《重磅！一文读懂 DeepSeek Harness 插件！》（2026-08-17）
> **原文**：https://mp.weixin.qq.com/s/r4C-RR1CthFooA7iZl6hzg
> **系列**：DeepSeek Harness 教程第 4 篇（前有保姆安装 / 插件教程 / 桌面版和 CLI）

> **一句话结论：DSH 不是"支持插件"，而是整个产品就是插件的组合——模型适配器、工具注册表、会话日志、agent loop 本身，每一部分都是插件，都能在运行时替换。** 核心是 Cordis 微内核，配套 Fiber（生命周期）/ inject（依赖）/ effect（清理）三个机制让"动态"可靠。

## 🎯 为什么这套架构值得关注

**任意环节可替换**是 DSH 与传统 Agent 框架最大的区别：
- 换一个**模型提供方**不用改源码；
- 换一个**沙箱后端**不用重启；
- 换 **agent loop 本身**也不用动框架；
- 连**呈现方式**都是插件：Web UI、CLI、API 是不同的前端插件，挂在同一棵插件树上。

---

## 一、DeepSeek Harness 是怎么组装出来的？

DSH 不是一个装好了就定型的应用。启动时按顺序叠加各层配置，在内存里形成一套完整插件组合开始运行。三层结构从底到顶：

```
Cordis 微内核（管单个插件加载/卸载/协作）
   ↓
三层组装机制（管整套组合怎么搭）：Profile / Bundle / Patch
   ↓
四种运行模式（管最终跑哪套组合）：Standard / Code / Minimal / Creator
```

### 底层：Cordis 微内核

Cordis 是专门管插件的框架，只管三件事：**插件什么时候加载、什么时候卸载、彼此怎么协作**。它最初由 Shigma 为聊天机器人框架 **Koishi** 编写，2022 年独立开源。**DeepSeek 把 Shigma 和 Cordis 一起雇了，把整套源码并进自己的仓库**——产品的每一部分（连驱动 Agent 的主循环）都是 Cordis 插件。

### 中层：三层组装机制

| 层 | 是什么 | 作用 |
| :--- | :--- | :--- |
| **Profile** | 存在本地的组装方案 | 列出要叠放哪些组合包 + 存用户自己写的配置补丁 |
| **Bundle** | 组合包 | 把一组插件和配置打包分发——装一个 = 装一组配套插件 |
| **Patch** | 补丁 | 按名字定位到具体插件，替换它的配置 |

**叠加顺序**：先装各组合包 → 打 profile 补丁 → 打本地补丁 → 启动命令里临时再加一层补丁。**每一层都能覆盖上一层的结果。**

![Profile / Bundle / Patch 三层组装](https://gitee.com/cheng-jiaqing/images/raw/master/dsh-plugin-guide-20260817-01.png)

**`dsh-base`** 是每个 profile 必装的第一层组合包——装上就有模型适配器、工具、数据持久化、安全策略、遥测等基础能力，后续组合包和用户补丁按名字覆盖它。

### 顶层：四种运行模式

| 模式 | 定位 |
| :--- | :--- |
| **Standard** | 功能最全 |
| **Code** | 专门面向编程 |
| **Minimal** | 只留 Shell 和文件编辑两个工具 |
| **Creator** | 开放给想做实验的人 |

> 切换模式 = 换一套插件组合，不改任何代码。

---

## 二、一切皆插件：Cordis 核心词汇表

### 插件三种写法（地位平等，加载/卸载同一条路）

```ts
// 函数写法：最简单，适合注册几个监听器
export const name = 'greeter'
export function apply(ctx: Context) { /* ... */ }

// 类的写法：适合提供给别人调用的能力
export class GreeterService extends Service {
  constructor(ctx: Context) { super(ctx, 'greeter') }
}
```

### 五个核心概念

| 概念 | 解决什么问题 |
| :--- | :--- |
| **Context** | 存放各种能力的地方（`ctx.tools` 工具、`ctx.llm` 模型、`ctx.sessions` 会话），别的插件按名字取用，不关心实现是谁、跑在本地还是远程 |
| **Service** | 一种可以被别人取用的能力；用类写会自动注册，插件卸载时自动注销 |
| **Fiber** | 每个插件的生命周期记录（动态加载的核心） |
| **inject** | 插件的依赖声明——框架等依赖都就位才启动，启动顺序由依赖关系决定，不由写的前后顺序决定 |
| **事件总线** | 插件之间不直接互相调用，全靠事件通信——发通知的插件不用知道谁在听 |

![Cordis 核心词汇](https://gitee.com/cheng-jiaqing/images/raw/master/dsh-plugin-guide-20260817-02.png)

### 事件五种分发方式

前几种是广播/定向通知，**最后一种 `waterfall` 是实现拦截的模式**：每个监听者拿到一个"继续往下传"的开关，调用它就把工作交给下一个监听者，不调用就直接拦下。DSH 的工具执行流水线就是四段 waterfall 串起来的。

![事件五种分发方式](https://gitee.com/cheng-jiaqing/images/raw/master/dsh-plugin-guide-20260817-03.png)

---

## 三、Fiber、inject、effect：插件动态能力的三个关键机制

### Fiber：加载与卸载都有迹可循

每个插件实例一条 Fiber，在六个阶段间转换：

```
排队等待 → 正在启动 → 正常运行 → 正在卸载 → 完全消失
                ↘ 启动失败
```

- 声明插件后，Cordis 检查依赖：满足就启动，不满足就**一直"排队等待"（静默，不报错）**；
- 热重载走同一条状态机：改代码保存 → 监视插件发现文件变了 → 先卸载旧实例到"完全消失" → 再加载新代码走一遍"排队→启动→运行"；
- 改 `cordis.yml` 也触发更新，loader 按名字比对，**只重新加载变化的部分**。

> ⚠️ "排队等待"最容易被误判成"插件坏了"——依赖没满足时它什么都不输出。排查方式：遍历所有 fiber，找 `FiberState.PENDING` 的。

```ts
for (const runtime of ctx.registry.values()) {
  for (const fiber of runtime.fibers) {
    if (fiber.state === FiberState.PENDING) {
      console.log(`${fiber.name} 在排队等待，缺少某个依赖`)
    }
  }
}
```

### inject：持续跟踪的依赖关系

**加载顺序无关紧要**：插件 A 声明需要 B 的能力，Cordis 让 A 排队直到 B 就位——配置里 A 写在 B 前还是后，结果一样。把 B 彻底移除，A 停在排队状态，不崩溃、不跑一半。

**运行中仍在跟踪**：某个能力消失（提供它的插件被卸载/热替换），所有依赖它的插件跟着卸载；能力恢复后它们再重新加载。这防止插件拿到某能力引用后，那个能力却突然不见了。

> **实际威力**：把默认的 Shell 执行插件换掉，挂上另一个提供方——所有用到 Shell 的插件自动重启、用上新实现，不用改这些插件一行代码，整条依赖链自动重排。

### effect：卸载了，副作用怎么办？

动态加载能跑不难，**难的是卸载干净**。插件启动时可能开了定时器、注册了监听器、挂载了子插件——Cordis 的办法：插件通过框架 API 做的所有注册都属于副作用，**卸载时框架自动撤销**。

框架没直接管的资源（定时器、网络连接、文件监视），包进 `ctx.effect()` 并返回清理函数：

```ts
export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => console.log('tick'), 200)
    return () => {
      clearInterval(timer)      // 卸载时跑这个
      console.log('heartbeat cleaned up')
    }
  })
}
```

**天生受 effect 管理、不用自己写清理的**：注册事件监听器（卸载自动移除）、挂载子插件（父卸载子跟着清）、往工具注册表注册工具（卸载自动注销）。

**清理执行纪律**：按注册的反序启动，多个异步清理并发跑；有先后顺序要求的放进同一个清理函数里依次执行。

![Fiber / inject / effect 三机制](https://gitee.com/cheng-jiaqing/images/raw/master/dsh-plugin-guide-20260817-04.png)

> **三个机制各管一段、互不重叠：Fiber 管生命周期，inject 管依赖，effect 管清理。**

---

## 四、一个真实例子：工具运行时如何串起整条链路

DSH 的工具运行时是前面所有概念/机制的总集成：
- 本身是一个**能力类**（Service），自动注册进系统；
- 往系统注册 **6 种事件**（4 种 waterfall 拦截型 + 2 种广播型）；
- 声明依赖"提示词组装"能力，等就位才启动，启动时立刻注册自己用的提示词片段；
- 提供"注册工具"方法——调用方每注册一个工具拿回一个**卸载函数**，想动态卸载某个工具时直接调用即可（"动态"在最细粒度的样子）。

**工具执行是四段 waterfall 流水线**：

![工具执行四段流水线](https://gitee.com/cheng-jiaqing/images/raw/master/dsh-plugin-guide-20260817-05.png)

- **审批插件**在"执行前"一环直接拦下，工具实际逻辑从没运行；
- **结果屏蔽插件**在"执行后"把成功结果改写成错误提示，模型看到的已是改写后的内容。

**同一个工具运行时服务不同 Agent**：每个 Agent 能看到哪几个工具，沿继承关系计算——先继承全局工具 → 叠加所在组合包层的工具 → 最后应用访问限制过滤。一个 Agent 被限制用某些工具，不影响另一个。

---

## 写在最后

DSH 的插件架构是目前开源 Agent 框架里**设计最彻底**的一个：不是"支持插件"，而是**整个产品就是插件的组合**，任意环节都可以在运行时替换。

- 不同场景需要不同工具集/上下文策略/执行流程，在 DSH 里都是**换一套插件组合**的事；
- 模型可换、工具可换、agent loop 可换、呈现方式（Web UI/CLI/API）可换；
- 后续方向：自定义 agent preset、第三方插件生态、场景化插件组合模板。

> 现状：Harness 还是 **v0.1 开发者预览版**，接口随时可能变，但架构方向已经很清晰。单看"任意环节都能在运行时替换"这一条，就非常看好它的架构设计。

---

## 📎 参考来源

- Datawhale《重磅！一文读懂 DeepSeek Harness 插件！》：https://mp.weixin.qq.com/s/r4C-RR1CthFooA7iZl6hzg
- Datawhale 系列：DeepSeek Harness 保姆安装教程 / 插件教程 / 桌面版和 CLI

---

## 📌 延伸思考

- **与架构拆解笔记互补**：[[Clippings/微信公众号/2026-08-14-DeepSeek-Harness拆解-一套能拼装的Agent架构]]（腾讯技术工程，作者 chino）深挖 Cordis 的实现机制（Fiber 六状态、effect 撤销栈、系统边界、遮蔽算法），本文是 Datawhale 从**概念入门**角度的科普梳理——两篇对照读，先看这篇建立心智模型，再看那篇深挖实现。
- **与插件教程互补**：[[Clippings/微信公众号/2026-08-14-DeepSeek-Harness拆解-一套能拼装的Agent架构]] 里的「🛠️ 实战」章节有写最小插件 greet 的完整代码 + 安装 Vision Toolkit 的步骤；本文把背后的 Fiber/inject/effect 概念讲清楚了——**概念 + 代码**拼起来就能自己写插件。
- **可迁移设计**：Fiber（生命周期）/ inject（依赖跟踪）/ effect（可逆副作用）三机制是通用插件系统设计范式；结合 DECO 的 Hook 治理（[[Clippings/微信公众号/2026-08-11-Agent治理-用Hook堵住LLM的偷懒越权与失忆]]），能看到"把横切逻辑挂到框架切面"的不同实现流派。
