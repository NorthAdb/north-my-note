---
name: claude-memory-vault
description: Claude Code 长期记忆已接入 north-my-note Obsidian vault,所有项目共用
metadata:
  type: reference
---

Claude Code 的长期记忆目录通过用户级 settings 中的 `autoMemoryDirectory` 指向 `E:\north-my-note\claude-memory`(Obsidian vault `north-my-note` 的子文件夹)。

要点:
- 配置位置:`C:\Users\Lenovo\.claude\settings.json` → `"autoMemoryDirectory": "E:\\north-my-note\\claude-memory"`,对机器上所有项目生效。
- 记忆文件是普通 markdown,无必填 frontmatter;可在 Obsidian 里直接查看、编辑、维护链接图谱。
- `MEMORY.md` 是索引,每次会话加载前 200 行 / 25KB;主题文件按需读取。
- vault 根目录 `.obsidian/` 是 Obsidian 配置,不会作为记忆读取;不要把无关普通笔记混入本目录。
