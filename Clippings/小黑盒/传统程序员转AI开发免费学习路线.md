---
created: 2026-07-21
updated: 2026-07-21
tags: [小黑盒, AI, 学习路线, 传统程序员, 转行]
source: "https://www.xiaoheihe.cn/app/bbs/link/16fe722e058c"
---

# 传统程序员转 AI 开发：一条免费学习路线

> 来源：小黑盒「前沿情报站」· 2天前发布
> GitHub 五个最好的免费项目，串成一条从零到 AI 开发的路

---

## 文章核心问题

> **「做 Java 三年了，怎么转 AI？」**
> **「前端想学 Agent 开发，从哪入手？」**
>
> 小黑盒上天天有人问。满屏 LLM、RAG、Agent、Fine-tuning，每个词都认识，连起来就不会了。缺的不是资源，是一条清晰的路线。

---

## 五步学习路线

### ① 建立认知 — microsoft/generative-ai-for-beginners ⭐113K

> 微软官方 21 节 AI 入门课

| 项目 | 数据 |
|------|:----:|
| 仓库 | [microsoft/generative-ai-for-beginners](https://github.com/microsoft/generative-ai-for-beginners) |
| ⭐ | 113k |
| 时长 | 每节 30-45 分钟 |
| 内容 | 从「什么是生成式 AI」→ Prompt 设计 → RAG → Agent → 安全 |

**价值**：帮你画了一张 AI 全景地图。不需要学完——看完前两节你就知道怎么跟 AI 对话了。

---

### ② 理解原理 — rasbt/LLMs-from-scratch ⭐99K

> 用 PyTorch 从零手写一个 GPT

| 项目 | 数据 |
|------|:----:|
| 仓库 | [rasbt/LLMs-from-scratch](https://github.com/rasbt/LLMs-from-scratch) |
| ⭐ | 99k |
| 内容 | 注意力机制 → Transformer → RLHF，每一个组件自己实现一遍 |

**价值**：不需要高等数学，公式都翻译成了 Python 代码。读完三个核心章节（Attention、架构、文本生成），AI 在你眼里不再是黑盒。

---

### ③ 掌握技能 — dair-ai/Prompt-Engineering-Guide ⭐76K

> Prompt 不是「跟 AI 说话」，是编程

| 项目 | 数据 |
|------|:----:|
| 仓库 | [dair-ai/Prompt-Engineering-Guide](https://github.com/dair-ai/Prompt-Engineering-Guide) |
| ⭐ | 76k |
| 内容 | 零样本、少样本、思维链、ReAct、自动优化 |

**价值**：读完基础三章（两小时），你之前写的 Prompt 你会发现基本全是错的。

---

### ④ 搭建平台 — langgenius/dify ⭐149K

> 国内团队开源的 AI 应用开发平台

| 项目 | 数据 |
|------|:----:|
| 仓库 | [langgenius/dify](https://github.com/langgenius/dify) |
| ⭐ | 149k |
| 功能 | 拖拽组件搭 AI 工作流 — LLM 串联、RAG 知识库、API 调用、条件分支 |

**价值**：不用写前后端，五分钟出一个原型。跑通了再翻源码学原理。转行的人先「看见能跑的东西」，再去补底层。

---

### ⑤ 做出成品 — harry0703/MoneyPrinterTurbo ⭐97K

> 中文项目！输入主题 → AI 自动生成完整短视频

| 项目 | 数据 |
|------|:----:|
| 仓库 | [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) |
| ⭐ | 97k |
| 流程 | 文案 (LLM) + 配音 (TTS) + 画面 (AI 生图) + 字幕自动合成 → mp4 |

**价值**：把你前面学的全串起来了：**LLM 调用 + Prompt 设计 + 工作流编排**。跑通了你就不是在「学 AI」了，是在「用 AI 做产品」。

---

## 📅 六周计划

| 周次 | 任务 | 目标 |
|:---:|------|------|
| 1-2 | 微软 AI 课 | 建立全局认知 |
| 3 | 手写 GPT（核心章节）+ Prompt 指南 | 理解 LLM 原理 + 学会跟 AI 高效协作 |
| 4-5 | Dify 搭工作流 | 能做出 AI 应用原型 |
| 6 | MoneyPrinter 跑通 | 第一个 AI 产品出炉 |

---

## 💬 精彩评论

> **RPG666**（Lv.20，2天前·北京 · 7赞）：
> 我自己找 ai 搞了个对话项目来学习，本地部署无审核模型，让 DeepSeek 做对话系统，加入了很多角色扮演功能。现在开始搞多 Agent 协作，不同 agent 是不同性格的角色。
> 项目完成后再看看每个接口后端代码怎么写的，很快入门了。不过你要找工作肯定不够。
>
> **前沿情报站** 回复：是这样的，工作涉及更多的业务场景，垂直领域
>
> **RPG666** 回复：小模型能力不太够，定义了几个魅魔跟我聊天，多人群聊共用记忆，agent 容易记忆混乱。不过刚开始几轮对话还是很涩的

> **不功**（Lv.19，2天前·山西 · 4赞）：
> 语言和技术栈对 AI 来说没有任何界限，所以想搞这个你需要学 AI 怎么用，需要去了解不同语言和技术栈大概什么情况。然后关注 AI 的实际应用，关注实际业务。
> 装个 Agent 玩容易，但搞清楚各种技术栈和实际业务需要很长时间的学习和积累。

---

## 🔑 文章金句

> **「你的工程能力在 AI 时代一点都不贬值——代码质量、系统设计、问题拆解，这些大模型替不了。你要加的就是这层 AI 知识框架。」**

---

## 🔗 关联笔记

- [[领域/AI Agent 智能体学习路线 2026.md]] — 你正在学习的 Agent 路线
- [[项目/大学四年比课本有用的GitHub仓库.md]] — 类似的 GitHub 仓库推荐
- [[Clippings/GitHub/2026-07-04-大学四年比课本有用的GitHub仓库.md]] — 校园到职场系列
