---
title: "用 Claude Code Skill 搭建个人预览服务：TikZ 渲染 + 墨水屏阅读器"
date: 2026-03-16
tags: ["claude-code", "skills", "tikz", "e-ink", "preview-server", "latex", "svg", "diagram", "ai-coding", "developer-tools", "markdown-reader", "http-server", "python", "automation", "anthropic"]
summary: "用 Claude Code Skill 搭建本地预览服务器：LaTeX TikZ 图表一键渲染为 SVG，Markdown 文档墨水屏友好阅读，AI 自动校验图表质量。支持架构图、流程图、状态机等可视化，内置分页翻页、目录导航、双模式切换。开源免费，5 分钟安装。"
draft: false
---

## 起因

我日常用 Claude Code 做开发，经常需要两类输出的可视化：

1. **TikZ 图表** —— 架构图、流程图、状态机这些，Claude 写好 TikZ 代码后我需要看到渲染结果
2. **Plan 文档** —— Claude Code 的 Plan Mode 会产出大量 markdown 计划文档，我想在墨水屏上舒服地阅读它们

以前的做法是手动编译 LaTeX、手动打开 markdown，非常割裂。于是我做了一个 Claude Code Skill，把这些整合成一个自动化的预览服务。

## 5 分钟安装

### 方式一：复制给你的 AI

```
请阅读 https://github.com/lldxflwb/claude-code-skills 帮我安装里面的 tikz skill
```

AI 会自动克隆仓库、安装依赖（tectonic + pdf2svg）、创建软链接。

### 方式二：手动安装

```bash
# 1. 安装依赖
brew install tectonic pdf2svg

# 2. 克隆仓库
git clone https://github.com/lldxflwb/claude-code-skills.git ~/claude-code-skills

# 3. 软链接到 Claude Code skills 目录
ln -s ~/claude-code-skills/skills/tikz ~/.claude/skills/tikz
```

安装完成后输入 `/tikz 画一个简单的流程图` 验证。

## 使用效果

### TikZ 渲染

在 Claude Code 里直接说你想画什么：

```
❯ /tikz 画一个 A → B → C 的流程图
```

Claude 会自动写 TikZ 代码、编译、返回一个可点击的 URL。整个过程你只需要等几秒：

```
⏺ Write(flowchart.tex)
  ⎿  Wrote 25 lines to flowchart.tex

⏺ Bash(python3 ~/.claude/skills/tikz/scripts/render_tikz.py flowchart.tex)
  ⎿  http://127.0.0.1:8073/flowchart_20260316_111125_932281.svg
      /Users/karl/.cache/tikz-skill/renders/flowchart_20260316_111125_932281.png

⏺ 渲染完成：http://127.0.0.1:8073/flowchart_20260316_111125_932281.svg
```

### AI 自动校验图表质量

这是我觉得最实用的功能。Claude 渲染完图表后，会派一个子 agent 去**读取 PNG 截图**，检查文字有没有重叠、布局是不是太挤、箭头是不是清晰。发现问题就自动改 TikZ 代码重新渲染，直到效果满意才返回最终 URL。

为什么需要这个？因为 TikZ 图表的一个常见问题是节点太多时文字会重叠 —— AI 写代码的时候不知道渲染出来长什么样。有了 PNG 校验，它能"看到"自己的产出，然后修正。

### Plan 文档阅读

打开预览服务首页（`http://127.0.0.1:8073/`），你会看到两个区域：

- **计划** —— `~/.claude/plans/` 下的所有 plan 文档，按创建时间倒序
- **TikZ 渲染** —— 所有渲染过的 SVG 缩略图

点击任意 plan 进入阅读页面，有两种模式可切换：

**墨水屏模式** —— 专门为墨水屏优化的阅读体验：
- Georgia 衬线字体，大字号加粗，高对比度配色
- 自动按视口高度分页，完全不需要滚动
- 点击屏幕左侧翻上一页，右侧翻下一页
- 代码块、表格超出一页时自动拆分到多页
- 顶部显示页码和阅读进度

**普通模式** —— 标准 web 阅读：
- 全宽布局，长页面滚动，可选择文字
- 右下角浮动按钮组：回到顶部、去到底部、打开目录侧边栏
- 目录侧边栏支持点击跳转，滚动时自动高亮当前章节

两种模式通过右上角按钮一键切换，选择会持久化到浏览器，下次打开还是上次的模式。

