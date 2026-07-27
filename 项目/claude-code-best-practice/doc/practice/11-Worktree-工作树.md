# Worktree（工作树）最佳实践

> 翻译整理自 `https://code.claude.com/docs/zh-CN/worktrees`
> 学习顺序：第 11 篇
> 官方文档：[Worktrees - 并行会话](https://code.claude.com/docs/zh-CN/worktrees)

---

## 什么是 Worktree？

**Worktree（工作树）**是 Git 的一个功能：它在同一个仓库中创建**多个独立的工作目录**，每个目录有自己的文件和分支，但共享相同的 `.git` 提交历史。

用"**分身术**"来理解：

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  主工作区        │    │  worktree A       │    │  worktree B       │
│  main 分支       │    │  feature-auth     │    │  bugfix-123       │
│                  │    │                  │    │                  │
│  D:\project\     │    │  D:\project\     │    │  D:\project\     │
│                  │    │  .claude\        │    │  .claude\        │
│                  │    │  worktrees\auth\ │    │  worktrees\bug\  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └─────────── 共享 .git ────────────────────────┘
```

**在 Claude Code 的场景下**：Worktree 让你可以同时运行多个并行会话，每个会话编辑不同的文件而互不干扰。

---

## Worktree 解决了什么问题？

| 问题 | 没有 Worktree | 有 Worktree |
|------|-------------|------------|
| **并行任务** | 一个 Claude 在改 A 文件，另一个也改 A 文件 → 冲突 | 互不干扰 |
| **上下文隔离** | 一个会话里塞了太多任务 → 混淆 | 每个 worktree 是干净的 git 状态 |
| **实验性修改** | 不敢让 Claude 大改，怕搞乱工作区 | 放心改，worktree 删了也不影响主工作区 |
| **多人协作** | 一个 Claude 在写文档，另一个在重构代码 → 互相打断 | 各干各的 |

---

## 使用方式

### 方式一：CLI 启动 worktree

```bash
# 创建 worktree 并在其中启动 Claude（会在 .claude/worktrees/feature-auth/ 创建）
claude --worktree feature-auth

# 简写 -w
claude -w feature-auth

# 省略名称 → Claude 自动生成（如 bright-running-fox）
claude --worktree
```

打开另一个终端，用不同的名称即可并行运行：

```bash
# 终端 1：构建功能
claude --worktree feature-auth

# 终端 2：修 Bug  
claude --worktree bugfix-123
```

### 方式二：会话中进入 worktree

在 Claude 会话中告诉它 **"在 worktree 中工作"**，它会使用 `EnterWorktree` 工具进入。进入后还可以直接切换到 `.claude/worktrees/` 下的另一个 worktree（之前的保留在磁盘上）。

在桌面应用中，每个新会话会自动创建一个 worktree。

---

## 选择基础分支

### 默认行为：从远程默认分支

```bash
claude --worktree feature-auth
```

↑ 从 `origin/HEAD`（远程默认分支）分出一个干净的 worktree。

### 从本地 HEAD 分支

如果希望 worktree **携带你还没推送的本地更改**（比如在已有功能分支上 work）：

```json
// .claude/settings.json
{
  "worktree": {
    "baseRef": "head"
  }
}
```

这样 worktree 从当前的 `HEAD` 分支，而不是远程默认分支。

### 从 PR 分支

```bash
# 按 PR 编号
claude --worktree "#1234"

# 或用完整的 PR URL
claude --worktree "https://github.com/owner/repo/pull/1234"
```

↑ 从 `origin` 拉取 PR 内容，在 `.claude/worktrees/pr-1234/` 创建 worktree。

---

## .worktreeinclude：复制 gitignored 文件

Worktree 是一个**全新的检出**，所以主仓库里的 `.env`、`.env.local` 等 gitignored 文件不存在。需要一个专门的配置文件来告诉 Claude 自动复制它们。

在项目**根目录**创建 `.worktreeinclude` 文件：

```text
# .worktreeinclude — 使用 .gitignore 语法
.env
.env.local
config/secrets.json
node_modules/.cache
```

**关键规则：**
- 使用 `.gitignore` 语法
- **只有被 gitignore 的文件才会被复制**（已跟踪的文件永不重复）
- 适用于 `--worktree`、子代理 worktrees、桌面应用并行会话

---

## 子代理隔离：`isolation: worktree`

上一章（第 2 篇）讲过 Subagent 的 `isolation` 字段。`isolation: worktree` 让 Subagent 在自己的临时 worktree 中运行：

```yaml
# .claude/agents/safe-refactorer.md
---
name: safe-refactorer
description: 重构代码，但不影响主工作区
isolation: worktree
---
重构 src/ 下的代码，保持 API 不变。
```

**效果：** 多个 Subagent 可以同时修改同一文件的不同版本，互不污染。

```yaml
# .claude/agents/code-writer-a.md
---
name: feature-writer
isolation: worktree
---
# 实现用户认证功能

# .claude/agents/code-writer-b.md
---
name: bug-fixer
isolation: worktree
---
# 修复支付模块的边界条件
```

**注意：** 子代理 worktrees 使用与 `--worktree` 相同的基础分支规则（`worktree.baseRef`），默认从远程默认分支分叉。

---

## Worktree 清理策略

| 场景 | 行为 |
|------|------|
| **无未提交更改 + 无新提交** | worktree 和分支自动删除 |
| **存在未提交更改 / 未跟踪文件 / 新提交** | Claude 提示："保留还是删除？" |
| **非交互式运行（`-p`）** | 不自动清理，需手动 `git worktree remove` |
| **子代理 worktree（无更改）** | 完成后自动删除 |
| **超过 `cleanupPeriodDays`** | 无更改/未推送的 worktree 会被自动扫描删除 |
| **`--worktree` 创建的** | 永远不会被自动扫描删除 |

**注意：** 运行代理时，Claude 会在 worktree 上执行 `git worktree lock`，防止并发清理误删。代理完成后释放锁。

---

## 手动管理

如果想完全控制 worktree 的位置和分支：

```bash
# 在新分支上创建 worktree
git worktree add ../project-feature-a -b feature-a

# 从现有分支创建 worktree
git worktree add ../project-bugfix bugfix-123

# 在 worktree 中启动 Claude
cd ../project-bugfix && claude

# 列出所有 worktrees
git worktree list

# 删除 worktree
git worktree remove ../project-feature-a
```

> **注意：** 在每个 worktree 中需要重新初始化开发环境——安装依赖、设置虚拟环境等。

---

## 非 Git 版本控制

对于 SVN、Perforce、Mercurial 等，需要配置 `WorktreeCreate` 和 `WorktreeRemove` hooks：

```json
{
  "hooks": {
    "WorktreeCreate": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'NAME=$(jq -r .name); DIR=\"$HOME/.claude/worktrees/$NAME\"; svn checkout https://svn.example.com/repo/trunk \"$DIR\" >&2 && echo \"$DIR\"'"
          }
        ]
      }
    ]
  }
}
```

> **注意：** 使用 hook 时，`.worktreeinclude` 不会自动处理。需要在 hook 脚本中自行复制配置文件。

---

## 最佳实践总结

| 场景 | 建议 |
|------|------|
| **并行功能开发** | 每个功能一个 `claude --worktree <功能名>` |
| **同时修 Bug** | 每个 Bug 一个 `claude --worktree bug-编号` |
| **重构（怕搞乱）** | Subagent 设置 `isolation: worktree` |
| **审查 PR** | `claude --worktree "#PR编号"` |
| **实验性修改** | worktree 用完即删，主工作区不受影响 |
| **传敏感配置** | 项目根目录加 `.worktreeinclude` |

### 重要提醒

1. **先在目录中运行一次 `claude` 接受信任** — `--worktree` 首次使用时需要信任确认
2. **将 `.claude/worktrees/` 加入 `.gitignore`** — 防止 worktree 目录在主检出中显示为未跟踪文件
3. **每个 worktree 需单独初始化环境** — 不是共享 node_modules/ 的
4. **使用 `git worktree lock` 保护运行中的 worktree** — Claude 会自动做，手动管理时需要注意

---

## 与相关概念的对比

| 工具 | 隔离什么？ | 如何并行？ |
|------|-----------|-----------|
| **Worktree** | 文件系统更改 | 多个终端 + `--worktree` |
| **Subagent** | 上下文窗口 | 在同一个会话中用 Agent 工具 |
| **Agent Teams** | 整个会话 | 自动协调多个 Claude 会话 |

三者可以配合使用：
- Worktree 处理**文件隔离**
- Subagent 处理**任务委派**
- Agent Teams 处理**会话协调**

---

## 关联文档

| 相关概念 | 文档 | 说明 |
|---------|------|------|
| Subagent | [`02-子代理-subagents.md`](./02-子代理-subagents.md) | `isolation: worktree` 字段详解 |
| Settings | [`05-设置-settings.md`](./05-设置-settings.md) | `worktree.baseRef` 等配置项 |
| CLI 启动参数 | [`08-CLI启动参数-cli-startup-flags.md`](./08-CLI启动参数-cli-startup-flags.md) | `--worktree` / `-w` 参数 |
| 实现参考 | [`../implementation/`](../implementation/) | 本仓库的 Worktree 实际配置 |

---

> ⏭️ 下一篇：你可以回到 [10-学习路径.md](./10-学习路径.md) 继续完整的学习路线
