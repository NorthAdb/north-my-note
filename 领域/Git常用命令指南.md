# Git 常用命令指南

## 一、配置相关

### 1. 查看配置
```bash
git config --list                # 查看所有配置
git config user.name             # 查看用户名
git config user.email            # 查看邮箱
```

### 2. 设置配置
```bash
git config --global user.name "你的用户名"      # 全局设置用户名
git config --global user.email "你的邮箱"       # 全局设置邮箱
git config user.name "用户名"                   # 仅当前仓库设置
```

---

## 二、仓库初始化

### 1. 创建新仓库
```bash
git init                         # 在当前目录初始化一个新的 Git 仓库
```

### 2. 克隆远程仓库
```bash
git clone <仓库地址>              # 克隆远程仓库到本地
git clone <仓库地址> <文件夹名>    # 克隆到指定文件夹
```

---

## 三、文件状态与暂存

### 1. 查看状态
```bash
git status                       # 查看当前仓库状态（哪些文件修改了、哪些待提交）
git status -s                    # 简洁模式显示状态
```

### 2. 添加文件到暂存区
```bash
git add <文件名>                  # 添加指定文件到暂存区
git add .                        # 添加所有修改的文件到暂存区
git add *.html                   # 添加所有 html 文件
```

### 3. 取消暂存
```bash
git reset <文件名>                # 将文件从暂存区移除（保留修改）
git restore --staged <文件名>     # 同上（新版本推荐）
```

---

## 四、提交

### 1. 提交到本地仓库
```bash
git commit -m "提交说明"          # 提交暂存区的文件
git commit -am "提交说明"         # 跳过 add，直接提交所有已跟踪文件的修改
```

### 2. 修改最后一次提交
```bash
git commit --amend -m "新的提交说明"   # 修改最近一次提交的说明
```

---

## 五、查看历史

### 1. 查看提交日志
```bash
git log                          # 查看完整提交历史
git log --oneline                # 简洁模式（每个提交一行）
git log --oneline -5             # 只显示最近 5 条
git log --graph                  # 图形化显示分支合并历史
git log --author="用户名"         # 查看指定作者的提交
```

### 2. 查看文件修改历史
```bash
git blame <文件名>                # 查看文件每一行是谁修改的
```

---

## 六、差异比较

```bash
git diff                         # 查看工作区与暂存区的差异
git diff --staged                # 查看暂存区与上次提交的差异
git diff <commit1> <commit2>     # 比较两次提交的差异
```

---

## 七、分支管理

### 1. 查看分支
```bash
git branch                       # 查看本地分支
git branch -a                    # 查看所有分支（包括远程）
git branch -v                    # 查看分支及最后一次提交
```

### 2. 创建与切换分支
```bash
git branch <分支名>               # 创建新分支
git checkout <分支名>             # 切换到指定分支
git checkout -b <分支名>          # 创建并切换到新分支
git switch <分支名>               # 切换分支（新版本推荐）
git switch -c <分支名>            # 创建并切换（新版本推荐）
```

### 3. 合并分支
```bash
git merge <分支名>                # 将指定分支合并到当前分支
```

#### 什么是分支合并？
分支合并就是把一个分支的代码变更**整合到另一个分支**上。





**示例场景：**
```
main 分支（主分支）：稳定的代码，可以上线
    │
    └── feature 分支（功能分支）：正在开发"登录功能"
```

**合并过程图示：**
```
时间线：
main:     A ────────────────── M (合并后)
           \                  /
feature:    └── B ── C ── D ──┘

A = 原来 main 的代码
B、C、D = 在 feature 分支上的提交
M = 合并后，main 拥有了 feature 的所有改动
```

**实际操作步骤：**
```bash
# 1. 先切换到目标分支（你想把代码合并到哪里）
git checkout main

# 2. 执行合并（把 feature 的内容合并过来）
git merge feature
```

#### 为什么需要分支合并？
| 场景 | 说明 |
|------|------|
| **功能开发** | 新功能在独立分支开发，完成后合并到主分支 |
| **Bug修复** | 修复分支完成后合并回主分支 |
| **团队协作** | 每个人在自己的分支开发，最后合并到一起 |
| **保护主分支** | 主分支始终保持稳定，开发在其他分支进行 |

#### 合并的三种情况
1. **快进合并 (Fast-forward)**：如果目标分支没有新提交，直接移动指针，最简单
2. **三方合并**：如果两个分支都有新提交，Git 会自动合并
3. **冲突 (Conflict)**：如果两个分支修改了同一处代码，需要手动解决冲突

### 4. 删除分支
```bash
git branch -d <分支名>            # 删除已合并的分支
git branch -D <分支名>            # 强制删除分支
```

---

## 八、远程仓库

### 1. 查看远程仓库
```bash
git remote                       # 查看远程仓库名称
git remote -v                    # 查看远程仓库详细信息（地址）
```

### 2. 添加远程仓库
```bash
git remote add origin <仓库地址>  # 添加远程仓库，命名为 origin
```

### 3. 推送到远程
```bash
git push origin <分支名>          # 推送到远程指定分支
git push -u origin main          # 首次推送并设置上游分支
git push                         # 之后可以直接 push
```

