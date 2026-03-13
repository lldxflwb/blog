---
title: "让 Claude Code 调用 Codex CLI：获取 GPT 的第二意见"
date: 2026-03-13
tags: ["claude-code", "codex", "ai-tools", "skills"]
summary: "在 Claude Code 中集成 Codex CLI，实现一键获取 GPT 模型的独立审查意见，并支持多轮对话的 session 恢复。"
draft: true
---

## 为什么需要第二意见？

用 Claude Code 做日常开发确实高效，但用久了会发现几个痛点：

**只修表面，不挖根因。** Claude Code 解决问题时倾向于"让报错消失"，而不是深入分析为什么会出错。比如一个类型错误，它可能直接加个 `as any` 或者一个条件判断绕过去，而不是追溯到上游数据流的问题。表面上修好了，实际上埋了雷。

**计划执行力差。** 即使你给了详细的实现计划，Claude Code 也经常走偏 —— 跳过步骤、遗漏边界条件、或者擅自"优化"你的方案。你以为它在按计划推进，回头一看发现偏了。

**Prompt 遵从度不稳定。** 明确写在指令里的要求，它有时会选择性忽略。尤其是多步骤复杂任务，越到后面遵从度越低。

这些问题的根源在于：单一模型容易陷入自己的思维惯性。它不会质疑自己的修复是否真的到位。

**解决方案：让另一个模型来查漏补缺。**

GPT 系列模型有不同的训练数据和推理风格，对同一段代码变更往往能发现不同的问题。如果能在 Claude Code 的工作流中一键获取 GPT 的审查意见，就能有效弥补单一模型的盲区。

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

**敏感信息过滤**：自动排除 `.env`、密钥文件，diff 中的 token 替换为 `[REDACTED]`。这不只是安全考虑 —— GPT 模型对敏感信息的审查策略比较严格，如果上下文中携带了 API key、密钥等内容，GPT 可能会直接拒绝执行整个任务。提前过滤掉这些信息，才能保证 Codex 正常工作。

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
