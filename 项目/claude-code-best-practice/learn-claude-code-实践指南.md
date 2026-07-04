# learn-claude-code 实践指南

> 基于 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) (67.5k ⭐)
> 从零构建一个 Claude Code 式的 Agent Harness

---

## 1. 项目核心哲学

### 1.1 你要学的是什么

这个项目要教你构建的是 **Agent 的操作环境（Harness）**，而不是训练模型本身：

```
Agent = Model（LLM）+ Harness（工具 + 知识 + 观测 + 权限）

模型负责智能  →  通过训练获得
Harness 负责操作 →  通过代码构建
```

### 1.2 核心 Agent Loop（贯穿全部 20 课）

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

**这 20 行代码从不改变。** 每节课只在这个 Loop 外围加一个 Harness 机制。

### 1.3 Harness 工程师的真正工作

| 职责 | 说明 | 本课对应 |
|------|------|----------|
| **实现工具** | 文件读写、Shell、API 调用、数据库查询 | s01-s02 |
| **管理知识** | 文档、设计决策、风格指南，按需加载 | s07 |
| **管理上下文** | Subagent 隔离、压缩、任务持久化 | s06, s08, s12 |
| **控制权限** | 沙箱隔离、审批流程、信任边界 | s03 |
| **采集训练数据** | 每次执行的 action 序列 = 下一代模型的训练信号 | 全部课 |

---

## 2. 环境准备

### 2.1 基础依赖

```bash
# 克隆仓库
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code

# Python 3.10+ 即可
pip install -r requirements.txt

# 配置 API Key
cp .env.example .env
# 编辑 .env 填入:
# ANTHROPIC_API_KEY=sk-ant-xxxxx
```

### 2.2 API Key 获取

