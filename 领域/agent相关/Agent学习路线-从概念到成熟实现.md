# Agent 工程学习路线：从概念到成熟实现

> **文档来源**：整理自一份面试复盘笔记（原作者面完 Agent 岗位后总结的六步学习路线），并结合 2026 年 10 月为止的一手资料，逐一对照了四个成熟 Agent 实现在每个知识点上的真实做法：
>
> - **cc** = Claude Code（Anthropic 终端编码代理）
> - **codex** = OpenAI Codex（CLI + 云端）
> - **pi** = Mario Zechner 的极简编码代理（github.com/badlogic/pi-mono，2026 年起迁移到 earendil-works/pi）
> - **hermes** = Nous Research 的开源 Hermes Agent CLI（github.com/NousResearch/hermes-agent）
>
> 另引用 Anthropic / OpenAI / Manus 的工程博客作为一手方法论来源。
>
> **原文路线图**：LLM 基础 → Agent Loop / Orchestration → Context & Retrieval → Runtime / Harness → Eval & Observability → Framework
>
> **原文面试原则**：简历写了什么，就一定要把什么吃透。面试官真正想知道的，不是你用了什么 Agent 框架，而是：你为什么这么设计，以及当它在生产环境里真的出问题时，你知不知道问题在哪、该怎么把它救回来。

---

## 目录

