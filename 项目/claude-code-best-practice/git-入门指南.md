# Git 仓库管理入门指南

## 1. Clone 仓库

```bash
# HTTPS 方式
git clone https://github.com/用户名/仓库名.git
cd 仓库名

# SSH 方式（需先配置 SSH Key）
git clone git@github.com:用户名/仓库名.git
cd 仓库名
```

> clone 会自动将 `origin` 设为远程仓库地址，一般无需手动指定。

---

## 2. 确认基本信息

```bash
git remote -v          # 查看远程仓库地址
git log --oneline -5   # 查看最近提交历史
git branch -a          # 查看本地和远程所有分支
git status             # 查看当前工作区状态
```

---

## 3. 设置用户信息

如果不想使用全局默认配置，可为当前仓库单独设置：

```bash
git config user.name "你的名字"
git config user.email "你的邮箱@example.com"
```

查看所有配置：

```bash
git config --list                   # 查看全部配置
git config user.name                # 查看单项配置
git config --list --show-origin     # 查看配置来源（系统/全局/本地）
```

---

## 4. 添加必要项目文件

| 文件 | 说明 |
|------|------|
| `.gitignore` | 排除不应提交的文件（venv、node_modules、\_\_pycache\_\_、.env 等） |
| `README.md` | 项目说明文档 |
| `LICENSE` | 开源许可协议（公开仓库建议添加） |

---

## 5. 管理远程仓库

```bash
# 修改远程仓库地址
git remote set-url origin 新地址

# 添加额外远程（如 forking 工作流的上游仓库）
git remote add upstream https://github.com/原作者/仓库.git

# 拉取上游更新
git fetch upstream
git merge upstream/main

# 删除远程
git remote remove 远程名称
```

---

## 6. 日常操作流程

```bash
# 拉取最新代码
git pull                     # = git fetch + git merge

# 创建新分支开发
git checkout -b my-feature

# 查看变更
git status                   # 查看文件状态
git diff                     # 查看具体改动

# 提交
git add 文件名               # 添加指定文件
git commit -m "feat: 描述信息"

# 推送新分支
git push -u origin my-feature

# 更新远程分支列表
git remote update origin --prune
```

---

## 7. 快速检查清单

Clone 一个新仓库后，建议依次确认：

- [ ] `git remote -v` — 远程仓库地址是否正确
- [ ] `git config user.name` / `git config user.email` — 用户信息是否合适（尤其是 fork 的项目）
- [ ] 是否存在 `.gitignore`，是否需要补充规则
- [ ] 确认分支策略：直接 main 开发，还是 feature branch + PR
- [ ] 如果是 fork，是否要添加 `upstream` 远程仓库

---

## 附录：常用 Git 配置参考

### 配置作用域（优先级从低到高）

| 级别 | 配置文件路径 | 命令 |
|------|-------------|------|
| 系统 | `C:/Program Files/Git/etc/gitconfig` | `git config --system` |
| 全局 | `~/.gitconfig` | `git config --global` |
| 本地 | `.git/config` | `git config`（无参数） |

### 典型全局配置（Windows）

```ini
[user]
    name = YourName
    email = your@email.com
[core]
    editor = code --wait          # 默认编辑器
    autocrlf = true               # Windows CRLF 自动转换
[init]
    defaultBranch = main
[credential]
    helper = manager              # Windows 凭据管理器
```
