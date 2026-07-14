---
created: 2026-07-06
updated: 2026-07-06
tags: [github, skill, AI, 学习, 信息聚合, claude-code]
source: "https://mp.weixin.qq.com/s/XYYV68sGT8KFxh6rCJZwkw"
---

# last30days-skill：用 5 万 Star 的 Skill 学习 AI 新知识

> 来源：微信公众号"逛逛GitHub" · 2026-07-06 发布
> 让 AI agent 帮你扒一遍近 30 天国外各大社交媒体的真实讨论，综合成带引用报告

---

## 项目信息

| 项目      |                                    数据                                     |
| ------- | :-----------------------------------------------------------------------: |
| 仓库      | [mvanhorn/last30days-skill](https://github.com/mvanhorn/last30days-skill) |
| ⭐ Stars |                                 **48.8k**                                 |
| 📝 语言   |                                  Python                                   |

---

## 这是什么？

`last30days-skill` 是一个 **Claude Code / AI agent skill**，输入一个话题，它会：

1. 跨平台搜索 Reddit、X、YouTube、TikTok、Hacker News、GitHub 等 10+ 海外平台
2. **深度爬取**：不只看帖子标题，连评论带点赞数一起扒下来
3. **按真实热度排序**：Reddit 几千 upvote 的讨论权重 > 没人看的博客
4. 生成一份**带引用的综合研究简报**

安装后，AI agent 就拥有了"知道最近 30 天全世界在讨论什么"的能力。

---

## 跟普通搜索的区别

| 维度 | 普通搜索 | last30days-skill |
|------|---------|-----------------|
| 覆盖范围 | 单篇网页 | 10+ 平台聚合 |
| 内容深度 | 只看正文 | 正文 + 高赞评论 + 互动数据 |
| 排序依据 | SEO 排名 | 真实热度（upvote/播放量） |
| 输出形式 | 链接列表 | 结构化报告 + 可直接追问的上下文 |

**核心价值**：AI 替你读完了所有相关讨论，你可以随便追问，它都能基于一手信息回答。

---

## 怎么用

```bash
# Claude Code 中安装
/plugin marketplace add mvanhorn/last30days-skill
/plugin install last30days

# Codex/Cursor/Copilot/Gemini CLI 中安装
npx skills add mvanhorn/last30days-skill -g
```

安装后输入：

```
/last30days  最近大家关于 [话题] 的讨论，比较有洞见的地方是啥？
```

然后 AI 会搜索 50+ 条海外信息来源，生成报告后你可以自由追问。

**免费源**：Reddit、Hacker News、Polymarket、GitHub 四个源完全免费，零配置即用。

---

## 学习场景示例

以最近很火的 **Loop Engineering** 为例：

1. 输入 `/last30days 最近关于 loop engineering 的讨论`
2. 等待 7 分钟，它深度搜索了 50 条来源（2 个 Reddit 帖子、10 个 YouTube 视频、15 条 TikTok、12 条 Instagram、10 条 HN、1 个 GitHub）
3. 生成结构化报告
4. 自由追问 AI，因为它已经吃透了所有上下文

> 这个感觉是普通搜索给不了的——普通搜索给你一篇文章读完就完了，你问它问题它不知道上下文。这个不一样，它已经替你读完了所有相关讨论。

---

## 🔗 关联笔记

- 已收录在 [[Clippings/Bilibili/2026-07-04-我的B站AI编程收藏清单.md]] 中
- 类似的 Agent Skills 参见 [[Clippings/GitHub/2026-07-04-GitHub高星项目速览.md]]
