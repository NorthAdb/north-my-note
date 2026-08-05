# 大型Monorepo的Skill策略（中文翻译）

> 中文翻译自 [reports/claude-skills-for-larger-mono-repos.md](../reports/claude-skills-for-larger-mono-repos.md)

# 理解 Claude Code 在大型 Monorepo 中的 Skill 发现机制

在 Monorepo 中使用 Claude Code 时，理解 Skill 如何被发现并加载到上下文中，对于有效组织项目级能力至关重要。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 与 CLAUDE.md 的重要区别

**Skill 的加载行为与 CLAUDE.md 文件不同。** CLAUDE.md 会沿目录树向上查找（祖先加载），而 Skill 使用不同的发现机制，侧重于项目内的嵌套目录。

## Skill 的发现方式

### 1. 标准 Skill 位置

Skill 从以下基于作用域的固定位置加载：

| 位置 | 路径 | 适用范围 |
|----------|------|------------|
| 企业级 | 托管设置 | 组织内所有用户 |
| 个人级 | `~/.claude/skills/<skill-name>/SKILL.md` | 你的所有项目 |
| 项目级 | `.claude/skills/<skill-name>/SKILL.md` | 仅当前项目 |
| 插件级 | `<plugin>/skills/<skill-name>/SKILL.md` | 启用该插件的环境 |

### 2. 从嵌套目录自动发现

当你处理子目录中的文件时，Claude Code 会自动发现嵌套 `.claude/skills/` 目录中的 Skill。例如，如果你正在编辑 `packages/frontend/` 中的文件，Claude Code 也会在 `packages/frontend/.claude/skills/` 中查找 Skill。

这支持了各包拥有自己 Skill 的 Monorepo 配置。

## 示例 Monorepo 结构

考虑一个包含多个独立包的典型 Monorepo：

```
/mymonorepo/
├── .claude/
│   └── skills/
│       └── shared-conventions/SKILL.md    # 项目级 Skill
├── packages/
│   ├── frontend/
│   │   ├── .claude/
│   │   │   └── skills/
│   │   │       └── react-patterns/SKILL.md  # 前端专属 Skill
│   │   └── src/
│   │       └── App.tsx
│   ├── backend/
│   │   ├── .claude/
│   │   │   └── skills/
│   │   │       └── api-design/SKILL.md      # 后端专属 Skill
│   │   └── src/
│   └── shared/
│       ├── .claude/
│       │   └── skills/
│       │       └── utils-patterns/SKILL.md  # 共享工具 Skill
│       └── src/
```

## 场景 1：刚在根目录启动 Claude（尚未编辑任何文件）

当你从 `/mymonorepo/` 运行 Claude Code 且尚未编辑任何文件时：

```bash
cd /mymonorepo
claude
# 刚启动——尚未编辑文件
```

| Skill | 在上下文中？ | 原因 |
|-------|-------------|------------|
| `shared-conventions` | **是** | 根目录 `.claude/skills/` 中的项目级 Skill |
| `react-patterns` | **否** | 尚未发现——尚未处理 `packages/frontend/` 中的文件 |
| `api-design` | **否** | 尚未发现——尚未处理 `packages/backend/` 中的文件 |
| `utils-patterns` | **否** | 尚未发现——尚未处理 `packages/shared/` 中的文件 |

## 场景 2：编辑包内文件后

在你要求 Claude 编辑 `packages/frontend/src/App.tsx` 之后：

| Skill | 在上下文中？ | 原因 |
|-------|-------------|------------|
| `shared-conventions` | **是** | 根目录 `.claude/skills/` 中的项目级 Skill |
| `react-patterns` | **是** | 编辑 `packages/frontend/` 中文件时发现 |
| `api-design` | **否** | 尚未发现——尚未处理 `packages/backend/` 中的文件 |
| `utils-patterns` | **否** | 尚未发现——尚未处理 `packages/shared/` 中的文件 |

**关键要点**：嵌套的 Skill 是**按需发现**的——当你处理这些目录下的文件时才会加载。它们不会在会话启动时预加载。

## 关键行为：描述 vs 完整内容

Skill 的描述会被加载到上下文中，使 Claude 知道有哪些可用能力，但**完整 Skill 内容仅在调用时加载**。这是一项重要的优化：

- **描述**：始终在上下文中（在字符预算范围内）
- **完整内容**：调用 Skill 时按需加载

> 注意：预加载了 Skill 的子代理（Subagent）的工作方式不同——完整 Skill 内容会在启动时注入。

## 优先级顺序（当 Skill 名称相同时）

当不同层级的 Skill 名称相同时，优先级高的位置胜出：

| 优先级 | 位置 | 作用域 |
|----------|----------|-------|
| 1（最高） | 企业级 | 组织范围 |
| 2 | 个人级（`~/.claude/skills/`） | 你的所有项目 |
| 3（最低） | 项目级（`.claude/skills/`） | 仅当前项目 |

插件 Skill 使用 `plugin-name:skill-name` 命名空间，因此不会与其他层级冲突。

## 此设计为何适用于 Monorepo

- **包专属 Skill 保持隔离**——在 `packages/frontend/` 中工作的前端开发者获得前端专属 Skill，而不会让后端 Skill 占用上下文。

- **自动发现减少配置**——无需显式注册包级 Skill；它们在你处理这些目录时自动被发现。

- **上下文经过优化**——初始只加载 Skill 描述，嵌套 Skill 按需发现。

- **团队可维护自己的 Skill**——每个包团队可以定义其领域专属的 Skill，而无需与其他团队协调。

## 字符预算考量

Skill 描述在字符预算内加载到上下文中（默认 15,000 字符）。在拥有大量包和 Skill 的大型 Monorepo 中，你可能会达到此限制。

- 运行 `/context` 检查是否有被排除的 Skill 的警告
- 设置 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 环境变量以增加限制

## 最佳实践

1. **将共享工作流放在根目录 `.claude/skills/` 中**——仓库级约定、提交工作流和共享模式。

2. **将包专属 Skill 放在包的 `.claude/skills/` 中**——框架专属模式、组件约定、该包独有的测试工具。

3. **对危险 Skill 使用 `disable-model-invocation: true`**——部署或破坏性操作需要用户显式调用。

4. **保持 Skill 描述简洁**——描述始终在上下文中（在字符预算内），冗长的描述会浪费上下文空间。

5. **在 Skill 名称中使用命名空间**——考虑使用包名前缀（例如 `frontend-review`、`backend-deploy`）以避免混淆。

## 对比：Skill 与 CLAUDE.md 的加载方式

| 行为 | CLAUDE.md | Skill |
|----------|-----------|--------|
| 祖先加载（向上） | 是 | 否 |
| 嵌套/子孙发现（向下） | 是（惰性） | 是（自动发现） |
| 全局位置 | `~/.claude/CLAUDE.md` | `~/.claude/skills/` |
| 项目位置 | `.claude/` 或仓库根目录 | `.claude/skills/` |
| 内容加载 | 完整内容 | 仅描述（调用时加载完整内容） |

---

## 来源

- [Claude Code 文档——通过 Skill 扩展 Claude](https://code.claude.com/docs/en/skills)
- [Claude Code 文档——从嵌套目录自动发现](https://code.claude.com/docs/en/skills#automatic-discovery-from-nested-directories)
