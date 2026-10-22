+++
title = "DeerFlow · 开源贡献"
description = "我在字节跳动开源的 DeerFlow 上的贡献：App Key 控制面和按 Agent 配置的链路追踪两个功能，以及脚本驱动的 Dynamic Workflows 等设计提案和部署评审。"
summary = "提交 App Key 控制面和按 Agent 配置的链路追踪两个功能（评审中）；提出脚本驱动的 Dynamic Workflows，已由社区分阶段实现，并做了多 Gateway 部署评审。"
weight = 3
card_label = "开源贡献 · 功能 + 设计"
highlights = ["DeerFlow", "Dynamic Workflows", "权限控制", "可观测性", "RFC"]
github = "https://github.com/bytedance/deer-flow/issues?q=author%3Arazerzhang"
date = 2026-10-20T09:00:00+08:00
draft = false
+++

[DeerFlow](https://github.com/bytedance/deer-flow) 是字节跳动开源的长任务 SuperAgent 框架，可以借助沙箱、记忆、工具、Skill 和子 Agent 完成研究、写代码等需要几分钟到几小时的任务。我在企业里基于它搭建智能体平台，下面是我在社区里提交的功能，以及参与的设计和评审。

## 功能贡献

### feat(tracing)：按 Agent 配置链路追踪，并支持导出到 Arize

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

### feat：App Key 控制面

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

## 设计提案与评审

### Dynamic Workflows：用脚本编排 Agent 调用

- 提案：[bytedance/deer-flow#6837](https://github.com/bytedance/deer-flow/issues/6837)
- 角色：**提出设计 + 部署评审**
- 进展：社区开发者按阶段实现，Phase 1（[#6858](https://github.com/bytedance/deer-flow/pull/6858)）和 Phase 2（[#6920](https://github.com/bytedance/deer-flow/pull/6920)）**已合并**，Phase 3、4 正在推进

**提案**：让主 Agent 针对用户的任务写一段小的编排脚本，由运行时在后台执行。下一步做什么由脚本里的控制流决定（顺序 `await`、有上限的并行、条件、循环和显式的错误处理），每一步具体的工作仍然交给 Agent 和模型完成。这样编排过程可以检查、可以复用，中间结果在脚本步骤之间传递，不用每一步都塞回主 Agent 的对话里。

我用产品合规检查报告做例子：合规 Agent 检索适用的要求和依据，成分检查 Agent 比对配方，评审 Agent 给出结构化结论，必要时退回修改，最后生成报告。每个关键产物都要经过用户确认。

提案里还定义了几条边界：

- 启动后马上返回 workflow-run ID，对话不被阻塞；支持查看状态、取消和明确约定的恢复。
- 复用现有的 AgentRuns、权限、Agent 配置、模型、工具和 Skill，不另起一套 Agent 运行时。
- 生成的脚本只能使用受限的宿主能力，不能绕过用户和 Agent 的权限，也拿不到额外的凭证、文件系统、shell 或网络访问。

**部署评审**：Phase 1、2 合并后，我对照 `main` 的代码做了一次部署评审（[评论](https://github.com/bytedance/deer-flow/issues/6837#issuecomment-6248477604)），指出当前实现只适合单进程：

- 工作流存储由单个进程独占，运行中的 worker、取消和审批事件、并发配额都只在本进程内有效，多个 Gateway 之间没有协调。
- 审批后的恢复依赖内存里的 worker，进程重启后未完成的工作流只会被标记为中断，不会接着执行。
- 工作流记录、子任务提交和外部副作用不在同一个事务里。

我提出的部署要求是：在负载均衡后面部署多个 Gateway 时，从 A 启动的工作流要能在 B 上查看、审批和取消；要定义清楚执行权归属、接管，以及如何防止过期的执行者继续写入或派发任务。实现方据此把“跨重启的持久化恢复”和“多 Gateway 部署”列为 Phase 3、4，并用两个真实 Gateway 进程、PostgreSQL、Redis 和 nginx 轮询做了验收测试。

### App Key：从访问控制 RFC 到控制面

- [RFC] Agent API Access Control：[bytedance/deer-flow#4310](https://github.com/bytedance/deer-flow/issues/4310)
- [RFC] App-Key Control Plane：[bytedance/deer-flow#4551](https://github.com/bytedance/deer-flow/issues/4551)
- 实现：上面的 [#4646](https://github.com/bytedance/deer-flow/pull/4646)

企业应用需要从自己的后端调用 Agent。如果把后端服务当成浏览器用户，要么共享用户会话，要么让一个长期凭证拿到整个 Gateway 的权限。第一版 RFC 提出给外部应用一个独立的身份边界：每个 App Key 只能使用明确授权的 Agent 和能力，遵循最小权限。社区对这份 RFC 做了详细的设计评审，指出了和现有代码的冲突。

第二版 RFC 吸收了这些意见，把配置文件驱动改成数据库驱动的控制面，解决几个实际问题：吊销需要重启才生效、多个 Pod 之间配置不一致、密钥明文存储、没有管理界面、没有审计、没法平滑轮换。

### Versioned Agent Bundles：可复现的 Agent 发布

- 提案：[bytedance/deer-flow#6862](https://github.com/bytedance/deer-flow/issues/6862)

提议把一个 Agent 的定义和依赖打包成不可变的版本（比如 `report-agent@1.2.0`），包括 SOUL、模型设置、Skill、工具和 MCP 选择、子 Agent 配置。这个版本先在 QA 环境评估，再原样发布到生产，绑定生产的资源和凭证后激活；之后可以查到每次运行用的是哪个版本，需要时回滚。

### 企业落地的痛点和建议

- 讨论：[bytedance/deer-flow#4357](https://github.com/bytedance/deer-flow/issues/4357)

结合在企业里搭建智能体平台的经验，整理了 7 个机会点：Agent 管理界面、Agent 配置的持久化、多租户、K8S 部署、应用接入控制、Skill 的动态加载，以及工具调用的管理、监控和评估。核心观点是 DeerFlow 正在走向 Agent 平台，企业场景缺少的是管理端。这个讨论和社区成员来回交流了多轮。

此外还提过几个较小的建议：Agent 的 Skill 和工具默认最小权限、按需开启（[#2122](https://github.com/bytedance/deer-flow/issues/2122)），不同 Agent 挂载不同的 Skill 和工具组（[#2269](https://github.com/bytedance/deer-flow/issues/2269)），自定义 Agent 的 SOUL.md 配置（[#2383](https://github.com/bytedance/deer-flow/issues/2383)），以及模型缓存命中率统计（[#2635](https://github.com/bytedance/deer-flow/issues/2635)）。
