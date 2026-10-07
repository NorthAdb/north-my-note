---
title: Function Calling、MCP 与 CLI：Agent 工具调用的区别与联系
tags:
  - AI-Agent
  - Agent-Harness
  - Function-Calling
  - MCP
  - CLI
  - Tool-Calling
  - Architecture
---

# Function Calling、MCP 与 CLI：Agent 工具调用的区别与联系

## 一、核心结论

Function Calling、MCP、CLI 都可以用于**让 Agent 获得并使用外部能力**，但它们并不是同一层次、也不是简单的“谁替代谁”。

最重要的区分：

> **Function Calling：调用函数 / Tool**  
> **MCP：通过标准协议调用外部 Tool Server**  
> **CLI：通过命令行调用程序**  
> **Skill：告诉 Agent 什么时候用、怎么用某种能力**

而：

> **stdio / HTTP 不是另一种能力，而是 MCP Server 与 MCP Client 之间的传输方式。**

---

# 二、先统一：Agent Tool Calling 的基本闭环

无论使用哪一种方式，典型 Agent Loop 都可以抽象成：

```text
用户请求
   ↓
模型获得“可用能力”的描述
   ↓
模型判断是否需要调用能力
   ↓
生成调用声明
   ↓
Agent Harness / Client 执行
   ↓
得到结果
   ↓
结果进入上下文
   ↓
模型继续推理
```

三者真正不同的是中间的：

```text
“能力是什么？”
“能力在哪里？”
“如何发现？”
“如何调用？”
“谁负责执行？”
```

---

# 三、Function Calling

## 3.1 是什么

Function Calling 可以理解为：

> **Agent Harness 在本地定义一组 Tool，并把它们的名称、用途和参数 Schema 告诉模型；模型返回结构化调用声明，Harness 再执行对应函数。**

例如本地程序定义：

```python
def get_weather(city: str):
    ...
```

对模型暴露：

```json
{
  "name": "get_weather",
  "description": "Get weather information",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string"
      }
    }
  }
}
```

模型推理后返回：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Tokyo"
  }
}
```

然后本地 Harness：

```text
模型
 ↓
tool call
 ↓
Tool Registry
 ↓
get_weather(...)
 ↓
执行
 ↓
tool result
 ↓
模型
```

---

## 3.2 关键特点

### Tool 定义在哪里？

通常就在当前 Agent / Harness 的代码中：

```text
Agent
├── read_file()
├── search()
├── get_weather()
└── ...
```

### 谁执行？

**Agent Harness。**

模型通常不直接执行函数，只负责：

```text
选择哪个 Tool
+
生成什么参数
```

---

## 3.3 Function Calling 的本质

可以把它理解为：

```text
模型
  ↓
“我要调用 get_weather”
  ↓
Harness
  ↓
真正执行 get_weather()
```

所以 Function Calling 更偏向：

> **模型与当前程序之间的 Tool 调用接口。**

---

# 四、MCP

## 4.1 是什么

MCP（Model Context Protocol）可以理解为：

> **把外部能力以标准化协议暴露给 Agent Host。**

典型结构：

```text
Agent / Host
    │
    │ MCP Client
    ▼
MCP Server
    │
    ├── Tool A
    ├── Tool B
    └── Tool C
```

例如 GitHub MCP：

```text
Cursor
  │
  │ MCP
  ▼
GitHub MCP Server
  │
  ├── list_issues
  ├── get_issue
  ├── list_pull_requests
  └── ...
```

---

## 4.2 MCP 和 Function Calling 的关系

二者在“模型调用 Tool”这一层其实非常相似：

```text
Function Calling
模型 → Tool Name + JSON Arguments

MCP
模型 → Tool Name + Structured Arguments
        ↓
     MCP Client
        ↓
     MCP Server
        ↓
       Tool
```

因此可以把 MCP 理解成：

> **把“Tool”从当前 Agent 进程中抽离出来，通过标准协议连接外部 Tool Server。**

---

## 4.3 MCP 的一个关键变化

Function Calling：

```text
Agent
├── Tool A
├── Tool B
└── Tool C
```

MCP：

```text
Agent
├── MCP Server A
│    ├── Tool A1
│    └── Tool A2
│
└── MCP Server B
     ├── Tool B1
     └── Tool B2
