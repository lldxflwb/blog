---
title: "让 Claude Code 调用 Codex CLI：获取 GPT 的第二意见"
date: 2026-03-13
tags: ["claude-code", "codex", "ai-tools", "skills"]
summary: "在 Claude Code 中集成 Codex CLI，实现一键获取 GPT 模型的独立审查意见，并支持多轮对话的 session 恢复。"
draft: false
---

## 为什么需要第二意见？

用 Claude Code 做日常开发确实高效，但用久了会发现几个痛点：

**只修表面，不挖根因。** 在多步骤修复和重构场景中，Claude Code 倾向于"让报错消失"，而不是深入分析为什么会出错。比如一个类型错误，它可能直接加个 `as any` 或者一个条件判断绕过去，而不是追溯到上游数据流的问题。表面上修好了，实际上埋了雷。

**计划执行力差。** 即使你给了详细的实现计划，Claude Code 也经常在复杂任务中走偏 —— 跳过步骤、遗漏边界条件、或者擅自"优化"你的方案。你以为它在按计划推进，回头一看发现偏了。

**Prompt 遵从度不稳定。** 明确写在指令里的要求，它有时会选择性忽略。尤其是多步骤复杂任务，越到后面遵从度越低。

这些问题的根源在于：单一模型容易陷入自己的思维惯性。它不会质疑自己的修复是否真的到位。

**解决方案：让另一个模型来查漏补缺。**

GPT 系列模型有不同的训练数据和推理风格，对同一段代码变更往往能发现不同的问题。如果能在 Claude Code 的工作流中一键获取 GPT 的审查意见，就能有效弥补单一模型的盲区。

