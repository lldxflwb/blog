---
title: "Skills /claude：让你的 Claude Code 会话们互相聊天"
date: 2026-03-23
tags: ["claude-code", "skills", "ai-tools", "ai-coding", "developer-tools", "anthropic", "multi-session", "cli", "session-resume", "context-bridging"]
summary: "每个 Claude Code 会话都有自己的上下文。通过 /claude skill + session resume，你可以从当前会话接入另一个已有会话，向它追问并带回结论；需要时也可以新开一个干净的 Claude 实例做独立分析。核心价值是复用已有 session 的上下文，而不是手动搬运聊天记录。"
draft: false
---

![/claude skill 工作机制](/blog/images/claude-bridge.svg)

## 从「第二意见」到「上下文桥接」

之前写过 [codex skill](/posts/codex-with-claude-code/) 和 [codex-plan](/posts/codex-plan-adversarial-review/)，核心都是在 Claude Code 里调用另一个模型获取外部视角。但用了一段时间后，我发现有个场景更常见：

**两个 Claude Code 会话各自积累了大量上下文，你想让它们协调。**

比如我最近同时推进两个项目：Session A 在做多仓库工作区管理工具（aw），Session B 在做 Claude Code 的 session 编辑功能。两个 session 各自聊了很久，各自有完整的项目理解和方案演进。

某天我发现 —— aw 在检出新工作区时，其实可以顺便把相关的 Claude Code session 迁移过去，这样新工作区一创建就能 `claude --resume` 继续之前的对话。而这恰好是 B 那边 session-edit 在做的事。

传统做法：翻 A 的聊天记录，提炼要点，粘贴到 B 的对话里。

**上下文搬运是最累的活。** 而且你搬的是降级版本 —— 你能复制的只是文字片段，不是那个会话完整的理解。

## 解决方案：当前会话充当桥接器

`/claude` skill 的机制是：当前会话通过 `claude -p --resume <session_id>` 接入另一个已有会话，把问题发过去，再把结果带回来。你复用的是对方已经积累好的上下文，不需要重新搬运。

```
❯ /claude 问 e2ec1031-204f-4ce3-b1df-572e2e9a91d5
    ~/Desktop/ai-space/claude-session-editor
    把 session 迁移到新工作区需要改哪些路径？

→ 不需要改任何路径。把 .jsonl 文件复制到 ~/.claude/projects/{新目录编码名}/ 下即可，
  文件名保持不变。Claude Code 在新目录 --resume 时会自动用当前 cwd 作为工作目录，
  session 里的旧路径不影响恢复。

❯ /claude 问 afd31f27-3f71-4ca0-b220-3f0b15ade5b1
    ~/Desktop/ai-space/claude-code-skills
    检出新工作区时顺便迁移 session 过去，技术方案是什么？

→ Claude Code 的会话数据存在 ~/.claude/projects/<项目路径编码>/ 下（路径中 / 替换为 -），
  核心是 .jsonl 对话记录文件。迁移方案：新 worktree 创建后，将旧路径对应的目录
  symlink 或 copy 到新路径对应的目录名下即可。最轻量的做法是直接 ln -s 软链接，
  这样两个 worktree 共享同一份会话状态。
```

你只搬运**结论**，而不是搬运原始上下文。每个 session 保持完整的项目理解，当前会话只是在它们之间架桥。

## /claude 的两种模式

1. **恢复模式**：接入一个已有 session，复用它积累的项目上下文。这是本文重点。
2. **新建模式**：起一个干净的 Claude 实例，拿到不受当前讨论影响的独立分析。

两种模式用法一样，区别只在于是否指定了 session ID。

## 一条命令安装

### 方式一：复制给你的 AI

```
请阅读 https://github.com/lldxflwb/claude-code-skills 帮我安装里面的 claude skill
```

### 方式二：手动安装

```bash
git clone https://github.com/lldxflwb/claude-code-skills.git /tmp/claude-code-skills
cp -r /tmp/claude-code-skills/skills/claude ~/.claude/skills/claude
rm -rf /tmp/claude-code-skills
```

对已在使用 Claude Code 的用户，无需再装第二个 CLI。

## 使用场景

### 跨项目协调（恢复模式）

两个项目各自推进了一段时间，突然发现有交集：

```
❯ /claude 问 e2ec1031 把 session 迁移到新工作区需要改哪些路径？

  ⏺ 查找 session e2ec1031...
    → 项目目录: ~/Desktop/ai-space/claude-session-editor

→ 不需要改任何路径。把 .jsonl 文件复制到新目录编码名下即可...

❯ /claude 问 afd31f27 检出新工作区时顺便迁移 session，技术方案是什么？

  ⏺ 查找 session afd31f27...
    → 项目目录: ~/Desktop/ai-space/claude-code-skills

→ 新 worktree 创建后，将旧路径目录 symlink 到新路径目录名下...
```

不需要复制粘贴任何上下文。A 和 B 各自知道自己项目的全部细节。

