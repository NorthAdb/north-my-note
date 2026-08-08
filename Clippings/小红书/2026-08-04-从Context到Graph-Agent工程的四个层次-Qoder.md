---
created: 2026-08-04
updated: 2026-08-04
platform: 小红书
author: Qoder（图片署名：朴云）
note_id: 6a6ff70c0000000008013752
source: "https://www.xiaohongshu.com/explore/6a6ff70c0000000008013752?xsec_token=CBY5HxyeIK43UhPj9rD1FNKowKoc1AMPMkwyvK4mUNYsE=&xsec_source=app_share"
likes: 74
collects: 70
comments: 34
image_count: 12
tags: [小红书, Agent, Context Engineering, Harness Engineering, Loop Engineering, Graph Engineering, Qoder]
---

# 从 Context 到 Graph：Agent 工程的四个层次

> 这篇图文以 Qoder CLI 和 Qoder Cloud Agents 为例，解释 Agent 工程从一次模型调用走向持续运行、多 Agent 协作时需要解决的四个层次。

## 一句话总结

```text
Context → Harness → Loop → Graph
```

- **Context**：一次模型调用能看到什么？
- **Harness**：Agent 在什么环境和边界中运行？
- **Loop**：Agent 如何持续推进、验证、重试并最终停止？
- **Graph**：多个 Agent 或 Loop 如何连接、委派和传递状态？

它们不是四种互相替代的架构，而是同一套 Agent 系统由内向外的四个工程层次。

## 一、为什么工程关注点会不断外移

过去的 AI 应用更多关注 Prompt：如何把一次指令说清楚。随着模型能力增强，问题逐渐变成：

1. **Prompt Engineering**：怎么告诉模型要做什么？
2. **Context Engineering**：给模型哪些信息、什么时候给、如何压缩？
3. **Harness Engineering**：模型在什么工具、权限、文件系统和沙箱中运行？
4. **Loop Engineering**：如何反复执行、验证结果、失败重试和恢复？
5. **Graph Engineering**：多个 Agent/Loop 如何协作和传递状态？

本质没有变，变化的是系统瓶颈：模型能力提升后，工程问题从“单次调用”扩展到了“运行环境、持续执行和协作编排”。

## 二、四层架构总览

| 层次 | 核心问题 | 关键能力 |
|---|---|---|
| Context | 一次调用看见什么 | 项目指令、Skills、历史、记忆、上下文压缩 |
| Harness | Agent 在哪里运行 | 工具、权限、文件系统、沙箱、状态、测试、可观测性 |
| Loop | 如何持续到目标达成 | 触发、目标、验证、重试、恢复、停止、人工接管 |
| Graph | 多个执行单元如何连接 | 节点、边、委派、反馈、状态传递、跨 Session 协作 |

底层仍然是不断重复的模型调用循环：

```text
读取上下文 → 模型推理 → 调用工具 → 写回结果 → 再次读取上下文
```

## 三、Context Engineering：管理一次调用的注意力

### 3.1 目标

Context Engineering 解决的不是“塞更多内容”，而是管理有限的注意力预算：

- 什么信息应该常驻？
- 什么信息应该按需加载？
- 什么信息需要压缩？
- 什么信息可以丢弃？
- 哪些信息应该隔离给 Subagent？

### 3.2 Qoder CLI 的做法

- 用 `AGENTS.md` 分层加载全局、项目和目录级约定。
- 先加载 Skill 名称和简介，任务命中后再完整读取 Skill。
- 用 Subagent 隔离代码探索、文件阅读和试错过程。
- 主上下文只接收必要的证据、结论和产物。

这是一种**渐进式披露（progressive disclosure）**：不是把所有信息一次塞给模型，而是按任务需要逐步展开。

### 3.3 Qoder Cloud Agents 的做法

云端 Agent 更强调：

- 持久化 Session
- 会话历史
- 跨任务上下文
- Memory
- 远程运行状态

本地 CLI 主要优化一次任务的上下文；云端平台还要解决跨时间、跨会话和跨用户的上下文连续性。

### 3.4 关键启示

> Context Engineering 的重点不是“更多”，而是“在正确的时间给正确的信息”。

## 四、Harness Engineering：构建模型的执行环境

