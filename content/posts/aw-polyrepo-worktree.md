---
title: "aw：让 AI Agent 看到你的所有仓库"
date: 2026-03-21
tags: ["aw", "git-worktree", "polyrepo", "ai-coding", "claude-code", "developer-tools", "cli", "go", "workspace", "context", "anthropic", "cross-repo"]
summary: "多仓库项目中 AI Agent 只能看到当前仓库的代码，跨仓协作全靠人肉复制粘贴。aw 用 git worktree 批量创建隔离工作区，自动软链接 AI 上下文文件（CLAUDE.md、.cursor/ 等），让 Agent 一次看到所有仓库的代码和指令。Go 单二进制，零依赖，支持 macOS/Linux/Windows。"
draft: false
---

## 问题：Agent 只能看到一个仓库

我桌面上有七八个仓库：blog、claude-code-skills、opencode……它们不在同一个 monorepo 里，而是各自独立的 git 仓库，散落在同一个目录下。

大多数人的项目不是"选择"了 polyrepo，而是自然就长成了 polyrepo。

这带来一个 AI 时代的新问题：**Agent 一次只能看到一个仓库的代码。** 我想让 Claude Code 同时改 blog 和 claude-code-skills 两个仓库的代码？做不到，除非我手动来回切换工作目录，或者把文件复制过去。

更麻烦的是 AI 上下文文件。我在工作目录根目录放了 `CLAUDE.md`、`.claude/` 目录，里面写了全局指令和记忆。但切到子仓库的 worktree 后，这些文件就不在了 —— Agent 丢失了所有上下文。

## aw 的解法

aw 做的事情很简单：**用 git worktree 批量创建隔离工作区，然后把 AI 上下文文件软链接过去。**

```
~/projects/                          /tmp/my-feature/
├── blog/          (.git)            ├── blog/          (worktree)
├── opencode/      (.git)     →      ├── opencode/      (worktree)
├── skills/        (.git)            ├── skills/        (worktree)
├── CLAUDE.md                        ├── CLAUDE.md      (symlink → 源文件)
└── .claude/                         └── .claude/       (symlink → 源目录)
```

每个 worktree 都是一个真实的 git 工作目录，有自己的分支，互不干扰。AI 上下文文件通过软链接共享，改一处处处生效。

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

几秒钟搞定。现在 `/tmp/my-feature/` 就是一个完整的工作区，Agent 在这里能看到所有仓库的代码，以及你的全局 AI 指令。

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

## AI 上下文文件

这是 aw 和普通 git worktree 命令的核心区别。aw 会自动扫描并链接这些 AI 上下文文件：

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

git worktree 和源仓库共享 `.git` 数据库，不需要重新下载代码。创建一个 worktree 比 clone 快几个数量级，而且磁盘占用几乎为零。

### 为什么不用 monorepo

monorepo 需要全部代码在一个仓库里。现实中大多数团队的仓库是各自独立的，把它们合并成 monorepo 的成本远高于收益。aw 不改变你的仓库结构，只在需要时创建临时工作区。

### workspace.json

每次 `aw new` 都会在工作区的 `.aw/workspace.json` 里记录完整状态：源路径、目标路径、分支名、所有仓库和上下文链接。`aw rm` 和 `aw status` 都依赖这个文件。格式如下：

```json
{
  "version": 1,
  "source": "/Users/karl/projects",
  "branch": "feature/login",
  "created_at": "2026-03-21T10:00:00+08:00",
  "repos": [
    {
      "name": "blog",
      "source_path": "/Users/karl/projects/blog",
      "worktree_path": "/tmp/my-feature/blog"
    }
  ],
  "context_links": [
    {
      "src": "/Users/karl/projects/CLAUDE.md",
      "dst": "/tmp/my-feature/CLAUDE.md",
      "type": "workspace",
      "method": "symlink"
    }
  ]
}
```

### --json 输出

所有命令都支持 `--json`，输出统一的 JSON 信封：

```json
{"schema_version": 1, "ok": true, "data": {...}, "error": null, "warnings": []}
```

Exit code 也标准化：0 成功、1 部分失败、2 用法错误。这让 aw 天然适合被上层工具（MCP Server、CI 脚本）集成。

### 并发锁

写操作（`aw new`、`aw rm`）会在 `.aw/lock` 上加文件锁。Unix 用 flock，Windows 用 exclusive file。防止两个 Agent 同时操作同一个工作区。锁超过 30 秒自动视为 stale 并回收。

## 限制

- **aw new 不完全幂等**：重复创建同名分支会被 git 拒绝，因为分支已存在。后续版本会检测已有分支并跳过
- **不支持嵌套 git 仓库**：只扫描一级子目录，不递归
- **软链接在 Windows 需要额外操作**：不是 bug，是 Windows 的限制，但 fallback + relink 的方案已经尽量平滑了

## 后续计划

aw 的定位是 **Agent 上游控制面** —— 先有好的工作环境，Agent 才能发挥最大能力。后续方向：

- **MCP Server** —— 让 Agent 直接调用 aw，不需要人类手动创建工作区
- **Skill 集成** —— `/aw` 一键创建工作区并切换 Agent 工作目录
- **aw.yml** —— 配置文件，指定哪些仓库需要、哪些排除、上下文文件自定义

---

项目地址：[aw](https://github.com/lldxflwb/aw)
