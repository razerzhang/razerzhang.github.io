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

**关键约定：**
- 构建输出目录是 `docs/`（不是默认的 `public/`），由 `config.toml` 的 `publishDir = "docs"` 控制
- `docs/` 目录需要提交到 git，GitHub Pages 直接读取此目录
- 模板使用 `{{ define "header" }}`、`{{ define "main" }}` 覆盖 `baseof.html` 中的同名 block

## 模板结构

```
layouts/
├── index.html              # 首页（覆盖 header + main）
├── _default/
│   ├── baseof.html         # 基础骨架，定义字体、CSS、footer
│   ├── single.html         # 文章详情页
│   ├── list.html           # 文章列表页
│   └── search.html         # 搜索页（layout: search）
└── partials/
    ├── nav.html            # 导航栏
    └── icons/              # 内联 SVG 图标（calendar, clock, tags, github, linkedin, x-twitter）
```

图标用 `{{ partial "icons/calendar.html" . }}` 引用，不使用 Font Awesome CDN。

## 内容结构

文章使用 Bundle 方式组织：`content/posts/文章名/index.md`

Front matter 字段：
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

## Giscus 评论配置

评论系统已集成但默认关闭。启用步骤：
1. 访问 giscus.app，填入 `razerzhang/razerzhang.github.io`，获取 `repoId` 和 `categoryId`
2. 在 `config.toml` 的 `[params.giscus]` 中填入值并设置 `enabled = true`

## 搜索（Pagefind）

搜索索引在 CI 中构建：`hugo --minify` 之后运行 `npx pagefind --site docs`。本地开发时搜索不可用（需手动运行 pagefind）。搜索页模板为 `layouts/_default/search.html`，内容入口为 `content/search.md`。
