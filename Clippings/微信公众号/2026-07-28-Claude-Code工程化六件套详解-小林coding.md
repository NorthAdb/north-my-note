---
created: 2026-07-28
updated: 2026-07-28
tags: [Claude Code, CLAUDE.md, Skills, Subagents, MCP, Hooks, Plugins, 工程化, AI编程, 小林coding]
source:
  - "https://mp.weixin.qq.com/s/7wkfAeMrsnrmVVGYAvyBjA"
  - "公众号：小林coding"
---

# Claude Code 工程化六件套详解

> **作者**：小林coding
> **来源**：微信公众号
> **日期**：2026-07-28

!($baseUrl01-cover.png)

---

## 为什么需要工程化

大部分人的 Claude Code 用法还停留在「打开终端，敲需求，等它干完」。每次开新会话，都要把项目背景重新交代一遍——用什么框架、构建命令是什么、哪些文件不能动。Claude Code 其实提供了六样东西来解决这些问题。

Claude Code 的**六件套**：

| 解决的问题 | 对应的能力 |
|-----------|-----------|
| 每次会话都要重新交代项目背景 | **CLAUDE.md** |
| 专项知识和流程，用时才需要 | **Skills** |
| 中间过程太多，主对话被污染 | **Subagents** |
| 需要访问 GitHub、数据库等外部系统 | **MCP** |
| 规则必须百分百执行，不能靠自觉 | **Hooks** |
| 配置要跨项目复用、团队共享 | **Plugins** |

---

## 01｜CLAUDE.md — 项目记忆

### 为什么需要

大模型本身没有记忆，每次会话都是一张白纸。CLAUDE.md 相当于给新员工的**入职手册**——每次开新会话自动加载，塞进对话最前面。

### 存放层级

!($baseUrl03-folder-structure.png)

```
~/.claude/
└── CLAUDE.md          # 全局，所有项目都加载

my-project/
├── CLAUDE.md          # 项目级，启动就加载
├── web/
│   └── CLAUDE.md      # 动到 web 模块的文件才加载
└── core/
    └── CLAUDE.md      # 动到 core 模块才加载
```

- **全局配置** (`~/.claude/CLAUDE.md`)：个人偏好，如「回答用中文」「commit 信息别写太长」
- **项目根目录**：技术栈、命令、规范
- **子目录**：各模块独立规范，按需加载

> 除了手写的 CLAUDE.md，Claude Code 还有自动记忆机制——会在干活过程中自己记下经验，存入记忆目录，下次会话自动想起。

### 写法要点

!($baseUrl05-claudemd-example.png)

```markdown
# 项目说明
Spring Boot 服务，JDK 8，禁止用高版本语法。

## 常用命令
- 单测：mvn test -pl web
- 打包：mvn clean package -DskipTests

## 铁律
- core 模块是待下线的祖传代码，只读，不许改
- 表结构变更必须走 Flyway，不许手写 ALTER TABLE
```

- 命令要写**完整可复制执行**的
- 禁令要用**绝对语气**——「不许改」而非「尽量不要改」
- 只放**每次会话都用得上**的信息，不要堆砌完整文档
- CLAUDE.md 是入职手册，不是公司图书馆

---

## 02｜Skills — 专项知识库

### 设计思想

!($baseUrl07-skills-concept.png)

Skill 的核心在于**拆两层加载**：

- **启动时**：只加载所有 Skill 的「一句话描述」（极低成本）
- **用时**：任务匹配到描述后，才完整读取正文

```
code-review    审查代码改动时用，带团队检查清单
deploy-check   上线发布前用，带发布步骤和回滚预案
db-migrate     改表结构时用，带 Flyway 操作规范
```

三行描述几十个字，装二十个 Skill 常驻成本也就千把字，但每个 Skill 背后可以挂几千字的正文和参考文档。

### Skill 结构

`.claude/skills/code-review/SKILL.md`：

```markdown
---
name: code-review
description: 审查代码改动时使用，包含团队的检查清单和输出格式
---

审查改动时按以下重点检查：
1. 有没有绕过 service 层直接查库
2. 新接口有没有做参数校验
3. 错误处理是吞掉了还是往上抛了

审查结果按「问题、位置、建议」三栏输出。
```

!($baseUrl10-skill-example.png)

- `description` 是门面，Claude 靠它判断何时唤起 Skill——要写清楚**什么时候用**
- 可附带参考文档和脚本在 Skill 目录中
- 可直接用 `/skill-name` 斜杠命令强制唤起

---

## 03｜Subagents — 小团队分工

### 为什么需要

!($baseUrl11-subagents-concept.png)

主对话越用越笨是因为上下文被中间过程垃圾灌满（搜索文件、分析日志等）。Subagent 相当于**给主管派活的组员**：

- Subagent 在**独立上下文**里干活，搜文件、试错都污染不到主对话
- 干完后**只把汇总结果交回来**
- 支持**并行**执行，效率翻倍

### 配置方式

`.claude/agents/` 目录下，一个 markdown 文件就是一个 Subagent：

```markdown
---
name: code-reviewer
description: 代码改动的专项审查，检查安全、性能与规范
tools: Read, Grep, Glob
model: haiku
---

你是团队的代码审查员，只做审查不做修改。
逐个检查改动文件，重点看安全漏洞与性能隐患。
最终只输出问题列表与修改建议，不要贴大段代码原文。
```

- `tools` 字段限制权限（如审查 agent 不给写权限）
- `model` 指定模型（体力活用便宜快的，深度推理上贵的）
- 核心原则：**只输出提纯后的结论**，不贴大段代码

### 适用判断

