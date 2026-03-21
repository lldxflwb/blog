---
title: "aw：一条命令给多仓库工作区切出新分支"
date: 2026-03-21
tags: ["aw", "git-worktree", "polyrepo", "ai-coding", "claude-code", "developer-tools", "cli", "go", "workspace", "context", "anthropic", "cross-repo"]
summary: "多仓库项目开新分支是个苦力活：七八个仓库，每个都要 git worktree add、每个都要敲一遍分支名。aw 把这件事缩成一条命令 —— 扫描目录下所有仓库，批量创建 worktree，统一分支名，顺带把 AI 上下文文件链接过去。做完了 aw rm 一键收尾。Go 单二进制，零依赖，macOS/Linux/Windows。"
draft: false
---

## 问题：开个分支要敲七遍命令

我桌面上有七八个仓库：blog、claude-code-skills、opencode……它们不在同一个 monorepo 里，而是各自独立的 git 仓库，散落在同一个目录下。

大多数人的项目不是"选择"了 polyrepo，而是自然就长成了 polyrepo。

想开一个 feature 分支做点事情，传统流程是这样的：

```bash
cd blog && git worktree add /tmp/feature/blog -b feature/login
cd ../opencode && git worktree add /tmp/feature/opencode -b feature/login
cd ../skills && git worktree add /tmp/feature/skills -b feature/login
# ... 还有四五个仓库
```

做完了还要一个一个删回去。七八个仓库，每次都是纯体力劳动。

如果你用 AI 编程工具，还有额外的问题：工作目录根目录下的 `CLAUDE.md`、`.claude/` 这些上下文文件，到了新的 worktree 目录里就不存在了 —— Agent 丢失了所有指令和记忆。

## aw 的解法

**一条命令，批量检出整个工作区的新分支。**

```bash
cd ~/projects
aw new --dir /tmp/my-feature -b feature/login
```

aw 扫描当前目录下的所有 git 仓库，对每个仓库执行 `git worktree add`，统一使用同一个分支名。完成后你得到一个完整的平行工作区：

```
~/projects/                          /tmp/my-feature/
├── blog/          (.git)            ├── blog/          (worktree)
├── opencode/      (.git)     →      ├── opencode/      (worktree)
├── skills/        (.git)            ├── skills/        (worktree)
├── CLAUDE.md                        ├── CLAUDE.md      (symlink → 源文件)
└── .claude/                         └── .claude/       (symlink → 源目录)
```

每个 worktree 都是一个真实的 git 工作目录，有自己的分支，互不干扰。AI 上下文文件通过软链接共享，改一处处处生效。

做完了？

```bash
aw rm --force --branch
```

一条命令，所有 worktree 删除，分支清理，目录回收。

## 安装

```bash
# macOS (Apple Silicon)
curl -L https://github.com/lldxflwb/aw/releases/download/v0.1.0/aw-darwin-arm64 -o aw
chmod +x aw && sudo mv aw /usr/local/bin/

# 或者用 Go 安装
go install github.com/lldxflwb/aw@latest
```

单二进制，零依赖。macOS、Linux、Windows 都有预编译版本。

## 使用

### 创建工作区

```bash
cd ~/projects
aw new --dir /tmp/my-feature -b feature/login
```

实际输出：

```
Found 7 repos: blog, claude-code-action, claude-code-skills, opencode, rtk, sing-box-subscribe, surfboard
Target: /tmp/my-feature
Branch: feature/login

[blog] creating worktree → /tmp/my-feature/blog (branch: feature/login)
[blog] OK

[claude-code-action] creating worktree → /tmp/my-feature/claude-code-action (branch: feature/login)
[claude-code-action] OK

...

== workspace context ==
  [link] CLAUDE.md
  [link] .claude
== repo context ==
---
Done. 7/7 repos, 2 context links.
```

几秒钟，七个仓库全部检出到新分支，上下文文件自动链接。

### 看状态

```bash
cd /tmp/my-feature
aw status
```

