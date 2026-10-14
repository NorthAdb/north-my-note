# Agent Eval：智能体评估体系、组成与主流框架

> [!summary] 核心结论
> **Agent Eval 不是某一个框架，而是一套用于验证 Agent 是否“正确、稳定、安全、高效”完成任务的工程体系。**
>
> 核心组成：
>
> **Dataset + Environment + Runner + Trace + Evaluator + Metrics + Regression + CI/Production Feedback**
>
> LangSmith、Promptfoo、Langfuse、Braintrust、Phoenix、Opik、DeepEval、Inspect AI 等，都是这套体系在不同层面的实现。

---

## 1. Agent Eval 是什么？

传统 LLM Eval 主要回答：

> **“模型回答得好不好？”**

典型流程：

```text
Input
  ↓
Prompt
  ↓
LLM
  ↓
Answer
  ↓
Score
```

而 Agent 是一个多步骤执行系统：

```text
User Task
    ↓
Agent
 ├── LLM Call
 ├── Tool Call
 ├── Tool Result
 ├── State Update
 ├── LLM Call
 ├── Retry
 └── Final Answer
```

因此 Agent Eval 不只评价最终答案，还评价：

- 是否完成任务
- 是否选择了正确工具
- 工具参数是否正确
- 工具调用顺序是否合理
- 是否出现无意义调用、循环、异常重试
- 是否正确修改系统状态
- 是否违反安全与权限策略
- Token、成本、延迟是否合理
- 多次执行是否稳定

> **Agent Eval = 对 Agent 整个执行过程和最终结果进行系统化评估。**

---

# 2. LLM Eval 与 Agent Eval

| 维度 | LLM Eval | Agent Eval |
|---|---|---|
| 输入 | Prompt | Task + Environment |
| 输出 | Text | Action Sequence + Final Output |
| 核心评价对象 | Response | Whole Execution |
| Tool Call | 通常没有 | 核心 |
| State | 通常不重要 | 核心 |
| Environment | 通常没有 | 核心 |
| Trajectory | 通常不关注 | 核心 |
| Judge | 常见 | 更复杂 |
| Reliability | 关注较少 | 非常重要 |
| Safety | 可能关注 | 核心 |
| Efficiency | Token / latency | Step / Tool / Token / Cost / Latency |

可以记成：

```text
LLM Eval
    ↓
回答好不好？

Agent Eval
    ↓
任务完成没有？
+
过程对不对？
+
工具调用对不对？
+
状态改对了吗？
+
安全吗？
+
稳定吗？
+
成本合理吗？
```

---

# 3. 为什么 Agent Eval 特别需要 Trace？

Agent 与普通 LLM 最大的区别之一，就是存在大量中间行为。

例如：

```text
用户：
“给 order_123 退款”

Agent：

1. LLM 分析任务
2. get_order(order_123)
3. get_user(user_456)
4. 检查用户身份
5. refund(order_123)
6. Final Answer
```

最终可能只返回：

```text
退款成功
```

但只看最终答案无法判断：

- 有没有验证用户身份？
- 有没有调用错误的订单？
- 有没有绕过权限？
- 有没有重复退款？
- 有没有进行无意义工具调用？
- 花了多少 Token？
- 是否经历了错误重试？

因此需要保存：

> **Trace = Agent 一次完整执行过程的结构化记录。**

---

# 4. Trace 的基本结构

```text
Trace
├── User Input
├── Agent Version
├── Model
├── Prompt Version
├── LLM Call
│   ├── Input
│   ├── Output
│   ├── Token Usage
│   └── Latency
├── Tool Call
│   ├── Tool Name
│   ├── Arguments
│   ├── Result
│   ├── Error
│   └── Latency
├── Retrieval
├── State Change
├── Retry
├── Final Output
├── Total Cost
└── Total Latency
```

> Trace 的重点是**记录 Agent 的外部执行行为和系统状态**，不等于记录模型内部 Chain-of-Thought。