1. 访问 [console.anthropic.com](https://console.anthropic.com)
2. 注册/登录 → API Keys → Create Key
3. 充值 $20-$50 用于练习（s08 上下文压缩实验会消耗较多 token）

### 2.3 验证

```bash
python s01_agent_loop/code.py
# 如果 agent 成功回复了你的消息，环境就绪
```

### 2.4 阅读每课文件结构

```
s08_context_compact/
  README.md       # 中文完整教程（带 inline 代码）
  README.en.md    # 英文版
  code.py         # 独立可运行的 Python 实现
  images/         # SVG 图解（复杂课有）
```

**建议：** 先读 README.md，理解核心概念；再读 code.py，对照实现；最后自己跑一遍。

---

## 3. 21 天学习计划

### 总体节奏

```
第一阶段: 核心能力（s01-s06）→  5 天
第二阶段: 记忆与恢复（s07-s11）→  4 天
第三阶段: 长期任务（s12-s17）→  5 天
第四阶段: 整合扩展（s18-s20）→  4 天
总结复盘:                     →  3 天
```

---

## 4. 第一阶段：让 Agent 能动起来（s01 - s06）

### s01 — Agent Loop + Bash

**核心：** "One loop + Bash is all you need"

```bash
python s01_agent_loop/code.py
```

**理解要点：**
- `while True` 循环是 Agent 的心跳
- `stop_reason == "tool_use"` → 继续执行工具
- `stop_reason == "end_turn"` → 对话结束

**实践任务：**
- [ ] 给 agent 加一个自定义 Bash 命令（如 `cat`），验证它能在循环中多次调用
- [ ] 数一数一个"帮我创建一个文件并写入内容"的请求触发了多少次 tool_use

---

### s02 — Tool Use

**核心：** "Adding a tool means adding one handler"

```bash
python s02_tool_use/code.py
```

**理解要点：**
- `TOOL_HANDLERS` 是工具调度表（dispatch map）
- 新增工具 = 在表中新增一个键值对
- Loop 完全不变，工具可以任意扩展

**实践任务：**
- [ ] 新增一个 `weather(city)` 工具，用 mock 数据返回
- [ ] 尝试同时注册 5+ 个工具，观察 agent 如何选择调用

---

### s03 — Permission System

**核心：** "Set boundaries first, then grant freedom"

```bash
python s03_permission/code.py
```

**理解要点：**
- `PermissionRule` 定义允许/禁止/需审批的规则
- 权限在工具调用**之前**判定
- 可以基于正则匹配命令内容

**实践任务：**
- [ ] 设置规则禁止 `rm -rf /` 和 `sudo` 命令
- [ ] 实现需审批模式：危险命令弹出确认

---

### s04 — Hook System

**核心：** "Hook around the loop, never rewrite the loop"

```bash
python s04_hooks/code.py
```

**理解要点：**
- `PreToolUse` — 工具调用前触发
- `PostToolUse` — 工具调用后触发
- Hook 是扩展点，不修改 Loop 本身

**实践任务：**
- [ ] 在 PreToolUse 中记录每次调用的时间戳
- [ ] 在 PostToolUse 中记录工具执行耗时（ms）
- [ ] 实现一个 Hook 统计每种工具的调用次数

---

### s05 — TodoWrite

**核心：** "An agent without a plan drifts"

```bash
python s05_todo_write/code.py
```

**理解要点：**
- `TodoItem` 结构：标题 + 状态（pending/in_progress/completed）
- 先出计划，再逐条执行
- 完成率翻倍（实践数据支持）

**实践任务：**
- [ ] 给一个复杂任务，对比有 Todo 和无 Todo 的执行差异
- [ ] 修改代码让 agent 在每完成一项时报告进度百分比

---

### s06 — Subagent

**核心：** "Big tasks split small, each subtask gets clean context"

```bash
python s06_subagent/code.py
```

**理解要点：**
- 每个 Subagent 有全新的 `messages[]`
- 上下文隔离防止噪音传播
- Subagent 只返回结果摘要

**实践任务：**
- [ ] 创建 2 个 Subagent 并行跑不同任务
- [ ] 验证主 Agent 看不到 Subagent 内部的工具调用细节

---

## 5. 第二阶段：让 Agent 有记忆和韧性（s07 - s11）

### s07 — Skill Loading

**核心：** "Load knowledge on demand, not upfront"

```bash
python s07_skill_loading/code.py
```

**理解要点：**
- 启动时只加载 Skill 名称和描述（~100 tokens/skill）
- 需要时才展开完整内容
- 避免上下文初始化时的大量 token 消耗

**实践任务：**
- [ ] 创建 3 个 Skill 文件（git、docker、testing），观察按需加载时机
- [ ] 对比全量加载 vs 按需加载的 token 差异

---

### s08 — Context Compact

**核心：** "Context always fills up - have a way to make room"

```bash
python s08_context_compact/code.py
```

**理解要点：**
- snipCompact — 裁剪旧对话
- microCompact — 裁剪长工具输出
- toolResultBudget — 限制工具输出长度
- autoCompact — 自动触发上下文压缩

**实践任务：**
- [ ] 跑一个超长任务直到触发 autoCompact
- [ ] 调整 toolResultBudget 的值，观察效果变化
- [ ] 画出每次压缩前后的 token 数量变化

---

### s09 — Memory System

**核心：** "Remember what matters, forget what doesn't"

```bash
python s09_memory/code.py
```

**理解要点：**
- Selection（选择）— 判断什么值得记住
- Extraction（提取）— 从对话中提取关键信息
- Consolidation（固化）— 将多段记忆合并为长期记忆

**实践任务：**
- [ ] 跑两个相关任务，验证第二个任务能复用第一个的记忆
- [ ] 实现一个简单的文件存储后端替换内存后端

---

### s10 — System Prompt Assembly

**核心：** "Prompts are assembled at runtime, not hardcoded"

```bash
python s10_system_prompt/code.py
```

**理解要点：**
- System Prompt 是多个 section 的动态拼接
- 可以根据场景按需加载不同的 section
- sections = [base, tools_desc, skills_desc, memory, rules]

**实践任务：**
- [ ] 实现根据当前分支名切换 coding style section
- [ ] 实现根据任务类型（debug/feature/refactor）选择不同 prompt 模板

---

### s11 — Error Recovery

**核心：** "Errors aren't the end, they're the start of a retry"

```bash
python s11_error_recovery/code.py
```

**理解要点：**
- token 溢出 → 自动压缩上下文后重试
- 工具执行失败 → 回退到更保守的模型
- 多次失败 → 换一条执行路径

**实践任务：**
- [ ] 故意创造一个会导致 token 溢出的长对话，观察恢复
- [ ] 模拟工具调用连续失败 3 次，观察降级策略

---

## 6. 第三阶段：让 Agent 跑长任务和协作（s12 - s17）

### s12 — Task System

**核心：** "Big goals break into small tasks, ordered, persisted to disk"

```bash
python s12_task_system/code.py
```

**理解要点：**
- `TaskRecord` — 任务记录：id、描述、状态、依赖
- `blockedBy` — 任务依赖图
- 任务持久化到磁盘，会话中断后可恢复

**实践任务：**
- [ ] 创建 5 个有依赖关系的任务，让 agent 按拓扑排序执行
- [ ] 中断执行，重新启动后验证任务恢复

---

### s13 — Background Tasks

**核心：** "Slow ops go background, agent keeps thinking"

```bash
python s13_background_tasks/code.py
```

**理解要点：**
- 耗时操作丢到后台线程
- 完成后通过通知队列注入结果
- Agent 不阻塞等待

**实践任务：**
- [ ] 同时发起 3 个后台 npm install，观察通知回调

---

### s14 — Cron Scheduler

**核心：** "Fire on schedule, no human kick needed"

```bash
python s14_cron_scheduler/code.py
```

**理解要点：**
- 持久化调度（durable scheduling）
- 会话级触发器
- 无需人工干预

**实践任务：**
- [ ] 设置 agent 每 5 分钟自动检查一次 git 状态
- [ ] 实现"每天早上 9 点生成项目日报"的 cron 任务

---

### s15 - s17 — Agent Teams

**核心：** 多 Agent 协作的三部曲

```bash
python s15_agent_teams/code.py      # 基础团队 + 信箱
python s16_team_protocols/code.py   # 通信协议
python s17_autonomous_agents/code.py # 自主领任务
```

**s15 理解要点：**
- `MessageBus` — 团队消息总线
- inbox — 每个 agent 有独立信箱
- permission bubbling — 权限向上传播

**s16 理解要点：**
- 固定请求-回复格式
- shutdown handshake — 安全关闭握手
- plan approval — 计划审批流程

**s17 理解要点：**
- 无 Leader 分配，Agent 自主认领
- idle cycle — 空闲时主动看板
- 自组织团队

**实践任务：**
- [ ] s15: 创建 3 个 specialist agent + 1 个 lead agent
- [ ] s16: 实现 agent 之间通过协议协商任务优先级
- [ ] s17: 5 个 agent 从共享任务板中自主抢任务执行

---

## 7. 第四阶段：整合与扩展（s18 - s20）

### s18 — Worktree Isolation

```bash
python s18_worktree_isolation/code.py
```

**理解要点：**
- 每个团队 agent 在独立 git worktree 中运行
- `WorktreeRecord` 绑定 task ↔ directory
- 互不干扰的并行开发

**实践任务：**
- [ ] 创建 2 个 agent 在不同 worktree 中同时修改不同文件

---

### s19 — MCP Plugin

```bash
python s19_mcp_plugin/code.py
```

**理解要点：**
- 多传输协议（stdio / HTTP / WebSocket）
- 外部工具接入同一个工具池
- MCP = 能力扩展的标准接口

**实践任务：**
- [ ] 对接一个真实的 MCP server（如 Brave Search 或 filesystem）

---

### s20 — Comprehensive Agent

```bash
python s20_comprehensive/code.py
```

**理解要点：**
- 所有 19 课的机制汇聚到一个完整 Harness
- "Many mechanisms, one loop"
- 这是你的毕业设计起点

**实践任务：**
- [ ] 跑通 s20，确认所有机制正常工作
- [ ] 在此基础上添加一个你自己的机制

---

## 8. 毕业设计

### 8.1 选题方向

1. **你的专属 Coding Agent** — 基于 s20 的完整 Harness，加上你的定制 tools，做一个自己的 Claude Code
2. **特定领域的 Agent** — 如数据库运维 Agent（tools: SQL 执行、索引分析、慢查询诊断）
3. **多 Agent 协作系统** — 5+ Agent 团队处理复杂任务（如：架构师 + 前端 + 后端 + 测试 + 代码审查）

### 8.2 进阶项目

做完 20 课之后，shareAI 还有另外两个项目可以继续深入：

| 项目 | 说明 | 地址 |
|------|------|------|
| **claw0** | 从被动会话到常驻助手：心跳 + Cron + IM 多通道 | https://github.com/shareAI-lab/claw0 |
| **Kode-Agent CLI** | 可直接使用的开源 Coding Agent CLI | https://github.com/shareAI-lab/Kode-Agent |
| **Kode-Agent SDK** | 可嵌入任何应用的 Agent SDK | https://github.com/shareAI-lab/kode-agent-sdk |

---

## 9. 常见问题

### Q: 需要什么基础？

- 基础的 Python 编程能力
- 了解 LLM API 的基本调用方式
- 不需要机器学习/深度学习背景

### Q: 预计花费多少 API 费用？

- 完整跑完 20 课大约 $15-30（取决于实验次数）
- s08（上下文压缩）和 s15-s17（Agent Teams）消耗较多

### Q: 每课大概需要多长时间？

- 简单课（s01-s04）：30-60 分钟
- 中等课（s05-s08, s10-s11）：1-2 小时
- 复杂课（s09, s12-s20）：2-3 小时

### Q: 可以用其他模型替代 Claude 吗？

- 课程使用 Anthropic API，但核心设计思想通用
- 可以尝试对接 OpenAI API 或其他兼容接口

### Q: 学完能做什么？

- 理解 Claude Code / OpenCode / Codex 等 Agent 产品的内部机制
- 能够独立设计并实现一个自定义 Agent 框架
- 具备多 Agent 系统设计能力
- 为 Agent 相关岗位面试做好技术储备

---

## 10. 学习检查清单

### 第一阶段 ✅

- [ ] 能解释 Agent Loop 为什么是 `while True`
- [ ] 能给 agent 新增自定义工具
- [ ] 能实现基本的权限控制
- [ ] 理解 Hook 的触发时机
- [ ] 能设计 Todo-driven 的任务流程
- [ ] 能实现 Subagent 并保持上下文隔离

### 第二阶段 ✅

- [ ] 理解 Skill 的按需加载机制
- [ ] 掌握至少 2 种上下文压缩策略
- [ ] 能实现跨会话记忆系统
- [ ] 理解动态 prompt 组装
- [ ] 能实现 error recovery 的降级链条

### 第三阶段 ✅

- [ ] 能设计带依赖的任务系统
- [ ] 理解后台任务的通信模型
- [ ] 能设置 cron 定时任务
- [ ] 理解 Agent 团队的消息信箱机制
- [ ] 能实现自组织的 Agent 团队

### 第四阶段 ✅

- [ ] 理解 git worktree 隔离
- [ ] 能对接外部 MCP server
- [ ] 跑通 comprehensive agent
- [ ] 完成至少一个毕业设计项目

---

## 11. 姊妹项目对照

| 维度 | learn-claude-code | claw0 |
|------|-------------------|-------|
| 定位 | Harness 内部机制 | 常驻 Agent 机制 |
| 核心 | Loop / Tool / Plan / Team | Heartbeat / Cron / IM / Memory |
| 生命周期 | 一次性会话 | 永久运行 |
| 适合 | 理解 Agent 原理 | 构建个人 AI 助手 |

两套教程共享相同的 `agent_loop` 核心，是同一哲学的两个分支。

---

> **"Agency comes from the model. The harness gives agency a place to land."**
>
> **"Bash is all you need. Real agents are all the universe needs."**