[Codex CLI](https://github.com/openai/codex) 是 OpenAI 的命令行工具，支持非交互式执行（`codex exec`），天然适合被集成。

## 5 分钟安装

### 方式一：复制给你的 AI

把下面这段话复制粘贴给 Claude Code（或任何 AI 编程助手），它会自动帮你完成安装：

```
请帮我安装 codex skill for Claude Code，按以下步骤执行：

1. 检查 codex CLI 是否已安装：which codex
2. 如果没有，运行：npm install -g @openai/codex
3. 安装完成后运行：codex login
4. 克隆 skill 仓库：git clone https://github.com/lldxflwb/claude-code-skills.git /tmp/claude-code-skills
5. 复制 skill 到 Claude Code：cp -r /tmp/claude-code-skills/skills/codex ~/.claude/skills/codex
6. 清理临时文件：rm -rf /tmp/claude-code-skills
7. 验证安装：ls ~/.claude/skills/codex/SKILL.md && echo "安装成功"
```

### 方式二：手动安装

```bash
# 1. 安装 Codex CLI
npm install -g @openai/codex

# 2. 登录
codex login

# 3. 安装 skill
git clone https://github.com/lldxflwb/claude-code-skills.git /tmp/claude-code-skills
cp -r /tmp/claude-code-skills/skills/codex ~/.claude/skills/codex
rm -rf /tmp/claude-code-skills
```

安装完成后，在 Claude Code 中输入 `/codex 你好` 验证是否正常工作。

## 使用示例

在 Claude Code 中直接输入：

```
/codex review 一下当前的代码变更
/codex 分析这个架构方案的优缺点
/codex 给这段代码写单元测试 @src/utils.ts
/codex 这个 bug 可能的原因是什么 @error.log
```

Claude Code 会自动收集相关上下文（git diff、文件内容等），发送给 Codex CLI，然后呈现 GPT 的审查意见。

一次真实的调用效果：

```
> /codex 0.11 和 0.9 哪个大？

Codex 任务:
- 指令: 比较 0.11 和 0.9 的大小
- 上下文: 无
- 模式: 新建会话

session id: 019ce6c8-...

## [Codex 结果]
0.9 更大。
```

## 实现原理

### Claude Code Skill 是什么

Claude Code Skill 是一个用 Markdown 定义的可调用能力，放在 `~/.claude/skills/<skill-name>/SKILL.md`。它告诉 Claude 在特定场景下该怎么做。

这个 codex skill 的结构：

```
~/.claude/skills/codex/
├── SKILL.md     # 主文件：定义调用流程
└── setup.md     # 环境检测与安装引导
```

SKILL.md 的 frontmatter 定义基本信息：

```yaml
---
name: codex
description: "调用 Codex CLI 执行任务，获取来自 GPT 模型的独立意见。"
user-invocable: true
argument-hint: "<要让 Codex 做的事情>"
---
```

### 调用流程

当你输入 `/codex review 代码` 时，Skill 指导 Claude Code 按这个流程执行：

1. **环境检测** — 检查 `codex` CLI 是否可用，不可用则提示安装
2. **理解意图** — 解析你的指令
3. **收集上下文** — 根据任务自动收集 git diff、文件内容等
4. **后台执行** — 调用 `codex exec` 将任务发送给 GPT 模型
5. **呈现结果** — 解析 Codex 输出并展示

核心命令：

```bash
# 新建会话
cat <<'EOF' | codex exec --sandbox read-only --skip-git-repo-check --json -o output.jsonl -
请审查以下代码变更...
EOF

# 恢复会话（多轮对话）
codex exec resume <SESSION_ID> --skip-git-repo-check --json -o output.jsonl "继续追问..."
```

### Session 恢复

Codex CLI 支持 `codex exec resume <SESSION_ID>` 恢复之前的对话。Skill 利用了这个特性：

- 首次调用时，从 Codex 的 JSONL 输出中提取 `thread_id`（session ID）
- 将 session ID 保留在 Claude Code 的对话上下文中
- 后续调用自动检测到已有 session，走 `resume` 路径

这意味着你可以和 Codex 进行多轮对话：

```
> /codex 0.11 和 0.9 哪个大？
→ Codex: 0.9 更大。

> /codex 你刚才说了啥？
→ Codex: 你刚才问的是 0.11 和 0.9 哪个更大。
```

## 关键设计决策

**后台执行**：Codex CLI 任务可能耗时较长（5-30 分钟），Skill 使用后台模式避免阻塞 Claude Code 的主流程。

**敏感信息过滤**：自动排除 `.env`、密钥文件，diff 中的 token 替换为 `[REDACTED]`。这不只是安全考虑 —— GPT 模型对敏感信息的审查策略比较严格，如果上下文中携带了 API key、密钥等内容，GPT 可能会直接拒绝执行整个任务。提前过滤掉这些信息，才能保证 Codex CLI 正常工作。

**只读沙箱**：Codex CLI 以 `read-only` 沙箱运行，不会修改你的任何文件。它只负责审查和建议。

## 限制与适用场景

**适合：**
- 代码变更的二次审查
- 方案评估和第二意见
- 补充测试建议
- 多角度分析问题

**不适合：**
- 高频低延迟的交互（Codex CLI 有启动开销）
- 超大上下文的任务（受 GPT 上下文窗口限制）
- 需要写入本地文件的操作（只读沙箱）

## FAQ

**Codex CLI 会不会修改我的文件？**
不会。Skill 强制使用 `--sandbox read-only` 模式，Codex CLI 无法写入任何文件。

**需要付费吗？**
Codex CLI 使用 OpenAI API，需要有效的 API 额度。具体成本取决于任务复杂度和使用的模型。

**Session 丢了怎么办？**
Claude Code 对话上下文被压缩时，session ID 可能丢失。丢失后下次 `/codex` 会自动创建新 session，不影响使用，只是失去之前的对话记忆。

**为什么要过滤敏感信息？**
GPT 模型遵循严格的安全审查策略。如果发送的上下文中包含 API key、密钥等敏感信息，GPT 可能直接拒绝执行整个任务。

## 开源

这个 skill 已开源在 [claude-code-skills](https://github.com/lldxflwb/claude-code-skills)，欢迎贡献更多 skill。
