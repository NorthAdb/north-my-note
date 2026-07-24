---
created: 2026-07-06
updated: 2026-07-06
tags: [claude-code, memory, agent, context-engineering, AI]
source:
  - "https://docs.anthropic.com/en/docs/claude-code/memory"
  - "https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents"
---

# Claude Code 记忆系统与 Agent 记忆架构

> 数据来源：Anthropic 官方文档 + 工程博客
> 2026-07-06 整理

---

## 概述

Claude Code 有两套互补的记忆系统，共同解决"每次对话都是一张白纸"的问题：

```
Claude Code 记忆体系
├── 📝 CLAUDE.md 文件（手动编写）
│   ├── 你写的内容
│   ├── 项目规范、编码标准、工作流程
│   └── 跨会话持久化
│
├── 🤖 Auto Memory（自动记忆）
│   ├── Claude 自己写的笔记
│   ├── 构建命令、调试经验、架构决策
│   └── 跨会话自动累积
│
└── 🧠 Context Window（工作记忆）
    ├── 当前会话的上下文
    ├── 有限容量（但支持大窗口）
    └── 通过 /compact 压缩管理
```

---

## 第一部分：Claude Code Auto Memory 详解

### 1.1 什么是 Auto Memory

Auto Memory 是 Claude Code 的**自动记忆系统**（v2.1.59+）。Claude 在工作过程中自动记录有价值的信息，跨会话保留，无需你手动编写任何东西。

Claude 会记录的内容：
- **构建命令**：项目的构建、测试、部署命令
- **调试经验**：排查过的 Bug 和解决方案
- **架构决策**：代码组织方式、设计模式
- **代码风格偏好**：缩进、命名约定等
- **工作流习惯**：你偏好的操作方式

> Claude 不会每轮对话都保存东西，它只会在判断"这个信息对未来的对话有用"时写入记忆。

### 1.2 启用与关闭

默认**开启**。几种控制方式：

```bash
# 方式一：通过 /memory 命令在会话中切换
/memory → 点击 Auto Memory 开关

# 方式二：在 settings.json 中设置
{
  "autoMemoryEnabled": false
}

# 方式三：环境变量
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1
```

### 1.3 存储位置

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 索引文件，每次会话加载前 200 行/25KB
├── debugging.md       # 调试模式笔记
├── api-conventions.md # API 设计决策
└── ...                # Claude 创建的其他主题文件
```

关键特性：
- **按仓库隔离**：每个 git 仓库有独立的记忆目录
- **跨 worktree 共享**：同一仓库的所有 worktree 共享 auto memory
- **机器本地**：不跨机器同步
- **纯 Markdown**：可以手动编辑和删除

### 1.4 加载机制

```
会话启动时加载：
  MEMORY.md 的前 200 行 或 前 25KB（先到者）
  → CLAUDE.md 文件（全部加载）

会话运行中按需加载：
  debugging.md、api-conventions.md 等主题文件
  → Claude 用标准文件工具按需读取
