---
created: 2026-08-07
updated: 2026-08-07
platform: 小红书
author: 得物技术
note_id: 6a75842b000000000502ba6e
source: "https://www.xiaohongshu.com/explore/6a75842b000000000502ba6e?xsec_token=CBfrWN2qrOUiTFSsqviVzM_-svhAvbhHTzXMUTO0ZKWjk=&xsec_source=app_share"
likes: 7
collects: 8
comments: 1
image_count: 17
tags: [小红书, Coding Agent, Agent Loop, Violin, Zig, Python, Lua, TCP, JSONL, Agent工程]
---

# 实战：从零开始构建一个 Coding Agent——Violin

> **来源**：得物技术 · 小红书
> **主题**：从底层循环出发，用 Zig、Python 和 Lua 构建一个可运行的 Coding Agent。

## 一句话总结

> Coding Agent 没有神秘魔法，底层就是一个受约束的循环：**调用模型 → 判断是否需要工具 → 执行工具 → 把结果写回上下文 → 再次调用模型**，直到模型给出最终答案。

```text
用户任务
  → LLM
  → 是否调用工具？
       ├─ 否 → 返回最终答案
       └─ 是 → 执行工具 → 追加 ToolResult → 再次调用 LLM
```

## 1. 背景：为什么要自己写一个 Agent

从 Gemini CLI、Claude Code，到 Codex、OpenCode、Pi、Qoder，Coding Agent 在短时间内快速演化。各类客服、数据分析和工作流 Agent，追根溯源，都是 Coding Agent 核心循环在不同场景下的变体。

作者借鉴 Pi 的架构思想，用 Zig 实现一个名为 **Violin** 的 toy agent，目标不是立即交付商业产品，而是验证一个判断：

> 理解 Coding Agent 的构建原理，也就掌握了理解其他 Agent 的一把钥匙。

## 2. Violin 的整体架构

Violin 采用分层设计，核心模块彼此解耦：

```text
客户端（Python / 其他语言）
        │ TCP + JSON-lines
        ▼
服务端（Zig daemon）
  ├─ AI 模型适配层
  ├─ Agent Core：状态机、Agent Loop、EventBus、Tools
  ├─ Product 层：Session、Compaction、Resources
  └─ Plugin 层：Lua 扩展与拦截
```

### 2.1 为什么使用 Zig + Python + Lua

- **Zig**：实现 Agent Loop、模型适配、会话管理等底层引擎，追求性能、内存可控和较小的运行时。
- **Python**：实现客户端和终端交互，利用成熟生态快速搭建 TUI/CLI。
- **Lua**：作为约 500KB 的嵌入式插件运行时，动态加载、调用简单、适合做 Hook。
- **TCP + JSON-lines**：客户端与服务端通过语言无关的协议通信，未来可以接入 Web 或其他语言客户端。

这里的关键不是“全栈只用一种语言”，而是让每层使用最合适的语言。

## 3. Agent Core：循环本身

Agent Loop 只关心运行时闭环，不关心消息从哪里来、结果存到哪里。

### 3.1 核心伪代码

```zig
while (turn < max_turns) : (turn += 1) {
    const assistant = try model.complete(.{
        .messages = messages.items,
        .tools = tool_registry.definitions(),
    });

    const has_tool_calls = assistant.toolCalls().len > 0;
    if (!has_tool_calls) break;

    for (assistant.content) |block| {
        if (block == .tool_call) {
            const result = tool_registry.execute(block.name, block.args);
            messages.append(result); // 把工具结果写回上下文
        }
    }
}
```

必须明确处理三件事：

1. `max_turns`：防止模型陷入无限工具循环，是安全阀。
2. ToolResult 回写 `messages`：模型只有看到结果，才能决定下一步。
3. `tools` 传给 `complete()`：模型必须预先知道可用工具及其参数 Schema。

### 3.2 错误与重试

| 错误类型 | 处理位置 | 策略 | 说明 |
|---|---|---|---|
| 网络错误 | `retryComplete` | 指数退避 × 3 | 请求可能根本没有发出 |
| Context Overflow | `agent.zig` | 压缩后完整重试 | 不是普通 retry，而是 compact + restart |
| LLM 返回错误 | `retryComplete` | 指数退避 × 3 | 请求成功但内容异常 |
| 工具调用错误 | Tool 层 | 封装为 `ToolResult` 回喂模型 | 由模型自行决定如何修正 |

