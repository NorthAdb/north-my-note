# 记忆系统（Memory）最佳实践

> 中文翻译整理自 `best-practice/claude-memory.md`
> 学习顺序：第 7 篇
> 官方文档：[Memory](https://code.claude.com/docs/en/memory)

---

## 什么是 Memory？

Memory 是 Claude Code 的**持久化记忆体系**。它让 Claude 在多次会话之间记住你的项目规范、代码风格和偏好设置。

---

## 三层记忆体系

```
第一层：CLAUDE.md        ← 项目级（显式声明）
第二层：.claude/rules/    ← 规则级（按路径加载）
第三层：Auto Memory       ← 自动级（自动学习）
```

---

## 第一层：CLAUDE.md（项目记忆）

**位置：** 项目根目录（或任意子目录）

**作用：** 这是 Claude 启动时首先读取的文件，相当于项目的"入职手册"。

**最佳长度：** 每份不超过 **200 行**。

### 应包含的内容

- 项目概述和技术栈
- 代码风格和命名约定
- 常用命令（构建、测试、部署）
- 架构决策记录
- 关键约定和规则

### Monorepo 中的 CLAUDE.md 加载机制

Claude Code 使用**两种加载机制**：

#### 1. 祖先加载（向上遍历）
启动时，Claude 从当前目录**向上**查找并加载遇到的所有 CLAUDE.md。

#### 2. 后代加载（向下延迟加载）
子目录中的 CLAUDE.md **不会在启动时加载**，只有当你操作该目录中的文件时才会加载。

**示例：**
```
/mymonorepo/
├── CLAUDE.md          # 始终在启动时加载
├── frontend/
│   └── CLAUDE.md      # 操作前端文件时加载
├── backend/
│   └── CLAUDE.md      # 操作后端文件时加载
```

**关键要点：**
- 祖先目录的 CLAUDE.md **总是启动时加载**
- 子目录的 CLAUDE.md **延迟加载**（操作相关文件时才加载）
- 兄弟目录的 CLAUDE.md **永不加载**（如你在 frontend/，不会加载 backend/ 的）
- 全局 CLAUDE.md 放在 `~/.claude/CLAUDE.md`，对所有项目生效

### 最佳实践

1. **共享约定放根目录** — 编码标准、提交信息格式
2. **组件专用指令放子目录** — 框架特性模式、测试约定
3. **个人偏好用 `.local.md`** — 加入 `.gitignore` 不提交

---

## 第二层：.claude/rules/（规则记忆）

**位置：** `.claude/rules/*.md`

**加载方式：**
- **无 `paths` 字段**：每次会话都加载，像 CLAUDE.md 一样
- **有 `paths` 字段**：只在你操作匹配路径时加载（延迟加载）

**示例：**
```markdown
---
name: presentation
description: 幻灯片开发规则
paths: presentation/**/*.html
---
# 演示文稿规则
- 使用 16:9 宽高比
- 主色调使用蓝色系
```

---

## 第三层：Auto Memory（自动记忆）

**位置：** `~/.claude/projects/<项目>/memory/`

**作用：** Claude 自动从对话中学习并记住：
- 用户的角色和偏好（`user` 类型）
- 反馈和偏好（`feedback` 类型）
- 项目上下文（`project` 类型）
- 参考信息（`reference` 类型）

**控制方式：**
- `/memory` — 查看和编辑记忆
- `autoMemoryEnabled: false` — 关闭自动记忆
- 在 Subagent 中设置 `memory: user|project|local` 控制范围

### ⚠️ 常见陷阱：官方 Auto Memory vs 自定义 Agent Memory

**问题：** 在实际项目中，容易把两个不同的"记忆"机制混为一谈。

|          | **官方 Auto Memory**                | **自定义 Agent Memory**                                 |
| -------- | --------------------------------- | ---------------------------------------------------- |
| **触发方式** | 运行时**自动触发**（感觉到值得记住的事实）           | Agent prompt 中**手写指令**（如 "Update your agent memory"） |
| **存储路径** | `~/.claude/projects/<项目>/memory/` | 由 prompt 决定，常见如 `.claude/agent-memory/<agent名>/`     |
| **文件格式** | Markdown + YAML frontmatter       | 自由格式 Markdown                                        |
| **谁控制**  | Claude Code 运行时                   | Agent 的定义者（写 prompt 的人）                              |

**本仓库的真实案例：**

`weather-agent` 的 frontmatter 声明了 `memory: project`，`/skills` 列表中也会显示该 agent 的状态和标签信息。

**如何避免混淆：**
1. `memory: project` 是**让运行时自动写入**官方路径——你不需要在 prompt 中写"Update your agent memory"
2. 如果在 prompt 中写了自定义记忆指令，模型会**自主推断路径**，可能创建 `.claude/agent-memory/` 这样的目录
3. 如果两者都用了（如本仓库），结果是：**官方路径空着没写入，自定义路径被模型自主创建并写入了**

---

## 配置优先级总结

```
最高优先级
  ├── 文件路径匹配的 .claude/rules/*.md（带 paths）
  ├── 项目根目录 CLAUDE.md
  ├── 会话中用户指令
  ├── 全局 ~/.claude/CLAUDE.md
  └── Auto Memory（最低持久化优先级）
```

---

## 实用技巧

1. **使用 `<important if="...">` 标签** — 包裹条件性规则，防止 Claude 在文件变长后忽略它们
2. **多个 CLAUDE.md 文件** — 在 monorepo 中利用祖先和后代加载机制
3. **用 settings.json 替代 CLAUDE.md** — 对于"不要加 Co-Authored-By"这样的规则，用配置比用提示更可靠
4. **`/compact` 手动触发** — 在 ~50% 上下文使用时手动 compact，比让自动 compact 触发更好

---

> 下一篇：[08-CLI启动参数-cli-startup-flags.md](./08-CLI启动参数-cli-startup-flags.md)