- [第 1 章 LLM 基础](#第-1-章-llm-基础)
- [第 2 章 Agent Loop / Orchestration](#第-2-章-agent-loop--orchestration)
- [第 3 章 Context & Retrieval](#第-3-章-context--retrieval)
- [第 4 章 Runtime / Harness](#第-4-章-runtime--harness)
- [第 5 章 Eval & Observability](#第-5-章-eval--observability)
- [第 6 章 Framework](#第-6-章-framework)
- [第 7 章 学习路线自测与面试准备](#第-7-章-学习路线自测与面试准备)
- [附录 A：术语速查表](#附录-a术语速查表)
- [附录 B：参考资料](#附录-b参考资料)

> 说明：文中标注「（非官方逆向）」的内容来自社区逆向工程，仅供参考；其余均给出官方或一手来源链接。Codex 现行文档在 learn.chatgpt.com（原 developers.openai.com/codex 301 跳转）。

---

# 第 1 章 LLM 基础

> 原文：先把最底层的东西搞懂：Token / Context Window / Message Protocol / Prompt / Structured Output / Function Calling / Sampling / 模型能力边界。不一定一开始就钻 Transformer 数学推导，但至少要知道：**模型吃进去的是什么，吐出来的是什么，为什么它的输出是不确定的**。

## 1.1 名词详解

### Token
- 模型读写的最小单位，不是"词"也不是"字符"，是 BPE 分词后的片段。1 个英文 token ≈ 0.75 个单词；中文通常 1 字 ≈ 1~2 token。
- **对 Agent 为什么重要**：计费、限速、上下文预算全都按 token 算。Agent 一场任务动辄几十万 token 输入（Manus 统计自己输入:输出 ≈ 100:1），token 管理就是成本管理。

### Context Window（上下文窗口）
- 模型单次推理能"看见"的全部 token 上限——系统提示 + 对话历史 + 工具定义 + 工具结果**全部**占这个额度，超了就必须丢或压缩。
- 现状（2026）：主流 200K；Claude 新模型、GPT-5.1-Codex-Max（400K）等已到 400K~1M。窗口变大不等于可以乱塞——见第 3 章 "context rot"（上下文腐烂）：token 越多，注意力被稀释，召回准确率反而下降。
- **常见误解**："窗口 1M，我就把所有文件都塞进去"。面试官听到这句基本就结束了。

### Message Protocol（消息协议）
- 模型 API 的输入输出结构：一个 `messages` 数组，每条消息有 `role`（system / user / assistant / tool）和 `content`（文本块、`tool_use` 块、`tool_result` 块等）。
- Agent 的本质之一：**一个不断往这个数组追加消息、反复调 API 的循环**。所谓"多轮对话"，只是把历史 messages 原样重发。
- 关键细节：`tool_result` 是放在 **user 角色消息**里回传的（Anthropic 协议），模型在下一次采样时看到它——这就是"Tool Result 为什么还要放回 Messages"的协议层答案。

### Prompt（提示）
- 广义：发给模型的一切内容。狭义：其中的指令部分（system prompt、用户指令）。
- Agent 里的 prompt 分层：系统提示（行为约束 + 环境信息 + 工具说明）、项目记忆（CLAUDE.md / AGENTS.md）、动态注入（system-reminder、hook 注入的上下文）。
- 原文观点落地：成熟实现里系统提示差异巨大——Claude Code 条件拼装多个 section；Codex 按模型版本各放一份提示文件（`gpt_5_codex_prompt.md` 等）；pi 只有约 20 行、加工具定义不到 1000 token。

### Structured Output（结构化输出）
- 强制模型按 JSON Schema 输出，而不是自由文本。实现方式：约束解码（grammar-constrained decoding）、JSON mode、tool-use 形式的"结构化输出工具"。
- **对 Agent**：路由分类、评测打分、最终交付物都靠它。Codex CLI 的 `codex exec --output-schema schema.json` 可以直接约束最终答案的 JSON Schema。

### Function Calling / Tool Calling（函数调用）
- 把函数的 name + description + JSON Schema 参数定义随请求发给模型；模型返回 `stop_reason: "tool_use"` 和一个 `tool_use` 块（含 `id`、`name`、`input`）；**客户端**执行函数，把结果作为 `tool_result`（带 `tool_use_id`）放回 messages，再发下一轮。
- 记住分工：**模型只"决定调什么、传什么参数"，执行永远在你的代码里**。工具描述就是给模型看的 API 文档——写得越像"给新人入职文档"，调用越准（Anthropic《Writing effective tools for agents》）。

### Sampling（采样）
- 模型输出是逐 token 生成的概率分布采样：`temperature`（分布平滑度）、`top_p`（截断累计概率）、`max_tokens`（生成长度上限）。
- **为什么输出不确定**：每一步都在分布里随机抽样，同一输入两次输出可以不同；temperature=0 只是近似确定（浮点/批处理仍可能有差异）。
- 对 Agent 的推论：**任何单次调用都可能错**，所以需要重试、验证、评测（第 4、5 章）。

### 模型能力边界
- 必须心里有数的几条：幻觉（编造事实/引用）；上下文腐烂（长上下文召回下降）；训练数据截止时间（不知道新 API）；数学与精确计算弱（所以要给计算器/代码工具）；"自我认知"不可靠（会说不知道或乱说）。
- Agent 工程的大部分设计（验证、评测、沙箱、人审）都是在**给能力边界兜底**。

## 1.2 成熟实现在这块怎么做

**cc（Claude Code）**
- 走标准 Messages API：`POST /v1/messages`，`stream: true` SSE 流式；停止原因三种——`end_turn`（自然结束）、`tool_use`（要调工具）、`stop_sequence`。见 [how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)。
- 系统提示**不是一段字符串**，而是按环境条件拼装的多个 section（环境信息、git 状态、工具描述、子代理提示等），社区有完整镜像：[Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)。
- CLI 直接暴露循环参数：`--max-turns`、`--system-prompt`、`--append-system-prompt`、`--model`、`--effort`（low→ultracode）。

**codex（OpenAI Codex）**
- Rust 核心（`codex-rs`，约 99 个 crate），**只走 Responses API**（`wire_api = "responses"` 是唯一支持的值）。
- 推理模型的思维链以 `encrypted_content` 形式由服务端加密保存、可回放但不暴露给客户端——客户端只看到 reasoning summary 流。
- `apply_patch` 工具不是 JSON 参数，而是 **grammar 约束的 freeform 工具**（lark 语法定义补丁格式）——用解码约束而非提示词来保证补丁格式正确，这是"结构化输出"的高级用法。
- 推理力度可调：`model_reasoning_effort = low | medium | high | xhigh | max | ultra`（xhigh 随 GPT-5.1-Codex-Max 引入）。

**pi**
- 作者的核心观察：市面上虽有无穷多模型，底层 wire API 只有四种（OpenAI Completions、OpenAI Responses、Anthropic Messages、Google GenAI），所以 `pi-ai` 做统一抽象，工具 schema 用 TypeBox/AJV 定义，跨提供商可迁移。
- 极简实证：系统提示约 20 行，系统提示 + 内置工具定义 **合计不到 1000 token**，内置只有 4 个工具（read / write / edit / bash）。作者用它证明"模型不需要一万 token 的系统提示也能干活"。

**hermes（开源工具调用事实标准）**
- 即使你说的是 Nous Research 的 Hermes Agent，也值得知道 **Hermes function-calling 格式**——开源权重模型工具调用的 de-facto 标准（vLLM 专门有 `--tool-call-parser hermes`，Qwen 等模型沿用）：
  - 系统提示里用 `<tools>[...JSON schemas...]</tools>` 嵌入工具定义；
  - 模型输出 `<tool_call>{"name": ..., "arguments": {...}}</tool_call>`；
  - 工具结果以 `tool` 角色包在 `<tool_response>...</tool_response>` 里回传；
  - 全部跑在 ChatML 模板（`<|im_start|>role ... <|im_end|>`）上。
- 这就是"Message Protocol"最裸露的形态：所谓协议，最后就是这些特殊 token 和模板。

## 1.3 自测问题

1. 一条 `tool_result` 消息放在哪个 role 里？为什么？
2. `temperature=0` 为什么仍不能保证 100% 确定？
3. 为什么 `apply_patch` 要用语法约束而不是 JSON Schema？
4. 你的 Agent 提示词 + 工具定义占多少 token？窗口预算怎么分？

---

# 第 2 章 Agent Loop / Orchestration

> 原文：接下来理解 Agent 到底是怎么"跑起来"的：Tool Calling / ReAct / Planning / Routing / Orchestration / Handoff / Reflection / Retry / Stop Condition。这一层搞懂之后，再看各种 Agent 框架会容易很多。

## 2.1 名词详解

### Agent Loop（代理循环）
Agent 的最小定义就是 Anthropic 那句话："**LLM using tools based on environmental feedback in a loop**"——在循环里用工具、根据环境反馈决定下一步。伪代码：

```python
messages = [system, user]
while True:
    resp = llm(messages, tools)          # 采样一次
    messages.append(resp.assistant_msg)
    if resp.stop_reason != "tool_use":   # Stop Condition
        break
    results = execute(resp.tool_calls)   # 执行工具（你的代码）
    messages.append(tool_results)        # 结果放回 messages
```

三个阶段反复交替：**收集上下文 → 采取行动 → 验证结果**（Claude Code 官方描述）。

### Tool Calling（循环中的细节）
- **串行 vs 并行**：模型可以在一条 assistant 消息里输出**多个** `tool_use` 块 → 客户端并发执行 → 在**同一条** user 消息里用多个 `tool_result`（按 `tool_use_id` 配对）回传。有依赖关系就自然退化为串行（下一轮才知道上一轮结果）。
- **失败也要回传**：工具报错不是抛异常终止，而是把错误文本作为 `tool_result` 给模型，让它自行调整——错误信息是模型的环境反馈。

### ReAct
Reasoning + Acting：把"思考轨迹"和"行动"交替写在上下文里（Thought → Action → Observation → …）。现代实现里，"Thought"就是推理模型的 reasoning summary 或模型自然写出的分析文本，协议层面已经内化了 ReAct，不需要显式模板。

### Planning（规划）
- 让 Agent 先分解任务再执行。两种形态：**内置**（推理模型自己想）和**外置**（显式的计划工具/文件）。
- 外置的代表：Claude Code 的 TodoWrite/TaskCreate 与 plan mode；Codex 的 `update_plan` 工具（把计划作为结构化状态回显给模型）；Manus 的 `todo.md` 重写（见第 3 章 "recitation"）。

### Routing（路由）
先分类再分发：按输入特征把请求导给不同的模型/子代理/流程。用途：简单问题走便宜模型、专业问题走专家 Agent。是"多 Agent"里最简单也最常用的形态。

### Orchestration（编排）
协调多个 LLM 调用/子代理的执行顺序与数据流。Anthropic《Building effective agents》的分类法必须背下来：
- **Workflow（写死流程）**：LLM 和工具通过**预定义代码路径**编排；
- **Agent（自主决策）**：LLM **动态决定**自己的流程和工具使用。
- 五种模式：① Prompt Chaining（链式 + 门槛检查）② Routing（路由）③ Parallelization（sectioning 分区 / voting 投票）④ **Orchestrator-Workers**（中央 LLM 动态拆解任务、派发 worker、汇总结果）⑤ Evaluator-Optimizer（生成者-批评者循环）。
- 选型原则：**能用单次调用 + 检索解决的，别上 Agent**；模式可以组合。

### Handoff（移交）
一个 Agent 把控制权连同上下文交给另一个 Agent。两种流派：
- **摘要式**（Claude Code 的 Agent 工具）：父代理把任务说明写给子代理，子代理跑完只回一段最终报告；
- **全量式**（OpenAI Agents SDK 的 handoff）：把整个对话历史移交给下一个 Agent 接管循环。

### Reflection（反思）
让模型审视自己的输出/轨迹并改进：生成-批评循环（evaluator-optimizer）、失败后自我修正、验证步骤（跑测试看结果再决定下一步）。注意 Reflection 是**花 token 买质量**，要有明确评分标准才值得。

### Retry（重试）
- 分三层：**API 层**（429/529 限流过载，指数退避）；**工具层**（网络抖动重试）；**语义层**（结果错误，让模型换思路重试）。
- 关键约束：**有副作用的工具能不能重试**？——不能盲目重试，必须配幂等性（第 4 章）。

### Stop Condition（停止条件）
循环必须有出口，否则烧钱且可能越跑越偏。常见：模型自然结束（`end_turn`）、达到最大轮数（`--max-turns` / `maxTurns`）、预算上限（`--max-budget-usd`）、人工打断、外部事件满足。pi 的激进观点："loop 就 loop 到 agent 说做完为止"——但那是本地 CLI 的特权，生产服务必须设预算护栏。

## 2.2 六个必答问题（原图原题，逐条回答）

**Q1：模型什么时候调用 Tool？**
- 机制上：模型在采样时输出 `tool_use` 块并给出 `stop_reason: "tool_use"`；是否调用由训练 + 工具的 name/description/参数 schema + 系统提示共同决定。
- 控制手段：工具描述写清适用场景（"何时用/何时不用"）；解码层控制——Manus 用三种 prefill 模式（auto / required / specified，预填到 `<tool_call>` 或函数名前缀，**强制或限定**模型调用某类工具）；Claude Code 用权限系统拦（模型想调，宿主可以拒）。
- 加分项：说明"模型不是每次需要信息都调工具，描述里的指导、few-shot 和 system-reminder 都影响调用时机"。

**Q2：Tool Result 为什么还要放回 Messages / Execution Context？**
- 因为 **API 是无状态的**：每次请求都要带全量历史。模型在下一轮采样时，只有看到 `tool_result` 才"知道"上一轮动作的结果，否则它会幻觉出一个结果或重复调用。
- 工程含义：结果放回的位置和格式直接影响下一轮决策质量；所以大结果要截断/蒸馏后再放（第 3 章），放回时要按 `tool_use_id` 精确配对。

**Q3：多个 Tool 怎么串行 / 并行？**
- 协议：一条 assistant 消息可含多个 `tool_use` 块（并行意图）；宿主并发执行后在一条 user 消息里批量回 `tool_result`。
- 真实实现：Claude Code 支持并行且可调（`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`）；v2.1.72 起一个 Read/WebFetch 失败不再连坐取消兄弟调用（只有 Bash 错误才级联）。Codex 的子代理还要求"每个子代理 3+ 个工具调用并行"以提速。
- 判断串并行的依据：**数据依赖**。有依赖必须串行；无依赖并行省时延。

**Q4：主 Agent 怎么调用 Sub-Agent？**
- 标准做法：**子代理就是一个工具**。Claude Code 的 Agent（旧名 Task）工具：输入是任务描述 + 可选工具/模型限制，子代理在**独立上下文窗口**里跑完整循环，只把最终报告返回给父代理。
- Claude Code 细节：子代理有自己的系统提示、工具白名单、独立权限；转录存为 `agent-{agentId}.jsonl`；并发上限 20（`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`）、嵌套深度 3。Codex 的 `spawn_agent` 明确"子代理看不到父对话轮次"（上下文隔离）。
- 为什么隔离：子过程要读几十万 token 的原始材料，若共享父窗口会立刻爆炸；父代理只需要蒸馏后的结论（Anthropic：子代理回传 1~2K token 摘要）。

**Q5：Agent 之间怎么传状态？**
- 反模式：把一个 Agent 的完整对话历史塞给下一个（"传话游戏"失真 + 爆窗口）。
- 正确做法三选一或组合：① **结构化摘要**（handoff 四要素：目标、输出格式、工具/来源指引、任务边界——Anthropic 多代理研究系统的实测结论，模糊的一句话任务会导致两个子代理重复搜索）；② **外部产物 + 轻量引用**（把大输出写到文件系统/对象存储，只传路径，"避免传话游戏"出自该文附录）；③ **共享外部状态**（数据库/任务列表/内存服务，如框架的 state/checkpointer）。
- Hermes Agent 的做法：独立子代理 + Python-RPC 工具调用组成"零上下文成本管线"。

**Q6：什么时候该写死 Workflow，什么时候让 Agent 自主决策？**
- Anthropic 的判据：**子任务序列能否预测**。能预测 → 写死（chaining/routing/parallelization）：更便宜、更快、更可控；不能预测（多文件改码、多源研究）→ orchestrator-workers 或自由 agent。
- 补充判据：错误代价（大 → 多加固定轨道与人审）、可验证性（有测试/评分器 → 放心给自主权）、任务频率（高频任务值得写死固化）。
- 面试金句："Workflow 是用确定性换灵活性，Agent 是用 token 和不确定性换灵活性；按子任务可预测性和错误代价选。"

## 2.3 成熟实现在这块怎么做

**cc（Claude Code）**
- 主循环：gather context → take action → verify work → repeat；模型自然结束或达到 `--max-turns` 退出（Agent SDK 返回 `error_max_turns`）。
- 并行真实存在：研究编排器"同时起 3~5 个子代理"；后台 shell（`run_in_background` + `Ctrl+B`）处理 dev server 这类长任务。
- 规划：TodoWrite/TaskCreate 工具 + plan mode（只读直到计划被批准）。
- 子代理生态：`Explore`（只读侦察）、`Plan`、general-purpose 三种内置；自定义子代理就是 `.claude/agents/*.md`（YAML frontmatter 定义 tools/model/permissionMode/maxTurns/isolation: worktree 等）。

**codex（OpenAI Codex）**
- 循环代码在 `codex-rs/core/src/session/turn.rs` 的 `run_turn()`：先做**预采样压缩**（pre-sampling compact），再进入对 `ResponseEvent` 流的处理循环（OutputItemDone / OutputTextDelta / ToolCallInputDelta / ReasoningSummaryDelta / Completed…）。
- **steering（转向）**：任务循环跑完后检查排队输入，有就继续跑——用户可以在长任务中途插话，请求进入队列而不是打断。
- `update_plan` 工具维持显式计划状态；`get_context_remaining` / `new_context_window` 是"上下文自感知"工具（模型可以查自己还剩多少窗口）。

**pi**
- `pi-agent-core` 的 `Agent` 类：状态管理、事件订阅、**消息队列分 steer（插话）与 follow-up（追加）两种**、传输抽象。**故意没有 max-steps 参数**："The loop just loops until the agent says it's done."
- 明确反对内置子代理："You have zero visibility into what that sub-agent does. It's a black box within a black box."（你需要时用 bash 让 pi 自己 spawn 自己：`pi --print`）——这是对"编排最小化"的一种代表性立场，与 Claude Code/Codex 形成鲜明对照，面试时引用会很加分。

**hermes（Nous Hermes Agent）**
- 隔离子代理（各自独立对话 + 独立终端）；40+ 内置工具；skills 自动从经验中创建并改进（"自带学习循环"是它的主打卖点）。

**Anthropic 多代理研究系统（一手工程经验，面试高引用率）**
- 拓扑：Lead agent（规划、派发、保存计划到 Memory，因为超 200K 会被截断）+ 3~5 个并行子代理 + CitationAgent 补引用。
- 效果：复杂查询研究时间降 90%；Opus 主导 + Sonnet 子代理比单 Opus 高 90.2%；代价是**多代理系统耗 15 倍于普通聊天的 token**（单代理 4 倍）。
- 工作量分级规则（写进提示词）：事实核查 = 1 个代理 3~10 次工具调用；比较 = 2~4 个子代理；深度研究 = 10+ 个子代理分片负责。
- 实测失败模式清单：简单任务也派 50 个子代理；对不存在的来源无限搜索；过度投资简单查询；查询写得过长（改为"先短宽、再窄"）；偏好 SEO 内容农场（加来源质量启发）；幻觉。

## 2.4 自测问题

1. 画出 Claude Code 一轮 "tool_use → tool_result" 的消息序列（含 role 和 block 类型）。
2. 为什么说"子代理是把上下文变成钱的手段"？（提示：15x token vs 90.2% 提升）
3. 你的场景该用 workflow 还是 agent？给出三个判据。

---

# 第 3 章 Context & Retrieval

> 原文：这也是我这轮面试里被问得很深的一块：Context Engineering / Memory / RAG / Embedding / Retrieval / Rerank / Context Compression / Progressive Disclosure。本质上都在解决同一个问题：**Agent 怎么在有限的 Context Window 里，操作远大于 Context 的信息**。比如 Tool 一次返回几十 MB 的日志、JSON、文件怎么办？你不可能全部塞给模型。可能需要 Sandbox、Artifact、grep、Python、Embedding Retrieval……所以 **RAG 只是这里面的一个解法**。

## 3.1 名词详解

### Context Engineering（上下文工程）
- 定义（Anthropic）："在 LLM 推理期间**策划并维护最优 token 集合**的艺术"，目标是"最小的高信号 token 集"。
- 与 prompt engineering 的区别：prompt engineering 优化一次调用的内容；context engineering 优化**整个循环过程**中什么该进窗口、什么时候进、以什么形态进、何时清除。

### Context Rot / Attention Budget（上下文腐烂 / 注意力预算）
- Chroma 的实验：token 数增加时，所有模型的召回准确率都会下降；transformer 注意力是 n² 成对计算，且训练数据偏短序列。窗口 1M ≠ 有效注意力 1M。
- 推论：上下文管理不是"能不能塞下"的问题，是"塞进去模型还能不能用好"的问题。

### Memory（记忆）
- 分两层理解：
  - **短期**：当前对话窗口内的内容（messages 数组本身）。
  - **长期**：窗口外的持久化知识。三种典型形态：① 文件式（CLAUDE.md / AGENTS.md / MEMORY.md——可读可写，人和 Agent 都能改）；② 结构化存储（数据库/向量库）；③ 模型平台记忆（ChatGPT Memory：显式保存的事实 + 隐式引用聊天历史两层）。
- 学术语源：MemGPT（论文《MemGPT: Towards LLMs as Operating Systems》，arXiv 2310.08560）——把窗口当 RAM、外部存储当磁盘，**让 LLM 自己通过函数调用在层级间换页**（self-editing memory），商业化为 Letta。

### RAG（Retrieval-Augmented Generation，检索增强生成）
- 预检索管线：文档 → 切块（chunking）→ 向量化 → 入库；查询时检索 top-K 塞进 prompt。
- 2024~2026 的实践共识：**混合检索**（BM25 关键词 + dense 向量，RRF 融合）打底，因为 BM25 能抓 embedding 漏掉的精确 token（如 "Error code TS-999"）；再用 **Rerank（重排）** 精修。
- **重要观念（原文的观点）**：RAG 只是"有限窗口操作超窗口信息"的一个解法。对代码这类精确符号密集的语料，agentic search（grep/glob + 迭代读文件）往往更好；对海量非结构化文档，检索 + 子代理蒸馏才合适。

### Embedding（向量化）
- 把文本映射为语义向量，余弦相似度近 = 语义近。用于语义检索、去重、聚类。
- 关键实践：Anthropic Contextual Retrieval——给每个 chunk 前面加一段 50~100 token 的 LLM 生成"该块在全文中的位置说明"再入库，检索失败率显著下降（数字见 3.3）。

### Retrieval（检索）
- 广义：Agent 获取外部信息的一切手段——向量检索、关键词检索、**grep/代码搜索**、SQL、API 调用、打开 URL。
- Anthropic 的关键区分：**预检索（pre-retrieval）vs 即时检索（just-in-time）**。JIT = Agent 只持有轻量标识符（文件路径、查询词、URL），运行时按需加载——Claude Code 就是这个流派（CLAUDE.md 常驻 + grep/glob 按需导航）。

### Rerank（重排）
- 检索是"先求召回，再求精度"：初筛（BM25+dense，宽）→ **cross-encoder 重排**（精）。Cohere Rerank、BGE-reranker 是常用件。开源重排代表 BGE-reranker；注意 rerank 不是万能——有评测显示部分跨语言场景反而变差，要在自己的数据上测。

### Context Compression（上下文压缩）
- 窗口快满时的处理：**Compaction（压实/摘要）**——把旧对话总结成摘要重启窗口；**工具结果清理**——旧 tool result 先被清成占位符；**外部化**——大内容落盘成 artifact，上下文里只留引用。
- Manus 的关键补充：压缩必须**可还原**——丢掉页面内容但保留 URL，丢掉文档但保留沙箱路径。

### Progressive Disclosure（渐进式披露）
- 不把所有信息一次性塞进上下文，而是分层：常驻的只有"目录/索引"（一行描述），需要时才加载全文。
- 落地形态：Claude Code 的 Agent Skills（SKILL.md 只有 name+description 常驻，调用时才载入正文；MCP 工具 schema 默认 deferred，用到才加载）；pi 对 MCP 工具同样做暴露模式控制。

## 3.2 原图问题的工程答案："Tool 一次返回几十 MB 日志/JSON/文件怎么办？"

分层手段（按优先级）：
1. **源头截断**：工具设计时就分页/过滤/限量。Claude Code 的 Bash 输出上限 ~30K 字符（超长只保留头尾），Glob 上限 100 个文件，WebFetch 走独立模型调用把 HTML 转成"针对问题的 Markdown 答案"而不是原文入库。
2. **落盘 + 引用（Artifact 模式）**：Claude Code 把 >50K 字符的工具结果写到磁盘，上下文里留路径；需要细节再用 Read/Grep 取——这就是"Sandbox、Artifact、grep、Python"清单的含义。
3. **换个工具取**：几十 MB 日志不要"读"，要 grep / tail / SQL / Python 处理后取结论。
4. **子代理蒸馏**：让子代理在独立窗口里消化原始材料，父代理只收 1~2K token 摘要。
5. **压缩/清理历史**：microcompact 清旧 tool 输出 → compaction 摘要对话（顺序很重要，先清工具输出，不够再整体摘要）。
6. **RAG/索引**：语料巨大且重复查询时才值得建索引（embedding + BM25 + rerank）。

## 3.3 成熟实现在这块怎么做

**cc（Claude Code）——目前公开资料里上下文工程最完整的生产实现**
- **两段式压缩**（官方表述）："先清除较旧的工具输出，仍有需要再把对话摘要化"。细节（非官方逆向）：microcompact 只清 Read/Bash/Grep/Glob/WebSearch/WebFetch/Edit/Write 的旧结果；有一条 cache-aware 路径用 API 的 context-editing 特性头，保住 prompt cache。
- **auto-compact**：token 化阈值（1M 窗口模型默认 ~967K 时触发），可用 `/autocompact 500k`、`autoCompactWindow`、`CLAUDE_CODE_AUTO_COMPACT_WINDOW` 调（100K~1M）；历史上按百分比（60%→80% 预警），v2.0.64 起"压缩即时完成"。连续 3 次压缩失败会熔断。
- **`/compact [指令]`**：手动压实，可带指令（如 `/compact 聚焦 auth bug 的修复`）。**压缩后重载**：系统提示、CLAUDE.md、auto memory、git 状态、计划，还会**重读最多 5 个最近修改的文件**（>5000 token 的文件降级为路径引用），调用过的技能正文按每个 ≤5K、合计 ≤25K token 重注入。
- **`/context`**：彩色网格可视化当前上下文构成；`/clear` 直接清空。
- **CLAUDE.md 层级**：企业策略 → `~/.claude/CLAUDE.md` → 项目 `./CLAUDE.md` → `CLAUDE.local.md`；子目录文件按需加载；`@path` 导入最深 4 跳；以 user 消息身份注入而非 system prompt。
- **记忆**：auto memory（Claude 自己写的笔记，`~/.claude/projects/<项目>/memory/`，索引文件 MEMORY.md 前 200 行每会话常驻）+ 平台 memory tool（`memory_20250818`，六个命令 view/create/str_replace/insert/delete/rename，客户端存储，API 自动注入"先查记忆、假定会被打断"的系统提示）。
- **渐进披露三件套**：Skills（描述 1536 字符内常驻）；MCP 工具 schema deferred；大工具结果落盘。

**codex（OpenAI Codex）**
- 自动压缩 + `/compact`；摘要以**固定前缀的 user 消息**注入："Another language model started to solve this problem and produced a summary..."（前缀常量在 `codex-rs/prompts/templates/compact/summary_prefix.md`），摘要本体上限 20K token；`is_summary_message()` 靠前缀识别摘要消息——这就是"压缩态如何进入协议"的具体答案。GPT-5.1-Codex-Max 起还有**模型侧原生 compaction**（官方宣称支撑 24 小时级任务、跨数百万 token）。
- **AGENTS.md 发现规则**（值得背）：全局 `~/.codex/AGENTS.md` + 项目内从 git 根目录走到 cwd 的所有 AGENTS.md **依次拼接**（后者覆盖前者），单文件优先 `AGENTS.override.md`，总量默认 32KiB 封顶，每会话只发现一次。
- 会话 = JSONL "rollout" 文件（`~/.codex/sessions/年/月/日/rollout-<时间戳>-<uuid>.jsonl`），`codex resume --last` / `codex fork`。

**pi**
- **会话即 JSONL 树**：每条消息记录父节点引用，构成树；只有活跃分支发给模型。`/tree` 在树里导航（选一条 user 消息改写重发 = 开新分支）、`/fork` 分叉、`/clone` 复制分支——把"上下文版本管理"做成了显式用户功能。
- `/compact` 手动 + 近上限自动压缩；压缩插的是"摘要节点"，**原始消息永不删除**（随时能回到压缩前）。
- 哲学：现有 harness 都"在你背后注入东西"，pi 让你**完整看见并自己做上下文工程**——他的口号是"Twitter 上全是 context engineering 帖子，但没有一个 harness 真正让你做 context engineering"。

**hermes（Nous Hermes Agent）**
- 四层记忆（社区归纳）：`MEMORY.md`（事实）/`USER.md`(用户画像)/`SOUL.md`（人格）+ `hermes_state_*` 状态模块；用 **SQLite FTS5 全文检索 + LLM 摘要**搜索过往对话；会话压缩命令 `/compress`；靠 Honcho 做 dialectic 用户建模。特色是"自我学习循环"：从经验里自动创建 skills、用的时候改进 skills。

**Manus 六条经验（2025-07 博客，工程含金量极高）**
1. **围绕 KV-cache 设计**："KV-cache 命中率是生产级 Agent 最重要的单一指标"。他们输入:输出 ≈ 100:1；Claude Sonnet 缓存输入 $0.30/MTok vs 未缓存 $3.00/MTok——**10 倍价差**。做法：提示前缀字节级稳定（别放每秒变化的时间戳！）、上下文 **append-only**（不改历史动作/观察——改了既毁缓存又毁模型对自己历史的信任）、JSON 序列化确定性（key 顺序稳定）。
2. **Mask 而不是 Remove**：动态增删工具会毁缓存并迷惑模型；用状态机在解码时**屏蔽 logits**。三种 prefill 模式（auto/required/specified）+ 工具命名前缀约定（`browser_`、`shell_`）就能按组约束。
3. **文件系统即终极上下文**：一切压缩要可还原（留 URL/路径，不留内容）。
4. **Recitation（复述）**：平均一个任务 50 次工具调用；持续重写 `todo.md` 让目标始终出现在上下文末端，对抗 lost-in-the-middle，减少目标漂移。
5. **把错误留在上下文里**：不要清 trace、不要重置状态、别急着调 temperature——失败动作和堆栈是模型修正信念的证据；错误恢复是核心 agent 能力。
6. **别被 few-shot 带节奏**：连续同构的动作-观察对会诱发"韵律模仿"（20 个连续简历评阅 → 行为复读）；注入结构化变化（换序列化模板、措辞、顺序噪声）。

**Anthropic《Effective context engineering》三技术 + 选型**
- Compaction（对话流）、结构化笔记（里程碑式工作）、子代理架构（并行探索），混合策略 = CLAUDE.md 常驻 + JIT 检索。Claude Code 早期真用过向量 RAG，2025 年中**删掉了 embedding 和本地向量库**，因为 agentic search（ripgrep+读文件）效果更好——这是"RAG 只是解法之一"的最有力例证。

**RAG 数字备查（Anthropic Contextual Retrieval，2024-09）**
- top-20 检索失败率：上下文化 embedding 单用降 **35%**（5.7%→3.7%）；+ 上下文化 BM25 降 **49%**（→2.9%）；再加 Cohere Rerank（top-150 → top-20）降 **67%**（→1.9%）。另注：整个知识库若塞得进 ~200K token，直接全塞 + 缓存即可，别上 RAG。

## 3.4 自测问题

1. 你的 Agent 上下文 1M，为什么还会"变笨"？怎么量化？（context rot / 注意力预算）
2. auto-compact 和 microcompact 的先后与分工？为什么先清工具输出？
3. KV-cache 命中率为什么是第一指标？列出三个破坏缓存的常见写法。
4. 什么场景你会给编码 Agent 上向量 RAG？什么场景坚决不上？（提示：精确符号 vs 语义模糊查询）

---

# 第 4 章 Runtime / Harness

> 原文：Runtime 更关注任务怎么**执行、调度和恢复**；Harness 更关注如何把模型放进一套**受控的工具、环境、约束和反馈闭环**里完成任务。这一部分是最值得重点学的：State / Sandbox / Tool / Permission / Async Task / Streaming / Concurrency / Timeout / Retry / Idempotency / Recovery / Verification。
> Demo 能跑起来只是第一步。**怎么让 Agent 稳定地跑起来，才是真正的工程问题。**

## 4.1 名词详解

### Runtime vs Harness（两个词的分工，面试常考）
- **Runtime**：任务执行基础设施——进程管理、调度、持久化、崩溃恢复、并发控制。回答"任务跑在哪、断了怎么办"。
- **Harness**：模型周围的受控闭环——系统提示、工具集、权限、验证、反馈注入。回答"模型被放进什么样的笼子、看到什么、能做什么"。
- 一个产品两者都有：Claude Code 的 JSONL 会话/重试是 runtime，工具+权限+hooks 是 harness；Codex 云端的容器编排是 runtime，Seatbelt 沙箱+审批是 harness。

### State（状态）
- Agent 运行时的一切可变数据：消息历史、任务列表、文件工作区、变量。三个工程问题：**存在哪**（内存/JSONL/数据库）、**怎么恢复**（持久化 + 重放）、**谁可见**（隔离 vs 共享）。

### Sandbox（沙箱）
- 给模型生成的代码/命令一个受限执行环境：文件系统白名单、网络开关、资源限额、超时。谱系：语言级（受限解释器）→ 容器（Docker/gVisor/Firecracker microVM）→ OS 原生（macOS Seatbelt、Linux Landlock/seccomp/bubblewrap、Windows 受限 token）。

### Permission（权限）
- 对工具调用做"执行前闸门"：白名单规则、模式（只读/自动批准编辑/需确认/全放行）、人审。设计要点：默认拒绝危险操作、规则可分层（企业>用户>项目>本地）、审批可委托给自动审查器。

### Async Task（异步任务）
- 长任务不阻塞会话：后台 shell、云端任务、定时任务。配套需求：任务状态机、完成通知、结果回取。

### Streaming（流式）
- SSE/WebSocket 逐 token 推给 UI，首 token 延迟决定体感。Agent 层面还要流"事件"（工具开始/结束、思考摘要）而非只有文本。

### Concurrency（并发）
- 并行工具调用、并行子代理、多会话。控制点：并发上限、错误隔离（一个失败不连坐）、资源竞争（两个 Agent 同时改一个文件）。

### Timeout（超时）
- 每个工具调用都要有超时（Claude Code Bash 默认 2 分钟、上限 10 分钟；Codex exec 默认 yield 10 秒 + 输出 1MiB 截断）。超时后：返回错误给模型让它换方案，而不是挂死。

### Retry（重试）
- API 层指数退避（Claude Code 最多 10 次，可到 15；Codex 请求重试 4 次/流重试 5 次）；语义层重试要配合反思。**重试必须回答幂等问题**（下条）。

### Idempotency（幂等性）
- 同一操作执行多次与一次效果相同。Agent 场景：任务恢复、重试、用户刷新页面都可能**重复触发同一个有副作用工具**（付款、发邮件、写数据库）。
- 标准方案（Stripe 模型）：客户端生成唯一 idempotency key → 服务端按 key 存首次响应并重放；参数不一致报错。给 Agent 的推论：**凡是有副作用的工具，要么设计成幂等，要么包一层幂等键**。

### Recovery（恢复）
- 崩溃/断线/刷新后从**断点**继续而不是从头再来：Anthropic 的研究代理"从错误发生的位置恢复"；实现靠检查点 + 事件溯源重放（durable execution）。
- 关键配套：**会话持久化**（Claude Code 的 JSONL + `--resume`/`--continue` + rewind 检查点；Codex 的 rollout 文件 + `codex resume`；pi 的会话树）。

### Verification（验证）
- 把"Agent 说做完了"和"真的做完了"分开：跑测试、编译、diff 审查、评分器。编码 Agent 之所以是好场景，正因为**可验证性强**（Anthropic 原话）。

## 4.2 原图生产问题清单，逐条对答案

| 生产问题 | 行业答案 |
|---|---|
| Tool 超时怎么办？ | 每工具必设超时；超时作为错误反馈给模型换路；长任务转后台任务（CC 的 run_in_background、Codex 云端） |
| 有副作用的 Tool 能不能 Retry？ | 盲目重试=灾难。要么工具幂等，要么幂等键包装；不可幂等的（发邮件类）只允许人审后单次执行 |
| Agent 跑十分钟，中间断线怎么办？ | 断点续跑：会话 JSONL/rollout 落盘 + resume；流断开重连（Codex 指数退避 5s→60s、可降级 WebSocket 传输、尊重 Retry-After）；CC 中断时保留已生成的部分输出 |
| 页面刷新以后任务怎么恢复？ | 服务端持有状态而非前端：任务状态机（Codex cloud: Pending/InProgress/Completed/Failed/Cancelled）+ 按 id 查询重连；云端任务状态保留 7 天 |
| Sub-Agent 怎么隔离？ | 独立上下文窗口 + 独立工具/权限集 + 独立转录文件；并发与深度上限；产物落盘只传摘要（CC：20 并发、深度 3） |
| 代码、文件、Shell 怎么安全执行？ | OS 级沙箱（Seatbelt/bubblewrap/Windows 受限 token）+ 默认禁网 + 文件白名单 + 审批策略 + hooks 事前拦截（下节细节） |

## 4.3 成熟实现在这块怎么做

**cc（Claude Code）**
- **权限**：六种模式 `default / acceptEdits / plan / auto / dontAsk / bypassPermissions`；规则语法 `Bash(npm run *)`、`Read(./.env)`、`WebFetch(domain:example.com)`；评估顺序**严格 deny → ask → allow**（allow 不能从 deny 里抠例外）；配置优先级：managed > 用户 > 项目 > 本地。
- **Hooks**：30+ 生命周期事件（SessionStart / UserPromptSubmit / PreToolUse / PostToolUse / Stop / PreCompact / SessionEnd…）；exit code 2 = 阻断该工具调用；stdout JSON 可返回 `permissionDecision: allow|deny|ask`、改写 `updatedInput`、注入 `additionalContext`。hooks 不能绕过 deny 规则。
- **沙箱**：v2.0.24（2025-10）给 Bash 工具上沙箱，基于开源 `@anthropic-ai/sandbox-runtime`：macOS Seatbelt，Linux/WSL2 bubblewrap + 网络代理域名白名单（初始为空！）；写权限限 cwd/TMPDIR；`.git`、`~/.claude` 属保护路径。
- **恢复**：会话是明文 JSONL（`~/.claude/projects/<项目>/<session>.jsonl`）；`--continue` 接最近会话、`--resume` 按会话 id/路径接；**rewind 检查点**（v2.0.0，Esc Esc / `/rewind`）：每个 prompt 存快照，可只回滚代码/只回滚对话/都回滚——注意只跟踪"通过编辑工具做的修改"，bash 里 `rm` 的不认。
- **重试**：瞬时错误指数退避最多 10 次；CI 可开 `CLAUDE_CODE_RETRY_WATCHDOG=1` 无限重试 429/529；压缩失败 3 次熔断；headless 模式会发 `api_retry` 事件（attempt/max_retries/retry_delay_ms）。

**codex（OpenAI Codex）**
- **沙箱矩阵**（全网最完整）：macOS Seatbelt；Linux 原生 Landlock+seccomp，0.115+ 迁移到 bubblewrap；Windows 双模式（elevated：专用低权沙箱用户 + ACL + 防火墙；unelevated：受限 token + 私有桌面）。`workspace-write` 模式**默认禁网**，按配置开 `[sandbox_workspace_write] network_access = true`。
- **审批**：`approval_policy` 从 untrusted/on-failure/on-request/never 演进到 2026 的**粒度化配置**（sandbox_approval / rules / mcp_elicitations / request_permissions / skill_approval 分开设）；2026 新增 `approvals_reviewer = "auto_review"`——把审批提示交给一个自动审查代理（高危要求用户授权、构建失败即拒绝=fail closed）；GPT-6 Astra 异步安全监控可在发现不安全行为时**暂停任务**。
- **工具可靠性细节**：`exec_command` 带 `yield_time_ms`（默认 10s）+ `max_output_tokens`（默认 10K）+ PTY 会话（长命令返回 session id 持续交互）；输出硬上限 1MiB；`apply_patch` 语法约束杜绝格式错误。
- **重试/重连**：请求重试 4、流重试 5（`request_max_retries` / `stream_max_retries`），空闲超时 5 分钟；连接失败 5s→60s 指数退避，尊重服务端 Retry-After；流重试耗尽可**换传输协议**续命。

**pi**
- 立场鲜明的反面教材（也有价值）：**"pi 全程 YOLO 模式，权限大多是 security theater"**——替代方案是文档建议你自己在 Docker/micro-VM 里跑，加供应链加固（锁版本、`min-release-age=2` 拒绝发布不到 2 天的包、`--ignore-scripts` 安装）。
- **project trust 机制**：项目的扩展/技能/AGENTS.md 首次加载前要求用户显式信任（扩展与 pi 同进程、同 OS 权限——所以必须先信）。
- **RPC 模式**（`pi --mode rpc`）：stdin/stdout 上跑 JSONL，ID 关联请求响应，`agent_settled` 事件标记回合完成——这就是它作为 runtime 被 IDE/其他程序驾驶的方式；TypeScript SDK 的 `createAgentSession()` 同理。

**hermes（Nous Hermes Agent）**
- 终端后端可插拔：local / **Docker / SSH / Singularity / Modal / Daytona / Vercel Sandbox**（7 种），带容器加固与命名空间隔离；命令审批可开；serverless 后端支持休眠省钱；内置 cron（定时任务）+ 消息网关（Telegram/Discord/Slack/WhatsApp/Signal/Email）——它把"runtime 的任务入口"扩展成了全渠道。

**Durable Execution（持久化执行，生产级 runtime 的通用答案）**
- **Temporal**：工作流代码事件溯源 + 确定性重放，代理从检查点恢复（官方有 OpenAI Agents SDK 集成课程）。
- **DBOS**：Postgres 上的轻量方案——`@DBOS.workflow()` 编排 `@DBOS.step()`，每步输出落库，进程被杀后重放已完成步骤不重复执行副作用（官方 demo：杀进程后退款不会退两次）。
- **Inngest**：step 函数、每步独立重试计数、`step.waitForEvent()` 等人工审批（可等 7 天）。
- **LangGraph Platform**：托管 checkpointer，持久化免自建。
- 部署配套：Anthropic 用 **rainbow deployment**（新旧版本同时在线，让进行中的 agent 在原版本上跑完）——有状态长任务不能裸发布。

## 4.4 自测问题

1. 你的工具要给用户转账。重试策略？幂等键放哪？崩溃恢复时怎么保证不双花？
2. 对比 Seatbelt / Landlock / bubblewrap / Docker 四种沙箱的隔离强度与接入成本。
3. Claude Code 的 rewind 为什么不跟踪 `rm -rf`？这个设计说明了什么权衡？
4. 为什么 Codex 要把审批做成"粒度化 + 自动审查代理"而不是一个总开关？

---

# 第 5 章 Eval & Observability

> 原文：最后还需要回答：**你怎么知道 Agent 真的变好了？** Eval Dataset / Task Success / Trajectory / Tool Accuracy / LLM-as-a-Judge / Trace / Failure Taxonomy / Regression / Cost / Latency。Agent 不能只是"感觉比以前聪明了"。

## 5.1 名词详解

### Eval Dataset（评测集）
- 固定的任务集 + 可验证的期望结果。从 ~20 条**真实业务查询**起步即可（Anthropic：早期修 bug 用 20 条真实查询，效果从 30% 提到 80%）。要点：任务来自真实分布、有确定性可查的结果、覆盖失败模式。

### Task Success（任务成功率）
- 最硬的指标：任务是否达成（测试通过/订单创建/答案正确）。**对会改状态的 Agent，评最终环境状态，不要评过程文本**（tau-bench 就是 diff 最终数据库状态）。

### Trajectory（轨迹）
- 定义（Anthropic）："一次完整尝试的记录——输出、工具调用、推理、中间结果"。轨迹评测看过程质量（步骤是否合理、有没有绕路、工具用得对不对），用于状态不落地的任务（如研究/咨询类）。
- 争议点：Anthropic 明确说"**通常评产出比评路径更好**"——轨迹评分容易奖励"看起来勤奋"。

### Tool Accuracy（工具准确率）
- 细分指标：该调没调、不该调乱调、参数错、结果被正确使用。Anthropic 工具评测四指标：accuracy、runtime、工具调用次数、token 消耗（+ 工具错误率）。

### LLM-as-a-Judge（模型当裁判）
- 用强模型按 rubric 给输出打分。MT-Bench 论文（arXiv 2306.05685）：强裁判与人类一致率 >80%（达到人类之间一致水平）。
- **三种已知偏差必须会背**：位置偏差（偏好某个顺序 → 成对评测时交换顺序）、冗长偏差（偏爱好长答案）、自增强偏差（偏好自己家族的输出）。缓解：rubric 结构化 + 提供"无法判断"出口（Anthropic 做法）+ 单裁判 0~1 分制反而胜过多裁判集成。
- 确定性可判的先用确定性判分（跑测试），LLM 裁判只兜语义层。

### Trace（追踪）
- 每次运行的完整事件流（每次 LLM 调用、工具调用、耗时、token）。观测三支柱映射到 Agent：metrics（成本/延迟/成功率）、traces（决策链）、logs（事件）。标准：OpenTelemetry GenAI 语义约定（`gen_ai.*`）。

### Failure Taxonomy（失败分类学）
- 把失败**归类**才能系统性修。学术参考 MAST（《Why Do Multi-Agent LLM Systems Fail?》，arXiv 2503.13657）：3 大类 14 小类——① 系统设计缺陷 ② 代理间失协（如任务重复、传话失真）③ 任务验证缺失（过早停止、幻觉完成）。由 150 条专家标注轨迹建系（kappa 0.88）。
- 工程做法：每个失败 trace 打标签，按类别修——prompt 问题、工具设计问题、上下文问题、模型能力问题各不同。

### Regression（回归）
- 改了 prompt/模型/工具后，旧任务集会不会变差？必须有**回归测试门禁**：评测集 + CI（promptfoo 这类工具做 prompt 的 CI 回归，断言可确定性可模型判）。

### Cost / Latency（成本 / 延迟）
- 成本 = token 用量 × 单价，注意缓存价差 10 倍（第 3 章 KV-cache）；延迟看 P50/P95 + 首 token 时间 + 总任务时长。二者是"效果提升"的约束项——效果提升 5% 成本翻 3 倍在生产上可能就是负收益。

## 5.2 成熟实现在这块怎么做

**评测方法论（Anthropic《Demystifying evals for AI agents》，2026-01）**
- 评**环境终态**而不是 prose；轨迹 = 完整尝试记录；"评产出常优于评路径"。
- LLM 裁判要校准：结构化 rubric + "Unknown" 逃生口。
- **pass^k** 一致性指标：同一任务跑 k 次全过的概率——tau-bench 实测 gpt-4o 单次 <50%，**pass^8 <25%**："单次运气"和"稳定可靠"是两回事。
- 编码评测先确定性判分（"代码能跑吗？测试过吗？"——SWE-bench Verified、Terminal-Bench 都是这个思路）。
- 警示案例：pass@100 为 0% "多半说明任务坏了"。

**基准测试速查（2026 数字）**
| 基准 | 测什么 | 参考数字 |
|---|---|---|
| SWE-bench Verified | 真实 GitHub issue 修复（500 条人工过滤） | codex-1 72.1% → GPT-5.1-Codex-Max 77.9% |
| Terminal-Bench 2.0 | 终端环境任务 | GPT-5.1-Codex 52.8% → Max 58.1% |
| tau-bench | 工具+用户模拟+领域策略（客服） | gpt-4o pass^8 <25%（零售域） |
| OSWorld | 真实电脑操作（369 任务） | 发布时人 72.36% vs 最好模型 12.24% |

**cc（Claude Code）**
- **OTel 指标**（`claude_code.*` 前缀）：session.count、token.usage、cost.usage（美元）、lines_of_code.count、pull_request.count、code_edit_tool.decision、active_time.total 等；trace 的 span 上带 `gen_ai.*` 属性（注意：它的 metrics 没用 gen_ai 前缀，gen_ai 只出现在 span 属性）。
- **成本**：`/usage`（原 `/cost`）本地按牌价计算；`--max-budget-usd` 给 print 模式设预算熔断。
- **headless 评测**：`claude -p "任务" --output-format json`（或 `stream-json`）拿结构化结果与成本元数据——这是把 Claude Code 当评测对象跑批的标准方式；`--bare` 去掉一切自定义保证确定性（官方说将来会成为 `-p` 默认）。
- **Claude Agent SDK**：把 Claude Code 的循环+工具+上下文管理开放成 Python/TS 库（`query()` 流式、hooks、子代理、权限回调 `canUseTool`、会话 resume/fork）——评测 harness 可以直接编程驾驶。

**codex（OpenAI Codex）**
- `codex exec --json` 输出 NDJSON 事件流（thread.started / turn.started / item.started / item.completed / turn.completed），item 类型包括 command_execution / agent_message / reasoning / file_change / mcp_tool_call / web_search / plan_update，`turn.completed` 带 token 明细（`{"input_tokens":24763,"cached_input_tokens":24448,...}`）——事件流即 trace。
- `--output-schema` 约束最终输出 JSON Schema（做结构化评测很方便）。
- 没有公开 eval harness（SWE-bench 数字出自内部 harness；Terminal-Bench 用外部 Harbor harness 跑 Codex）——面试可说"开源仓库里没有评测代码这件事本身也说明评测是各家内部能力"。

**平台生态**
- **OpenTelemetry GenAI 语义约定**（开放规范，Development 状态）：`gen_ai.operation.name`（chat / invoke_agent / execute_tool…）、`gen_ai.request.model`、`gen_ai.response.finish_reasons`、`gen_ai.usage.input_tokens / output_tokens / cache_read.input_tokens`、`gen_ai.tool.name / .call.arguments / .call.result`，指标如 `gen_ai.client.operation.duration`、`gen_ai.server.time_to_first_token`——厂商中立的 trace/metric 词表。
- **Langfuse**（开源可自部署）/ **LangSmith**（LangChain 系）/ **Braintrust**（"主动观测"，trace 上长评测）：三件事都做——trace 采集、数据集管理、在线/离线评测、prompt 版本管理、成本质量告警。
- **promptfoo**：开源 prompt 回归测试 CLI（YAML 配置 + 断言 + GitHub Action 门禁 + 红队插件）。

## 5.3 自测问题（对应原图三连问）

1. Prompt 改了一版，有没有 Regression？——答：评测集 + CI 门禁 + 版本化（promptfoo / Langfuse prompt labels）。
2. 失败到底发生在哪一步？——答：trace + 失败分类学（MAST 三类 14 模式）；把每条失败 trace 归因到 prompt/工具/上下文/模型。
3. 效果提升后 Cost 和 Latency 有没有一起爆掉？——答：token/cost/时延指标与质量指标同图监控；缓存命中率单列；用 effort 分级（简单任务降模型/降推理力度）做成本调度。

---

# 第 6 章 Framework

> 原文：最后再去学 Framework：LangChain / LangGraph / Claude Agent SDK / OpenAI Agents SDK……框架应该是：**把前面的概念映射成代码**，而不是反过来靠背框架 API 理解 Agent。

## 6.1 定位

把前五章的概念当作"词汇表"，框架只是每种词汇的具体拼法。看任何框架先问五个问题：循环在哪、状态存哪、怎么编排/移交、怎么持久化恢复、怎么评测观测。

## 6.2 主流框架对照

| 框架 | 核心抽象 | 概念映射 | 一句话定位 |
|---|---|---|---|
| **LangGraph** | `StateGraph`（节点返回状态增量，reducer 合并）+ 条件边 | 循环=节点回路；Handoff=`Command(goto=...)`；状态=checkpointer（Memory/Sqlite/Postgres，按 thread_id）；并行=`Send()` map-reduce；人审=`interrupt()`+`Command(resume=...)` | 低层编排框架+运行时，受 Pregel 启发；最接近"自己写循环"的框架 |
| **OpenAI Agents SDK** | `Agent` / `Runner` / **Handoff** / **Guardrail** 四原语 | 循环=Runner 跑到 final_output；Handoff=整段对话移交下一个 Agent；守卫=与主循环并行、fail fast；会话=Session 后端（SQLite/Redis/Mongo）；自带 tracing | Swarm 的生产版，轻；handoff 是全量移交流派代表 |
| **Claude Agent SDK** | Claude Code 的循环+工具+上下文管理，Python/TS 可编程 | 循环/工具/压缩=CC 同款；hooks/子代理/权限/技能全部开放；`settingSources` 控制 CLAUDE.md 装载 | "把一个成熟 harness 嵌进你的程序" |
| **Google ADK** | 多代理层级（LlmAgent + Sequential/Loop/Parallel 工作流代理） | 会话/状态/记忆三服务分离；artifacts；内置 eval 模块；A2A 协议跨语言互操作 | 企业全家桶，模型无关（Gemini/Claude/Ollama 均可） |
| **AutoGen → MS Agent Framework** | v0.4 事件驱动 actor 运行时（AgentRuntime，分布式 GrpcWorker） | GroupChat=多代理轮流发言；2025-10 与 Semantic Kernel 合并为 Agent Framework：图工作流+持久执行+HITL，MIT 协议 | 学术系多代理鼻祖转向企业框架 |
| **CrewAI** | 角色（role/goal/backstory）+ Task + Crew | 顺序/层级流程；Flows 做事件驱动骨架、Crew 只嵌在需要自治处；`max_iter` 默认 20（内置停止条件） | 角色扮演式多代理，上手最快 |
| **PydanticAI** | 类型化 `Agent[Deps, Output]` | 工具=`@agent.tool`（签名+docstring 即 schema）；output_type=Pydantic 模型、校验失败自动重试（反思的工程化）；持久执行支持 Temporal/DBOS/Prefect 等 8 引擎 | "FastAPI 哲学的 Agent 库"，类型安全 |
| **Vercel AI SDK** | `ToolLoopAgent`（循环+停止条件封装）/ `generateText` | 工具=zod schema + execute；`runtimeContext` 传递租户/凭证；`HarnessAgent` 可直接跑 Claude Code/Codex/pi 作为后端 | TS 全栈首选，UI 集成最好 |

## 6.3 选型建议

- **学习期**：先手写 100 行裸循环（第 2 章伪代码跑通），再用 LangGraph 或 OpenAI Agents SDK 重写一遍——理解每个抽象省掉了你什么。
- **做产品**：要深度定制 harness（权限/hooks/上下文策略）→ Claude Agent SDK 或 Vercel AI SDK 的 HarnessAgent；要复杂有状态图 + 人审 → LangGraph；轻量 handoff 多代理 → OpenAI Agents SDK。
- **反框架声音也要会讲**：pi 的立场——扩展机制（agent 给自己写工具）比预置框架更持久；" Prompts are code, .json/.md files are state"。面试官会欣赏你知道两边的论点。

---

# 第 7 章 学习路线自测与面试准备

## 7.1 六步路线（原文）+ 每步自测

```
LLM 基础
  ↓ 能说清：模型吃什么、吐什么、为什么不确定
Agent Loop / Orchestration
  ↓ 能手写循环；能答原图六问（何时调工具/结果为何回传/串并行/子代理/状态传递/workflow vs agent）
Context & Retrieval
  ↓ 能设计：大工具结果、长任务压缩、记忆分层、检索选型（RAG 只是解法之一）
Runtime / Harness
  ↓ 能回答：超时/副作用重试/断线/刷新恢复/子代理隔离/代码安全执行
Eval & Observability
  ↓ 能搭：评测集、pass^k、trace、失败分类、成本延迟监控、回归门禁
Framework
  ↓ 能把概念映射到 1~2 个框架的 API 上，并说清框架省掉了什么
```

## 7.2 面试原则（原文要点展开）

1. **简历写了什么，就一定要把什么吃透。**
   - 写了 RAG → 追问链：Embedding 选型、切块策略、Retrieval（混合检索）、Rerank、召回效果怎么量化（recall@k / 失败率）、什么时候 RAG 不是答案。
   - 写了 Multi-Agent → 追问链：Router 怎么实现、Handoff 传什么、Context 怎么隔离、Orchestration 怎么兜底、token 成本账。
2. **面试官真正想知道的不是你用了什么框架**，而是：
   - 你**为什么这么设计**（权衡：workflow vs agent、检索 vs agentic search、摘要式 vs 全量式 handoff）；
   - **生产环境真的出问题时，你知不知道问题在哪、该怎么把它救回来**（超时、幂等、断点恢复、沙箱逃逸、回归）。
3. 高引用一手材料清单（面试前精读）：Anthropic《Building effective agents》《How we built our multi-agent research system》《Effective context engineering for AI agents》《Writing effective tools for agents》《Demystifying evals for AI agents》、Manus《Context Engineering》、Claude Code 官方文档的 tools/context/permissions 章节、codex 官方 sandboxing/config 文档、pi 全系列博客。

---

# 附录 A：术语速查表

| 术语 | 一句话定义 |
|---|---|
| Token | 模型读写的最小单位；计费/限速/预算的计量 |
| Context Window | 单次推理可见 token 上限，系统提示+历史+工具+结果都占它 |
| Message Protocol | messages 数组 + role + content block 的 API 协议 |
| tool_use / tool_result | 模型的调用请求块 / 客户端回传的结果块，按 id 配对 |
| stop_reason | end_turn / tool_use / stop_sequence——循环出口的协议信号 |
| Sampling | 逐 token 概率采样；temperature/top_p 决定不确定性 |
| Structured Output | 用 schema/语法约束强制格式化输出 |
| ReAct | 思考-行动-观察交替的循环范式，已被现代协议内化 |
| Orchestrator-Workers | 主代理动态拆解派发、worker 执行、主代理汇总的模式 |
| Handoff | Agent 间移交控制权；摘要式（CC）vs 全量式（OA SDK） |
| Steering | 任务运行中插入新指令进入队列（Codex/pi 的术语） |
| Reflection | 模型自我审视改进；花 token 买质量 |
| Stop Condition | 最大轮数/预算/人工/自然结束等循环出口 |
| Context Engineering | 策划维护"最小高信号 token 集"的整个过程 |
| Context Rot | token 越多召回越差的注意力稀释现象 |
| Compaction | 临近窗口上限时把历史摘要化重启窗口 |
| Microcompact | 先清旧工具输出（占位符化），不够再整体摘要 |
| Progressive Disclosure | 只常驻索引/描述，用到才加载全文（Skills/MCP schema） |
| JIT Retrieval | 只持轻量标识符（路径/URL），运行时按需加载 |
| KV-cache 命中率 | 生产 Agent 第一指标；缓存价差 ~10 倍 |
| Append-only | 上下文只追加不修改，保缓存保信任（Manus） |
| Recitation | 持续重写 todo 把目标锚在上下文末端（Manus） |
| AGENTS.md / CLAUDE.md | 项目级常驻指令文件（Codex / CC 的发现层级） |
| Session Tree | 会话消息成树可分支导航（pi 的 JSONL） |
| Rollout | Codex 的会话 JSONL 持久化格式 |
| Sandbox | 受限执行环境：Seatbelt/bubblewrap/Landlock/容器 |
| Approval Policy | 工具执行前闸门策略（on-request/never/粒度化…） |
| Hooks | 生命周期钩子，可拦截/改写/注入（exit 2 阻断） |
| Idempotency Key | 幂等键，防重试/恢复时副作用重复 |
| Durable Execution | 步骤落库+重放的持久化执行（Temporal/DBOS/Inngest） |
| Rainbow Deployment | 新旧版本并行让进行中任务跑完的发布法 |
| Trajectory | 一次完整尝试的记录（输出+工具调用+推理+中间结果） |
| LLM-as-a-Judge | 模型按 rubric 打分；注意位置/冗长/自偏好偏差 |
| pass^k | 同任务跑 k 次全过的概率——一致性指标 |
| MAST | 多代理失败分类学：3 大类 14 小模式 |
| OTel GenAI | gen_ai.* 语义约定，厂商中立的观测词表 |
| pass@k / SWE-bench Verified / Terminal-Bench / tau-bench / OSWorld | 常用 Agent 基准 |

---

# 附录 B：参考资料

**官方文档**
- Claude Code：[how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works) · [tools-reference](https://code.claude.com/docs/en/tools-reference) · [sub-agents](https://code.claude.com/docs/en/sub-agents) · [context-window](https://code.claude.com/docs/en/context-window) · [memory](https://code.claude.com/docs/en/memory) · [skills](https://code.claude.com/docs/en/skills) · [permissions](https://code.claude.com/docs/en/permissions) · [hooks](https://code.claude.com/docs/en/hooks) · [sandboxing](https://code.claude.com/docs/en/sandboxing) · [checkpointing](https://code.claude.com/docs/en/checkpointing) · [errors](https://code.claude.com/docs/en/errors) · [monitoring-usage](https://code.claude.com/docs/en/monitoring-usage) · [headless](https://code.claude.com/docs/en/headless) · [agent-sdk](https://code.claude.com/docs/en/agent-sdk/overview) · [claude-directory](https://code.claude.com/docs/en/claude-directory)
- Codex：[CLI](https://learn.chatgpt.com/docs/codex/cli) · [sandboxing](https://learn.chatgpt.com/codex/sandboxing) · [agent-approvals-security](https://learn.chatgpt.com/codex/agent-approvals-security) · [config-reference](https://learn.chatgpt.com/docs/config-file/config-reference) · [agents-md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) · [non-interactive-mode](https://learn.chatgpt.com/docs/non-interactive-mode) · [cloud-environments](https://learn.chatgpt.com/codex/environments/cloud-environments) · [windows-sandbox](https://learn.chatgpt.com/codex/windows/windows-sandbox) · [openai/codex 仓库](https://github.com/openai/codex)
- pi：[pi.dev/docs](https://pi.dev/docs/latest/how-pi-works.md) · [sessions](https://pi.dev/docs/latest/sessions.md) · [extensions](https://pi.dev/docs/latest/extensions.md) · [skills](https://pi.dev/docs/latest/skills.md) · [mcp](https://pi.dev/docs/latest/mcp.md) · [rpc](https://pi.dev/docs/latest/rpc.md) · [sdk](https://pi.dev/docs/latest/sdk.md)
- Hermes Agent：[GitHub](https://github.com/NousResearch/hermes-agent) · [文档](https://hermes-agent.nousresearch.com/)；Hermes 工具调用格式：[Hermes-2-Pro 模型卡](https://huggingface.co/NousResearch/Hermes-2-Pro-Llama-3-8B)、[Hermes-Function-Calling](https://github.com/NousResearch/Hermes-Function-Calling)

**工程方法论（一手博客/论文）**
- Anthropic：[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)（2024-12）· [How we built our multi-agent research system](https://www.anthropic.com/engineering/built-multi-agent-research-system)（2025-06）· [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（2025-09）· [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)（2025-09）· [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)（2026-01）· [Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)（2024-09）· [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)（2025-09）
- Manus：[Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)（2025-07）
- Mario Zechner（pi）：[What I learned building an opinionated and minimal coding agent](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)（2025-11）· [What if you don't need MCP at all?](https://mariozechner.at/posts/2025-11-02-what-if-you-dont-need-mcp/)（2025-11）· [Armin is wrong and here's why](https://mariozechner.at/posts/2025-11-22-armin-is-wrong/) · [Year in Review 2025](https://mariozechner.at/posts/2025-12-22-year-in-review-2025/) · [I've sold out](https://mariozechner.at/posts/2026-04-08-ive-sold-out/)（2026-04）；Armin Ronacher：[pi 点评](https://lucumr.pocoo.org/2026/1/31/pi/)（2026-01）
- OpenAI：[Introducing Codex](https://openai.com/index/introducing-codex/)（2025-05）· [GPT-5-Codex](https://openai.com/index/gpt-5-codex/)（2025-09）· [GPT-5.1-Codex-Max](https://openai.com/index/gpt-5-1-codex-max/)（2025-11）· [OpenAI Agents SDK 文档](https://openai.github.io/openai-agents-python/)
- 论文：MemGPT（[arXiv 2310.08560](https://arxiv.org/abs/2310.08560)）· MT-Bench/LLM-as-judge（[arXiv 2306.05685](https://arxiv.org/abs/2306.05685)）· tau-bench（[arXiv 2406.12045](https://arxiv.org/abs/2406.12045)）· OSWorld（[arXiv 2404.07972](https://arxiv.org/abs/2404.07972)）· MAST 多代理失败分类（[arXiv 2503.13657](https://arxiv.org/abs/2503.13657)）· Hermes 4（[arXiv 2508.18255](https://arxiv.org/pdf/2508.18255)）
- 基准：[SWE-bench](https://www.swebench.com/) · [Terminal-Bench](https://www.tbench.ai/) · [OTel GenAI 语义约定](https://github.com/open-telemetry/semantic-conventions-genai)
- 框架文档：[LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) · [Google ADK](https://adk.dev/) · [AutoGen](https://microsoft.github.io/autogen/stable/) / [MS Agent Framework](https://microsoft.github.io/agent-framework/) · [CrewAI](https://docs.crewai.com/) · [PydanticAI](https://pydantic.dev/docs/ai/overview/) · [Vercel AI SDK](https://ai-sdk.dev/docs/introduction) · [promptfoo](https://www.promptfoo.dev/docs/intro/) · [Langfuse](https://langfuse.com/docs)

**非官方但高价值（注明为逆向/二手）**
- [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)（系统提示全文镜像）
- [openprogram.io：Claude Code 压缩机制逆向](https://openprogram.io/docs/reference/claude-code-compaction.html)
- 社区对 pi / hermes 的 star 数、LOC 数字等未经第一手验证，引用时注意。