Agent Loop 只负责“循环”；历史加载、持久化、上下文压缩和客户端展示等“循环之外”的事情由上层模块承担。

## 4. AI 模型适配层：统一 Provider 差异

适配层的职责只有一句话：把不同 LLM Provider 的 API 差异封装在统一的 `Model.complete()` 接口后面。

```text
Agent Loop
    ↓
Model.complete(input)
    ↓
OpenAI / Anthropic / DashScope / Mock
```

统一接口至少需要：

```text
complete(input) -> AssistantMessage
name()          -> Provider 名称
deinit()        -> 释放资源
```

`CompleteInput` 包含：

- system prompt
- messages
- tools
- max tokens
- temperature
- stream callback

Violin 主要适配两种协议：

| 适配器 | 请求端点 | 响应格式 | 工具调用差异 |
|---|---|---|---|
| OpenAI | `/chat/completions` | Chat Completion | `tool_calls[]` |
| Anthropic | `/v1/messages` | Messages API | `content[]` 中的 `tool_use` block |

两者还存在请求字段（`messages[]` / `content[]`）和流式协议（SSE `data:` / `event:`）的差异。适配层把这些差异统一成 `Message`、`Tool`、`AssistantMessageEvent` 和 `streamSimple()`，上层 Loop 不需要知道具体 Provider。

模型配置从 `~/.violin/agent/models.json` 加载，Violin 不内置具体模型，由客户端决定使用哪个模型。例如：

```json
{
  "providers": {
    "openai": {
      "base_url": "https://api.openai.com/v1",
      "api_key": "$OPENAI_API_KEY",
      "models": [{"id": "gpt-4o", "contextWindow": 128000}]
    }
  }
}
```

## 5. Tool System：Agent 的手和脚

模型适配层让 Agent Loop 可以调用任意模型；工具系统则让 Agent Loop 能够执行动作。

### 5.1 工具定义

```zig
pub const Tool = struct {
    name: []const u8,
    description: []const u8,
    parameters: []const u8, // JSON Schema
    execute: ToolExecuteFn,
};
```

Violin 的基础工具包括：

| 工具 | 作用 |
|---|---|
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `edit_file` | 局部编辑文件 |
| `bash` | 执行 Bash 命令 |
| `ripgrep` | 正则搜索文件内容 |
| `fd` | 快速查找文件 |

工具注册表使用 HashMap 按名称查找工具，并通过 `definitions()` 将工具列表编码为模型可识别的 JSON Schema。

## 6. Product 层：把循环变成可用产品

Product 层主要处理四类事情：

| 文件 | 责任 |
|---|---|
| `agent.zig` | 组合 loop、session、compaction，形成可用 Agent |
| `session.zig` | 用 JSONL 持久化会话，支持断点续聊 |
| `compaction.zig` | 上下文超限时自动压缩，保留关键信息 |
| `resources.zig` | 加载 `AGENTS.md`、Skills 和项目扩展 |

### 6.1 `agent.zig`：胶水层

```text
加载 Session 历史
  → 传给 loop.run()
  → loop 返回后保存新消息
  → ContextOverflow 时调用 compaction
  → 压缩后重新执行 loop
```

### 6.2 `session.zig`：对话的记忆

会话以 JSONL 存储：第一行是会话头，后续每行是一条消息。

```json
{"id":"sess_001","created_at":1717234567,"cwd":"/project","model":"gpt-4o"}
{"id":1,"parent_id":null,"timestamp":1,"role":"user","content":"帮我读 README.md"}
{"id":2,"parent_id":1,"timestamp":2,"role":"assistant","content":"我来帮你读..."}
```

每条消息带有 `parent_id`，因此不仅能保存线性历史，也可以支持树形对话、分支、fork 和回滚。

`SessionStore` 需要维护：

- `file_path`：JSONL 文件路径
- `entries`：消息索引
- `leaf_id`：当前对话叶子节点
- `next_id`：下一个消息 ID
- `header`：会话元信息

实现上用 Zig 的 `ArenaAllocator` 统一管理内存，在一次会话加载完成后整体释放，避免逐条 `free` 的复杂性。

### 6.3 `compaction.zig`：对话的“脑容量管理”

当 token 超过阈值时，把旧消息压缩为摘要，保留最近消息，然后重新执行 Agent Loop：

