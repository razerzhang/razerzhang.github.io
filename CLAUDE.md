# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 命令

```bash
hugo server          # 本地开发，热重载
hugo --minify        # 构建生产版本，输出到 docs/
```

Hugo 版本：0.143.1 (Extended)。不需要 npm/package.json，Hugo 是单一二进制。

## 架构

Hugo 静态博客，部署到 GitHub Pages。推送到 `main` 分支后，GitHub Actions 自动构建并部署。

**CI 流程**（`.github/workflows/hugo.yaml`）：checkout → hugo --minify → npx pagefind --site docs → upload-pages-artifact → deploy-pages。

**关键约定：**
- 构建输出目录是 `docs/`（不是默认的 `public/`），由 `config.toml` 的 `publishDir = "docs"` 控制
- `docs/` 目录需要提交到 git，GitHub Pages 直接读取此目录
- 模板使用 `{{ define "header" }}`、`{{ define "main" }}` 覆盖 `baseof.html` 中的同名 block

## 模板结构

```
layouts/
├── index.html              # 首页（覆盖 header + main），聚合 posts/projects/methodology 三个 section
├── _default/
│   ├── baseof.html         # 基础骨架，定义字体、CSS、footer
│   ├── single.html         # 文章详情页
│   ├── list.html           # 文章列表页
│   └── search.html         # 搜索页（layout: search），基于 JSON index 的客户端搜索
├── search/
│   └── single.html         # 搜索页备用模板（同 _default/search.html）
├── methodology/
│   ├── skill-map.html      # 框架地图总览页（layout: skill-map），数据来自 front matter 的 frameworks/categories
│   └── framework-detail.html # 框架详情页（layout: framework-detail），通过 ?id= 参数选择框架，JS 动态渲染
└── partials/
    ├── nav.html            # 导航栏
    └── icons/              # 内联 SVG 图标（calendar, clock, tags, github, linkedin, x-twitter）
```

图标用 `{{ partial "icons/calendar.html" . }}` 引用，不使用 Font Awesome CDN。

## 内容结构

站点有三个内容 section：

- **posts**：文章，Bundle 方式组织 `content/posts/文章名/index.md`
- **projects**：项目展示，单文件 `content/projects/项目名.md`，front matter 含 `card_label`、`highlights`、`github` 字段
- **methodology**：方法论模块，`content/methodology/` 下的 `.md` 文件。其中 `skill-framework-map.md` 用 front matter 的 `frameworks` 和 `categories` 数组存储所有框架数据（非独立 md 文件），`framework-detail.md` 作为详情页入口

文章 front matter 字段：
```yaml
title: "标题"
date: 2025-02-16T09:00:00+08:00
draft: false
description: "meta description，用于 SEO"
summary: "摘要，显示在列表页和首页卡片（优先于自动截断）"
tags: ["标签"]
categories: ["分类"]
```

## CSS 规范

所有颜色通过 `static/css/site.css` 中的 CSS 变量管理，不要硬编码颜色值：

```css
--text-color: #000000
--muted-text-color: #333333
--section-bg: #ffffff
--heading-color: #0f172a
--highlight-text: #1d3b75
--highlight-bg: rgba(37, 99, 235, 0.08)
--card-border: rgba(37, 99, 235, 0.1)
--footer-bg: #0f172a
--footer-text: rgba(148, 163, 184, 0.85)
--card-shadow / --card-shadow-hover
--transition-speed: 0.3s
```

## Giscus 评论

评论系统已启用，配置在 `config.toml` 的 `[params.giscus]`。

## 搜索

两套搜索实现：
1. **Pagefind**（生产环境）：CI 中 `hugo --minify` 之后运行 `npx pagefind --site docs` 生成索引。本地开发时需手动运行
2. **JSON 全文搜索**（fallback）：`layouts/_default/search.html` 中基于 `/index.json`（由 `home = ["HTML", "RSS", "JSON"]` 输出）的客户端 JS 搜索

搜索页内容入口为 `content/search.md`。
