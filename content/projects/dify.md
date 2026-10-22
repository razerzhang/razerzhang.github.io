+++
title = "Dify · 开源贡献"
description = "我给 Dify 贡献的功能：为指定工作流单独配置日志保留周期的定时清理任务；以及 LLM Cache 命中统计的设计提案。"
summary = "给 Dify 增加按工作流单独设置保留周期的日志定时清理任务，高频工作流可以更快清理，不影响其他工作流。已合并。"
weight = 2
card_label = "开源贡献 · 已合并"
highlights = ["Dify", "Celery", "数据治理", "开源贡献"]
github = "https://github.com/langgenius/dify/issues?q=author%3Arazerzhang"
date = 2026-02-11T12:54:36+08:00
draft = false
+++

[Dify](https://github.com/langgenius/dify) 是一个开源的 LLM 应用开发平台，可以在同一个工作区里搭建 Agent 工作流和 RAG 流水线。下面是我给它贡献的功能和设计提案。

## 功能贡献

### feat(api)：为指定工作流设置独立的日志清理策略

- PR：[langgenius/dify#31843](https://github.com/langgenius/dify/pull/31843)
- 状态：**已合并**（2026 年 2 月）
- 关联 issue：[#31883](https://github.com/langgenius/dify/issues/31883)

**问题**：有些工作流调用量很大，运行日志增长得很快。但 Dify 原有的日志清理只有一个全局保留周期，没法单独给这些高频工作流设置更短的保留时间。

**做法**：新增一个 Celery 定时任务，只清理指定工作流的运行日志，保留周期单独配置：

- `SPECIFIC_WORKFLOW_LOG_CLEANUP_ENABLED`：开启或关闭这个任务
- `SPECIFIC_WORKFLOW_IDS`：要清理的工作流 ID，用逗号分隔
- `SPECIFIC_WORKFLOW_LOG_RETENTION_DAYS`：保留天数，默认 7 天

任务每天凌晨 3 点执行，会级联删除这些运行记录关联的数据，其他工作流的日志不受影响。

## 设计提案

### LLM Cache 命中统计

- 提案：[langgenius/dify#35653](https://github.com/langgenius/dify/issues/35653)

建议增加模型缓存命中情况的统计。知道哪些调用命中了缓存，才能有针对性地调整 prompt 结构和工具的使用方式，优化成本和性能。