```

因此 MCP 更强调：

- Tool Discovery
- 标准化协议
- 外部 Server
- 多客户端复用
- Tool / Resource / Prompt 等统一能力模型

---

## 4.4 一个重要修正

“模型先通过 MCP 客户端发现 MCP Server，然后把这个 Server 列表直接塞进 messages 里”这种说法不够准确。

更准确的是：

```text
MCP Client
   ↓
连接 MCP Server
   ↓
通过 MCP 协议发现 Tool
   ↓
拿到 Tool Schema
   ↓
Host 再把这些能力以模型 API 需要的形式提供给模型
```

具体落到模型 API 时，Tool 信息可能位于：

- `tools`
- system/developer context
- Host 自己构造的上下文

取决于具体 Agent Harness / SDK。

因此：

> **MCP 负责“发现与调用标准”；至于 Tool Schema 最终怎么进入模型上下文，是 Host 的实现细节。**

---

# 五、CLI

## 5.1 是什么

CLI（Command-Line Interface）本质上是：

> **一个支持命令行参数的程序接口。**

例如 GitHub CLI：

```bash
gh issue list
gh pr view 123
gh repo clone owner/repo
```

其调用方式可以概括为：

```text
命令
+
参数
+
flags
```

例如：

```bash
gh pr view 123 --json title,author
```

最终程序可以理解到类似：

```text
argv[0] = gh
argv[1] = pr
argv[2] = view
argv[3] = 123
argv[4] = --json
argv[5] = title,author
```

---

## 5.2 Agent 使用 CLI 时发生什么

典型流程：

```text
模型
  ↓
决定执行某个命令
  ↓
Shell / Terminal Tool
  ↓
Shell 启动 CLI 程序
  ↓
CLI 解析 argv
  ↓
执行
  ↓
stdout / stderr
  ↓
Agent
  ↓
模型继续推理
```

例如：

```text
Agent
 ↓
gh issue list --limit 10
 ↓
Shell
 ↓
gh.exe
 ↓
stdout
 ↓
Agent
```

---

## 5.3 一个重要修正

严格来说，通常不是：

> “模型生成一条 CLI 调用声明，本地程序再把它解析成结构化调用声明。”

更常见的是：

```text
模型
 ↓
生成命令文本
 ↓
Shell / Command Executor
 ↓
启动 CLI
 ↓
CLI 自己解析 argv
```

也就是说，模型面对的是：

> **命令行接口**

而不是：

> **一个标准化 Tool Schema**

---

# 六、三者的核心区别

## 6.1 一张总表

| 维度 | Function Calling | MCP | CLI |
|---|---|---|---|
| 本质 | Tool / Function 接口 | 标准化 Tool Server 协议 | 命令行程序接口 |
| 面向对象 | Agent Harness | Agent / Host | 人 / Shell / Script / Agent |
| 能力载体 | 函数 | MCP Server 中的 Tool | CLI 程序 |
| Tool 定义位置 | 当前 Agent 程序 | 外部 MCP Server | CLI 程序自身 + 文档 |
| 发现方式 | Tool Schema | MCP Tool Discovery | help / 文档 / Skill |
| 参数形式 | JSON | Structured JSON | command + argv |
| 执行主体 | Agent Harness | MCP Server | CLI 进程 |
| 返回形式 | Tool Result | MCP Result | stdout / stderr |
| 是否有统一协议 | Function Calling 协议由模型 API / SDK 定义 | MCP | 没有统一 CLI 协议 |
| Agent 原生适配 | 强 | 强 | 取决于 Shell 能力 |
| 人直接使用 | 一般 | 不方便 | 很强 |

---

# 七、最重要的不是“谁更强”，而是接口层不同

不要简单理解成：

```text
Function Calling < MCP < CLI
```

或者：

```text
CLI 被 MCP 替代
```

这都是不准确的。

更合理的是：

```text
                  Agent 能力
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
   Function Call     MCP         CLI
       │              │           │
    本地 Tool      外部 Tool     外部程序
       │              │           │
     函数          Server       Command