```
REPO                BRANCH         STATUS  AHEAD  BEHIND  LAST COMMIT
------------------  -------------  ------  -----  ------  ------------------------------------------
blog                feature/login  1M      0      0       a300ba9 Add codex-plan blog post
claude-code-action  feature/login  clean   0      0       3428ca8 chore: bump Claude Code to 2.1.71
opencode            feature/login  clean   0      0       6f90c3d chore: update nix hashes
...
```

一眼看到所有仓库的分支、修改状态、ahead/behind。比挨个 `cd` 进去 `git status` 快多了。

### 清理

```bash
aw rm --force --branch
```

一键删除所有 worktree 和分支，清理 `.aw/` 目录。不加 `--force` 会检查是否有未提交的修改，防止误删。`--dry-run` 可以先预览会删什么。

## AI 上下文链接

批量 worktree 创建是 aw 的核心，上下文链接是锦上添花 —— 但确实很实用。

aw 会自动扫描并链接这些 AI 上下文文件：

- `CLAUDE.md`、`AGENTS.md`、`codex.md`
- `.claude/`、`.codex/`、`.cursor/`、`.cursorrules`
- `aw.yml`

**两级链接：**

1. **工作区级** —— 源目录根下的上下文文件（如 `~/projects/CLAUDE.md`）链接到工作区根
2. **仓库级** —— 各仓库内的上下文文件，**只链接 git 未跟踪的**。已跟踪的文件 worktree 里本来就有，不需要重复

这个设计意味着你可以在源目录随时修改 `CLAUDE.md`，工作区里的 Agent 立刻能看到最新内容。

## Windows 兼容

Windows 默认不支持创建软链接（需要开发者模式或管理员权限）。aw 的处理方式是：

**软链接失败时自动 fallback 到复制文件**，并打印修复提示：

```
[warn] symlink failed for CLAUDE.md: ..., falling back to copy
[hint] to fix: enable Developer Mode (Windows Settings → System → For developers), then run 'aw relink'
```

开启开发者模式后，运行 `aw relink` 就能把所有复制文件升级为软链接。如果 relink 仍然失败，会自动恢复复制并提示具体原因。

## 设计决策

### 为什么用 git worktree 而不是 clone

git worktree 和源仓库共享 `.git` 数据库，不需要重新下载代码。创建一个 worktree 比 clone 快几个数量级，而且磁盘占用几乎为零。更重要的是，worktree 的分支和源仓库的分支是同一个 —— 不需要 push/pull 来同步。

### 为什么不用 monorepo

monorepo 需要全部代码在一个仓库里。现实中大多数团队的仓库是各自独立的，把它们合并成 monorepo 的成本远高于收益。aw 不改变你的仓库结构，只在需要时创建临时工作区。

### workspace.json

每次 `aw new` 都会在工作区的 `.aw/workspace.json` 里记录完整状态：源路径、目标路径、分支名、所有仓库和上下文链接。`aw rm` 和 `aw status` 都依赖这个文件，不需要重新扫描或猜测。

### --json 输出

所有命令都支持 `--json`，输出统一的 JSON 信封：

```json
{"schema_version": 1, "ok": true, "data": {...}, "error": null, "warnings": []}
```

Exit code 也标准化：0 成功、1 部分失败、2 用法错误。这让 aw 天然适合被脚本、CI、MCP Server 等上层工具集成。

### 并发锁

写操作（`aw new`、`aw rm`）会在 `.aw/lock` 上加文件锁。Unix 用 flock，Windows 用 exclusive file。防止两个进程同时操作同一个工作区。锁超过 30 秒自动视为 stale 并回收。

## 限制

- **aw new 不完全幂等**：重复创建同名分支会被 git 拒绝，因为分支已存在。后续版本会检测已有分支并跳过
- **不支持嵌套 git 仓库**：只扫描一级子目录，不递归
- **软链接在 Windows 需要额外操作**：不是 bug，是 Windows 的限制，但 fallback + relink 的方案已经尽量平滑了

## 后续计划

- **MCP Server** —— 让 Agent 直接调用 aw 创建工作区
- **Skill 集成** —— `/aw` 一键创建工作区并切换 Agent 工作目录
- **aw.yml** —— 配置文件，指定哪些仓库需要、哪些排除、上下文文件自定义

---

项目地址：[aw](https://github.com/lldxflwb/aw)