对于 Agent Harness 来说，更重要的是：

```text
LLM
 ↓
Decision
 ↓
Tool
 ↓
Result
 ↓
State
 ↓
Next Decision
```

---

# 5. Agent Eval 的核心组成

## 5.1 Dataset：测试集

Dataset 是 Eval 的输入。

最简单：

```json
{
  "task": "查询订单123",
  "expected": "paid"
}
```

Agent Eval 往往需要更丰富的信息：

```json
{
  "task": "给订单123退款",

  "initial_state": {
    "order_status": "paid",
    "user_verified": true
  },

  "expected_state": {
    "order_status": "refunded"
  },

  "constraints": [
    "must_verify_user",
    "must_not_refund_twice"
  ]
}
```

因此 Agent Dataset 不一定只有：

```text
input
expected_output
```

还可能包含：

```text
task
initial_state
available_tools
expected_state
constraints
reference_trajectory
metadata
```

### Dataset 的来源

```text
人工编写
    ↓
历史生产 Trace
    ↓
失败案例
    ↓
用户反馈
    ↓
人工标注
    ↓
Regression Dataset
```

---

# 6. Environment：Agent 所处的环境

这是 Agent Eval 与普通 LLM Eval 的一个关键区别。

例如 Coding Agent：

```text
Agent
 ├── File System
 ├── Shell
 ├── Git
 ├── Network
 └── Package Manager
```

客服 Agent：

```text
Agent
 ├── CRM
 ├── Order DB
 ├── Payment API
 └── User Profile
```

Eval 需要能够控制这些环境。

例如：

```text
Case 1
order = paid

Case 2
order = already_refunded

Case 3
user = not_verified

Case 4
payment_api = timeout

Case 5
get_order = 500
```

然后观察 Agent：

```text
Tool Error
    ↓
Agent
 ├── Retry
 ├── Fallback
 ├── Stop
 └── Infinite Loop
```

这类测试可以验证：

> **Agent 的错误恢复能力。**

---

# 7. Runner：执行 Agent

Runner 负责真正把测试 Case 跑起来。

```text
Dataset
   ↓
Runner
   ↓
Reset Environment
   ↓
Run Agent
   ↓
Capture Trace
   ↓
Evaluator
   ↓
Score
```

伪代码：

```python
for testcase in dataset:
    reset_environment(testcase)

    trace = run_agent(
        task=testcase["task"],
        tools=tools,
        state=testcase["initial_state"]
    )

    score = evaluate(
        trace,
        testcase
    )

    save_result(trace, score)
```

Runner 本质上需要解决：

> **如何让被测试的 Agent 在可控、可重复的环境中执行。**

被测试对象通常称为：

> **SUT（System Under Test）**

---

# 8. Evaluator：如何判断 Agent 是否正确？

Evaluator 是 Agent Eval 的核心。

主要可以分成以下几类。

---

## 8.1 Code Evaluator

通过确定性代码判断。

```python
def evaluate(result):
    return result["order_status"] == "refunded"
```

可以检查：

```text
JSON 是否合法
Tool 是否正确
参数是否正确
文件是否存在
文件内容是否正确
数据库状态是否正确
Exit Code 是否为 0
```

基本原则：

> **能用代码判断的问题，优先不要用 LLM Judge。**

原因：

- 确定性高
- 成本低
- 可解释性强
- 适合 CI

---

## 8.2 Trajectory Evaluator

Agent 的执行轨迹：

```text
A → B → C → D
```

可以直接进行评估。

例如要求：

```text
get_user
    ↓
check_permission
    ↓
refund
```

实际却是：

```text
get_order
    ↓
refund
```

应该失败。

但通常不应该要求 Agent 100% 复刻某个专家轨迹，因为可能存在多个合理解法。

### Strict

完全匹配：

```text
Expected:
A → B → C

Actual:
A → B → C
```

