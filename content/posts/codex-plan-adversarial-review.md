---
title: "codex-plan：让 GPT 当你的计划审查官，多轮对抗式质疑直到无懈可击"
date: 2026-03-20
tags: ["claude-code", "codex", "ai-tools", "skills", "gpt", "openai", "plan-review", "adversarial", "ai-coding", "developer-tools", "multi-model", "anthropic", "cli", "architecture"]
summary: "基于 codex skill 的进阶版本：codex-plan 自动驱动 Claude 与 GPT 的多轮对抗式计划审查。Claude 写计划，GPT 质疑，Claude 评估并修改，循环往复直到 GPT 说 LGTM。不再是一次性的第二意见，而是真正的红蓝对抗。开源免费，5 分钟安装。"
draft: false
---

## 从「第二意见」到「对抗式审查」

之前写过一篇 [让 Claude Code 调用 Codex CLI：获取 GPT 的第二意见](/posts/codex-with-claude-code/)，介绍了 codex skill —— 在 Claude Code 中一键调用 GPT 模型做代码审查。

用了一段时间后发现一个问题：**一次性的第二意见不够用。**

典型场景是写完一份实现计划，让 Codex 审查一下，它提了 5 条意见。然后呢？你手动改计划，再手动 `/codex` 让它看修改后的版本，它又提了 3 条……来回几轮你就烦了。

更大的问题是：**你在充当中间人。** Claude 写计划、GPT 提意见、你评估意见合不合理、你决定改不改、你把修改后的版本喂回去。整个流程的瓶颈是你。

codex-plan 解决的就是这个问题：**把你从循环里移出去，让 Claude 和 GPT 自动对抗。**

## codex-plan 和 codex 的区别

先说清楚：codex-plan 是**基于 codex skill 构建的**，底层调用的是同一个 Codex CLI，同样以只读沙箱运行，同样支持 session 恢复。区别在于上层流程：

| | codex | codex-plan |
|---|---|---|
| **定位** | 通用工具 —— 问什么都行 | 专用工具 —— 只做计划审查 |
| **交互模式** | 单轮：你问，GPT 答 | 多轮自动循环：质疑 → 修改 → 再质疑 |
| **谁评估反馈** | 你自己 | Claude 自动评估（采纳/驳回） |
| **谁改计划** | 你自己 | Claude 自动修改计划文件 |
| **终止条件** | GPT 回答完就结束 | GPT 说 LGTM 才结束 |
| **输出** | GPT 的原始回复 | 每轮摘要 + 最终修改总结 |

简单说：codex 是一把瑞士军刀，codex-plan 是一台自动化的红蓝对抗机器。

## 工作流程

输入 `/codex-plan` 后，完整流程是这样的：

```
你的计划
  ↓
Claude 收集项目上下文
  ↓
┌─────────────────────────────────────┐
│ Claude 把计划 + 上下文提交给 Codex   │
│          ↓                          │
│ Codex 扮演严格审查者，按优先级质疑    │
│   🔴 高优先级 / 🟡 中 / 🔵 低       │
│          ↓                          │
│ Claude 逐条评估：                    │
│   ✅ 采纳 → 直接修改计划文件          │
│   ❌ 驳回 → 附理由                   │
│          ↓                          │
│ Claude 把修改后的计划回传给 Codex     │
│          ↓                          │
│ Codex 继续审查修改后的版本            │
│          ↓                          │
│ 循环，直到 Codex 说 "LGTM"          │
└─────────────────────────────────────┘
  ↓
输出最终摘要：改了什么、保留了哪些分歧
```

整个过程你不需要介入。当然，你随时可以发消息打断，Claude 会立即停下来响应你。

## 使用示例

### 审查已有计划

```
❯ /codex-plan @docs/migration-plan.md

⏺ Skill(codex-plan)
  ⎿  Successfully loaded skill

⏺ 计划文件: docs/migration-plan.md (87行)
  收集项目上下文...

⏺ Bash(codex exec --sandbox read-only ...)
  ⎿  Running in the background

--- 第 1 轮 ---
Codex 提出 6 条质疑
✅ 采纳 4 条：
  - 🔴 缺少回滚策略 → 新增 "回滚方案" 章节
  - 🔴 数据库迁移无锁表评估 → 补充锁表影响分析
  - 🟡 缺少监控指标 → 新增关键指标清单
  - 🔵 步骤 3 和 4 可以并行 → 调整依赖关系
❌ 驳回 2 条：
  - 🟡 建议用蓝绿部署（理由：当前规模不需要，滚动更新足够）
  - 🔵 建议增加负载测试（理由：已在 CI 中覆盖）
📝 计划已更新

session id: a1b2c3d4-...【用于codex恢复记录，压缩时请保留】

⏳ 继续下一轮...

--- 第 2 轮 ---
Codex 提出 2 条质疑
✅ 采纳 1 条：
  - 🟡 回滚方案缺少数据一致性验证步骤 → 补充
❌ 驳回 1 条：
  - 🔵 建议拆分为两个 PR（理由：变更耦合度高，拆分反而增加风险）
📝 计划已更新

⏳ 继续下一轮...

--- 第 3 轮 ---
Codex: LGTM

--- 讨论完成 ---
总轮次：3
计划文件：docs/migration-plan.md
主要修改：
  - 新增回滚方案章节（含数据一致性验证）
  - 补充数据库锁表影响分析
  - 新增监控指标清单
  - 调整步骤 3/4 为并行执行
保留分歧：无
最终状态：共识达成 ✅
```

