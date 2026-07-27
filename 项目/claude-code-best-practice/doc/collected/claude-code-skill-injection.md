# ClaudeCode 的 Skill 注入上下文分析

> 来源：[小黑盒 - ClaudeCode的skill注入上下文分析](https://www.xiaoheihe.cn/app/bbs/link/1de3720946d0)
> 作者：锝嘚得（Lv.6）
> 日期：3天前（2026-06-18 左右）
> 收藏日期：2026-06-21

---

## 文章背景

作者接到需求：**计算 skill 的 token 用量**。

但 token 用量不是独立工作流或 agent 的属性——"skill 激活期间的会话 token 用量" 难以界定，因为不确定 skill 何时不再发挥作用。在探索这个问题的过程中，作者顺便研究了 Claude Code 是如何把 skill 注入上下文的。

---

## Skill 注入的三种方式

### 第一种：Slash Command 触发

在 Claude Code 中输入已注册的 skill 命令，例如：

```
/find-skills react testing
```

**运行时行为：** 不会把这段文本原样发给模型。Claude Code 会：

1. 先把本地命令解析成**结构化命令消息**（非纯文本）
2. 紧接着注入 **skill 正文**
3. 之后可能出现**权限声明**
4. 最后 assistant 在 skill 上下文中回复

**典型事件链：**

```
User: /find-skills react testing
        ↓
[解析为结构化命令消息]
        ↓
[注入 skill 正文 + 指令]
        ↓
[权限声明（如需）]
        ↓
Assistant 回复（在 skill 上下文中）
```

### 第二种：Assistant 自主调用 Skill 工具

Claude Code 提供一个内置工具，名字就是 `Skill`。当 assistant 判断某个 skill 适用时，它会发起**结构化工具调用**：

```
Tool Call: Skill(skill: "skill-name")
        ↓
[返回工具结果]
        ↓
[注入 skill 正文到上下文]
        ↓
Assistant 继续在 skill 上下文中工作
```

**大致流程：**

```
Assistant 判断需要某个 skill
        ↓
调用 Skill 工具（结构化调用）
        ↓
运行时返回工具结果
        ↓
注入 skill 正文
        ↓
Assistant 继续执行
```

这是**最智能的方式**——不需要用户手动输入斜杠命令，assistant 自主决定何时需要 skill。

### 第三种：Compact 后 invoked_skills 恢复

当会话过长发生 **/compact** 时，Claude Code 需要在续接时恢复仍然重要的上下文。如果某些 skill 已经被加载，续接消息附近会出现 `invoked_skills` 字段。

**这个字段的含义：** compact 之后，运行时把这些 skill 作为**快照恢复**进上下文。

**两个关键特点：**

1. **这不是一次新的用户调用** — 而是上下文续接机制
2. **不一定包含历史上所有加载过的 skill** — 只包含运行时认为续接仍需要保留的 skill

因此正确的标记方式应为：

```
event_type = "skill_snapshot"
is_new_call = false
```

而不是：

```
event_type = "skill_called"
```

---

## 技术要点总结

| 注入方式 | 触发者 | 场景 |
|----------|--------|------|
| Slash Command | 用户 | 用户主动使用 `/skill-name` |
| AI 自主调用 Skill 工具 | Assistant | AI 判断需要某个 skill |
| Compact 后 invoked_skills 恢复 | 运行时 | 上下文压缩后的 skill 快照恢复 |

---

## 附录：评论区精选

> **玩家67118204（Lv.15）：** 我去，给我干哪来了，这还是xhh吗，还以为在l站呢，质量这么高
>
> **changesh：** 不好说现在的L站质量还有多高，现在首页全是这个公益站用不了那个公益站打不开的
>
> **水墨藍（Lv.17）：** 直接做反代，然后从原始的请求里提取出skill部分。
>
> **作者回复：** 问题就是在于那部分可以被算到skill上啊，如果同时调了多个skill怎么分呢，skill的作用什么时候结束又是以什么作为标准呢，这些不好分啊
>
> **达达利鸭鸭（Lv.19）：** 这个游戏的分区里面现在全都是codex的帖子了

---

## 关联

- 参见本仓库的 skill 实现：[doc/implementation/03-skill-实现.md](../implementation/03-skill-实现.md)
- 参见 skill 概念讲解：[doc/practice/04-技能-skills.md](../practice/04-技能-skills.md)
