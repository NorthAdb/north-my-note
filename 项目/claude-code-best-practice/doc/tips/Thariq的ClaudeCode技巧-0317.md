# 构建Claude Code的经验教训：我们如何使用技能——Thariq（中文翻译）

> 中文翻译自 [tips/claude-thariq-tips-17-mar-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-thariq-tips-17-mar-26.md)

关于Anthropic内部如何使用技能的全面指南，由Thariq（[@trq212](https://x.com/trq212)）于2026年3月17日分享。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 背景

技能已成为Claude Code中使用最广泛的扩展点之一。它们灵活、易于制作且易于分发。但这种灵活性也使得很难知道什么效果最好。Thariq分享了在Anthropic大量使用技能（有数百个技能在活跃使用中）的经验教训。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/1.png" alt="Thariq 介绍推文" width="50%" /></a>

---

## 什么是技能？

一个常见的误解是技能"只是markdown文件"，但最有趣的部分在于它们是**文件夹**，可以包含脚本、资源、数据等——代理可以发现、探索和操作的东西。技能还有各种各样的配置选项，包括注册动态钩子。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/2.png" alt="什么是技能？" width="50%" /></a>

---

## 技能的类型

在整理了所有技能后，团队注意到它们聚集为9个重复出现的类别。最好的技能能干净地归入其中一个；那些更令人困惑的则横跨多个类别。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/3.png" alt="技能类型网格" width="50%" /></a>

---

### 1/ 库和API参考

解释如何正确使用库、CLI或SDK的技能。这些可以是针对内部库，或Claude Code有时处理不好的常见库。它们通常包含一个参考代码片段文件夹和一份编写脚本时要避免的陷阱列表。

**示例：** billing-lib, internal-platform-cli, frontend-design

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/4.png" alt="库和API参考" width="50%" /></a>

---

### 2/ 产品验证

描述如何测试或验证代码是否正常工作的技能。这些通常与外部工具如Playwright、tmux等配对使用。验证技能对于确保Claude的输出正确性极其有用。花一周时间让工程师专门打磨你的验证技能是非常值得的。

**示例：** signup-flow-driver, checkout-verifier, tmux-cli-driver

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/5.png" alt="产品验证" width="50%" /></a>

---

### 3/ 数据获取与分析

连接到你的数据和监控堆栈的技能。这些可能包括获取数据所需的库及凭证、特定的仪表板ID等，以及常见工作流或获取数据方式的说明。

**示例：** funnel-query, cohort-compare, grafana

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/6.png" alt="数据获取与分析" width="50%" /></a>

---

### 4/ 业务流程与团队自动化

将重复性工作流自动化为一个命令的技能。这些通常是比较简单的指令，但可能依赖于其他技能或MCP，依赖关系较为复杂。将之前的结果保存在日志文件中可以帮助模型保持一致并对之前的工作流执行情况进行反思。

**示例：** standup-post, create-\<ticket-system\>-ticket, weekly-recap

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/7.png" alt="业务流程与团队自动化" width="50%" /></a>

---

### 5/ 代码脚手架与模板

为代码库中的特定功能生成框架模板代码的技能。你可以将这些技能与可组合的脚本结合使用。当你的脚手架有纯代码无法覆盖的自然语言需求时，它们尤其有用。

**示例：** new-\<framework\>-workflow, new-migration, create-app

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/8.png" alt="代码脚手架与模板" width="50%" /></a>

---

### 6/ 代码质量与审查

在组织内强制执行代码质量并帮助审查代码的技能。这些可以包括确定性脚本或工具以获得最大的稳健性。你可能希望将这些技能作为钩子的一部分或GitHub Action中自动运行。

**示例：** adversarial-review, code-style, testing-practices

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/9.png" alt="代码质量与审查" width="50%" /></a>

---

### 7/ CI/CD与部署

帮助你在代码库中获取、推送和部署代码的技能。这些技能可能引用其他技能来收集数据。

**示例：** babysit-pr, deploy-\<service\>, cherry-pick-prod

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/10.png" alt="CI/CD与部署" width="50%" /></a>

---

### 8/ Runbook（操作手册）

接受一个症状（如Slack线程、警报或错误签名），进行多工具调查，并生成结构化报告的技能。

**示例：** \<service\>-debugging, oncall-runner, log-correlator

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/11.png" alt="Runbook（操作手册）" width="50%" /></a>

---

### 9/ 基础设施运维

执行日常维护和操作流程的技能——其中一些涉及可以从护栏中受益的破坏性操作。这使得工程师更容易在关键操作中遵循最佳实践。

**示例：** \<resource\>-orphans, dependency-management, cost-investigation

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/12.png" alt="基础设施运维" width="50%" /></a>

---

## 制作技能的技巧

编写有效技能的9个最佳实践，以及关于分发和度量的指南。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/13.png" alt="制作技能技巧网格" width="50%" /></a>

---

### 技巧1：不要陈述显而易见的事

Claude Code对你的代码库了解很多，Claude对编码也了解很多，包括许多默认的偏好。如果你发布的技能主要是关于知识方面的，试着专注于推动Claude跳出其正常思维方式的信息。前端设计技能就是一个很好的例子——它通过跟客户反复迭代来改进Claude的设计品味，避免像Inter字体和紫色渐变这样的经典模式而构建的。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/14.png" alt="不要陈述显而易见的事" width="50%" /></a>

---

### 技巧2：建立"陷阱"部分

任何技能中最有价值的信号就是"陷阱"部分。这些部分应该从Claude在使用你的技能时遇到的常见失败点中积累起来。理想情况下，你会随时间更新你的技能以捕捉这些陷阱。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/15.png" alt="建立陷阱部分" width="50%" /></a>

---

### 技巧3：利用文件系统和渐进式披露

技能是一个文件夹，而不仅仅是一个markdown文件。你应该将整个文件系统视为一种上下文工程和渐进式披露的形式。告诉Claude你的技能中有哪些文件，它会在适当的时候读取它们。最简单的形式是指向其他markdown文件——例如，将详细的函数签名和用法示例拆分到 `references/api.md` 中。你可以有引用、脚本、示例等文件夹。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/16.png" alt="渐进式披露" width="50%" /></a>

---

### 技巧4：避免给Claude设定死板路线

Claude通常会尽量遵循你的指示，而且由于技能的可重用性很强，你需要注意不要过于具体。给Claude所需的信息，但也要给它适应情况的灵活性。与其给出规定性的逐步指令，不如给出目标和约束条件。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/17.png" alt="避免给Claude设定死板路线" width="50%" /></a>

---

### 技巧5：仔细考虑设置

有些技能可能需要用户提供上下文才能进行设置。一个好的模式是将这些设置信息存储在技能目录中的 `config.json` 文件中。如果配置未设置，代理可以向用户询问信息。你可以指示Claude使用AskUserQuestion工具进行结构化的选择题问答。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/18.png" alt="仔细考虑设置" width="50%" /></a>

---

### 技巧6：描述字段是为模型写的

当Claude Code启动会话时，它会构建一个所有可用技能及其描述的列表。Claude扫描这个列表来决定"有这个请求对应的技能吗？"这意味着描述字段不是摘要——而是**何时触发**此技能的描述。要为模型编写它。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/19.png" alt="描述 = 触发条件" width="50%" /></a>

---

### 技巧7：记忆与数据存储

一些技能可以通过在自身内部存储数据来包含某种形式的记忆。你可以将数据存储在简单到只追加的文本日志文件或JSON文件中，或复杂到SQLite数据库中。存储在技能目录中的数据可能在升级技能时被删除，所以使用 `${CLAUDE_PLUGIN_DATA}` 作为每个插件的稳定文件夹来存储数据。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/20.png" alt="记忆与数据存储" width="50%" /></a>

---

### 技巧8：存储脚本和生成代码

你可以给Claude的最强大工具之一就是代码。给Claude脚本和库可以让Claude将其轮次用于组合，决定下一步做什么，而不是重构样板代码。Claude可以即时生成脚本来组合这些功能，用于更高级的分析。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/21.png" alt="存储脚本和生成代码" width="50%" /></a>

---

### 技巧9：按需钩子

技能可以包含只有在技能被调用时才激活的钩子，持续时间为会话期间。用于那些你不想一直运行但有时极其有用的更有主见的钩子。

**示例：**
- `/careful` —— 通过PreToolUse在Bash上匹配，阻止 rm -rf、DROP TABLE、force-push、kubectl delete
- `/freeze` —— 阻止任何不在特定目录中的Edit/Write

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/22.png" alt="按需钩子" width="50%" /></a>

---

## 分发技能

与团队分享技能的两种方式：
- **检入你的仓库**（放在 `.claude/skills` 下）——最适合在相对少量仓库上工作的小型团队
- **制作插件**并建立一个Claude Code插件市场，让用户可以上传和安装插件

每个检入的技能也会为模型的上下文增加一点负担。随着规模扩大，内部插件市场允许你分发技能，让你的团队决定安装哪些。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/23.png" alt="分发技能" width="50%" /></a>

---

## 管理市场

没有一个中心化团队决定哪些技能进入市场。相反，尝试有机地发现最有用的技能。上传到GitHub中的沙箱文件夹，并在Slack或其他论坛中引导大家使用。一旦某个技能获得了关注（由技能所有者自行决定），他们可以提交PR将其移入市场。发布前的策展很重要，以避免冗余的技能。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/24.png" alt="管理市场" width="50%" /></a>

---

## 组合技能

你可能希望拥有相互依赖的技能。例如，一个文件上传技能用于上传文件，和一个CSV生成技能用于制作CSV并上传。这种依赖管理在原生层面尚未内置于市场或技能中，但你只需通过名称引用其他技能，如果它们已安装，模型就会调用它们。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/25.png" alt="组合技能" width="50%" /></a>

---

## 度量技能

为了了解技能的表现如何，使用一个PreToolUse钩子，让你在公司内部记录技能使用情况。这意味着你可以发现哪些技能受欢迎，或者哪些技能与预期相比触发不足。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/26.png" alt="度量技能" width="50%" /></a>

---

## 结论

技能是代理极其强大和灵活的工具，但尚处于早期阶段，我们都在摸索如何最好地使用它们。把这更多地看作是我们已经看到有效的一些有用技巧的集合，而非一份权威指南。理解技能的最佳方式就是开始使用、实验，并看看什么对你有用。我们的大多数技能都是从几行代码和一个陷阱开始的，并随着人们不断添加而变得更好——因为Claude会遇到新的边缘情况。

<a href="https://x.com/trq212/status/2033949937936085378"><img src="assets/thariq-26-3-17/27.png" alt="结论" width="50%" /></a>

---

## 来源

- [Thariq (@trq212) on X — March 17, 2026](https://x.com/trq212/status/2033949937936085378)
- [Skilljar — Agent Skills course](https://code.claude.com/docs/en/skills)
- [Skill Creator](https://code.claude.com/docs/en/skills)