```

它们关注的问题不同：

- **Function Calling**：模型怎么调用当前程序里的 Tool？
- **MCP**：不同 Agent / Host 怎么标准化发现和调用外部 Tool？
- **CLI**：程序怎么通过命令行接受输入并输出结果？

---

# 八、GitHub：`gh CLI` vs GitHub MCP

这是理解三者最好的实际案例。

## 8.1 gh CLI

```text
Cursor
  ↓
Shell
  ↓
gh pr view 123
  ↓
gh.exe
  ↓
GitHub API
```

特点：

> Agent 会使用一个“命令”。

例如：

```bash
gh pr view 123
```

模型需要理解：

```text
gh
 → pr
 → view
 → 123
```

---

## 8.2 GitHub MCP

```text
Cursor
  ↓
MCP Client
  ↓
GitHub MCP Server
  ↓
GitHub API
```

模型看到的是类似：

```text
get_pull_request
list_pull_requests
get_issue
...
```

调用：

```json
{
  "name": "get_pull_request",
  "arguments": {
    "number": 123
  }
}
```

特点：

> Agent 会使用一个“结构化 Tool”。

---

## 8.3 两者功能可以高度重叠

例如：

```text
GitHub MCP
    ↓
get_issue(123)
```

完全可能等价于：

```bash
gh issue view 123
```

因此：

> **MCP 的价值不是“实现 CLI 做不到的功能”，而主要在于标准化的 Tool Discovery、结构化参数、Agent 友好的调用模型、工具级组织与权限控制。**

---

# 九、为什么很多人说“有 CLI 用 CLI”

这是有道理的。

例如：

```text
git status
git diff
git grep
gh issue list
gh pr checkout
docker ps
kubectl get pods
psql
```

这些程序本来就已经有非常成熟的 CLI。

对于 Agent：

```text
Agent
 ↓
Shell
 ↓
CLI
```

已经完全可用。

尤其在代码 Agent 中：

```text
git
npm
pnpm
docker
kubectl
gh
rg
jq
python
```

CLI 是非常自然的能力接口。

---

# 十、那为什么还需要 MCP？

当你开始拥有大量外部服务：

```text
GitHub
Slack
Notion
Jira
Linear
Postgres
Browser
...
```

全部通过：

```text
Shell
 ↓
CLI
```

会变成：

```text
Agent
 └── Shell
      ├── gh
      ├── slack-cli
      ├── jira-cli
      ├── ...
```

模型需要自己理解大量命令：

```text
命令树
+
flags
+
输出格式
+
错误信息
```

而 MCP 可以统一成：

```text
Agent
 ├── GitHub MCP
 │    ├── get_issue
 │    ├── create_issue
 │    └── ...
 │
 ├── Slack MCP
 │    ├── search_messages
 │    └── ...
 │
 └── DB MCP
      ├── query
      └── ...
```

于是 Agent 更容易进行：

```text
Tool Discovery
      ↓
Tool Selection
      ↓
Structured Arguments
      ↓
Tool Execution
```

---

# 十一、stdio / HTTP 到底属于哪里？

这个非常容易和 MCP 本身混淆。

## MCP 是协议层

```text
MCP
 │
 ├── Tool
 ├── Resource
 └── Prompt
```

## stdio / HTTP 是传输层

```text
MCP
 │
 └── Transport
      ├── stdio
      └── HTTP
```

所以：

```text
MCP ≠ stdio
MCP ≠ HTTP
```

而是：

```text
MCP
 └── 可以使用不同 Transport
```

---

# 十二、stdio MCP

Cursor 自己启动 MCP Server：

```text
Cursor
  │
  │ 启动子进程
  ▼
skillspector.exe mcp
  │
  │ stdin / stdout
  │ MCP messages
  ▼
Cursor
```

特点：

- 不需要手动启动端口
- Server 通常作为 Cursor 的子进程存在
- 生命周期通常由 Host 管理
- 非常适合本机 MCP

例如：

```json
{
  "mcpServers": {
    "skillspector": {
      "command": "skillspector",
      "args": ["mcp"]
    }
  }
}
```

---

# 十三、HTTP MCP

独立运行 Server：

```bash
skillspector mcp \
  --transport http \
  --host 127.0.0.1 \
  --port 8000
```

然后：

```text
Cursor
   │
   │ HTTP
   ▼
127.0.0.1:8000
   │
   ▼