### Subset

要求某些关键步骤必须出现：

```text
A、B 必须出现
```

实际允许：

```text
X → A → Y → B
```

### Superset

只要求实际轨迹包含最基本的必要步骤：

```text
A → B
```

实际可以：

```text
A → X → Y → B
```

核心问题：

> **哪些行为是必须的，哪些行为可以自由变化？**

---

## 8.3 LLM-as-a-Judge

对于难以用规则判断的问题：

```text
这个回答是否完整？
这个方案是否合理？
有没有明显幻觉？
工具使用是否适当？
是否真正解决了用户问题？
```

可以让另一个 LLM 作为 Judge。

```text
Agent Trace
    ↓
Judge LLM
    ↓
Rubric
    ↓
Score
```

例如：

```text
评分标准：

1. 是否正确完成任务？
2. 工具是否使用合理？
3. 是否存在未经授权的操作？
4. 最终回答是否准确？
```

输出可以是：

```json
{
  "score": 0.85,
  "pass": true,
  "reason": "..."
}
```

### 优点

- 灵活
- 能处理复杂语义
- 能评价开放式任务

### 缺点

- Judge 本身可能判断错误
- 成本更高
- 存在 Judge Bias
- 稳定性不如确定性代码

---

## 8.4 Agent-as-a-Judge

进一步发展：

```text
普通 LLM Judge

Trace
 ↓
LLM
 ↓
Score
```

变成：

```text
Evaluator Agent
 ├── 查看 Trace
 ├── 分析 Tool Call
 ├── 检查 State
 ├── 调用辅助工具
 └── 输出 Score
```

例如判断：

> “Agent 是否违反退款政策？”

Evaluator Agent 可能需要：

```text
读取 Trace
    ↓
找到 refund Tool Call
    ↓
查看用户身份
    ↓
查询退款政策
    ↓
检查订单状态
    ↓
综合判断
```

也就是说：

> **Evaluator 本身也是一个 Agent。**

---

## 8.5 Human Evaluation

自动 Eval 不是万能的。

典型流程：

```text
Automatic Eval
      ↓
发现疑难 Case
      ↓
Human Review
      ↓
人工标注
      ↓
加入 Dataset
```

尤其对于：

```text
高风险任务
安全
权限
金融
医疗
复杂业务规则
```

不能完全依赖自动 Judge。

---

# 9. Agent Eval 的主要评价维度

可以把核心指标记成：

```text
Outcome
Trajectory
Tool
State
Safety
Efficiency
Reliability
```

---

## 9.1 Outcome：任务成功率

最重要的指标。

```text
100 个 Task
82 个成功

Task Success Rate = 82%
```

---

## 9.2 Trajectory：路径质量

关注：

```text
Tool Selection
Tool Ordering
Planning
Unnecessary Steps
Loops
Retry
```

---

## 9.3 Tool Correctness：工具调用正确性

检查：

```text
Tool Name
Tool Arguments
Tool Order
Tool Result Handling
```

例如：

```json
{
  "tool": "refund_order",
  "order_id": "123"
}
```

检查：

```text
Tool 是否正确？
参数是否正确？
参数是否完整？
是否调用非法工具？
```

---

## 9.4 State Correctness：最终状态正确性

这是 Agent Eval 非常重要的指标。

例如：

```text
Agent：
“订单已经成功退款”
```

不能只看文本，还应该检查实际系统状态：

```sql
SELECT status
FROM orders
WHERE id = '123';
```

是否真的变成：

```text
refunded
```

因此：

> **State-based Evaluation 往往比只检查 Final Answer 更可靠。**

---

## 9.5 Safety / Policy

例如：

```text
未认证用户
    ↓
要求退款

Agent
    ↓
直接 refund
```

虽然最终回答：

```text
退款成功
```

但 Eval 应该：

```text
FAIL
```

原因：