### 从零开始

没有现成的计划文件也行。告诉它你要做什么，它会先生成初始计划再开始对抗：

```
❯ /codex-plan 重构用户认证模块，从 session 迁移到 JWT
```

Claude 会分析项目现有代码，生成一份初始计划保存到 `~/.claude/plans/`，然后自动进入对抗循环。

### 继续上次的讨论

```
❯ /codex-plan
```

不带参数时，自动找到 `~/.claude/plans/` 下最近修改的计划文件继续。

## 关键设计

### 角色分工明确

**Codex（GPT）只质疑，不修改。** 它以只读沙箱运行，没有写入权限。所有文件修改都由 Claude 完成。这确保了计划文件始终在你的 Claude Code 工作区内，版本可控。

**Claude 评估并执行修改。** Claude 不是 Codex 的传声筒 —— 它会独立判断每条质疑是否合理。不合理的会驳回并附理由，驳回理由也会回传给 Codex，让它知道为什么被拒。

### 终止条件

三种情况会终止循环：

1. **Codex 说 LGTM** —— 最理想的结果，双方达成共识
2. **Codex 没有新质疑** —— 全是之前提过的重复内容，说明已经充分审查
3. **连续 2 轮全部驳回** —— 双方分歧无法调和，记录分歧点后停止

第三种情况很重要。不是所有分歧都需要解决 —— 有时候是合理的技术分歧，记录下来让人类决策就好。

### 每轮摘要保持透明

每轮结束后 Claude 都会输出摘要：采纳了哪些、驳回了哪些、改了什么。你不需要看完整的 GPT 输出，但你始终知道发生了什么。如果某条驳回你不同意，随时打断告诉 Claude。

### Session 恢复

和 codex skill 一样，codex-plan 利用 Codex CLI 的 `resume` 机制维持多轮对话。每轮摘要都包含 session ID，即使 Claude Code 的上下文被压缩，也不会丢失对话状态。

## 什么时候用 codex，什么时候用 codex-plan

**用 codex 的场景：**
- 快速问一个问题："这段代码有什么问题？"
- 让 GPT review 一下 diff
- 需要一个不同视角的分析
- 不涉及计划文档的任意任务

**用 codex-plan 的场景：**
- 有一份实现计划，想在执行前充分验证
- 架构设计需要严格审查
- 重大重构或迁移的方案评审
- 希望自动化多轮质疑，不想手动当中间人

简单说：如果你只想听听意见，用 codex；如果你想让计划被锤到没有破绽，用 codex-plan。

## 5 分钟安装

### 方式一：复制给你的 AI

```
请阅读 https://github.com/lldxflwb/claude-code-skills 帮我安装里面的 codex-plan skill
```

前提是你已经安装了 codex skill（codex-plan 依赖 Codex CLI）。

### 方式二：手动安装

```bash
# 1. 确保 Codex CLI 已安装
which codex || npm install -g @openai/codex

# 2. 安装 skill
git clone https://github.com/lldxflwb/claude-code-skills.git /tmp/claude-code-skills
cp -r /tmp/claude-code-skills/skills/codex-plan ~/.claude/skills/codex-plan
rm -rf /tmp/claude-code-skills
```

## 限制

- **依赖 Codex CLI**：需要 OpenAI 账号和额度
- **耗时较长**：多轮对抗通常需要 5-15 分钟，取决于计划复杂度和轮次
- **只针对计划文档**：不适合代码审查（代码审查用 codex 就够了）
- **Claude 的判断不一定对**：它可能错误地驳回合理意见，或采纳不合理的建议。所以每轮摘要是关键 —— 你随时可以介入纠正

## 开源

codex-plan 和 codex 一起开源在 [claude-code-skills](https://github.com/lldxflwb/claude-code-skills)。
