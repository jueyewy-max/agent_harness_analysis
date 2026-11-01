# 关键能力对比

[返回首页](../README.md) · [架构图](ARCHITECTURE_DIAGRAMS.zh-CN.md) · [详细分析与源码证据](../ARCHITECTURE_ANALYSIS.zh-CN.md)

比较日期：2026-11-01。拆分为三张窄表，便于在 GitHub 阅读。以下比较结构与机制，不做未经实测的性能排名。

**阅读口径：** DeepSeek、Codex、OpenCode、Gemini CLI 以本次固定的官方源码提交为准；Claude Code 核心以第三方保存的 **2.1.88 历史源码**为准，较新官方 mods 单独标注。未检查的能力不等于不支持。

## 1. 执行与上下文

| 应用 | 执行核心 | 并发机制 | 上下文管理 |
|---|---|---|---|
| **DeepSeek Harness** | Agent Loop 本身是 Cordis 插件 | 执行器控制批次；PTC 安全调用并行，独占调用形成屏障 | 日志投影、工具结果裁剪、可组合压缩插件 |
| **Codex** | Rust Session / `run_turn` | 按工具并发声明使用共享/独占门控 | token 预算、压缩及上下文更新机制 |
| **Claude Code 2.1.88** | `query()` 异步生成器与显式循环 | 流式工具执行受 gate 控制；安全工具并行，结果按接收顺序交付 | Snip、Microcompact、Autocompact；压缩失败熔断 |
| **OpenCode** | Effect Session Runner 与 continuation | Effect 组织并发与生命周期；本次未系统审计工具冲突调度 | 保留近期原文；支持后台预计算摘要 |
| **Gemini CLI** | GeminiClient + Turn + Scheduler | 工具状态机与并发安全判断 | 压缩、工具输出遮蔽、循环检测 |

## 2. 扩展、权限与状态

| 应用 | 扩展边界 | 权限机制 | 状态与恢复 |
|---|---|---|---|
| **DeepSeek Harness** | 模型、工具、执行器、存储、循环均可组合 | 可重写 Hooks 与只收紧权限的 Guards 分开；执行隔离由后端承担 | 追加日志、历史投影、fork / resume |
| **Codex** | MCP、Hooks、扩展与客户端协议；核心协调执行生命周期 | 编排器统一审批、网络许可和平台沙箱；升级权限重新审查 | Rollout、持久记录、会话恢复 |
| **Claude Code** | 历史版统一工具管道与 Hooks；较新官方仓库另有函数 mods | 历史版校验、Hooks、`canUseTool`；新版 `sec-default` 保护组织策略 | 历史版消息与 transcript、子代理上下文与恢复逻辑 |
| **OpenCode** | 模型适配、Effect 服务、插件 Hooks、多客户端 | action/resource 规则、询问/拒绝、无人值守处理 | SQLite；Runner 显式处理迁移/重载 continuation |
| **Gemini CLI** | 工具、MCP、Hooks、扩展与声明式策略 | PolicyEngine 分层优先级；非交互默认决策更保守 | Chat Recording 与历史恢复相关服务 |

**权限判断不等于沙箱隔离。** 插件接口隔离、子代理上下文隔离也不等于容器或虚拟机隔离。上述表格不构成安全审计。

## 3. 为什么这样设计，以及付出的代价

| 应用 | 优先解决的问题 | 设计收益 | 主要工程代价 |
|---|---|---|---|
| **DeepSeek Harness** | 如何替换模型、运行环境甚至整个循环 | 新能力可通过组合接入，适合研究与平台化 | 插件顺序、作用域、卸载和日志兼容性复杂 |
| **Codex** | 如何约束长任务、子进程与实际执行权限 | 协议、审批和执行生命周期有明确边界 | 跨平台实现多；核心变更需协调多个模块 |
| **Claude Code** | 如何让编码工作流持续推进并减少等待 | 中心循环统一协调流式执行、恢复和压缩 | 核心职责集中；副作用与回退处理复杂；历史源码不能代表最新版 |
| **OpenCode** | 如何复用多模型、多客户端与运行位置 | 服务复用、依赖显式、适配层可扩展 | Effect 学习成本；模型差异仍需逐项适配 |
| **Gemini CLI** | 如何治理工具作业、确认和企业规则 | 策略配置与调度执行分离，等待确认状态明确 | 调度、Hooks、审批和取消之间状态组合较多 |

## 4. 按目标选择参考实现

| 你的目标 | 优先阅读 | 阅读重点 |
|---|---|---|
| 研究可替换的 Agent Loop 或程序化工具调用 | DeepSeek Harness | Cordis、Session 投影、Native / PTC 执行器 |
| 实现受控命令执行与多客户端引擎 | Codex | ToolOrchestrator、平台沙箱、app-server 协议 |
| 优化流式执行、上下文和子代理体验 | Claude Code 历史存档 | query、StreamingToolExecutor、runAgent |
| 开发多模型、多界面的开源产品 | OpenCode | Effect 服务图、模型适配、服务端与 SQLite |
| 设计策略配置与工具作业状态机 | Gemini CLI | Scheduler、PolicyEngine、确认和取消 |

源码提交、具体路径与证据边界见[完整报告第 1、3—7、9 节](../ARCHITECTURE_ANALYSIS.zh-CN.md)。其中 Claude Code 两份镜像的六个关键文件与所附 source map 一致，但官方 npm 来源校验未完成。