| 适合拆 | 不适合拆 |
|--------|---------|
| 过程重、结论轻的任务 | 正在反复讨论的方案 |
| 大范围搜索、日志分析 | 需要完整上下文的任务 |
| 边界干净的专项审查 | 改到一半的需求 |

---

## 04｜MCP — 外部系统连接

### MCP 是什么

!($baseUrl16-mcp-concept.png)

MCP（Model Context Protocol）是 Anthropic 牵头的统一协议，给 AI 和外部工具之间定一个**统一插口**——类似 USB-C 标准。

### 架构

```
Claude Code (Client)  ←→  MCP Server (工具方)
```

Server 对外声明自己有哪些 **tools**（工具），每个工具带着名字、说明和参数定义。Claude 按需调用，Server 执行并返回结果。

### Skill vs MCP

!($baseUrl19-mcp-vs-skill.png)

| Skill | MCP |
|-------|-----|
| 给知识和流程（说明书） | 真正动手操作外部系统（手） |
| 告诉你怎么做 | 替你做 |

两者常配合使用：Skill 里写着「审查完提交到工单系统」是流程，真到提交那一步动手的是 MCP。

### 接入示例

GitHub MCP 接入：

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer 你的GitHub令牌"
```

接入后 Claude 可以直接拉取 PR 改动、结合 Skill 审查清单逐条过、把审查意见评论回 PR。

> 如果工具本身就有好用的命令行（如 git、gh），Claude 直接在终端敲命令即可，不一定非要 MCP。

---

## 05｜Hooks — 铁的纪律

### 为什么需要

!($baseUrl21-hooks-concept.png)

提示词（CLAUDE.md / Skill）本质上是**建议，不是法律**。模型是概率生成的，上下文一长规则就被稀释了。Hooks 是 **门禁系统**——在关键节点上挂 shell 命令，节点一到命令就跑，没有商量余地。

### 挂载点

Claude Code 会话生命周期中可挂 Hook 的关键事件：

- 会话启动
- 用户提交提示词
- **工具调用之前**（PreToolUse）— 拦截不合规操作
- **工具调用之后**（PostToolUse）— 自动善后
- Claude 准备结束回复

### 配置方式

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
      }]
    }]
  }
}
```

### 退出码含义

- `exit 0`：放行
- `exit 2`：拦截操作，报错信息回传给 Claude，它会自动调整做法

### 示例：自动格式化

文件被编辑后自动跑 prettier，无需在 CLAUDE.md 里再念叨格式化。

### 示例：敏感文件保护

`~/.claude/hooks/protect.sh`：

```bash
#!/bin/bash
file=$(jq -r '.tool_input.file_path')
if [[ "$file" == *".env"* || "$file" == *"secret"* ]]; then
  echo "敏感文件禁止修改" >&2
  exit 2
fi
```

效果：Claude 想动 .env 文件 → Hook 拦截 → Claude 收到报错 → 自动换方案。

---

## 06｜Plugins — 打包分发

### 为什么需要

!($baseUrl27-plugins-concept.png)

六个配置工具散落项目各处，手动复制导致版本混乱、团队对齐困难。Plugin 就是 **Claude Code 的包管理**——将 Skills、Subagents、Hooks、MCP 配置打包进一个盒子，发布出去，一条命令安装。

### Plugin 结构

```
my-review-kit/
├── .claude-plugin/plugin.json   # 名字、版本、描述
├── skills/        # Skill 们
├── agents/        # Subagent 们
├── hooks/         # Hook 配置
└── .mcp.json      # MCP 接入配置
```

没有新概念，Plugin 纯粹是**收纳盒**。

### 分发

基于 marketplace（可以是 git 仓库）：

```bash
/plugin marketplace add your-team/claude-plugins
/plugin install my-review-kit@team-marketplace
```

> **安全提醒**：Plugin 的 Hook 能在机器上执行任意命令，MCP 能连外部服务。装前看来源，团队内部或靠谱开源项目再装。

---

## 07｜六件套完整配合示例

!($baseUrl31-workflow-overview.png)

以一次**代码审查 + 修改**为例：

1. **会话开启** → CLAUDE.md 自动进场（项目规范就位）
2. 用户输入「审一下今天这个 PR」
3. **MCP** 把 PR 改动从 GitHub 拉下来
4. 审查工作派给 **code-reviewer Subagent**
5. Subagent 持有 **code-review Skill** 的团队清单，在独立上下文中逐文件审查
6. **主对话只收到**一份干净的审查结论
7. Claude 动手修问题 → **Hook** 自动格式化 + 拦截敏感文件操作
8. 审查意见通过 **MCP** 评论回 PR → 收工

> 全程用户体验：敲一句话 → 等一会儿 → 收到结果。背后六件套各司其职。

!($baseUrl34-summary-table.png)

---

## 核心洞见

> 模型决定 AI **「能」** 做到什么。能不能**「稳定」**做到，看你搭的脚手架。

六件套背后的思路全都不新鲜——入职手册、翻书查资料、团队分工、标准接口、门禁系统、包管理——全是软件工程和团队管理里玩了几十年的老经验。

> **AI 变强了，但让 AI 好好干活的方法，还是那些让人好好干活的方法。**

所以别光盯着模型跑分，回去把脚手架搭起来。工具明年可能又换一茬，但这个思路丢不了。

---

## 相关阅读

- [[2026-07-21-Claude-Code-Skill机制源码剖析]]
- [万字长文图解 Claude Code 入门](https://mp.weixin.qq.com/s/...)
- [万字长文图解 Claude Code 源码：Skill 机制](https://mp.weixin.qq.com/s/...)
- [万字长文图解 Claude Code 源码：多Agent实现机制](https://mp.weixin.qq.com/s/...)
- [万字长文图解 Claude Code 源码：上下文窗口管理机制](https://mp.weixin.qq.com/s/...)