同一个模型，直接调用 API 和放进成熟 Harness 后，执行质量可能差异很大。Harness 是模型之外、直接影响执行质量的环境层，通常包括：

- 上下文组装
- 工具注册与调用
- 权限检查
- 文件系统
- 沙箱
- 状态反馈
- 测试与验证
- Hooks
- 可观测性

### 4.1 Qoder CLI 的 Harness 能力

图中列出的本地能力包括：

- Plan Mode：约束分析和规划状态
- Permission Mode：决定工具调用自动放行、询问确认还是拒绝
- 受信目录：限制 Agent 可操作的项目范围
- 工具和 settings：定义可用能力与配置
- Hooks：注入项目约束、确定性检查和自动化动作

本地 Harness 的重点是：**让 Agent 在一个受控的开发环境里工作**。

### 4.2 Qoder Cloud Agents 的扩展

云端 Harness 需要额外处理多租户和远程运行问题：

- Agent：定义 Agent 能力
- Environment：提供具体运行环境
- Session：承载一次任务
- Sandbox：隔离执行环境
- Vault：安全注入凭证和敏感配置
- Event / SSE：暴露执行过程
- Managed Agents：平台托管的 Agent
- Template：复用运行配置
- Identity：区分不同终端用户
- Forward Mode：面向业务交付的上层模式

### 4.3 裸 API 与 Harness 的区别

```text
裸 API：Prompt → LLM → Text

Harness：
LLM + 工具 + 权限 + 文件系统 + 沙箱
    + 状态反馈 + 测试 + Hooks + 可观测性
```

安全不能只依赖 Prompt，还需要运行时权限、工具白名单、沙箱和执行规则共同约束。

## 五、Loop Engineering：让系统持续运行到目标达成

Loop Engineering 关注一次 Agent 运行以外的闭环：

- 谁触发下一轮？
- 什么条件算完成？
- 谁验证结果？
- 失败后是否重试？
- 重试几次？
- 什么时候恢复？
- 什么时候交给人？
- 如何跨多个 Session 继续？

### 5.1 本地 CLI

本地 CLI 中，人通常仍然是外层 Loop：

```text
人提出目标
  → Agent 运行
  → 人检查结果
  → 决定继续 / 修改目标 / 停止
```

当目标、验证、重试和停止条件被固化后，系统才从“人不断提示 Agent”变成“人设计一个可持续推进的循环”。

### 5.2 云端 Agent

Qoder Cloud Agents 把外层 Loop 平台化：

- 持久化 Session 保存状态
- Event 和 SSE 暴露执行过程
- Schedule 按时间触发
- IM Channel 通过外部消息触发
- Batch 批量运行同类任务
- 外部系统可以取消当前 Turn 或继续下一轮
- 控制者可以是业务系统、监控规则或审批流程

```text
Schedule / IM / Batch
          ↓
      Agent 运行
          ↓
      结果验证
     ↙    ↓    ↘
  重试   继续   停止
          ↑
       人工接管
```

### 5.3 Loop 的核心设计

一个可靠 Loop 至少需要：

1. 明确目标
2. 可观察状态
3. 可验证结果
4. 有限重试
5. 错误恢复
6. 停止条件
7. 人工接管入口

## 六、Graph Engineering：连接多个 Agent 或 Loop

Graph Engineering 的基础元素是：

- **State**：节点之间需要共享的信息
- **Nodes**：代码函数、模型调用、Agent、Subagent 或完整 Loop
- **Edges**：执行顺序、条件分支、委派和结果反馈

### 6.1 Subagent 如何成为 Graph 节点

可以把多 Agent 协作抽象成：

```text
Coordinator
  ├─ 委派 → Agent A
  │          └─ 结果 → Coordinator
  └─ 委派 → Agent B
             └─ 结果 → Coordinator
```

- Subagent 是节点
- 委派和结果反馈是边
- 任务说明、证据和结论是动态 State
- Coordinator 负责拆分任务、汇总结果和决定下一步

### 6.2 Qoder CLI 与 Qoder Cloud 的差异

| 层次 | Qoder CLI | Qoder Cloud Agents |
|---|---|---|
| Context | `AGENTS.md` 分层、Skills 按需、Subagent 隔离 | Skills、Session 历史、上下文和 Memory |
| Harness | Plan/Permission Mode、受信目录、工具、settings、Hooks | Sandbox、Vault、Forward Mode、Template、Identity |
| Loop | 目标、状态、验证、重试和人工接管 | Schedule、Batch、IM Channel 自动触发 |
| Graph | Subagent 是节点，委派和反馈是边 | 多 Agent 委派、Coordinator、API 上层编排 |

