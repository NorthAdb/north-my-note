---
created: 2026-07-31
updated: 2026-08-13
tags: [Matt Pocock, Skills, 仓库清单, 分类汇总, engineering, productivity, Agent工作流]
source:
  - "https://github.com/mattpocock/skills"
  - "Skills For Real Engineers"
---

# Matt Pocock Skills 仓库全量分类汇总

> **仓库**：[mattpocock/skills](https://github.com/mattpocock/skills) | **理念**：Skills For Real Engineers（真实工程，不是 vibe coding）
> **规模**：21.6 万 star（215,920）· 1540 万 installs · 35 个 skills（工程 18 + 生产力 7 + misc 4 + 实验 6；插件含前 25 个）
> **安装**：`claude plugins install mattpocock-skills`（Claude Code 官方插件，自动更新）或 `npx skills@latest add mattpocock/skills`（可编辑副本）

---

## 📅 本次更新（2026-08-13，对比 07-31 快照）

| 类型 | 变化 |
| :--- | :--- |
| **新增** | `wait-what`（productivity）：一句话纠正模型啰嗦 |
| **改名 + 重构** | `writing-great-skills` → **`writing-for-agents`**（覆盖所有 agent 消费的文档，改 model-invoked，旧名无别名） |
| **转正** | `wizard`：in-progress → **engineering**（model-invoked）；`to-questionnaire`：in-progress → **productivity** |
| **移除** | `edit-article`、`obsidian-vault`（原 Personal 类整体消失）、`batch-grill-me`、`design-an-interface`、`qa`、`request-refactor-plan`、`ubiquitous-language`（deprecated 桶已清空） |
| **大改** | `prototype`（单一 HTML 文件 + primary source）、`wayfinder`（decision ticket + research 子 agent 并行）、`ask-matt`（phase 决策树 + 新路由）、`diagnosing-bugs`（脱敏）、`improve-codebase-architecture`（YAGNI 范围过滤） |
| **安装** | 新增官方 Claude Code 插件（`claude plugins install mattpocock-skills`，v1.2.3，25 个 skill，自动更新）；skills 全部带 Codex 元数据，双 harness 可用 |

---

## 设计哲学

> **「Approaches like GSD, BMAD, and Spec-Kit try to own the process. But while doing so, they take away your control. These skills are designed to be small, easy to adapt, and composable. They work with any model.」**

- 小而可组合，不搞大而全的流程框架
- 基于数十年工程经验，不依赖特定模型
- 大多需**手动调用**（不自动加载），描述精简精准；`wizard` / `writing-for-agents` 为 model-invoked（agent 可自动触发）

---

## 🏗️ Engineering 工程类（18 个）⭐ 核心

### 探索与规划

| Skill               | 作用                                                        | 何时用               |
| :------------------ | :-------------------------------------------------------- | :---------------- |
| **grill-with-docs** | 有状态盘问打磨方案，边聊边生成 ADR + 术语表                                 | 开始设计/规划前          |
| **domain-modeling** | 建立并打磨项目领域模型（DDD 统一语言、ADR）                                 | 想敲定领域术语时          |
| **wayfinder**       | 规划超大会话容量的工作：把大块工作变成 issue tracker 上的决策 ticket map，逐个解决    | 一个 agent 会话装不下的任务 |
| **research**        | 针对高可信一手资料做调查，结果存为 Markdown                                | 需要调研/查证           |
| **triage**          | 把 issue 和外部 PR 推过分类状态机：归类→验证→按需 grill→写 agent-ready brief | 管理 issue 流        |
| **to-spec**         | 把当前对话合成为 spec 发布到 issue tracker（只综合，不采访）                  | 讨论完要固化 spec       |
| **to-tickets**      | 把 plan/spec/对话拆成 tracer-bullet tickets，声明 blocking 边      | 拆解实现任务            |

### 设计

| Skill               | 作用                         | 何时用               |
| :------------------ | :------------------------- | :---------------- |
| **codebase-design** | 深度模块设计的共享词汇表：设计模块接口、找深化机会  | 设计/改进模块接口         |
| **prototype**       | 做一次性原型回答设计问题（状态模型/逻辑是否对）   | 需要 sanity-check 时 |
| **tdd**             | 测试驱动开发（red-green-refactor） | 想测试先行时            |

### 实现与质量

| Skill | 作用 | 何时用 |
|:------|:-----|:-------|
| **implement** | 基于 spec/tickets 实现一块工作 | 有 spec 或 ticket 后 |
| **code-review** | 从固定点（commit/branch/tag/merge-base）审查改动，两个维度：Standards（是否遵循仓库标准）+ Spec（是否符合原始 issue/spec），并行子 agent 双审 | 提交前 |
| **diagnosing-bugs** | 硬 bug 和性能回归的诊断循环 | debug/诊断 |
| **improve-codebase-architecture** | 扫描代码库找深化机会 → HTML 报告 → 挑一个 grill | 定期重构 |
| **resolving-merge-conflicts** | 解决进行中的 git merge/rebase 冲突 | 冲突时 |
| **ask-matt** | 技能路由器：根据情况推荐该用哪个 skill/流程 | 不知道用哪个时 |
| **setup-matt-pocock-skills** | 配置仓库：issue tracker、triage 标签词汇、领域文档布局（首次使用前跑一次） | 首次配置 |
| **wizard** | 生成交互式 bash 向导，带人走完**只有人能做**的步骤（配置密钥/CI secret、走陌生第三方面板、一次性迁移） | 遇到"只能人做"的步骤时（model-invoked） |

---

## 🛠️ Productivity 生产力类（7 个）

| Skill                    | 作用                            | 何时用               |
| :----------------------- | :---------------------------- | :---------------- |
| **grill-me**             | 无状态盘问打磨计划/设计                  | 想 stress-test 想法时 |
| **grilling**             | 关于计划/决策/想法的持续盘问               | 任何 grill 触发词      |
| **handoff**              | 把当前对话 compact 成交接文档给另一个 agent | 换会话/交接            |
| **teach**                | 在当前工作区教用户新技能/概念（有状态教学工作区）     | 想学东西时             |
| **wait-what**            | 🆕 一句话纠正模型啰嗦："停，这句没听懂，重新讲"（三行小 skill，user-invoked） | 模型输出没落地时           |
| **to-questionnaire**     | 🆕 把无法独自回答的决策变成问卷，给别人（异步/会议上）填 | 只有别人能回答时          |
| **writing-for-agents**   | 写 agents 消费的文档（skills、AGENTS.md/CLAUDE.md、指针文档）的参考，model-invoked | 写/改 skill 或 agent 文档时 |

---

## 🧰 Misc 杂项类（4 个）

| Skill                          | 作用                                                                    |
| :----------------------------- | :-------------------------------------------------------------------- |
| **git-guardrails-claude-code** | 设置 Claude Code hooks 拦截危险 git 命令（push、reset --hard、clean、branch -D 等） |
| **migrate-to-shoehorn**        | 把测试文件从 `as` 类型断言迁移到 @total-typescript/shoehorn                        |
| **scaffold-exercises**         | 创建练习目录结构（sections/problems/solutions/explainers）通过 lint               |
| **setup-pre-commit**           | 设置 Husky pre-commit hooks（Prettier + 类型检查 + 测试）                       |

---

## 👤 Personal 个人类（已移除 🗑️）

> 原 `edit-article`、`obsidian-vault` 两个 skill 已从仓库删除（2026-08-13 前）。仓库现在只有 engineering / productivity / misc / in-progress 四个正式桶，不再有 personal 桶。

---

## 🧪 In-progress 实验类（6 个）

| Skill | 作用 | 状态 |
|:------|:-----|:-----|
| **claude-handoff** | 把当前对话交给一个全新的后台 agent 立即接续 | 实验 |
| **loop-me** | 针对要构建的工作流 spec 盘问 | 实验 |
| **setup-ts-deep-modules** | 把 dependency-cruiser 接入 TS 仓库，让每个包成为深度模块（实现藏在子目录、只能通过入口文件触达） | 实验 |
| **writing-beats** | 写作·exploit：把原材料组装成节奏（beats）旅程 | 实验 |
| **writing-fragments** | 写作·explore：挖掘原始片段，还没结构 | 实验 |
| **writing-shape** | 写作·exploit：把原材料逐段塑造成文章 | 实验 |

> 写作三件套（fragments → shape → beats）是 Matt 在打磨的写作工作流：先无结构探索，再塑形，再组装成节奏。
> ⚠️ 原 `batch-grill-me` 已被移除；`to-questionnaire` 和 `wizard` 已转正（见上）。

---

## 🗑️ Deprecated 已废弃类（空）

> 仓库的 deprecated 桶已清空。新规则：**退役即删**——一个 skill 不再使用时直接删除，由对应 changeset 注明替代品。原 4 个已删除：
> `design-an-interface`（→ codebase-design）、`qa`（→ diagnosing-bugs）、`request-refactor-plan`（→ improve-codebase-architecture / codebase-design）、`ubiquitous-language`（→ domain-modeling）

---

## 🔗 主流程链路（这些 skills 如何配合）

```
wayfinder（决策 ticket map：把超大工作拆成 issue tracker 上的决策）   grill-with-docs（打磨方案）
      │                              │
      └──────► to-spec（合成 spec）
                    │
              to-tickets（拆 tickets）
                    │
              implement（逐个实现）
                    │
              code-review（审查）
```

**围绕主链路的辅助**：
- `research`：为决策做调研（wayfinder 的 research tickets 会派 `/research` 子 agent 并行烧毁，是"每会话一个 ticket"的唯一例外）
- `prototype`：需要可运行答案时提保真度（产物变成单一 HTML 文件 + throwaway branch）
- `domain-modeling`：敲定统一语言，让 spec/实现更精准
- `triage`：管理 issue 流
- `code-review` + `improve-codebase-architecture`：持续保证质量（后者现在按 YAGNI 只扫活跃开发的路径）
- `ask-matt`：不知道用哪个时的路由器（覆盖 phase 边界决策 + 新路由 /grilling、/resolving-merge-conflicts）
- `wizard`：流程卡在"只能人做"的步骤时，由 agent 自动拉起（model-invoked）

---

## 📌 快速选型

| 我想…         | 用这个                               |
| :---------- | :-------------------------------- |
| 打磨一个模糊想法    | **grill-me / grill-with-docs**    |
| 规划一个大项目     | **wayfinder**                     |
| 敲定领域术语      | **domain-modeling**               |
| 调研一个问题      | **research**                      |
| 看看设计/逻辑对不对  | **prototype**                     |
| 把讨论变 spec   | **to-spec**                       |
| 拆实现任务       | **to-tickets**                    |
| 动手实现        | **implement**                     |
| 提交前检查       | **code-review**                   |
| 修一个 bug     | **diagnosing-bugs**               |
| 让代码库更好改     | **improve-codebase-architecture** |
| 学新东西        | **teach**                         |
| 交接给别的 agent | **handoff**                       |
| 模型输出没落地/太啰嗦 | **wait-what**                     |
| 需要别人才能回答的决策 | **to-questionnaire**              |
| 流程卡在"只能人做"的步骤 | **wizard**                        |

---

## 证据与原文位置

- **仓库**：https://github.com/mattpocock/skills（通过 gh API 读取目录结构 + SKILL.md frontmatter + CHANGELOG）
- **docs 目录**：`docs/engineering/`（18 个 skill 各有 doc）+ `docs/productivity/`（7 个）
- 各 SKILL.md frontmatter 的 description 字段为汇总依据

## 局限

- ⚠️ In-progress 的 skill 可能随时变化/移除；deprecated 桶新规则为"退役即删"
- ⚠️ 描述为英文原文的精简翻译，完整用法以仓库 SKILL.md 为准
- ⚠️ 仓库持续更新（Matt 每周都在 ship 改动），本清单基于 **2026-08-13** 抓取（插件 v1.2.3）

## 关联笔记

- [[Matt-Pocock/Matt-Pocock方法论精华-完整工作流与技能体系]] — 方法论精华 + 主流流程
- [[Matt-Pocock/2026-07-31-Matt-Pocock-Skills原作者手把手教程]] — 官方安装配置与主流程教程
- [[Matt-Pocock/2026-07-31-Matt-Pocock直播-SlopWatch从零搭建完整实录]] — GrillMe/domain-model 实战