```text
Policy Violation
```

常见安全评估包括：

```text
Prompt Injection
权限越权
Tool Misuse
敏感信息泄漏
危险操作
Policy Violation
```

---

## 9.6 Efficiency：效率

不仅判断：

```text
成功 / 失败
```

还要判断：

```text
Steps
Tool Calls
Token
Cost
Latency
Retries
```

例如：

| Agent | Success | Steps | Cost | Latency |
|---|---:|---:|---:|---:|
| A | ✅ | 6 | $0.03 | 4s |
| B | ✅ | 34 | $0.62 | 47s |

两者成功率一样，但 Agent A 更高效。

---

## 9.7 Reliability：稳定性

Agent 有明显的随机性。

一次成功：

```text
run 1 → success
```

并不能证明稳定。

应该多次运行：

```text
run 1 → success
run 2 → success
run 3 → fail
run 4 → success
run 5 → fail
```

关注：

```text
Pass Rate
Consistency
Pass@k
Pass^k
```

---

# 10. 一个完整的 Agent Eval 架构

```text
                    ┌───────────────┐
                    │    Dataset    │
                    │   Test Cases  │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Eval Runner   │
                    └───────┬───────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │      Agent       │
                  │ Prompt + Model   │
                  │ Harness + Tools  │
                  └────────┬─────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           LLM Call      Tool Call    State Change
              │            │            │
              └────────────┼────────────┘
                           ▼
                     ┌───────────┐
                     │   Trace   │
                     └─────┬─────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   Evaluators    │
                  ├─────────────────┤
                  │ Code            │
                  │ Trajectory      │
                  │ LLM Judge       │
                  │ Agent Judge     │
                  │ Human           │
                  └────────┬────────┘
                           │
                           ▼
                         Scores
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Analysis      Regression      CI Gate
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                      Production
                           │
                           ▼
                    New Failure Case
                           │
                           └────→ Dataset
```

最重要的闭环：

> **Production Failure → Dataset → Regression Test**

这意味着 Eval 不是一次性的测试，而是持续积累。

---

# 11. Agent Eval 与 Agent Harness 的关系

可以从架构层看：

```text
                 Agent System
                      │
       ┌──────────────┼───────────────┐
       │              │               │
     Runtime        Harness         Tools
       │              │               │
       └──────────────┼───────────────┘
                      │
                Observability
                      │
                     Trace
                      │
                  Agent Eval
                      │
             ┌────────┼────────┐
             │        │        │
          Dataset    Judge    Metrics
             │        │        │
             └────────┼────────┘
                      │
                    CI/CD
                      │
                  Production
                      │
                      └────→ Regression Dataset
```

可以简单理解：

```text
Agent Harness
= 负责让 Agent 执行

Agent Eval
= 负责判断 Agent 执行得对不对
```

二者通过：

```text
Trace
Events
Environment State
```

连接。

---

# 12. LangSmith 是什么位置？

LangSmith 可以理解为：

> **Agent / LLM Observability + Evaluation Platform**

```text
LangSmith
├── Tracing
├── Dataset
├── Evaluation
│   ├── Code Evaluator
│   ├── LLM Judge
│   ├── Human Evaluation
│   └── Trajectory Evaluation
├── Experiments
├── Comparison
├── Online Evaluation
└── Monitoring
```

它并不等于：

```text
Python Agent
    ↓
必须使用 LangSmith
```

LangSmith 同时支持 Python 和 TypeScript，也可以用于自定义 Agent，而不要求 Agent 一定基于 LangChain。

---

# 13. Promptfoo 是什么位置？

Promptfoo 更接近：

> **Evaluation / Testing / Assertion / Red Team Framework**

其思路很像：

```text
pytest
```

即：

```text
Test Case
    ↓
Run Target
    ↓
Assertion
    ↓
PASS / FAIL
```

例如：