```

加载阈值设计原因：MEMORY.md 是索引，保持精简。详细内容放到独立主题文件中，需要时才加载，不浪费上下文空间。

### 1.5 读写时机

当你在 Claude Code 界面中看到以下提示时，就是 Auto Memory 在工作：

| 提示                   | 含义              |
| -------------------- | --------------- |
| "Writing memory..."  | Claude 正在写入新记忆  |
| "Recalled memory..." | Claude 正在读取已有记忆 |

### 1.6 查看与编辑

```bash
/memory  # 列出所有已加载的记忆文件
```

你也可以直接打开 `~/.claude/projects/<project>/memory/` 目录，里面的文件都是纯 Markdown，可以随意编辑或删除。

---

## 第二部分：CLAUDE.md 文件体系

### 2.1 CLAUDE.md 的五层作用域

| 层级 | 位置 | 用途 | 共享范围 |
|------|------|------|---------|
| 🔒 **Managed Policy** | `/etc/claude-code/CLAUDE.md` | 组织级强制指令 | 全部用户 |
| 👤 **User** | `~/.claude/CLAUDE.md` | 个人全局偏好 | 仅自己（所有项目） |
| 📁 **Project** | `./CLAUDE.md` | 项目级规范 | 团队成员（版本控制） |
| 🔐 **Local** | `./CLAUDE.local.md` | 个人项目偏好 | 仅自己（需 .gitignore） |
| 📂 **Rules** | `.claude/rules/*.md` | 模块化/路径限定指令 | 按需加载 |

### 2.2 CLAUDE.md vs Auto Memory

| 维度   | CLAUDE.md   | Auto Memory  |
| ---- | ----------- | ------------ |
| 谁写的  | **你**       | **Claude**   |
| 内容   | 规则和指令       | 学习和模式        |
| 作用域  | 项目/用户/组织    | 每个仓库         |
| 加载方式 | 每次会话全部加载    | 前 200 行/25KB |
| 用途   | 编码规范、工作流、架构 | 构建命令、调试经验、偏好 |

### 2.3 目录加载顺序

```
从工作目录向上查找每个目录的 CLAUDE.md
加载顺序：根目录 → ... → 工作目录
同一目录下：CLAUDE.md 在前，CLAUDE.local.md 在后

子目录中的 CLAUDE.md：
  不预先加载
  Claude 读取该目录文件时才按需加载
```

### 2.4 Path-Scoped Rules（路径限定规则）

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 开发规则

- 所有 API 端点必须包含输入验证
- 使用标准错误响应格式
- 包含 OpenAPI 文档注释
```

仅在 Claude 操作匹配路径时加载，节省上下文空间。

### 2.5 /init 自动生成

```bash
/init  # 分析代码库，自动生成 CLAUDE.md
```

会读取已有的构建命令、测试配置、项目约定。如果 CLAUDE.md 已存在，则建议改进而非覆盖。

### 2.6 @import 引用机制

```markdown
参考 @README 了解项目概述
参考 @package.json 了解可用命令
```

支持递归导入，最深 4 层。导入的文件在启动时展开到上下文中。

### 2.7 AGENTS.md 兼容

如果项目已有 AGENTS.md（其他 AI 工具使用），可以：

```markdown
# CLAUDE.md
@AGENTS.md

## Claude Code 专属指令
- 对 src/billing/ 目录使用 plan 模式
```

或直接用软链接：`ln -s AGENTS.md CLAUDE.md`

---

## 第三部分：Context Engineering（上下文工程）

### 3.1 核心问题

LLM 的上下文窗口是**有限资源**：

- **Context Rot（上下文衰退）**：随 token 数增加，模型准确回忆信息的能力下降
- **注意力预算**：每个新 token 都在消耗模型的注意力资源
- **n² 复杂度**：Transformer 中每个 token 需要关注所有其他 token

### 3.2 有效上下文的原则

```
最少的 token + 最高信号 = 最佳结果
```

对每个上下文组件都要问：**"这个 token 对想要的结果有多大贡献？"**

### 3.3 系统提示词的最佳高度

```
过于笼统 ← 最佳平衡点 → 过于硬编码
     "请写得好的代码"          "if x > 5 then do y else do z"
         ↓                         ↓
    缺乏具体指引              脆弱、难以维护
```

最佳实践：
- 使用 XML 标签或 Markdown 标题分隔章节
- 从最小化提示开始测试，根据失败模式逐步添加
- 追求"刚好够用"的最小信息集

### 3.4 /compact 压缩机制

当上下文接近限制时：

```bash
/compact  # 压缩对话历史，保留关键信息
```

- 项目根目录的 CLAUDE.md 在 /compact 后**会自动重新加载**
- 嵌套子目录的 CLAUDE.md **不会自动重新加载**（需要重新读取该目录文件）
- 纯对话中的指令会丢失 → 重要指令应写入 CLAUDE.md

### 3.5 Prompt Caching

Claude Code 使用 prompt caching 优化：
- 系统提示词、CLAUDE.md、工具定义被缓存
- 后续轮次只需传递新消息
- 大幅降低延迟和成本

---

## 第四部分：Agent 记忆系统宏观设计

### 4.1 记忆的五层模型

AI Agent 的记忆系统通常分为五层：

```
人类记忆类比          Agent 记忆             技术实现
─────────────────────────────────────────────────────
🧠 工作记忆      ←  上下文窗口（Context）  ←  Transformer 的注意力机制
                                   
📝 短期记忆      ←  会话历史（Messages）  ←  消息列表 + 滑动窗口
                                   
📖 情景记忆      ←  事件记录（Episodic）  ←  时间索引的日志/数据库
                                   
📚 语义记忆      ←  知识存储（Semantic）  ←  向量数据库 + RAG
                                   
🔧 程序记忆      ←  技能/工具（Procedural）←  Skills 定义 + 工具调用
```

### 4.2 各层详解

#### ① 工作记忆（Context Window）
- **容量**：当前模型支持 100K~200K token
- **特点**：速度最快，但容量有限，存在 context rot
- **Claude Code 实现**：大型上下文窗口 + prompt caching + /compact

#### ② 会话历史（Message History）
- **容量**：整个对话的消息列表
- **特点**：线性增长，需要压缩机制
- **Claude Code 实现**：消息列表 + 自动 /compact 建议

#### ③ 情景记忆（Episodic Memory）
- **内容**：过去的事件、操作、结果
- **存储**：时间戳索引的结构化记录
- **代表项目**：Mem0、Letta（MemGPT）
- **Claude Code 实现**：Auto Memory 中的经验记录（如 debugging.md）

#### ④ 语义记忆（Semantic Memory）
- **内容**：事实、概念、知识
- **存储**：向量嵌入 + 知识图谱
- **检索**：RAG（检索增强生成）、Hybrid Search
- **代表项目**：cognee、MemPalace、GraphRAG

#### ⑤ 程序记忆（Procedural Memory）
- **内容**：技能、工具使用模式、工作流
- **存储**：Skills 定义、Tool 定义
- **Claude Code 实现**：Skills 系统（按需加载）、MCP 工具

### 4.3 主流 Agent 记忆框架对比

| 框架                          | 侧重      | 特点                    |
| --------------------------- | ------- | --------------------- |
| **Claude Code Auto Memory** | 工程实践    | Markdown 文件，简洁实用，机器本地 |
| **Mem0**                    | 长期记忆层   | 可插拔，支持 LLM 应用的长期记忆    |
| **Letta (MemGPT)**          | 虚拟上下文管理 | 分层记忆，自主管理上下文          |
| **LangGraph**               | 图状态记忆   | 基于图的 Agent 工作流状态管理    |
| **GraphRAG**                | 知识图谱检索  | Microsoft 出品，全局语义理解   |
| **cognee**                  | 端到端记忆平台 | 知识图谱 + 语义检索，开源自托管     |

### 4.4 2026 年趋势

1. **从无状态 RAG 到持续学习 Agent**
   - Agent 不仅能检索，还能写入更新记忆
   - 写回循环（Write-back loop）成为标配

2. **记忆评估标准化**
   - LOCOMO、LongMemEval 等基准测试
   - 遗忘机制和隐私控制

3. **多 Agent 共享记忆**
   - 跨 Agent 知识图谱
   - 企业级共享记忆系统

4. **知识图谱 + 向量检索融合**
   - 减少关系查询中的幻觉
   - 结构化与语义检索互补

5. **记忆合并与压缩**
   - 定期自动摘要、去重、重要性评分
   - 防止记忆存储膨胀

---

## 第五部分：实践建议

### 5.1 如何用好 Claude Code 的记忆系统

```
① 先用 /init 生成项目 CLAUDE.md
② 在对话中自然使用，让 Auto Memory 自动积累
③ 定期用 /memory 查看 Auto Memory 内容
④ 手动纠正 Auto Memory 中的错误记录
⑤ 重要规则直接写入 CLAUDE.md（比 Auto Memory 更可靠）
⑥ 当 CLAUDE.md 过大时，拆分为 .claude/rules/ 路径限定规则
```

### 5.2 哪些该放 CLAUDE.md，哪些该让 Auto Memory 处理

```
放入 CLAUDE.md：                           让 Auto Memory 处理：
├── 编码规范（缩进、命名）                  ├── 构建命令
├── 项目架构决策                            ├── 调试经验
├── 必须遵守的安全规则                      ├── 你纠正过的偏好
├── 测试和部署流程                          ├── 常用操作模式
└── API 设计约定                            └── 项目特有配置
```

### 5.3 记忆健康检查清单

- [ ] CLAUDE.md 是否在 200 行以内？
- [ ] 是否有冲突的指令？（同级 CLAUDE.md 互相矛盾）
- [ ] Auto Memory 是否记录了有价值的内容？
- [ ] 是否定期检查 /memory 中的内容？
- [ ] 旧规则是否已清理？

---

## 🔗 关联笔记

- [[Clippings/Bilibili/2026-07-04-我的B站AI编程收藏清单.md]] — 你收藏的 Claude Code 教程
- [[2026-07-04-GitHub高星项目速览]] — cognee、MemPalace 等记忆项目
- [[.claudian/claudian-settings.json]] — 你的 Claudian 配置中的记忆相关设置
