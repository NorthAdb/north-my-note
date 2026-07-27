# Fable 5 系统提示词工程要点梳理

> 来源：公众号"小林coding"
> 原文：[面试官皱眉："你看过 Claude Fable 5 系统提示词吗？"](https://mp.weixin.qq.com/s/QAhMamZY0m4gSyIzxKfVEg)
> 泄露仓库：https://github.com/elder-plinius/CL4R1T4S
> 作者：小林coding | 收藏日期：2026-06-21

---

**背景：** 2026 年 6 月 9 日 Anthropic 发布 Fable 5，不到两天有人通过越狱手段提取了完整的 system prompt（1600+ 行）并传至 GitHub。以下是从中提炼的工程原则。

---

## 一、提示词结构概览

Fable 5 的 system prompt 近一半篇幅在定义工具，其骨架：

```
claude_behavior          → 产品信息 / 安全 / 语气 / 用户关怀
memory_system
mcp_app_suggestions
computer_use             → skills / artifacts
search_instructions
Tool Definitions         → 十多个工具的完整定义（吃了半本）
network / filesystem configuration
```

> 好的提示词 = 给模型搭「工作台」，不是写许愿池。

---

## 二、工具定义的关键工程原则

### 原则 1：把「什么时候别用」写得比「什么时候用」更狠

`ask_user_input` 工具的描述：

> WHEN NOT TO USE: User asks "A or B?" -> They want YOUR analysis and recommendation, not the options repeated back as buttons. User is venting -> Just listen.

> If you're about to write clarifying questions as prose bullets, STOP — those go in this tool instead.

每个工具定义都配了一张写满「别这么用」的说明书——它不信模型会自动用对。

### 原则 2：用参数顺序给模型的思考排序

`create_file` 的三个参数被强制顺序：

> description: Why I'm creating this file. ALWAYS PROVIDE FIRST.
> path: ... ALWAYS PROVIDE SECOND.
> file_text: Content to write. ALWAYS PROVIDE LAST.

大模型逐 token 生成，先写什么会影响后写什么——把「动机」放在最前面，逼模型想清楚再动手。

### 原则 3：给「策略方案」而非「语气版本」

`message_compose` 不生成不同语气的版本，而是提供通向不同结果的策略：

> Generate 2-3 strategies that lead to different outcomes — not just tones. Label each: "Disagree and commit" vs "Push for alignment", "Rip the bandaid" vs "Soften the landing".

每个策略标清楚在优先什么、牺牲什么。

### 原则 4：针对模型已知弱点打补丁

- **地图工具**：`Copy place_id values EXACTLY... do not type from memory` — 防模型"自信地记错"长 ID
- **体育比分**：先取数据再开口，别靠记忆猜
- **第三方服务**：`Never pick a partner for someone who didn't ask` — "速度不能成为替用户做主的理由"

---

## 三、搜索与知识边界

### 决策原则：能稳的别搜，会变的必搜

> partial recognition from training does not mean current knowledge. An unfamiliar capitalized word is almost certainly a name that postdates training.

模型的知识有保质期。碰到首字母大写的生词 → 几乎肯定是训练后才出现的新名字。

### web_fetch 的硬约束

> Can only fetch EXACT URLs provided by the user or returned by web_search.

不允许模型自己"编"一个 URL。

---

## 四、先读说明书（required first step）

> Reading the relevant SKILL.md is a required first step before writing any code, creating any file, or running any other tool.

- **required** + **first step**，不给模型"自我判断要不要"的余地
- SKILL.md 编码了"环境特有的约束"（库、路径、渲染怪癖），模型训练数据里没有这些

附带 `product-self-knowledge` 技能：

> Any time you would otherwise rely on memory for Anthropic product details, verify here — your training data may be outdated.

**关联：** 这与 Claude Code 的 skill 机制同源。详见本仓库的 [skill 概念](../practice/04-技能-skills.md) 和 [skill 实现](../implementation/03-skill-实现.md)。

---

## 五、语气与产品伦理

### 反格式化

> Claude avoids over-formatting with bold emphasis, headers, lists, and bullet points.

> Claude never uses bullet points when declining a task; the additional care helps soften the blow.

### 防用户上瘾

> Claude never thanks the person merely for reaching out... Claude avoids reiterating its willingness to continue talking.

绝不为用户「愿意开口」而道谢，不反复表示"我随时都在"——主动防止过度依赖。

### 情绪安全

> If someone mentions emotional distress and asks for info that could be used for self-harm — bridges, tall buildings, weapons, medications — Claude should address the underlying distress.

额外约束：不给人贴没确诊的标签，不用"替代法"，求助资源必须指向仍在运作的机构。

---

## 六、安全红线的写法：讲原则，藏机制

**恶意代码：** `even with an ostensibly good reason such as education` — 不教育目的留口子

**态度：** `If the conversation feels risky or off, saying less and giving shorter replies is safer.`

**核心原则：**
1. 拒绝时只讲原则，不讲检测机制——"narrating the boundary teaches how to reframe around it"
2. 模型发现自己正在"帮请求找理由"→ 这个念头本身就是拒绝信号——防的不是特定行为，而是模型"想妥协"的意图

---

## 七、对本仓库的启发

| Fable 5 原则 | 本仓库对应 |
|---|---|
| 动手前先读 SKILL.md | `SKILL.md` + `allowed-tools` + 精确指令 |
| 工具分寸写死 | skill 的 `description` 和 frontmatter |
| 拒绝时不泄露检测逻辑 | weather-agent 的 fail-closed guardrail |
| 参数排序影响思考顺序 | 同上思路：CLAUDE.md 和 skills 的结构设计 |

---

## 关联

- [小黑盒：Claude Code 的 skill 注入上下文分析](./claude-code-skill-injection.md)
- [本仓库 skill 概念讲解](../practice/04-技能-skills.md)
- [本仓库 skill 实现剖析](../implementation/03-skill-实现.md)