```yaml
tests:
  - vars:
      input: "查询订单123"

    assert:
      - type: contains
      - type: is-json
      - type: javascript
      - type: llm-rubric
```

目前也已经覆盖：

```text
LLM Eval
RAG Eval
Agent Eval
Coding Agent Eval
Trajectory Eval
Security / Red Team
```

---

# 14. “Python → LangSmith，TS → Promptfoo”应该如何理解？

这种说法：

```text
Python → LangSmith
TypeScript → Promptfoo
```

**不是严格的技术绑定。**

更准确：

```text
                    Agent Eval
                        │
        ┌───────────────┴────────────────┐
        │                                │
 Platform-oriented                Test-oriented
        │                                │
   LangSmith                         Promptfoo
   Langfuse                          DeepEval
   Braintrust                        Inspect
   Phoenix                           ...
```

LangSmith：

```text
Python ✅
TypeScript ✅
```

Promptfoo：

```text
TypeScript / Node 原生 ✅
Python 集成 ✅
```

真正的区别不是：

```text
Python vs TypeScript
```

而更接近：

```text
Platform
vs
Testing Framework
```

---

# 15. 主流 Agent Eval 框架

| 框架 | 主要定位 | Agent Eval | Trace | Dataset | Judge | CI | 开源/自托管 |
|---|---|---:|---:|---:|---:|---:|---:|
| **LangSmith** | Observability + Eval Platform | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | 部分 |
| **Promptfoo** | Eval + Testing + Red Team | ★★★★★ | ★★★★ | ★★★★ | ★★★★★ | ★★★★★ | ✅ |
| **Langfuse** | Observability + Eval | ★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★ | ✅ |
| **Braintrust** | Eval + Observability | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | 部分 |
| **Arize Phoenix** | Open-source Observability + Eval | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | ★★★★ | ✅ |
| **Opik** | Open-source Eval + Tracing | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★ | ★★★★ | ✅ |
| **DeepEval** | Python Eval Framework | ★★★★★ | ★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | ✅ |
| **Inspect AI** | Research / Benchmark / Agent Eval | ★★★★★ | ★★★★ | ★★★★★ | ★★★★★ | ★★★★ | ✅ |
| **OpenAI Evals** | Eval Framework / Benchmark | ★★★ | ★★ | ★★★★★ | ★★★★ | ★★★ | ✅ |

---

# 16. Agent Eval Framework ≠ Agent Benchmark

这是另一个很容易混淆的概念。

## Agent Eval Framework

解决：

```text
如何组织实验？
如何运行 Agent？
如何记录 Trace？
如何评分？
如何比较版本？
如何进行 Regression？
```

例如：

```text
LangSmith
Promptfoo
Langfuse
DeepEval
Inspect
Opik
Phoenix
```

---

## Agent Benchmark

解决：

> **“拿什么题目测试 Agent？”**

例如：

```text
SWE-bench
WebArena
OSWorld
τ-Bench
AgentBench
```

所以：

```text
               Agent Evaluation
                       │
             ┌─────────┴──────────┐
             │                    │
       Evaluation Engine       Benchmark
             │                    │
      LangSmith              SWE-bench
      Promptfoo              WebArena
      Inspect                OSWorld
      DeepEval               τ-Bench
      ... 
```

Benchmark 更接近：

> **考试题库 + 环境**

Eval Framework 更接近：

> **考试系统 + 判卷系统 + 成绩系统**

---

# 17. 从 0 搭一个最小 Agent Eval

其实完全没必要一开始就上完整平台。

可以先自己实现一个最小版本：

```text
eval/
├── dataset.jsonl
├── runner.py
├── evaluator.py
├── metrics.py
└── results.jsonl
```

### dataset.jsonl

```json
{"task":"查询订单123","expected_status":"paid"}
{"task":"退款订单456","expected_status":"refunded"}
{"task":"查询订单789","expected_status":"cancelled"}
```

### runner.py