```text
压缩前：[消息1] [消息2] ... [消息N-10] [消息N-9] ... [消息N]
压缩后：[摘要：之前讨论的要点] [消息N-9] ... [消息N]
```

图中给出的默认策略是：

- token 阈值：100K
- 保留最近：10 条消息
- 摘要目标长度：500 token

压缩分为两层：

| 函数 | 场景 | 效果 |
|---|---|---|
| `compactContext` | 运行时内存压缩 | 替换消息列表，继续当前会话 |
| `compactSession` | 持久化压缩 | 写入 JSONL，下次加载时生效 |

### 6.4 `resources.zig`：把规则和 Skills 注入模型

资源加载器从文件系统读取项目规则与技能：

- 项目规则：`AGENTS.md`、`CLAUDE.md`
- 全局规则：`~/.violin/agent/AGENTS.md`、`~/.violin/agent/CLAUDE.md`
- 项目 Skills：`{cwd}/.agent/skills/*/SKILL.md`、`{cwd}/.agents/skills/*/SKILL.md`
- 全局 Skills：`~/.violin/agent/skills/*/SKILL.md`

每个 `SKILL.md` 的 YAML frontmatter 会被解析，再组织成 XML 格式的可用技能清单，注入 system prompt，让模型知道有哪些能力可用。

## 7. Event System 与 Lua 插件

Agent Loop 在运行，但外界需要知道它运行到了哪一步；插件还需要在工具执行前拦截危险命令、在上下文中注入指令。EventBus 就是连接二者的基础设施。

每个重要动作都发布事件，例如：

- Agent 开始 / 结束一轮
- Session 创建、加载、持久化
- Context 压缩前后
- 工具调用开始 / 结束
- 流式 token 输出

### 7.1 为什么选择 Lua

| 方案 | 隔离性 | 性能 | 复杂度 |
|---|---:|---:|---:|
| 动态连接库 `.so` | 中 | 高 | 中 |
| WebAssembly 沙箱 | 高 | 中 | 高 |
| Lua 脚本 | 低 | 高 | 低 |

最终选择 Lua，不是因为它绝对最好，而是因为它体积小、嵌入成熟、调用路径短。对于 Zig toy agent，500KB 运行时和一个 C 函数调用就足够。

### 7.2 插件生命周期

```text
扫描 ~/.violin/plugins/*.lua
  → PluginLoader 加载脚本
  → LuaBridge 调用 load()
  → PluginRegistry 注册插件
  → EventBus 分发事件
  → 插件执行 hook
```

EventBus 为 `agent`、`session`、`compaction` 提供回调槽。`install()` 会保存原始回调，再包装成自己的 dispatch 函数：先执行原回调，再遍历已注册插件，调用对应 hook。

### 7.3 插件能做什么

例如 `bash-guard.lua` 可以：

- 在 `on_tool_start` 中拦截 `rm -rf` 等危险命令
- 修改命令，自动加入安全前缀
- 在 `on_tool_end` 中把错误结果改写成已处理状态
- 在 `on_context` 中向 system prompt 注入安全提示
- 在 `on_agent_start` 中阻止 Agent 启动
- 在 `on_session_before_compact` 中阻止手动压缩

> 图片同时暴露了一个尚未解决的边界：插件目前缺少权限隔离，Lua 理论上可以做任何事情。

## 8. 网络层：客户端与服务端分离

Violin 采用 C/S 架构：

```text
Python TUI Client
      │ TCP / JSON-lines，端口 9877
      ▼
Zig Server Daemon
  ├─ Command & Session Manager
  ├─ Skills & Tools Orchestrator
  ├─ Model Gateway
  ├─ Streaming Engine
  └─ Session Store
```

选择 C/S 而不是把所有东西放在一个 CLI 进程中，主要有三点好处：

1. 服务端负责重计算、工具、模型和会话，客户端专注交互。
2. 客户端语言无关，Python、Web 或其他语言都可以接入。
3. 服务端可以长期运行，客户端可以断开后重新连接。

### 8.1 握手

```json
{"type":"handshake","cwd":"/home/user/project/violin"}
```

服务端返回模型列表和技能列表：

```json
{"type":"models_result","models":[...],"default":"deepseek-v4-flash"}
{"type":"skills_result","global_skills":[...],"project_skills":[...]}
```

