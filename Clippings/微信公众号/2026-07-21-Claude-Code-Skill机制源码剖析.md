---
created: 2026-07-21
updated: 2026-07-21
tags: [Claude Code, Skill, 源码分析, Agent, 渐进式披露, AI编程]
source:
  - "https://mp.weixin.qq.com/s/jH3xzCZW3keKOgTgA9sHkA"
  - "公众号：小林coding"
---

# Claude Code Skill 机制源码剖析

> **来源**：公众号「小林coding」2026-07-21  
> **核心主题**：Skill 不是普通的提示词，而是一套精密的「上下文懒加载系统」

---

## 面试题：Skill 和提示词有什么区别？

Skill 的载体确实就是几个 Markdown 文件，但面试官的追问才是关键：

> 「既然就是提示词，那我直接把内容塞进 system prompt，跟用 skill 有什么区别？」

**核心答案**：Skill 的内容确实就是提示词，但区别不在内容，在**加载方式**。

| 对比       | 提示词（直接塞）  | Skill（渐进式披露）     |
| -------- | --------- | ---------------- |
| **加载方式** | 一股脑全塞进上下文 | 分层按需加载           |
| **常驻内容** | 全文        | 只常驻一行描述（~250 字符） |
| **触发时机** | 始终存在      | 模型自己判断，用时才注入     |
| **扩展文档** | 无         | 第三层懒加载，用才读       |

---

## 一、Skill 到底是什么？

### 文件结构

```
pdf/
├── SKILL.md          # 主文件，必须叫这个名字
├── reference.md      # 进阶操作的详细文档
├── forms.md          # 表单填写的专项说明
└── scripts/
    ├── fill_form.py  # 现成的表单填写脚本
    └── ...
```

### SKILL.md 的核心格式

```markdown
---
name: pdf
description: 处理 PDF 文件的技能。当用户需要读取、
  生成、合并 PDF，或者填写 PDF 表单时使用。
---

# PDF 处理指南

读取 PDF 文本用 pdftotext 命令...
表单填写这种复杂操作，先阅读 forms.md...
```

- **frontmatter**：`name`（技能名）+ `description`（描述，模型选它的唯一依据）
- **正文**：操作指南，告诉模型用什么工具、遇到复杂情况去读哪个补充文档

> 打个比方：Skill 就是 Claude 的「入职手册」。模型能力没变，变的是手头多了一份你家业务的说明书。

---

## 二、五分钟写一个最小 Skill

### 三步搞定

```bash
# 第一步：建目录
mkdir -p ~/.claude/skills/style-check
```

创建 `~/.claude/skills/style-check/SKILL.md`：

```markdown
---
name: style-check
description: 检查代码是否符合团队风格规范。
  当用户要求检查代码风格、评审代码时使用。
---

# 代码风格检查清单

逐条检查以下规则，输出违反项和修改建议。

1. 变量名要有业务含义，不许用 a、b、tmp 这种
2. 魔法数字必须提成有名字的常量
3. 捕获异常不许静默吞掉，至少打一行日志
```

### 两种使用方式

| 方式      | 操作               | 说明                 |
| ------- | ---------------- | ------------------ |
| **手动挡** | 敲 `/style-check` | 你自己召唤              |
| **自动挡** | 说「帮我看看这段代码规范吗」   | 模型自己翻出 style-check |

---

## 三、渐进式披露（Progressive Disclosure）—— 核心机制

这是 Skill 系统的灵魂。Claude Code 把加载拆成三层：

### 第一层：常驻的只有「书脊」（~2000 token）

所有 skill 的**名字和描述**拼成清单，跟着 system-reminder 送进上下文。

```markdown
- style-check: 检查代码是否符合团队风格规范...
- pdf: 处理 PDF 文件的技能。当用户需要读取...
- commit: 根据暂存区变更生成规范的提交信息...
```

**预算控制**（源码中的关键常量）：

```typescript
// src/tools/SkillTool/prompt.ts
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01    // 整个清单只占上下文 1%
export const MAX_LISTING_DESC_CHARS = 250            // 单条描述硬上限 250 字符
```

**降级方案**：装两百个 skill 装不下时，先等比截短描述，最极端时第三方 skill 只留名字，官方内置 skill 永远不被截断。

### 第二层：选中了，才把整本书拿出来

模型判断需要某个 skill → 发起调用 → SKILL.md 完整正文以 **user message**（标记 `isMeta: true`）注入对话流。

```typescript
// src/tools/SkillTool/SkillTool.ts
newMessages: tagMessagesWithToolUseID(
  [createUserMessage({ content: finalContent, isMeta: true })],
  toolUseID,
),
```

> **为什么走 user message 而不是改 system prompt？**  
> 改动 system prompt 会导致提示词缓存大面积失效，每轮重新计费。追加一条消息是纯增量，缓存不伤。

### 第三层：附录，连第二层都懒得进

SKILL.md 里只写指路的话，更细的文档（如参考文档、脚本）等模型真正需要时才自己去 Read 进来。不需要任何专门机制，靠的就是 Claude 本来就会读文件。

### 效果对比

