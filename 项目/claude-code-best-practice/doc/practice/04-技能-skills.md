# 技能（Skills）最佳实践

> 中文翻译整理自 `best-practice/claude-skills.md`
> 学习顺序：第 4 篇
> 官方文档：[Skills](https://code.claude.com/docs/en/skills)

---

## 什么是 Skill？

Skill 是 Claude Code 中的**可复用知识包**。它是一份 Markdown 文件，告诉 Claude 如何完成某个特定领域的任务。

**想象一下：** Skill 就像一本"菜谱"。当你想做一道菜时，你翻开对应的菜谱，照着步骤做。做完后合上菜谱，不影响你做下一道菜。

---

## 前置元数据字段（17 个）

每个 Skill 定义文件（`.claude/skills/<名称>/SKILL.md`）使用 YAML frontmatter 配置：

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | 否 | 显示名称和 `/命令`。省略时使用目录名 |
| `description` | string | 推荐 | 技能功能描述——**这是触发条件，不是摘要** |
| `when_to_use` | string | 否 | 何时调用的额外上下文——触发短语和示例 |
| `argument-hint` | string | 否 | 自动补全时的提示（如 `[issue-number]`） |
| `arguments` | string/list | 否 | 位置参数名，用于 `$name` 替换 |
| `disable-model-invocation` | boolean | 否 | `true` 则阻止 Claude 自动调用，变为纯手动 skill。配合 `user-invocable` 控制 `/skills` 显示标签。详见下方 [可见性控制](#skill-可见性控制) |
| `user-invocable` | boolean | 否 | `false` 则从 `/` 菜单和 `/skills` 中完全隐藏。和 `skillOverrides` 配合控制可见性。详见下方 [可见性控制](#skill-可见性控制) |
| `allowed-tools` | string | 否 | 免确认的工具 |
| `disallowed-tools` | string/list | 否 | 禁止的工具 |
| `model` | string | 否 | 运行时的模型 |
| `effort` | string | 否 | 推理力度覆盖 |
| `context` | string | 否 | `fork` 则在独立子代理中运行 |
| `agent` | string | 否 | `context: fork` 时的子代理类型 |
| `background` | boolean | 否 | 仅与 `context: fork` 配合使用。设为 `false` 则等待分支子代理的结果，而不是在后台运行（默认 `true`）。需 v2.1.218+ |
| `hooks` | object | 否 | 生命周期钩子 |
| `paths` | string/list | 否 | Glob 模式限制自动激活范围 |
| `shell` | string | 否 | `` !`command` `` 块的 shell |

---

## Skill 可见性控制

Skill 的可见性由三个层面决定：**两个 frontmatter 字段** + **一个 settings 覆盖**。

### 两个控制字段的分工

| 字段 | 控制什么 | 默认值 |
|------|---------|-------|
| `disable-model-invocation` | **Claude 能否自动调用**这个 skill | `false`（能自动调） |
| `user-invocable` | **用户能否 `/` 调用**这个 skill | `true`（能 / 调） |

两个字段独立工作。它们的四种组合产生了 `/skills` 列表中看到的不同标签。

### `/skills` 中看到的三种标签

在 `/skills` 交互界面中，每个 skill 前面有一个状态标记：

```
✔ on         code-review · project · ~140 tok       ← Claude 能自动调，用户也能 / 调
🔒 user-only ask-matt · project · ~30 tok            ← 只有用户能 / 调，Claude 不能自动调
（完全隐藏） weather-fetcher                           ← 用户看不到也调不了
```

#### 标签一：`✔ on`（Model-Invoked Skill）

```yaml
---
name: code-review
# disable-model-invocation: 未设置（默认 false）
# user-invocable: 未设置（默认 true）
---
```

| 用户 `/` 调用 | Claude 自动调用 |
|:---:|:---:|
| ✅ 可以 | ✅ 可以 |

**效果：** Claude 能在对话中根据上下文自动触发这个 skill，用户也可以手动 `/` 调用。

---

#### 标签二：`🔒 user-only · locked by author`（User-Invoked Skill）

```yaml
---
name: ask-matt
disable-model-invocation: true   # ← Claude 不能自动调
# user-invocable: 未设置（默认 true）
---
```

| 用户 `/` 调用 | Claude 自动调用 |
|:---:|:---:|
| ✅ 可以 | ❌ 不行 |

**效果：** 只有用户才能用 `/` 调用。Claude 不会在对话中自动触发它。这类 skill 的 description 是面向人类的，不是给模型看的触发条件。

**适用场景：** 不想让 Claude 自作主张调用的 skill，比如：
- `/ask-matt` — 需要用户自己做选择
- `/implement` — 影响太大的操作，必须用户确认
- `/edit-article` — 编辑文章这种需要用户主观判断的

---

#### 标签三：完全隐藏（Agent-Only Skill）

```yaml
---
name: weather-fetcher
user-invocable: false              # ← 用户看不到
# disable-model-invocation: 未设置
---
```

| 用户 `/` 调用 | Claude 自动调用 |
|:---:|:---:|
| ❌ 不可见 | ❌ 不行 |

**效果：** 既不在 `/skills` 列表中显示，也不能 `/` 调用。仅在 Subagent 通过 `skills:` 字段预加载时使用。

**适用场景：** 底层数据获取模块、仅供 agent 内部使用的知识包。

---

### 总结：两个字段四种组合

| `disable-model-invocation` | `user-invocable` | `/skills` 标签 | 用户 `/` 调用 | Claude 自动调 | 典型 skill |
|---|---|---|---|:---:|:---:|
| 未设置（false） | 未设置（true） | `✔ on` | ✅ | ✅ | `code-review` |
| `true` | 未设置（true） | `🔒 user-only` | ✅ | ❌ | `ask-matt`、`implement` |
| 未设置（false） | `false` | **完全隐藏** | ❌ | ❌ | `weather-fetcher` |
| `true` | `false` | **完全隐藏** | ❌ | ❌ | （极罕见） |

**两个思想流派：**
- 来自 `writing-great-skills` 的说明：
  > - **Model-invoked skill**（`✔ on`）：省略 `disable-model-invocation`，写一个面向模型的 description，Claude 可以自动发现并调用。description 在每个会话中都占用上下文 token。
  > - **User-invoked skill**（`🔒 user-only`）：设置 `disable-model-invocation: true`，description 变成面向人类的说明。零上下文开销，但需要你自己记住它存在。

### 上下文 Token 的实际影响

**核心结论：`🔒 user-only` 的 skill 描述不进上下文，`✔ on` 的进。**

每次会话，Claude 把 skill 列表加载到上下文中。受 `skillListingBudgetFraction`（默认 1%）控制：

| 模型 | 上下文窗口 | skill 列表预算 |
|-----|:---------:|:-------------:|
| Sonnet/Opus | 200K tokens | ~2000 tokens |

**加载时的行为差异：**

```
✔ on 的 skill（model-invoked）：
    code-review    "Review the changes since a fixed point along two axes..."   ← 完整描述
    tdd            "Test-driven development. Use when the user wants..."        ← 完整描述

🔒 user-only 的 skill（user-invoked）：
    ask-matt       （仅名称）                                                    ← 不占 token
    implement      （仅名称）                                                    ← 不占 token
```

当 skill 列表超出预算时，**最不常用的 `✔ on` skill 也会被折叠为仅名称**——Claude 知道它们存在但看不到描述，也就无法自动触发。

**现场审计结果：当前 36 个 Skill 中**

| 加载描述（占 token） | 不加载描述（零 token） |
|:---:|:---:|
| 17 个 — `code-review`、`tdd`、`prototype` 等 | 19 个 — `ask-matt`、`implement`、`handoff` 等 |

**一条典型 skill 描述的 token 成本：**
- `code-review`：~70 tokens
- `diagnosing-bugs`：~40 tokens
- 17 个 model-invoked 合计：~600-800 tokens/会话

**设计权衡：**
> User-invoked skill **用你的记忆力换取 Claude 的 token 预算**——你记得它存在、手动调它，它就不消耗上下文。Model-invoked skill 让 Claude 替你记、替你做自动化决策，但要付 token。
> 
> ——来自 `writing-great-skills` 的模式说明

---

### 用户覆盖：`skillOverrides`

在 `settings.json` 中，用户可以**覆盖** skill 作者的 frontmatter 声明：

```json
{
  "skillOverrides": {
    "weather-fetcher": "off",
    "code-review": "on",
    "ask-matt": "user-invocable-only"
  }
}
```

四种覆盖值：

| 值 | 效果 | 使用场景 |
|----|------|---------|
| `"on"` | 强制显示在 `/skills` 和 `/` 菜单 | 把隐藏的 skill 放出来 |
| `"name-only"` | 仅显示名字，不加载描述（节省 token） | 不常用但不想隐藏 |
| `"user-invocable-only"` | 只显示 `user-invocable: true` 的 | 只看自己能调的 |
| `"off"` | 完全隐藏 | 彻底禁用某个 skill |

### 三层的协作关系

```
Skill 作者 frontmatter             用户 skillOverrides           最终结果
───────────────────────           ────────────────────          ─────────
disable-model-invocation: 未设置   未设置                          ✔ on
user-invocable: 未设置

disable-model-invocation: true    未设置                          🔒 user-only
user-invocable: 未设置

user-invocable: false             未设置                          完全隐藏

user-invocable: false             "on"                           强制显示
                                  （用户覆盖优先级最高）
```

**一句话总结：**
- `disable-model-invocation` 控制 **Claude 能不能自动调**
- `user-invocable` 控制 **用户能不能 `/` 调**
- `skillOverrides` **用户说了算**——覆盖上面两个字段的效果

---

## 官方内置 Skill（13 个）

| # | Skill | 用途 |
|---|-------|------|
| 1 | `code-review` | 审查代码变更中的正确性错误 |
| 2 | `batch` | 批量操作多个文件 |
| 3 | `debug` | 调试失败的命令或代码 |
| 4 | `loop` | 按间隔重复执行 |
| 5 | `claude-api` | 构建 Claude API / Anthropic SDK 应用 |
| 6 | `fewer-permission-prompts` | 减少权限提示 |
| 7 | `run` | 启动并驱动项目应用 |
| 8 | `verify` | 构建并运行应用以验证更改 |
| 9 | `run-skill-generator` | 教 `/run` 和 `/verify` 如何构建和启动项目 |
| 10 | `simplify` | 审查代码简化机会 |
| 11 | `design-sync` | 将 React 设计系统同步到 Claude Design。仅在 Anthropic API 可用 |
| 12 | `dataviz` | 设计图表、图形和仪表板，含色板验证器。v2.1.198 引入 |
| 13 | `doctor` | Claude Code 配置的安装/健康检查。唯一不受 `disableBundledSkills` 影响的 bundle skill。v2.1.205 从内置命令转为 bundle skill |

---

## 最佳实践

### Skill 设计的核心理念

> 来自 Thariq（Anthropic 团队）：

1. **description 是触发条件，不是摘要**
   - 错误的写法：`"代码审查技能"`
   - 正确的写法：`"Review the current diff for correctness bugs — use when changes are ready for review"`

2. **Skill 是目录，不是文件**
   - 可以包含 `references/`、`scripts/`、`examples/` 子目录
   - Claude 会自动加载这些子目录内容

3. **建立 Gotchas 章节**
   - 记录 Claude 在这个领域常犯的错误
   - 随时间积累，成为最高信号密度的内容

4. **不说显而易见的事**
   - 重点写哪些让 Claude 偏离默认行为的信息
   - 不需要写 "You are an AI assistant..."

5. **给目标，不给步骤**
   - 错误的写法："第 1 步做 X，第 2 步做 Y"
   - 正确的写法："目标是确保所有 API 端点都有认证检查。你可以决定如何实现。"

6. **包含脚本和库**
   - Claude 应该组合你的工具，而不是自己重写样板代码

7. **使用 `!command` 动态注入**
   - 在 SKILL.md 中嵌入 `!ls src/`，Claude 运行 Skill 时会执行该命令并将结果注入提示

### Skill 的两大模式

**模式 1：普通 Skill（共享上下文）**
```yaml
---
name: my-skill
description: 对当前上下文注入领域知识
---
```

**模式 2：Fork Skill（隔离上下文）**
```yaml
---
name: my-heavy-skill
description: 在隔离环境中执行，不干扰主会话
context: fork
agent: general-purpose
---
```
适合：需要大量工具调用或文件操作的任务，主会话只看到最终结果。

---

### Skill vs Command vs Subagent

| | Skill | Command | Subagent |
|--|-------|---------|----------|
| **本质** | 知识/指令包 | 工作流入口 | 独立助手 |
| **运行方式** | 注入到当前上下文或 fork | 在当前上下文执行 | 独立上下文 |
| **最适合** | 告诉 Claude "怎么做" | 告诉 Claude "做什么" | 让另一个 Claude "替你做" |
| **复用方式** | 跨项目复制 | 跨项目复制 | 跨项目复制 |

---

## 一个典型的 Skill 示例

```yaml
---
name: api-patterns
description: 实现 REST API 端点时使用的模式
---
## API 设计约定

### 路由命名
- 资源名使用复数：`/api/users`，`/api/posts`
- 嵌套路由不超过 2 层

### 错误响应格式
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "用户友好的错误描述",
    "details": {}
  }
}
```

### Gotchas
- 使用 `request.body` 前始终做类型验证
- 始终设置 `Content-Type: application/json` 响应头
```

### 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Subagents（子代理） | [`02-子代理-subagents.md`](./02-子代理-subagents.md) | 在 Agent 中用 `skills` 字段预加载 Skill |
| Commands（命令） | [`03-命令-commands.md`](./03-命令-commands.md) | Skill 也可通过 `/命令` 调用，用法类似 Command |
| MCP 服务器 | [`06-MCP服务器-mcp.md`](./06-MCP服务器-mcp.md) | Skill 可与 MCP 工具组合使用 |
| 记忆系统（Memory） | [`07-记忆系统-memory.md`](./07-记忆系统-memory.md#⚠️-常见陷阱官方-auto-memory-vs-自定义-agent-memory) | 注意 `memory: project` 与自定义 agent 记忆的区别 |
| 本仓库 Skill 列表 | 项目结构总览见 [`01-CONCEPTS-概念总览.md`](./01-CONCEPTS-概念总览.md#项目结构总览) | 本仓库安装了 35+ 个 Matt Pocock Skills |
| 实现参考 | [`../implementation/03-skill-实现.md`](../implementation/03-skill-实现.md) | 本仓库 Skill 的实际配置示例 |

---

> 下一篇：[05-设置-settings.md](./05-设置-settings.md)