`cwd` 用于加载项目级 Skills 和系统提示词。

### 8.2 聊天请求与事件流

客户端发送：

```json
{
  "type": "chat",
  "content": "列出目录下文件",
  "model": "deepseek-v4-flash",
  "session_id": "sess_001",
  "temperature": 0.2,
  "max_tokens": 4096,
  "system_prompt": "...",
  "images": []
}
```

服务端流式返回 8 类事件：

| 事件 | 含义 |
|---|---|
| `turn_start` | 新一轮开始，携带模型与会话信息 |
| `delta` | 增量文本或 token |
| `tool_start` | 工具开始执行 |
| `tool_end` | 工具执行完成 |
| `turn_end` | 一轮结束 |
| `result` | 最终结果和 token 用量 |
| `error` | 错误信息 |
| `abort` | 用户取消当前任务 |

此外还有 `ping/pong` 心跳消息。

## 9. 目前的边界与未完成事项

图文总结中明确指出，这个项目距离成熟 Coding Agent 还有一些坑：

- `buildJson` 中的 tools 参数还没有序列化完整，模型可能收不到工具定义。
- 插件缺少权限隔离，Lua 目前拥有过大的能力范围。
- ACP 协议还没有接入，暂时只能自定义协议。
- 这是一个 toy agent，不以商业交付为目标。

这些缺口并不削弱项目的学习价值，反而清晰地标出了一个 Agent 从“能跑”走向“可用、可控、可扩展”时需要继续补齐的工程问题。

## 10. 我的理解

这篇图文最值得记住的不是某一段 Zig 代码，而是**职责边界**：

| 模块 | 核心问题 |
|---|---|
| Model Adapter | 问谁、如何统一不同模型协议 |
| Agent Core | 什么时候继续循环、什么时候停止 |
| Tool System | Agent 能做什么 |
| Session | 如何记住和恢复对话 |
| Compaction | 上下文太长时保留什么 |
| Resources | 如何把规则和 Skills 注入模型 |
| EventBus / Plugin | 如何扩展、观察和拦截运行时 |
| Network | 客户端与 Agent 引擎如何解耦 |

可以把它抽象成一个最小可行 Agent 的演进路径：

```text
模型调用
  → Agent Loop
  → Tool Calling
  → 错误重试与 max_turns
  → Session 持久化
  → Context Compaction
  → Resources / Skills
  → EventBus / Plugin
  → C/S 协议与多客户端
```

这与 [[领域/AI Agent 智能体学习路线 2026]] 中“先理解 Agent Loop，再学习 Tool、Memory、Harness 和生产部署”的路径一致，也可以和 [[2026-07-30-AI-Agent落地铁三角-Graph-Loop-Harness]] 放在一起理解：Violin 主要展示的是单个 Agent 内部的 **Loop + Harness 基础设施**，Graph 则是更上层的任务编排问题。

## 原图（已下载到 vault）

### 1. 封面

![封面](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-02.webp)

### 2. 背景：Coding Agent 的兴起

![背景：Coding Agent 的兴起](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-03.webp)

### 3. 效果预览与整体架构

![效果预览与整体架构](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-04.webp)

![整体架构续](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-05.webp)

### 4. 为什么选择 Zig 与分层设计

![为什么选择 Zig](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-06.webp)

### 5. Agent Loop

![Agent Loop](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-07.webp)

### 6. AI 模型适配层

![AI 模型适配层](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-08.webp)

### 7. Tool System 与 Product 层

![Tool System 与 Product 层](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-09.webp)

### 8. Agent、Session 与 Compaction

![Agent、Session 与 Compaction](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-10.webp)

![SessionStore 与 Compaction](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-11.webp)

![Compaction 与 Resources](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-12.webp)

### 9. Resources 与 Event System

![Resources 与 Event System](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-13.webp)

### 10. Lua 插件

![Lua 插件架构](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-14.webp)

![Lua 插件与网络层](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-15.webp)

### 11. 网络层、握手与 Python Client

![网络层](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-16.webp)

![握手、事件流与 Python Client](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-17.webp)

### 12. 总结

![总结](https://gitee.com/cheng-jiaqing/images/raw/master/coding-agent-01.webp)

## 原文标签

#AI新手村 #人工智能 #AI编程 #程序员 #大模型 #AI工具 #干货分享 #科技 #程序员日常 #得物技术
