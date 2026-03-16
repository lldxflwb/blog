---
title: "用 Claude Code Skill 搭建个人预览服务：TikZ 渲染 + 墨水屏阅读器"
date: 2026-03-16
tags: ["claude-code", "skills", "tikz", "e-ink", "preview-server"]
summary: "一个 Claude Code Skill 实现的本地预览服务：TikZ 图表实时渲染为 SVG，Plan 文档墨水屏友好阅读，自动分页翻页，一套基础设施服务多种内容。"
draft: false
---

## 需求起点

我日常用 Claude Code 做开发，经常需要两类输出的可视化：

1. **TikZ 图表** —— 架构图、流程图、状态机这些，Claude 写好 TikZ 代码后我需要看到渲染结果
2. **Plan 文档** —— Claude Code 的 Plan Mode 会产出大量 markdown 计划文档，我想在墨水屏上舒服地阅读它们

以前的做法是手动编译 LaTeX、手动打开 markdown，非常割裂。于是我做了一个 Claude Code Skill，把这些整合成一个自动化的预览服务。

## 架构设计

整个 skill 的核心思路是 **"脚本负责生产，server 负责展示"**：

```
skills/tikz/
├── SKILL.md              # Claude 读取的指令
├── config.json           # host/port/默认模式配置
├── setup.md              # 依赖安装引导
└── scripts/
    ├── render_tikz.py     # TikZ → PDF → SVG
    ├── tikz_server.py     # HTTP 预览服务
    ├── ensure_server.py   # 自动启停管理
    └── config.py          # 配置读取（env > config.json > defaults）
```

**配置优先级链**：环境变量 > config.json > 硬编码默认值。这样不同机器可以通过 config.json 配置不同的 IP 和端口，不依赖 shell 环境变量是否加载。

## TikZ 渲染流程

Claude 只需要两步：

1. 写一个 `.tex` 文件到项目目录
2. 调用 `render_tikz.py <文件路径>`

脚本内部做了这些事：

- 自动检测 TeX 引擎（tectonic 优先，pdflatex 兜底）
- 自动包裹 `standalone` 文档类（如果不是完整文档的话）
- 识别 `\usepackage`、`\usetikzlibrary`、`\definecolor`、`\tikzset` 等 preamble 命令
- 编译 PDF，转换 SVG（pdf2svg）
- 带微秒时间戳输出到缓存目录，避免文件名冲突
- 自动启动预览 server（如果没在运行的话）
- 输出一行 URL，完事

```bash
$ python3 render_tikz.py architecture.tex
http://10.8.0.2:9094/architecture_20260316_111125_932281.svg
```

Server 管理完全透明 —— `render_tikz.py` 内部调用 `ensure_server.py`，AI 和用户都不需要操心。

## 预览服务

`tikz_server.py` 是一个基于 Python 标准库的轻量 HTTP 服务，绑定 `0.0.0.0`，不依赖 Flask 或 FastAPI。

### 首页索引

访问根路径显示两个区域：

- **计划** —— 列出 `~/.claude/plans/` 下的所有 plan 文档，按创建时间倒序，显示标题和摘要
- **TikZ 渲染** —— 列出所有 SVG 缩略图，可点击查看大图或下载 PDF

### Plan 阅读器

点击任意 plan 进入阅读页面，支持两种模式：

**墨水屏模式**（默认可配置）：
- Georgia 衬线字体，大字号（1.35rem），加粗（font-weight: 600）
- 高对比度配色，所有边框、背景都加深，适合彩色墨水屏
- 自动按视口高度分页，**完全不需要滚动**
- 点击左侧 35% 区域翻上一页，右侧 35% 翻下一页
- 键盘左右箭头也能翻页
- 代码块自动换行（`white-space: pre-wrap`），不水平滚动
- 超长表格按行拆分，每页保留表头
- 超长代码块按行拆分到多页

**普通模式**：
- 系统无衬线字体，长页面滚动
- 可选择文字
- 标准 web 阅读体验

两种模式通过右上角按钮切换，状态存储在 `localStorage`，刷新和换页都保持。

## 分页的坑

墨水屏分页是这个项目里最折腾的部分。踩了好几个坑：

**离屏测量不准。** 最初用一个隐藏的 div 测量元素高度，但字体大小、行高、padding 和真实渲染环境不一致，导致分页偏差。最终方案是直接把内容渲染到真实容器里，逐个添加元素，用 `scrollHeight > clientHeight` 判断是否溢出。

**嵌套代码块不分页。** marked.js 把列表里的代码块生成为 `<ol><li><pre>`，分页逻辑只检查顶层 children 的 `tagName === 'PRE'`，永远匹配不到。修复方案是递归遍历容器，找到嵌套的 `pre/table` 并拆分（感谢 Codex 的诊断）。

**base64 中文乱码。** `atob()` 只能处理 Latin-1 编码，中文 UTF-8 字节会乱码。用 `TextDecoder` 解码修复。

**f-string 和 JS 冲突。** Python f-string 里的 `\n` 会被解释为真换行，破坏 JS 字符串。markdown 内容中的反引号和 `</script>` 也会破坏页面。最终用 base64 编码嵌入内容，彻底避免转义问题。

## Server 生命周期

`ensure_server.py` 的设计目标是 **"调用方不需要关心 server 状态"**：

1. 先对端口做 healthcheck（探测 3 次，每次 2 秒超时）
2. 如果有响应 → 直接复用
3. 如果端口被占但不响应 → 杀掉僵尸进程，重新启动
4. 如果端口空闲 → 启动新 server，轮询 healthcheck 直到就绪

`--stop` 支持显式停止，先查状态文件，再按端口 lsof 兜底，确保总能找到进程。

## 安装

```bash
# 安装依赖
brew install tectonic pdf2svg

# 安装 skill（软链接，改代码自动生效）
ln -s /path/to/skills/tikz ~/.claude/skills/tikz
```

config.json 配置示例：

```json
{
  "host": "10.8.0.2",
  "port": 9094,
  "default_view_mode": "eink"
}
```

## 使用

```
# TikZ 渲染
/tikz 画一个神经网络架构图

# 查看所有渲染和计划
浏览器打开 http://<your-host>:9094/

# 停止服务
python3 ~/.claude/skills/tikz/scripts/ensure_server.py --stop
```

## 后续

目前 server 主要服务 TikZ 和 Plan 两种内容。后续如果有更多 skill 需要预览（比如 HTML 报告、CSV 可视化），可以把 server 抽成独立的预览基础设施。但现阶段，够用就好。

---

项目地址：[claude-code-skills](https://github.com/lldxflwb/claude-code-skills)