SkillSpector Server
```

特点：

- Server 独立运行
- 有网络地址
- 可以由多个客户端连接
- 更适合远程服务或独立服务部署

---

# 十四、所以“本地运行”并不能区分 CLI 和 stdio

这是一个非常重要的结论。

两者都完全可以是本地进程：

```text
CLI:
Agent → Shell → gh.exe

MCP stdio:
Agent → MCP Client → github-mcp-server.exe
```

真正区别不是：

```text
本地 vs 非本地
```

而是：

```text
接口不同
```

CLI：

```text
命令 + argv
```

MCP stdio：

```text
MCP Protocol
+
stdin/stdout Transport
```

---

# 十五、一个更完整的四层模型

可以把整个体系理解成四层：

```text
                    Agent Harness
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
       Function         MCP            Shell
       Calling           │              │
          │              │              ▼
          │              │             CLI
          │              │
          │              ▼
          │        MCP Server
          │              │
          │        ┌─────┴─────┐
          │        ▼           ▼
          │      stdio        HTTP
          │
          ▼
    Local Tool Function
```

这里：

- **Function Calling**：Tool 调用接口
- **MCP**：外部 Tool 的标准协议
- **stdio / HTTP**：MCP 的传输机制
- **CLI**：程序自己的命令行接口

---

# 十六、Skill 又处在哪里？

Skill 最好不要和 MCP / CLI / Function Calling 放在完全同一层。

Skill 更偏向：

> **告诉模型“什么时候使用某个能力、怎么使用、执行步骤是什么”。**

例如：

```text
Skill
└── skill-inspector
    └── SKILL.md
```

可能写：

```text
在安装第三方 Skill 之前：

1. 使用 SkillSpector 扫描
2. 查看风险等级
3. 检查 findings
4. 高风险时不要安装
```

而真正执行扫描的可能是：

```text
CLI:
skillspector scan xxx
```

或者：

```text
MCP:
scan_skill(...)
```

因此：

```text
                   Agent
                     │
               Skill / Guidance
                     │
                     ▼
             “应该调用什么？”
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
 Function Call      MCP           CLI
      Tool         Tool         Command
```

---

# 十七、Skill + MCP + CLI 可以同时存在

以 SkillSpector 为例，可以同时拥有：

```text
                 Cursor Agent
                       │
                 skill-inspector
                     SKILL.md
                       │
               “先进行安全扫描”
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
        CLI                        MCP
skillspector scan          scan_skill(...)
          │                         │
          ▼                         ▼
    CLI Program               MCP Server
          │                         │
          └────────────┬────────────┘
                       ▼
                  Scan Engine
```

所以：

> **Skill 决定“怎么用”；MCP / CLI 决定“怎么调用能力”。**

---

# 十八、最终心智模型

## 一句话版

```text
Skill
= 告诉 Agent 怎么做

Function Calling
= 调用当前程序里的函数

MCP
= 通过标准协议调用外部 Tool Server

CLI
= 通过命令行调用程序

stdio / HTTP
= MCP Server 与 Client 的传输方式
```

## 按“调用对象”记忆

```text
Function Calling
    ↓
Function

MCP
    ↓
MCP Tool / Server

CLI
    ↓
Command / Program
```

## 按“调用形式”记忆

```text
Function Calling
    ↓
function_name + JSON arguments

MCP
    ↓
MCP request + structured arguments

CLI
    ↓
command + argv
```

## 按“执行者”记忆

```text
Function Calling
    → Agent Harness

MCP
    → MCP Server

CLI
    → CLI 子进程
```

---

# 十九、最值得记住的辨析

> **CLI 不等于低级版 MCP，MCP 也不等于高级版 CLI。**

二者可以实现大量相同的业务能力，例如：

```text
GitHub MCP → get_issue(123)

gh CLI → gh issue view 123
```

真正的区别是：

```text
CLI
= 程序接口
= 面向命令行

MCP
= Agent 工具协议
= 面向结构化 Tool Discovery / Tool Calling
```

所以在 Agent Harness 中，两者更像是：

```text
                 Agent Harness
                  /          \
                 /            \
            Shell              MCP Client
              │                    │
              ▼                    ▼
             CLI              MCP Server
```

它们都是 Agent 的“能力来源”，只是采用了不同的接口范式。
