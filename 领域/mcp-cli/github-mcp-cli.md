# GitHub MCP、gh CLI，以及 tool calling

同一件事——查 PR、发评论、建仓库——在本机可以走两条路。它们打的是同一套 GitHub API，差别在于模型怎么把请求交出去。

本文记录 2026-10-07 在这台机器上看到的情况：

- `gh` 2.96.0，已登录 GitHub 账号 **NorthAdb**（权限包括 `repo`、`read:org`、`gist`）
- Cursor 里的 GitHub MCP 命名空间 `user-github` 处于可用状态

## 三层关系

一次对话里，模型并不自己访问 GitHub。它产出一段结构化请求，由 Cursor 执行，再把结果送回模型。这个机制叫 **tool calling**。

```text
你：列出这个仓库未合并的 PR
        │
        ▼
模型判断：单靠对话答不了，需要外部数据
        │
        ▼
发出一次 tool call（名字 + JSON 参数）
        │
        ├─ 路径 A：GitHub MCP 工具 list_pull_requests
        │           Cursor → MCP 服务 → GitHub API → JSON
        │
        └─ 路径 B：内置 Shell 工具
                    Cursor 执行 gh pr list → 终端文本
        │
        ▼
结果写回对话，模型再用自然语言总结
```

**Function calling** 是 tool calling 里最常见的一种调用形态：模型选出一个名字，并填好 JSON 参数。早期接口就叫 function call。后来能力变多了——跑命令、搜网页、读文件、连 MCP——统一改称 **tool**。

- 一次 function call：按约定模式调用某一个函数（名字 + 参数）。
- tool calling：整套循环。发现工具、发出调用、执行、把结果喂回模型，直到模型能够直接回答。

在 Cursor 里，`Read`、`Shell`、`CallDynamicTool` 都是 tool。GitHub MCP 上的 `list_pull_requests` 也是 tool。`gh` 本身不是模型直接看到的 tool，它是 Shell 这条 tool 里跑起来的一个程序。

走 MCP 时，模型侧发出的内容大致是：

```json
{
  "name": "list_pull_requests",
  "arguments": {
    "owner": "NorthAdb",
    "repo": "ai-research-hub",
    "state": "open",
    "perPage": 10
  }
}
```

走 CLI 时，模型调用的是 Shell，GitHub 命令只是参数里的字符串：

```json
{
  "name": "Shell",
  "arguments": {
    "command": "gh pr list --repo NorthAdb/ai-research-hub --state open --limit 10"
  }
}
```

MCP 的参数由 JSON Schema 约束：缺 `owner` 就发不出去，枚举字段只能取规定的值。CLI 靠模型记住 `gh` 的旗标和引号，写错了要等终端报错再改。

## GitHub MCP 是什么

MCP（Model Context Protocol）是一套让编辑器连接外部工具服务的协议。GitHub 提供一个 MCP server，把 API 包成一批带说明和参数表的工具。模型先读到这些工具的 schema，再按字段填参数。认证配在 Cursor 的 MCP 连接上，和这场对话绑在一起。

`user-github` 里当前能用的工具包括：

| 类别 | 工具 |
|---|---|
| 身份与仓库 | `get_me`、`search_repositories`、`create_repository`、`fork_repository`、`list_branches`、`list_commits` |
| Issue | `list_issues`、`search_issues`、`issue_read`、`issue_write`、`add_issue_comment`、`sub_issue_write` |
| Pull request | `list_pull_requests`、`search_pull_requests`、`pull_request_read`、`create_pull_request`、`update_pull_request`、`merge_pull_request`、`pull_request_review_write` |
| 文件与代码 | `get_file_contents`、`create_or_update_file`、`push_files`、`delete_file`、`search_code` |
| 发布与协作 | `list_releases`、`get_latest_release`、`list_tags`、`get_teams`、`request_copilot_review` |

服务端还给了用法约定：宽泛列举用 `list_*`，带条件的查找用 `search_*`；能分页就分页，一批大约 5–10 条；暂时用不到的字段用 `minimal_output`。搜 issue 的查询串只放筛选条件，排序用单独参数，不把 `sort:` 写进查询串。

## gh CLI 是什么

`gh` 是本机上的命令行程序。人在终端能跑的命令，模型通过 Shell 也能跑。认证在系统钥匙串里。输出默认是表格；加上 `--json` 后是文本形式的 JSON。它能和 `git`、管道、脚本、CI 放在一起用。

常见命令：

