---
created: 2026-07-21
updated: 2026-07-21
tags: [GitHub, 开源, 新手, Git, PR, 教程]
source:
  - "https://github.com/firstcontributions/first-contributions"
  - "https://firstcontributions.github.io/"
---

# first-contributions：开源初体验指南

> 🚀✨ **Helping beginners to contribute to open source projects**

---

## 项目概况

| 项目          | 信息                                                                                                  |
| ----------- | --------------------------------------------------------------------------------------------------- |
| **仓库**      | [firstcontributions/first-contributions](https://github.com/firstcontributions/first-contributions) |
| **Star**    | ⭐ ~55k                                                                                              |
| **Fork**    | 🍴 ~106k（学习型项目中最高的 fork/star 比）                                                                     |
| **License** | MIT                                                                                                 |
| **官网**      | [firstcontributions.github.io](https://firstcontributions.github.io/)                               |
| **创建时间**    | 2016-09-20                                                                                          |

> **核心理念**：通过一个简单的练习，让零基础新手体验完整的 GitHub 开源贡献流程。

---

## 为什么有 106k Fork？

这个项目的 **fork 数远超 star 数**（约 2:1），因为每个学习者都需要 fork 仓库来提交练习 PR，完成后再保留 fork。它本质上是**全球最大规模的 Git 实操课堂**。

---

## 项目内容

仓库包含三个核心文件：

| 文件 | 作用 |
|------|------|
| `README.md` | 多语言教程（50+ 语言） |
| `Contributors.md` | 贡献者名单（练习 PR 就是把自己的名字加到这里） |
| `docs/translations/` | 各国语言翻译文件夹 |

---

## 贡献流程（完整步骤）

### Step 1：Fork 仓库
在 GitHub 页面点击 Fork 按钮，将仓库复制到自己的账号下。

### Step 2：Clone 到本地
```bash
git clone git@github.com:<你的用户名>/first-contributions.git
cd first-contributions
```

### Step 3：创建分支
```bash
git switch -c add-你的名字
```
> 旧版 Git 可使用 `git checkout -b add-你的名字`

### Step 4：修改文件
打开 `Contributors.md`，在文件中间任意位置加上你的名字。

### Step 5：提交修改
```bash
git add Contributors.md
git commit -m "Add 你的名字 to Contributors list"
```

### Step 6：Push 到 GitHub
```bash
git push -u origin add-你的名字
```

### Step 7：提交 Pull Request
在 GitHub 上点击 **Compare & pull request** 按钮，填写说明后提交。

### Step 8：等待合并
项目维护者会合并你的 PR，收到通知邮件即完成 🎉

---

## 适合人群

| 人群 | 说明 |
|------|------|
| **Git 新手** | 第一次使用 Git/GitHub 的人 |
| **未参与过开源的人** | 想为开源做贡献但不知道怎么开始 |
| **需要练习 PR 流程的人** | 团队协作前熟悉 fork → clone → PR 流程 |
| **教学中** | 高校/培训机构用来教 Git 工作流 |

---

## 关联组织

| 项目 | Star | 说明 |
|------|:----:|------|
| [first-contributions](https://github.com/firstcontributions/first-contributions) | 55k | 主项目 |
| [firstcontributions.github.io](https://github.com/firstcontributions/firstcontributions.github.io) | 454 | Web 应用，支持浏览适合新手的开源项目 |
| [Open-Source-Programs](https://github.com/firstcontributions/Open-Source-Programs) | 14 | 开源项目列表 |

### Web 应用功能

官网网站提供：
- **Project Discovery**：浏览适合初学者的开源项目
- **Issue Integration**：直接从 GitHub 获取 "good first issue" 和 "help wanted" 标签
- **Bento Layout**：现代卡片式布局
- **实时数据**：从 GitHub 仓库获取最新 Issue

---

## 心得与建议

### 完成后可以做什么

1. **到官网分享首次贡献**：`https://firstcontributions.github.io/#social-share`
2. **继续练习**：试试 [code-contributions](https://github.com/roshanjossey/code-contributions) 项目
3. **找真实项目贡献**：在官网浏览 "good first issue" 列表

### 常见问题

| 问题                 | 解决                                                           |
| ------------------ | ------------------------------------------------------------ |
| `git switch` 找不到命令 | 使用 `git checkout -b` 代替                                      |
| Push 时认证失败         | GitHub 2021 年起不再支持密码认证，配置 SSH Key 或 Personal Access Token    |
| Clone 用了 HTTPS     | 设置 SSH：`git remote set-url origin git@github.com:用户名/仓库.git` |

---

## 🔗 关联笔记

- [[领域/AI Agent 智能体学习路线 2026]] — 开源学习路径中的实践项目
- [[项目/Javase/JavaSE 学习清单]] — 用 Git 管理学习进度的实践
- [[Clippings/GitHub/2026-07-04-GitHub高星项目速览]] — GitHub 高星项目汇总