### 4. 从远程拉取（git pull）
```bash
git fetch                        # 获取远程更新（不合并）
git pull                         # 获取并合并远程更新 = fetch + merge
git pull origin <分支名>          # 拉取指定分支
git pull --rebase                # 拉取并用 rebase 方式合并：会将本地提交“重放”到远程更新之后（不会生成 merge commit，历史更线性）。注意：rebase 会重写提交历史，不建议在多人协作的共享分支上使用，以免影响他人。
#### git fetch vs git pull 区别
| 命令 | 作用 | 适用场景 |
|------|------|----------|
| `git fetch` | 只下载远程更新到本地，不自动合并 | 想先查看更新内容再决定是否合并 |
| `git pull` | 下载远程更新 + 自动合并到当前分支 | 日常更新，快速同步远程代码 |

**git pull 使用场景：**
- 每天开始工作前，先拉取最新代码
- 推送前先 pull，避免冲突
- 团队协作时同步同事的代码

**注意事项：**
- 如果本地有未提交的修改，pull 可能会产生冲突
- 建议先 commit 或 stash 本地修改，再执行 pull

### 5. 修改/删除远程仓库
```bash
git remote set-url origin <新地址>   # 修改远程仓库地址
git remote remove origin             # 删除远程仓库
```

---

## 九、撤销与回退

### 1. 撤销工作区修改
```bash
git checkout -- <文件名>          # 撤销文件修改（恢复到暂存区或上次提交）
git restore <文件名>              # 同上（新版本推荐）
```

### 2. 回退版本
```bash
git reset --soft HEAD~1          # 回退一个版本，保留修改在暂存区
git reset --mixed HEAD~1         # 回退一个版本，保留修改在工作区（默认）
git reset --hard HEAD~1          # 回退一个版本，丢弃所有修改（危险！）
git reset --hard <commit-id>     # 回退到指定提交
```

### 3. 撤销某次提交（生成新提交）
```bash
git revert <commit-id>           # 撤销某次提交的内容（不改变历史）
```

---

## 十、暂存工作区（stash）

```bash
git stash                        # 暂存当前修改（工作区变干净）
git stash list                   # 查看暂存列表
git stash pop                    # 恢复最近一次暂存并删除记录
git stash apply                  # 恢复最近一次暂存（保留记录）
git stash drop                   # 删除最近一次暂存记录
```

---

## 十一、标签管理

```bash
git tag                          # 查看所有标签
git tag v1.0.0                   # 创建轻量标签
git tag -a v1.0.0 -m "说明"       # 创建附注标签
git push origin v1.0.0           # 推送标签到远程
git push origin --tags           # 推送所有标签
git tag -d v1.0.0                # 删除本地标签
```

---

## 十二、常用工作流程示例

### 日常开发流程
```bash
# 1. 拉取最新代码
git pull

# 2. 创建功能分支
git checkout -b feature/新功能

# 3. 开发并提交
git add .
git commit -m "完成新功能"

# 4. 切回主分支并合并
git checkout main
git merge feature/新功能

# 5. 推送到远程
git push

# 6. 删除功能分支
git branch -d feature/新功能
```

---

## 十三、实用技巧

### .gitignore 文件
在项目根目录创建 `.gitignore` 文件，指定不需要 Git 跟踪的文件：
```
node_modules/
*.log
.env
dist/
.DS_Store
```

### 查看某个命令的帮助
```bash
git help <命令>                   # 例如: git help commit
git <命令> -h                     # 简短帮助
```

---

## 十四、Pull Request（拉取请求）

### 什么是 Pull Request？

**Pull Request (PR)** 不是 Git 命令，而是 GitHub/GitLab/Gitee 等平台的**协作功能**。

**核心含义**：请求项目维护者**拉取（pull）你的代码**并合并。

### 为什么叫 Pull Request 而不是 Push Request？

| 角色 | 动作 |
|------|------|
| 你 | 我改好了代码，**请求**你来拉取合并 |
| 维护者 | 好的，我来 **pull（拉取）** 你的代码 |

**名字是从维护者角度起的**：你请求对方来 pull，所以叫 Pull Request。

### Push vs Pull Request 区别

| 操作 | 类型 | 适用场景 |
|------|------|----------|
| `git push` | Git 命令 | 推送到**你有权限**的仓库 |
| Pull Request | 平台功能 | **没有权限**直接推送，需要审核 |

**类比**：
- `push` = 你有钥匙，自己开门进去
- `Pull Request` = 你没钥匙，按门铃请主人开门 🚪

### Pull Request 流程

```
你改代码 → 提交到你的分支/仓库 → 发起 Pull Request
                                      ↓
                              审核者查看代码
                                      ↓
                         ┌─── 通过 → 合并到目标分支 ✅
                         │
                         └─── 不通过 → 继续修改 🔄
```

### 为什么需要 Pull Request？

| 作用 | 说明 |
|------|------|
| **代码审核** | 合并前让他人检查代码质量 |
| **讨论交流** | 在 PR 中评论、提建议 |
| **保护主分支** | 防止未审核的代码直接进入 |
| **记录历史** | 每个功能的修改和讨论都有迹可循 |

---

> 📝 **提示**：多练习是掌握 Git 的最好方法！
