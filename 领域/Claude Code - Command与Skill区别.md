# Claude Code：Command 与 Skill 的区别

> 结论：在当前版本中，custom slash commands 已并入 skills 系统，二者在**形式**和**调用方式**上基本统一；仅在能力范围和历史遗留部分仍有区别。

## 一、历史上的区别（旧版本）

| | Command | Skill |
|---|---|---|
| 触发方式 | 仅支持手动 `/xxx` | 仅支持 Claude 自主语义匹配触发 |
| 内容形态 | 单个 md 文件，纯 prompt 模板 | 目录 + SKILL.md，可带脚本/资源 |
| 设计初衷 | 高频操作的快捷方式 | 补充模型自身缺乏的领域知识/工具链 |

## 二、当前状态：两者已合并

- 推荐格式统一为 `.claude/skills/<name>/SKILL.md`
- 该格式**同时支持**：
  1. 手动 `/name` 显式调用（原 command 的能力）
  2. Claude 根据 frontmatter 中的 description 自主判断触发（原 skill 的能力）
- 若新旧格式重名（如 `.claude/commands/review.md` 与 `.claude/skills/review/SKILL.md` 同时存在），**skill 版本优先生效**
- 旧的 `.claude/commands/*.md` 文件仍可正常工作，只是官方建议逐步迁移到 skills 格式

**一句话**：现在的自定义 slash command，本质上就是"没有额外特性的 skill"。

## 三、现在还剩下的真实区别

1. **Skill 是能力的超集**
   - 可以是目录，能打包脚本、参考资料等附属文件
   - frontmatter 可配置自动触发时机、可调用范围等
   - Command（单文件形式）不具备这些扩展能力

2. **内置固定命令是独立存在的第三类**
   - 如 `/clear`、`/compact`、`/help`、`/model`
   - 硬编码在 CLI 内部，**不涉及 AI 推理**，不可自定义
   - 与 skills 系统完全无关，不会被合并

3. **平台适用范围不同**
   - Skill 不局限于 Claude Code，Claude.ai / Claude Desktop 也支持，可跨平台复用
   - Command（严格意义上）主要是 Claude Code 内的概念

## 四、实践建议

新建自定义能力时，直接使用 `.claude/skills/<name>/SKILL.md` 格式即可：
- 触发方式（手动 `/` 还是自动语义匹配）只是 frontmatter 里的配置项
- 不再需要在"做成 command 还是 skill"之间做架构层面的选择
