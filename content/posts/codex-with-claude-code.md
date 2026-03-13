---
title: "让 Claude Code 调用 Codex CLI：获取 GPT 的第二意见"
date: 2026-03-13
tags: ["claude-code", "codex", "ai-tools", "skills"]
summary: "在 Claude Code 中集成 Codex CLI，实现一键获取 GPT 模型的独立审查意见，并支持多轮对话的 session 恢复。"
draft: true
---

## 为什么需要第二意见？

用 AI 辅助编程时，单一模型的视角有时会有盲区。Claude 擅长推理和代码质量，GPT 在某些场景下有不同的思路。如果能在 Claude Code 的工作流中一键获取 GPT 的看法，就能两全其美。

[Codex CLI](https://github.com/openai/codex) 是 OpenAI 的命令行工具，支持非交互式执行（`codex exec`），天然适合被集成。

## 实现思路

Claude Code 支持自定义 Skill —— 一个 Markdown 文件，定义 Claude 该如何响应特定指令。我写了一个 `codex` skill，让 Claude Code：

1. 接收用户指令（如 "review 一下代码变更"）
2. 自动收集相关上下文（git diff、文件内容等）
3. 调用 `codex exec` 提交给 GPT 模型
4. 解析并呈现结果

## 核心实现

### Skill 结构

```
~/.claude/skills/codex/
└── SKILL.md
```

Skill 的 frontmatter 定义了基本信息：

```yaml
---
name: codex
description: "调用 Codex CLI 执行任务，获取来自 GPT 模型的独立意见。"
user-invocable: true
argument-hint: "<要让 Codex 做的事情>"
---
```

### 调用方式

在 Claude Code 中输入：

```
/codex review 一下当前的代码变更
/codex 分析这个架构方案的优缺点
/codex 给这段代码写单元测试 @src/utils.ts
```

Claude 会自动收集上下文、构建提示词、后台执行 Codex，然后呈现结果。

### Session 恢复

Codex CLI 支持 `codex exec resume <SESSION_ID>` 恢复之前的对话。Skill 利用了这个特性：

- 首次调用时，从 Codex 的 JSONL 输出中提取 `thread_id`
- 将 session ID 保留在 Claude Code 的对话上下文中
- 后续调用自动检测到已有 session，走 `resume` 路径

这意味着你可以和 Codex 进行多轮对话：

```
/codex 0.11 和 0.9 哪个大？
→ Codex: 0.9 更大。

/codex 你刚才说了啥？
→ Codex: 你刚才问的是 0.11 和 0.9 哪个更大。
```

### 关键细节

**后台执行**：Codex 任务可能耗时较长（5-30 分钟），使用后台模式避免阻塞。

**敏感信息过滤**：自动排除 `.env`、密钥文件，diff 中的 token 替换为 `[REDACTED]`。

**沙箱隔离**：Codex 以 `read-only` 沙箱运行，不会修改任何文件。

## 安装

```bash
# 1. 安装 Codex CLI
npm install -g @openai/codex

# 2. 登录
codex login

# 3. 安装 skill
git clone https://github.com/lldxflwb/claude-code-skills.git
cp -r claude-code-skills/skills/codex ~/.claude/skills/codex
```

## 开源

这个 skill 已开源在 [claude-code-skills](https://github.com/lldxflwb/claude-code-skills)，欢迎贡献更多 skill。
