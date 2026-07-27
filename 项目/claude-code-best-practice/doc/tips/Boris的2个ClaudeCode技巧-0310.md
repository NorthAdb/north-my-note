# 代码审查和测试时计算——来自Boris Cherny的技巧（中文翻译）

> 中文翻译自 [tips/claude-boris-2-tips-10-mar-26.md](https://github.com/shanraisshan/claude-code-best-practice/blob/main/tips/claude-boris-2-tips-10-mar-26.md)

Claude Code创建者Boris Cherny（[@bcherny](https://x.com/bcherny)）于2026年3月10日分享的见解总结。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 1/ 引入代码审查

Claude Code新功能：**代码审查**。一个代理团队对每个PR进行深度审查。

- 首先为Anthropic自己的团队打造——今年每位工程师的代码产出提升了**200%**，而审查成了瓶颈
- Boris已经使用了几周，发现它能捕捉到许多他原本不会注意到的真实bug
- 当PR打开时，Claude会派遣一个代理团队寻找bug

<a href="https://x.com/bcherny/status/2031089411820228645"><img src="assets/boris-26-3-10/0.png" alt="Boris Cherny 宣布代码审查" width="50%" /></a>

---

## 2/ 测试时计算与多上下文窗口

粗略来说，你投入到编码问题上的token越多，结果就越好。Boris称之为**测试时计算**。

- 使用**单独的上下文窗口**会让结果更好——这就是子代理起作用的原因，也是为什么一个代理可能引入bug，而另一个（使用完全相同的模型）能找到它们
- 类似于工程团队：如果Boris引入了一个bug，他同事审查代码可能比他本人更可靠地发现它
- 最终，代理可能会写出完美无bug的代码——在此之前，**多个不相关的上下文窗口**通常是一个好方法

<a href="https://x.com/bcherny/status/2031151689219321886"><img src="assets/boris-26-3-10/1.png" alt="Boris Cherny 关于测试时计算" width="50%" /></a>

---

## 来源

- [Boris Cherny (@bcherny) on X — March 10, 2026](https://x.com/bcherny)