### 在 Plan 中嵌入图表

写计划文档时如果需要配图，Claude 会先渲染 TikZ 图表，然后用相对 URI 嵌入到 markdown 里：

```markdown
![Pipeline Overview](/pipeline_20260316_120000_123456.svg)
```

Plan viewer 会自动内联展示，图片居中、宽度自适应。

## 架构设计

![Preview Server 架构图](/blog/images/preview-server/architecture.svg)

整个 skill 的核心思路是 **"脚本负责生产，server 负责展示"**：

```
skills/tikz/
├── SKILL.md              # Claude 读取的技能指令
├── config.json           # host/port/默认模式配置
├── setup.md              # 依赖安装引导
└── scripts/
    ├── render_tikz.py     # TikZ → PDF → SVG/PNG
    ├── tikz_server.py     # HTTP 预览服务
    ├── ensure_server.py   # 自动启停管理
    └── config.py          # 配置读取（env > config.json > defaults）
```

几个设计决策值得说一下：

**配置优先级链**：`环境变量 > config.json > 硬编码默认值`。这样不同机器通过 config.json 配不同的 IP 和端口，不依赖 shell 环境变量。我之前踩过坑 —— Claude Code 的 Bash 工具不一定会 source `.zshrc`，导致环境变量读不到。config.json 跟着 skill 走，彻底解决了这个问题。

**Server 完全透明**：`render_tikz.py` 内部自动调 `ensure_server.py`，AI 和用户都不需要手动管理 server。已有 server 就复用，没有就启动，僵尸进程就杀掉重来。

**静态目录安全**：server 的静态根目录限定在 `renders/`，不会暴露 `server.log`、`server.json` 等内部文件。

**子 agent 校验**：渲染时同时生成 PNG（200 DPI），Claude 派子 agent 读取 PNG 验证图表质量，在子 agent 里闭环修正，不污染主 agent 上下文。

## 踩坑记录

墨水屏分页是这个项目里最折腾的部分。跟 Codex（GPT 模型）联合诊断了好几轮才搞定：

**离屏测量不准。** 最初用一个隐藏的 div 测量元素高度，但字体大小、行高、padding 和真实渲染环境不一致，导致分页偏差。最终方案是直接把内容渲染到真实容器里，逐个添加元素，用 `scrollHeight > clientHeight` 判断是否溢出 —— 零误差，所见即所得。

**嵌套代码块不分页。** marked.js 把列表里的代码块生成为 `<ol><li><pre>`，分页逻辑只检查顶层 children 的 `tagName === 'PRE'`，永远匹配不到嵌套的代码块。修复方案是递归遍历容器（`flattenOversized`），找到任意层级的 `pre/table` 并拆分。这个 bug 是让 Codex 诊断出来的 —— 自己盯了半天没看出来，换个模型一下就定位了。

**base64 中文乱码。** JavaScript 的 `atob()` 只能处理 Latin-1 编码，中文 UTF-8 字节会乱码。用 `TextDecoder` 解码修复。

**f-string 和 JS 冲突。** Python f-string 里的 `\n` 会被解释为真换行，破坏 JS 字符串。markdown 内容中的反引号和 `</script>` 也会破坏页面。最终用 base64 编码嵌入内容，彻底避免转义问题。

## 配置参考

`config.json` 示例：

```json
{
  "host": "192.168.1.100",
  "port": 8073,
  "default_view_mode": "normal"
}
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `host` | `127.0.0.1` | 展示 URL 使用的地址（server 始终绑定 0.0.0.0） |
| `port` | `8073` | 服务端口 |
| `default_view_mode` | `normal` | 新用户默认阅读模式（`eink` 或 `normal`） |

环境变量 `TIKZ_SERVER_HOST`、`TIKZ_SERVER_PORT`、`TIKZ_DEFAULT_VIEW_MODE` 优先级高于 config.json。

## 后续想法

目前 server 服务 TikZ 和 Plan 两种内容。后续如果有更多 skill 需要预览（HTML 报告、CSV 可视化），可以把 server 抽成独立的通用预览基础设施。和 Codex 讨论过这个方案 —— 用 `manifest.json` 驱动索引页、每个 artifact 独立目录。但现阶段，够用就好，不过早抽象。

---

项目地址：[claude-code-skills](https://github.com/lldxflwb/claude-code-skills)