| 方案    |   50 个 skill 的开销    |
| ----- | :-----------------: |
| 全文常驻  |  ~15 万 token（75%）   |
| 渐进式披露 | **~2000 token（1%）** |

> **本质**：Skill 骨子里就是一套「上下文的懒加载系统」。

---

## 四、Claude 怎么知道该用哪个 Skill？

**答案：没有检索系统。全靠模型自己读。**

模型每轮生成回复前会通读上下文。清单就在里面，用户需求也在里面——这就是语言模型最擅长的阅读理解。

```typescript
// src/tools/SkillTool/prompt.ts
const desc = cmd.whenToUse
  ? `${cmd.description} - ${cmd.whenToUse}`
  : cmd.description
```

- `description`：说清「我是干什么的」
- `when_to_use`（可选）：补一句「什么时候该用我」

### 强制调用机制

```typescript
// 工具说明中的强硬指令：
// "When a skill matches the user's request, this is a
//  BLOCKING REQUIREMENT: invoke the relevant Skill tool
//  BEFORE generating any other response about the task"
```

有 skill 匹配就必须先调 skill，再干别的。

> **哲学**：能靠模型自身理解力解决的问题，绝不外挂一套额外系统。模型的阅读理解能力是随着模型升级白嫖变强的。

---

## 五、斜杠命令和 Skill 是同一个东西

```typescript
// 源码内部，slash command 和 skill 已经统一成了一套 Command 体系
// 用的同一个数据结构，加载和执行走同一条路
// "When users reference a 'slash command' or '/<something>',
//  they are referring to a skill."
```

### 两个入口 + 两个开关

| 入口     | 手动                            | 自动                                      |
| ------ | ----------------------------- | --------------------------------------- |
| **方式** | 敲 `/名字`                       | 模型自主匹配                                  |
| **开关** | `user-invocable: false` 从菜单隐藏 | `disable-model-invocation: true` 禁止模型调用 |

**应用场景举例：**
- `disable-model-invocation: true` → 「一键发布上线」skill，必须人扣扳机
- `user-invocable: false` → 「数据库表结构说明」这类纯背景知识，人敲没意义

---

## 六、动态内容：参数和命令执行

Skill 的正文不是写死的，有两个动态花样：

### 1. 参数替换

```markdown
# 代码风格检查

检查 $ARGUMENTS 文件
```
敲 `/style-check UserService.java` → `$ARGUMENTS` 被替换为 `UserService.java`

### 2. 命令预执行

```markdown
待检查的代码如下：

!`cat $ARGUMENTS`
```

调用时先执行 `cat` 命令，把输出填进正文再交给模型。

### 安全防线

```typescript
// src/skills/loadSkillsDir.ts
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(...)
}
```

来自 MCP 服务器的远程 skill **永远不执行**嵌入的命令。本地才执行——自己放的文件，出了事怪不着别人。

---

## 七、Skill 的五个来源

| 来源 | 路径 | 范围 |
|------|------|------|
| **官方内置** | Claude Code 自带 | 全局 |
| **用户级** | `~/.claude/skills/` | 全局 |
| **项目级** | `.claude/skills/` | 仅当前项目（会进 git）|
| **插件** | 随插件打包 | 全局 |
| **MCP** | 远程服务器提供 | 动态 |

> **项目级的亮点**：skill 跟代码一起进 git 仓库。新同事克隆项目，团队沉淀的所有操作手册自动就位。

**命名冲突**：按固定顺序扫描，先到先得。项目级排在用户级后面，所以不要在项目里试图覆盖用户级的同名 skill。

---

## 八、怎么写好一个 Skill？三条心法

### ① description 按广告词写
- 模型全靠它做匹配，进清单时最多留 250 字符
- ❌「这是一个功能强大的代码质量工具，支持多种语言」
- ✅「检查代码是否符合团队风格规范。当用户要求检查代码风格、评审代码时使用」

### ② SKILL.md 管今天，references 管细节
- SKILL.md 只写高频主干流程
- 低频/超长细节拆到 references 目录，正文里留指路的话
- **经验标准**：SKILL.md 超 500 行就该拆附录了

### ③ 重复劳动写成脚本
- 固定的机械操作（如格式转换）直接写脚本放 scripts 目录
- 模型现想有失败率，脚本跑一万次一个样
- 能固化成脚本的就别让模型现想

---

## 总结

> **Skill 内容确实就是提示词，但区别不在内容，在加载方式。  
> 提示词是知识本身，Skill 是知识的加载策略。**

| 维度       | 实现                             |
| -------- | ------------------------------ |
| **放得下**  | 三层渐进式披露，50 个 skill 常驻开销压到 1%   |
| **选得准**  | 一行 description + 模型阅读理解，不上检索系统 |
| **命令统一** | 斜杠命令和 skill 是同一个东西的两个入口        |
| **动态能力** | 参数插值 + 命令预执行，不只是一份静态文档         |

---

## 🔗 关联笔记

- [[领域/AI Agent 智能体学习路线 2026]] — Stage 6 涵盖 Skills 生态
- [[领域/Claude Code记忆系统与Agent记忆架构]] — Claude Code 记忆机制
- [[领域/Claude Code - Command与Skill区别]] — 对比分析