```powershell
gh auth status
gh pr list --repo OWNER/REPO --state open --limit 10
gh issue list --repo OWNER/REPO --state open --search "登录失败"
gh issue comment 42 --repo OWNER/REPO --body "补充复现步骤"
gh pr checkout 12
gh pr create --title "标题" --body "说明"
gh api repos/OWNER/REPO/contents/README.md
```

## 两边怎么不同

| | GitHub MCP | `gh` CLI |
|---|---|---|
| 模型怎么碰到它 | 直接调用具名工具 | 先调用 Shell，再执行命令字符串 |
| 输入 | 按 schema 填的 JSON | 旗标、位置参数、引号 |
| 输出 | 结构化 JSON | 终端文本，或 `--json` |
| 谁维护登录 | Cursor 里的 MCP 配置 | `gh auth login` / 钥匙串 |
| 离开对话之后 | 那次调用留在对话记录里 | 同一条命令可以抄进脚本或 CI |
| 和本地仓库的关系 | 面向 GitHub 上的对象 | 可以 checkout、改工作区、再推回去 |
| 参数写错时 | 调用前就被 schema 拦住 | 命令跑起来后由 `gh` 报错 |

## 什么时候选哪一个

用 **MCP**，当这一步是对话里的一次离散 GitHub 操作，而且希望参数被类型卡住：

- 读远端文件，不必先 clone：`get_file_contents`
- 按条件搜 issue、PR、代码：`search_issues`、`search_pull_requests`、`search_code`
- 给已有 PR 写行评、改 issue、合并：字段多，schema 比一长串旗标更稳
- 结果还要被下一步推理使用（先搜重复 issue，再决定要不要新建）

用 **CLI**，当这条命令本身就是交付物，或者要碰本机仓库：

- 希望留下一条人可以自己再跑的命令
- 要和本地 git 连着做：`gh pr checkout`、改代码、跑测试、再 `gh pr create`
- MCP 当前这批工具没包进去的能力：Actions workflow、gist、用 `gh api` 打任意 REST、把 `gh pr diff` 管道给别的程序
- 写进脚本、定时任务、CI
- 一次要串很多本地步骤（建分支、提交、推送、开 PR）

两边都能做的事（列 PR、评论、建仓），对话里用 MCP 更不容易把引号写错。要变成可复制的操作记录时用 `gh`。在本仓库里，代理替人开 pull request 时约定走 `gh`，因为整条命令和 diff 都能在终端里核对。

## 例子

### 1. 列出未合并的 PR

MCP：调用 `list_pull_requests`，参数是 `owner`、`repo`、`state: "open"`、`perPage: 10`。回来的是 PR 对象列表，模型按标题、作者、链接总结。

CLI：

```powershell
gh pr list --repo NorthAdb/ai-research-hub --state open --limit 10
```

回来的是表格。加上 `--json number,title,author,url` 就接近 MCP 那种结构，解析仍然发生在命令输出之后。

### 2. 连续两步：先查重复 issue，再评论

这是完整的 tool calling 循环。每一步都是一次 function call。

1. 调用 `search_issues`。查询串只放筛选条件，例如 `repo:NorthAdb/ai-research-hub is:open 登录失败`。
2. 宿主把命中列表送回。模型看到 `#42` 已经在谈同一件事。
3. 再调用 `add_issue_comment`：`owner`、`repo`、`issue_number: 42`、`body`。
4. 两次结果都齐了，模型才回复：「没有新建，已评论在 #42。」

用 CLI 是同一步骤、换执行器：

```powershell
gh issue list --repo NorthAdb/ai-research-hub --state open --search "登录失败"
gh issue comment 42 --repo NorthAdb/ai-research-hub --body "补充复现步骤：..."
```

模型每一轮仍然只是在调用 Shell。GitHub 的语义藏在命令字符串里，没有单独的参数校验。

### 3. 本地闭环用 CLI

「把 12 号 PR 拉下来，跑测试，修完再推回去」需要工作区：

```powershell
gh pr checkout 12
npm test
git add -A
git commit -m "fix: 修正分页偏移"
git push
```

MCP 可以读 PR、改远端文件、开新 PR。它不持有这份本地工作区，也跑不了仓库里的测试命令。

### 4. 只读远端文件用 MCP

「看一下 `cli/cli` 仓库 `main` 分支的 README，不必 clone」：调用 `get_file_contents`，填 `owner: "cli"`、`repo: "cli"`、`path: "README.md"`。文件内容以工具结果回来。

CLI 对应的是：

```powershell
gh api repos/cli/cli/contents/README.md
```

内容要再做一层 base64 解码，模型更容易把查询串拼错。这种只读、字段固定的请求，MCP 更合适。
