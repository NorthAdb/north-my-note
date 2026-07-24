---
created: 2026-07-07
updated: 2026-07-07
tags: [小红书, claude-code, skills, AI, 教程]
source: "https://www.xiaohongshu.com/discovery/item/6a4355dd0000000022019eb8"
---

# 小红书"点点"：Claude Code 大全 - AI Agent Skills 教程

> 来源：小红书用户"点点" · 2026-07-07 采集
> 笔记主题：Claude Code Skills 的全面指南

---

## 一、Claude Code 是什么

Claude Code 是 Anthropic 推出的 **Agent 化 CLI 工具**，让 Claude 不只是聊天，而是能：

- 直接读写本地文件
- 执行 Shell / Bash 命令
- 跑测试、部署代码
- 调用 MCP（Model Context Protocol）工具
- 在终端、VS Code、JetBrains、桌面 App 等多个环境运行

---

## 二、核心概念：Skills（技能）

Skills 是让 Agent "专业化" 的方式，本质是一组**结构化的提示词 + 工具调用模板**。

### Skills 的三大组成

| 组件 | 作用 |
|------|------|
| **系统提示（System Prompt）** | 定义 Agent 角色与行为边界 |
| **工具声明（Tool Declaration）** | 告诉模型能用什么（读文件 / 写文件 / 跑命令） |
| **Few-shot 示例** | 教模型如何正确组合工具完成任务 |

### Skills 的加载方式

- **按需加载**：Skills 不会全部加载到上下文中，只在需要时调用
- **路径限定**：可以指定 Skill 只对特定目录/文件类型生效
- **市场安装**：通过 `/plugin marketplace add` 或 `npx skills add` 安装社区 Skill

---

## 三、常见 Skills 分类

| 分类 | 示例 Skills |
|------|------------|
| **代码类** | code-review、refactor、test-generator、code-format |
| **文档类** | doc-writer、changelog、README-generator、api-doc |
| **运维类** | deploy-helper、log-analyzer、docker-manager |
| **数据类** | sql-runner、csv-explorer、data-analyzer |
| **研究类** | web-search、paper-summarizer、last30days |
| **工程类** | tdd、grilling、wayfinder、adr |

---

## 四、写一个 Skill 的最小模板

```markdown
# Skill: xxx

## 角色
你是 [具体角色]，擅长 [具体能力]。

## 可用工具
- read_file
- write_file
- bash

## 工作流
1. 先分析用户需求
2. 调用对应工具
3. 验证结果
4. 输出结论

## 输出格式
- 必须包含 ...
```

---

## 五、最佳实践

| 原则 | 说明 |
|------|------|
| ✅ **单一职责** | 一个 Skill 只做一件事 |
| ✅ **明确边界** | 写清楚"不能做什么" |
| ✅ **提供示例** | 减少模型幻觉 |
| ✅ **可验证** | 输出能跑测试 / 截图证明 |
| ❌ **避免超长 prompt** | >2k tokens 会稀释注意力 |
| ✅ **版本记录** | 在 Skill 文件头标注版本和变更 |

---

## 六、学习路线建议

1. 先跑通官方 `claude` CLI，体验基础功能
2. 读 Anthropic 官方的 Skills 示例
3. 从改造现有 Skill 开始（不要从零写）
4. 在 GitHub 上 fork 别人的 `.claude` 目录练手
5. 关注社区生态：addyosmani/agent-skills、mattpocock/skills 等合集

---

## 🔗 关联笔记

- [[领域/Claude Code记忆系统与Agent记忆架构.md]] — Claude Code 记忆系统详解
- [[领域/AI Agent 智能体学习路线 2026.md]] — 完整学习路线
- [[领域/Python vs TypeScript AI时代编程语言之争.md]] — Agent 开发语言选择
- [[2026-07-04-GitHub高星项目速览]] — Skills 相关高星项目