### 干净上下文的独立分析（新建模式）

有时不需要接入已有 session，而是需要一个没有思维惯性的新视角：

```
❯ /claude 从零分析这个模块的设计问题 @src/auth/

→ [新实例没有之前讨论的包袱，直接指出结构性问题]
```

### 指定模型

默认在 Claude 模型家族内，可选 sonnet/opus：

```
/claude 用 sonnet 快速扫一下有没有明显问题 @src/parser.py
/claude 用 opus 从安全角度分析这段逻辑 @src/auth.ts
```

### 多轮对话

和 [codex skill 的 session 恢复](/posts/codex-with-claude-code/#session-恢复)一样，首次调用后 session ID 保留在上下文中，后续自动 resume：

```
❯ /claude 分析一下 skills/ 目录的整体架构
→ [给出架构评价]

❯ /claude 那 codex 和 codex-plan 之间的抽象层次合理吗？
→ [基于上一轮继续深入]
```

## 和 /codex 的关系

`/claude` 并不是唯一支持 session resume 的 skill。[/codex](/posts/codex-with-claude-code/) 也支持 `codex exec resume`，"跨 session 复用已有上下文"不是 `/claude` 的独有能力。两者真正的差异不在于能不能 resume，而在于你要不要引入另一家模型。

| 维度 | /codex | /claude |
|---|---|---|
| **底层** | Codex CLI (`codex exec` / `resume`) | Claude Code CLI (`claude -p` / `--resume`) |
| **新建干净上下文** | 支持 | 支持 |
| **恢复已有 session** | 支持 | 支持 |
| **主要价值** | 引入 GPT/Codex 的外部视角 | 留在 Claude 生态内复用已有会话 |
| **依赖** | 需要额外安装 Codex CLI + OpenAI 账号 | 对 Claude Code 用户无需额外安装 |
| **执行限制** | `--sandbox read-only` | `--allowedTools` 白名单 |

[/codex](/posts/codex-with-claude-code/) 解决"换个模型看问题"，[/codex-plan](/posts/codex-plan-adversarial-review/) 解决"自动化多轮对抗审查"，`/claude` 解决"在 Claude 生态内复用已有 session 做上下文桥接"。三者按需组合。

如果当前会话同时掌握 Claude 和 Codex 两边的 session ID，还可以做跨 CLI 的 relay —— 把 Claude session 的结论转发给 Codex session，反之亦然。

## 实现原理

### 核心命令

```bash
# 新建会话
claude -p \
  --output-format json \
  --dangerously-skip-permissions \
  --allowedTools "Read,Glob,Grep" \
  -- "你的指令..." < /dev/null

# 接入已有会话
claude -p \
  --output-format json \
  --dangerously-skip-permissions \
  --allowedTools "Read,Glob,Grep" \
  --resume <SESSION_ID> \
  -- "继续对话..." < /dev/null
```

关键是 `--resume <SESSION_ID>`。Claude Code 的每个 session 都有一个 UUID，通过这个 ID 就能接入已有会话的完整上下文。

如果目标 session 来自不同项目（不同工作目录），skill 会从 `~/.claude/sessions/` 中查找该 session 的原始工作目录，并 `cd` 过去再执行。

`--output-format json` 返回结构化数据，从中提取 `session_id` 保存在当前对话上下文里：

```json
{
  "session_id": "c3e6ccf8-9cce-49c7-b07e-1d420fb73bc0",
  "result": "回复内容...",
  "total_cost_usd": 0.058
}
```

### 踩过的坑

**白名单而非黑名单。** 一开始用 `--disallowedTools "Edit,Write,NotebookEdit"` 禁写入工具，后来发现 Bash 还是可用的 —— 子进程能执行 `rm` 或 `echo >`。改成 `--allowedTools "Read,Glob,Grep"` 白名单模式后，Bash、Edit、Write 全部不可用，才是真正的只读。

**`--` 分隔符不能省。** `--allowedTools` 是变参选项，会吞掉后面所有参数。不加 `--` 的话，你的 prompt 会被当成工具名，然后报"找不到输入"。

**`< /dev/null` 关掉 stdin。** 在我的测试里，不显式关闭 stdin 时 `claude -p` 会短暂等待输入，加上 `< /dev/null` 可以避免这段等待。

**不要默认使用 `--bare`。** 如果你依赖 OAuth/keychain 登录，`--bare` 会跳过这些认证来源，子进程会表现为未登录。只有改用环境变量（`ANTHROPIC_API_KEY`）等 bare-compatible 认证方式时才适合用它。

## 限制

- **只读**：白名单模式下子进程不能修改文件，只能分析和建议
- **Session ID 需要你自己找**：跨项目 resume 时，你需要知道目标 session 的 UUID（可以通过 `claude --resume` 交互式选择器查找）

## 开源

和 codex、codex-plan 一起开源在 [claude-code-skills](https://github.com/lldxflwb/claude-code-skills)。
