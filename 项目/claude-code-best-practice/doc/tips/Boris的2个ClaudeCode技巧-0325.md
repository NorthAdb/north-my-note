# Squash合并与PR大小分布——来自Boris Cherny的技巧（中文翻译）

> 中文翻译自 [tips/claude-boris-2-tips-25-mar-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-2-tips-25-mar-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年3月25日分享的见解总结。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 1/ 单日266次贡献——始终使用Squash

Boris分享了他的GitHub贡献图，显示**3月24日有266次贡献**——来自 **141个PR，始终squash合并**，每个PR中位数**118行**。

- Squash合并将分支中的所有提交合并为目标分支上的一个提交——保持历史清晰和线性
- 每个PR = 一个提交，便于回滚整个功能并简化 `git bisect`
- 在高速度AI辅助工作流中（每天141个PR），squash是务实的选择——分支内单独的"修复lint"、"试试这个"等提交只是噪音

<a href="https://x.com/bcherny/status/2038552880018538749"><img src="assets/boris-26-3-25/1.png" alt="Boris Cherny —— 266次贡献，始终squash" width="50%" /></a>

---

## 2/ PR大小分布——保持PR小巧

Boris分享了那141个PR的大小分布，总计**45,032行变更**（增删合计）：

| 指标 | 行数（增+删） | 含义 |
|--------|---------------:|---------|
| **p50** | **118** | PR大小中位数——一半的PR不超过118行 |
| p90 | 498 | 90%的PR在500行以内 |
| **p99** | **2,978** | 只有约1个PR超过了~3K行 |
| 最小值 | 2 | 最小的PR——一个快速的2行修复 |
| 最大值 | 10,459 | 最大的单个PR——可能是迁移或生成的代码 |

- **中位数118行**意味着大多数PR是聚焦且可审查的，即使每天141个PR
- 分布严重右偏——偶尔的大PR不可避免（批量重命名、迁移），但常态是小而精
- 小PR降低合并冲突风险，更容易审查，与squash合并完美配合以实现干净的撤销

<a href="https://x.com/bcherny/status/2038552880018538749"><img src="assets/boris-26-3-25/2.png" alt="Boris Cherny —— PR大小分布表" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — March 25, 2026](https://x.com/bcherny)