### 6.3 Graph 什么时候有价值

当系统出现以下需求时，应该从单 Loop 升级到 Graph：

- 多个 Agent 需要分工
- 任务有清晰的前置依赖
- 需要并行执行后汇总
- 需要跨 Session 传递状态
- 需要不同权限边界
- 需要失败恢复和审计
- 需要把 Agent 接入业务系统 API

## 七、四层不是替代关系

这四层是同一套 Agent 系统的嵌套关系：

```text
Graph：多个 Agent / Loop 如何协作
  └── Loop：单个任务如何持续推进
       └── Harness：Agent 在什么环境中运行
            └── Context：一次调用能看到什么
                 └── Model Call：推理与工具调用循环
```

每一层解决不同问题：

- Context 控制模型注意力
- Harness 控制执行环境和边界
- Loop 控制任务推进和结果验证
- Graph 控制多个执行单元之间的协作

因此不能用“上层替代下层”来理解：一个 Graph 内部仍然包含多个 Loop，每个 Loop 又依赖 Harness，每次运行都要构建 Context。

## 八、对 Java + Agent 开发的启示

如果用 Java/Spring 实现类似系统，可以按层拆分：

| 层次 | Java 后端对应模块 |
|---|---|
| Context | ContextBuilder、TokenBudget、PromptTemplate、MemoryRetriever |
| Harness | ToolRegistry、PermissionService、SandboxAdapter、HookRunner |
| Loop | AgentRunService、RetryPolicy、Verifier、Checkpoint、HumanApproval |
| Graph | Workflow/StateGraph、Coordinator、SubagentDispatcher、EventBus |

建议先做一个最小闭环：

```text
ContextBuilder
  → Agent Loop
  → Tool Calling
  → Result Verifier
  → Checkpoint
```

再逐层增加：

1. Context：项目规则、Skill 按需加载和上下文压缩。
2. Harness：工具权限、沙箱、Hook、状态和观测。
3. Loop：重试、验证、恢复、定时任务和人工接管。
4. Graph：并行 Subagent、状态传递和 Coordinator。

## 九、结论

> **Context 决定模型看见什么，Harness 决定 Agent 在什么环境里运行，Loop 决定任务如何持续推进，Graph 决定多个 Agent 如何协作。**

真正的 Agent 工程，不是单独选择 Context、Harness、Loop 或 Graph，而是根据当前系统最突出的瓶颈，优先补齐对应的一层。

## 原图

![图1：封面](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-01.webp)

![图2：文章引言](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-02.webp)

![图3：从 Prompt 到 Graph 的工程演进](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-03.webp)

![图4：四层架构与 Qoder 对比](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-04.webp)

![图5：Context Engineering](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-05.webp)

![图6：Context 窗口与渐进式披露](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-06.webp)

![图7：Harness Engineering](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-07.webp)

![图8：云端 Harness 与本地 Harness](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-08.webp)

![图9：Loop Engineering](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-09.webp)

![图10：本地与云端 Loop](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-10.webp)

![图11：Graph Engineering](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-11.webp)

![图12：四种 Engineering 组成一套 Agent 系统](https://gitee.com/cheng-jiaqing/images/raw/master/2026-08-04-从Context到Graph-Agent工程四个层次-12.webp)

## 信息边界

- 本文是对小红书图文观点的整理与工程化延伸，不是 Qoder 官方技术文档。
- Qoder CLI 与 Qoder Cloud Agents 的具体能力、API 和实现细节应以官方文档和源码为准。
- “四个层次”更适合作为架构分析框架，不应理解为所有 Agent 系统都必须采用的固定产品架构。


---

## 🔗 关联笔记

- [[2026-07-31-Agent系统架构设计-Harness-Loop-Graph怎么选]] — 同主题视频深度版
- [[2026-07-30-AI-Agent落地铁三角-Graph-Loop-Harness]] — 企业落地铁三角
- [[2026-07-31-10分钟讲透AI-Agent-8种主流架构]] — 8 种架构总览