```python
for case in dataset:
    reset_env(case)

    trace = run_agent(case["task"])

    score = evaluate(
        trace,
        case
    )

    save_result(
        trace,
        score
    )
```

### evaluator.py

```python
def evaluate(trace, case):
    return {
        "task_success": check_final_state(trace, case),
        "tool_correct": check_tools(trace),
        "policy_ok": check_policy(trace),
        "steps": len(trace.steps),
    }
```

### metrics.py

```text
task_success_rate
tool_error_rate
policy_violation_rate
average_steps
average_latency
average_token_usage
average_cost
```

### CI

```text
Git Commit
    ↓
Run Eval
    ↓
100 Test Cases
    ↓
Calculate Metrics
    ↓
Compare Baseline
    ↓
PASS / FAIL
```

例如：

```text
Baseline Success Rate = 91%
New Version           = 87%

↓
Regression Detected
↓
CI FAIL
```

这就是基础的：

> **Agent Regression Testing**

---

# 18. 推荐学习路径

结合 Agent Harness / Pi / Claude Code / Codex 这类 Agent，可以按下面顺序。

## 第一阶段：手写最小 Eval

先不用大型框架。

理解：

```text
Dataset
Runner
Environment
Trace
Evaluator
Metrics
Regression
```

目标：

> 能完整解释一次 Agent Eval Run 到底发生了什么。

---

## 第二阶段：Promptfoo

重点研究：

```text
Test Case
Target
Assertion
LLM Judge
Trajectory
Red Team
CI
```

建立：

> **Eval 就是 Agent 的测试系统**

---

## 第三阶段：LangSmith / Langfuse

重点研究：

```text
Tracing
Dataset
Experiments
Evaluation
Online Evaluation
Human Feedback
Monitoring
```

建立：

> **Eval 不只是离线测试，也包括生产环境 Observability。**

---

## 第四阶段：Inspect AI / Benchmark

进一步研究：

```text
Benchmark
Environment
Simulator
Agent
Grader
Scorer
Leaderboard
```

进入 Agent Research / Benchmark 方向。

---

# 19. 最终心智模型

```text
                    Agent
                      │
                      ▼
                 Execution
                      │
                      ▼
                    Trace
                      │
                      ▼
                    Eval
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      Outcome     Trajectory      Safety
        │             │             │
        ▼             ▼             ▼
      State         Tools         Policy
        │             │             │
        └─────────────┼─────────────┘
                      ▼
                    Score
                      │
                      ▼
                 Regression
                      │
                      ▼
                     CI
                      │
                      ▼
                Production
                      │
                      ▼
                New Failures
                      │
                      └────→ Dataset
```

---

# 20. 一句话区分几个概念

| 概念 | 一句话理解 |
|---|---|
| **LLM Eval** | 模型回答得好不好 |
| **Agent Eval** | Agent 是否正确完成任务 |
| **Trace** | Agent 到底做了什么 |
| **Evaluator** | 如何判断它做得对不对 |
| **Dataset** | 拿什么题测试 |
| **Environment** | Agent 在什么世界里执行 |
| **Benchmark** | 一套标准化考试题 |
| **LangSmith** | Eval + Trace + Observability 平台 |
| **Promptfoo** | Eval + Testing + Assertion + Red Team |
| **Regression Eval** | 新版本有没有把旧能力搞坏 |
| **Online Eval** | 线上真实 Agent 是否持续表现良好 |

> **最终可以把 Agent Eval 理解成：给 Agent 建立一套类似软件工程中 `pytest + observability + CI/CD + benchmark` 的质量保障体系。**

---

## 21. 最值得记住的公式

```text
Agent Harness
= 负责执行

Agent Eval
= 负责验证

Trace
= 连接执行与验证的证据

Benchmark
= 标准化测试任务 / 环境

LangSmith / Promptfoo / Langfuse / ...
= 将上述能力工程化的工具或平台
```
