+++
title = "DeerFlow · 开源贡献"
description = "我给字节跳动开源的 DeerFlow 提交的功能：App Key 控制面，以及按 Agent 配置的链路追踪。"
summary = "给 DeerFlow 提交两个较大的功能：面向外部调用的 App Key 控制面，以及按 Agent 独立配置、支持 Arize 导出的链路追踪。正在评审中。"
weight = 3
card_label = "开源贡献 · 评审中"
highlights = ["DeerFlow", "Agent 平台", "权限控制", "可观测性", "开源贡献"]
github = "https://github.com/bytedance/deer-flow/pulls?q=is%3Apr+author%3Arazerzhang"
date = 2026-10-20T09:00:00+08:00
draft = false
+++

[DeerFlow](https://github.com/bytedance/deer-flow) 是字节跳动开源的长任务 SuperAgent 框架，可以借助沙箱、记忆、工具、Skill 和子 Agent 完成研究、写代码等需要几分钟到几小时的任务。下面是我给它提交的功能。

## feat(tracing)：按 Agent 配置链路追踪，并支持导出到 Arize

- PR：[bytedance/deer-flow#7179](https://github.com/bytedance/deer-flow/pull/7179)
- 状态：**评审中**
- 关联 issue：[#7162](https://github.com/bytedance/deer-flow/issues/7162)

**问题**：原来的链路追踪是整个部署共用一套配置。两个同时运行的 Agent 没法把追踪数据发到不同的目的地，给某个 Agent 轮换凭证也会影响其他 Agent 的运行。在界面上保存的设置，还需要在多个 Gateway 副本之间以统一的时间点生效。

**做法**：

- 管理员可以维护加密存储的追踪服务配置，每个 Agent 再选择继承、关闭或使用指定配置。管理凭证和选择配置是两种分开的权限。
- 配置变更按版本发布，用乐观并发控制，再加一道共享数据库的栅栏，保证运行中的任务和排队中的任务看到一致的配置。
- 每次运行都会冻结一份加密的配置快照，委派任务、后台任务、排队、检查点改写和恢复执行都沿用这份快照，不会中途切换。
- 新增 Arize 的 OTLP HTTP/gRPC 导出，LangSmith 和 Langfuse 也按 Agent 隔离路由，支持凭证轮换。
- 前端增加 Agent 的追踪设置界面：凭证脱敏编辑、显示当前生效的版本和状态，以及冲突后的恢复。

## feat：App Key 控制面

- PR：[bytedance/deer-flow#4646](https://github.com/bytedance/deer-flow/pull/4646)
- 状态：**评审中**
- 设计讨论：[#4551](https://github.com/bytedance/deer-flow/issues/4551)

**问题**：外部系统调用 DeerFlow 时，需要一种可以管理、可以随时吊销、权限范围可控的访问凭证。

**做法**：

- 新增只有平台管理员能用的 App Key 管理：Profile、哈希存储的凭证、能力授权、审计事件，以及对应的数据库迁移。
- 凭证只在生成时显示一次，数据库里只存 SHA-256 哈希。
- 每次请求都直接查数据库校验，不加本地或 Redis 缓存。这样吊销凭证或停用 Profile 后，每个 Gateway 实例的下一次请求就会生效。
- App Key 只能访问 5 个指定的接口，可用的 Agent、模型、Skill、工具组和工具都按白名单限定。
- 在 Workspace 里新增 App Key 管理页面：编辑权限范围、生成凭证、吊销，以及紧急停用。
