# CLAUDE.md 使用指南 — 文章总结

> 原文：《面试官皱眉："Claude Code 你用了半年，CLAUDE.md 平时怎么维护？"》
> 作者：小林

---

## 一句话总结

**CLAUDE.md 是给 Claude 的「入职手册」，不是给开发者看的 README。核心原则：200 行黄金线、短具体告诉为什么、持续更新。**

---

## 1. CLAUDE.md 是什么？

- 项目根目录下的一个 markdown 文件，每次 Claude Code 启动时**自动加载**进上下文。
- 它不是「可选的提示」，而是**默认的前提（ground truth）**——在输入第一个问题之前，Claude 就已经读过它了。
- README 是写给人看的，CLAUDE.md 是写给 Agent 看的，读者不同、密度不同。
- 加载机制：从工作目录一路往上爬到根目录，每层读取 `CLAUDE.md` 和 `.claude/CLAUDE.md`，全部合并。

---

## 2. 写多了反而废

| 行数 | 规则遵守率 |
|------|-----------|
| ≤ 200 行 | 约 92% |
| ≥ 400 行 | 明显下降 |
| 200 行拆成 5×30 行模块化 | **约 96%** |

### 原因
- **Token 经济**：每行都消耗 token，挤压对话和思考空间。
- **注意力稀释**：规则越多，每条规则的权重越被摊薄。

### 三类该删的「负资产」
| 类型 | 例子 | 问题 |
|------|------|------|
| 复述型 | 把架构文档复制进去写 100 行 | 架构会变，规则过时 |
| 愿望型 | "希望测试覆盖率达到 90%" | Claude 无法判断愿望和现实的差距 |
| 术语表型 | "Repo 指 repository" | Claude 本来就知道通用术语 |

---

## 3. 有效规则的四个原则

### 短
控制在 200 行以内，给后续规则留空间。

### 具体（可验证）
| 无效 ❌ | 有效 ✅ |
|----------|---------|
| 测试一下你的修改 | 提交前跑 `npm test` |
| 保持目录整洁 | API 处理函数放在 `src/api/handlers/` |
| 用好的命名 | 组件用 PascalCase，工具用 kebab-case |

### 告诉为什么
规则要附上理由。例如：
> "不要在测试里写入生产数据库 —— 因为去年有一次测试不小心把 users 表清空了。"

Claude 知道原因后，能在边界情况做出正确判断，而不是机械执行。

### 持续更新
- Claude 在哪儿错两次以上 → 加一条防御规则。
- 老规则要删，**错误的规则比没有规则更糟**。
- 建议每 1-2 周 review 一次。

---

## 4. CLAUDE.md 的层级结构

```
~/.claude/CLAUDE.md          # 全局：跨项目个人偏好（如"用中文回复"）
my-project/
├── CLAUDE.md                # 项目根：技术栈、命令、硬约束（每次必加载）
├── frontend/
│   └── CLAUDE.md            # 模块级：前端约定（按需加载）
├── backend/
│   └── CLAUDE.md            # 模块级：后端约定（按需加载）
└── .claude/
    └── rules/
        ├── testing.md       # path-scoped：只在改测试文件时加载
        ├── api-design.md    # path-scoped：只在改 API 文件时加载
        ├── security.md
        └── ui-components.md
```

### path-scoped rules 示例
```yaml
---
paths: ["**/*.test.ts", "**/*.spec.ts"]
---
# 测试规则
- 用 describe / it，不用 test()
- mock 外部依赖必须用 vi.mock
- 每个测试只写一个断言
```

---

## 5. 维护工具

| 命令/方式 | 作用 |
|-----------|------|
| `/init` | 自动扫描代码库，生成 CLAUDE.md 草稿 |
| `/memory` | 会话中快速编辑 CLAUDE.md |
| `Plan Mode`（Shift+Tab×2） | 动手前先出计划，计划质量依赖 CLAUDE.md |
| 规则触发标准 | Claude 错两次以上 → 加一条新规则 |

---

## 6. 跨工具兼容：AGENTS.md

- OpenAI Codex 使用 `AGENTS.md`，作用和 CLAUDE.md 几乎一样。
- 同时用两个工具时：把所有规则写在 `AGENTS.md` 里，`CLAUDE.md` 只写一行 `@AGENTS.md` 引用它。
- 一份文件、两个工具、零重复维护。

---

## 7. 推荐模板结构（6 段，50 行以内）

```markdown
# CLAUDE.md

## 1. Project Overview（2-3 行：技术栈 + 定位）
## 2. Commands（常用命令：安装/启动/测试/检查）
## 3. Architecture（三句话指路，不展开详情）
## 4. Conventions（团队真实在用的命名、格式约定）
## 5. Hard Constraints（硬约束，越界一次就补，附带 why）
## 6. Gotchas（踩过的坑，Claude 无法从代码推断的经验）
```

---

## 三句话精华

1. **CLAUDE.md 是给 Agent 的入职手册，不是给人的 README。**
2. **200 行是黄金线，每行都吃 token，多写不如不写。**
3. **具体可验证、告诉 why、持续更新 —— 三条铁律压过一切技巧。**

---

## 参考资料

- [Anthropic 官方文档：CLAUDE.md 使用指南](https://docs.anthropic.com/en/docs/claude-code/claude-md)
- [Anthropic Help Center：Give Claude context with CLAUDE.md](https://support.claude.com/en/articles/14553240)
- [claudeguide.io：How to Write Effective CLAUDE.md Files](https://claudeguide.io/claude-md-effective-patterns)
- [claude-codex.fr：Mastering CLAUDE.md（6 段式结构）](https://claude-codex.fr/en/prompting/claude-md/)
- [SFEIR Institute：The CLAUDE.md Memory System Deep Dive（实测数据来源）](https://institute.sfeir.com/en/claude-code/claude-code-memory-system-claude-md/deep-dive/)
