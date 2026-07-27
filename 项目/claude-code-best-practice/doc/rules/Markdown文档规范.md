# Markdown文档规范（中文翻译）

> 中文翻译自 [.claude/rules/markdown-docs.md]

---
paths:
  - "**/*.md"
---

## 文档标准

- 保持文件聚焦且简洁 — 每个文件一个主题
- 文档之间使用相对链接（例如 `../best-practice/claude-memory.md`），而不是绝对 GitHub URL
- 在最佳实践和报告文档的顶部包含返回导航链接（参见现有文件的模式）
- 添加新概念或报告时，更新 README.md 中的相应表格（概念或报告）

## 结构约定

- 最佳实践文档放在 `best-practice/`
- 实现文档放在 `implementation/`
- 报告放在 `reports/`
- 技巧放在 `tips/`
- 更新日志跟踪放在 `changelog/<category>/`

## 格式

- 使用表格进行结构化比较（参见 README 概念表作为参考）
- 在链接最佳实践或实现文档时，使用 `!/tags/` 的徽章图片以保持视觉一致性
- 保持标题层级 — 不要跳级（例如，不要从 `##` 跳到 `####`）
