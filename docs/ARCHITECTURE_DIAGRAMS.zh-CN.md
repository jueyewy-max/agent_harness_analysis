# 应用架构图

[返回首页](../README.md) · [关键能力对比](CAPABILITIES.zh-CN.md) · [完整报告](../ARCHITECTURE_ANALYSIS.zh-CN.md)

调研日期：2026-11-01。每张图只保留主要职责与关键回路，未穷举模块。**架构图不是完整调用追踪，也不是安全边界证明。**

颜色统一：蓝色为入口，绿色为执行核心，橙色为权限与策略，紫色为持久状态。实线通常表示主路径，虚线的含义在每图说明中列明。

图以 SVG 展示，便于直接在 GitHub 查看和放大；每图附 Mermaid 源码，可在支持 Mermaid 的编辑器中修改。

## 0. 共同执行骨架

![共同执行骨架](diagrams/overview.svg)

共同抽象，不代表每个项目的精确调用顺序。关键职责是上下文、模型、工具权限、执行及状态回写。

为什么需要这些层：模型负责选择动作，确定性程序负责记录事实、控制权限和恢复执行。

[放大 SVG](diagrams/overview.svg) · [Mermaid 源文件](diagrams/overview.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["用户输入 / 项目指令"] --> C["上下文组装与预算管理"]
    C --> M["模型请求与流式响应"]
    M --> D{"需要调用工具？"}
    D -->|是| P["工具校验与权限决策"]
    P --> A["必要的审批 / 执行隔离"]
    A --> E["执行工具并收集结果"]
    E --> H["会话记录 / 历史更新"]
    H --> C
    D -->|否| F["检查完成条件 / 等待输入"]
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U input
    class C,M,D,E,F core
    class P,A policy
    class H state
```

</details>

## 1. DeepSeek Harness：由插件组合整套运行时

![DeepSeek Harness：由插件组合整套运行时](diagrams/deepseek.svg)

图中实线为主要调用或数据路径；虚线为插件装配关系。核心组件本身也是插件。

为什么这样实现：替换运行环境或编排策略时，尽量更换能力供应者，而不是复制产品。代价是配置顺序、生命周期和事件语义更复杂。

[放大 SVG](diagrams/deepseek.svg) · [Mermaid 源文件](diagrams/deepseek.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["Web / Headless / SDK / ACP"] --> B["Profile + Bundles + 配置补丁"]
    B --> X["Cordis 插件树与服务上下文"]
    X -.-> L["Agent Loop 插件"]
    X -.-> T["工具注册与执行器"]
    X -.-> S["Session 追加式日志"]
    L --> C["提示词 / 工具集 / 历史投影"]
    C --> M["LLM 适配器与模型"]
    M --> T
    T --> P["Pre Hook → 审批 → Guards"]
    P --> E["Native 或 PTC 工具执行"]
    E --> R["Post Hook / 结果规范化"]
    R --> S
    S --> C
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U,B input
    class X,L,T,C,M,E,R core
    class P policy
    class S state
```

</details>

## 2. Codex：核心引擎协调协议、审批与执行

![Codex：核心引擎协调协议、审批与执行](diagrams/codex.svg)

实线展示主要执行路径；虚线表示恢复及向客户端反馈事件。图中的协议入口是概念归纳，各客户端的具体入口不同。

为什么这样实现：把模型请求与实际执行权限分开，集中处理审批和沙箱升级，并支持多个界面复用引擎。代价是核心及跨平台维护成本较高。

[放大 SVG](diagrams/codex.svg) · [Mermaid 源文件](diagrams/codex.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["CLI / app-server 客户端"] --> I["输入与事件协议"]
    I --> L["Rust Session / run_turn"]
    L --> C["上下文预算与压缩"]
    C --> M["模型请求与响应流"]
    M --> T["Tool Router / 并发门控"]
    T --> P["Tool Orchestrator"]
    P --> A["审批 / 网络许可 / 沙箱选择"]
    A --> E["平台执行器与工具运行时"]
    E --> R["工具结果与事件"]
    R --> L
    L --> H["Rollout / 持久会话记录"]
    H -.-> L
    R -.-> I
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U,I input
    class L,C,M,T,E,R core
    class P,A policy
    class H state
```

</details>

## 3. Claude Code 2.1.88：中心循环与流式工具执行

![Claude Code 2.1.88：中心循环与流式工具执行](diagrams/claude-code.svg)

仅代表第三方保存的 2.1.88 历史实现。流式工具执行受 gate 控制。虚线表示可选子代理及记录路径，不表示 OS 隔离。

为什么这样实现：一套 query 循环统一协调流、工具、压缩和恢复，子代理复用同一执行语义。代价是中心职责较多，流回退不能自动撤销已发生的工具副作用。

[放大 SVG](diagrams/claude-code.svg) · [Mermaid 源文件](diagrams/claude-code.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["终端 / 非交互消费者"] --> Q["query() 异步生成器"]
    Q --> C["Snip → Microcompact → Autocompact"]
    C --> M["模型请求与响应流"]
    M --> T["工具块 / StreamingToolExecutor"]
    T --> P["统一执行管道：校验 / Hooks / 权限"]
    P --> E["Tool.call / 执行并发安全约束"]
    E --> R["结果缓冲 / 按接收顺序交付"]
    R --> Q
    M --> S["无工具时检查 Stop Hooks"]
    S -->|继续| Q
    S -->|完成| F["返回 / 等待输入"]
    E -.-> A["AgentTool：子代理上下文"]
    A -.-> Q
    Q -.-> H["消息与 Transcript 记录"]
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U input
    class Q,C,M,T,E,R,S,F,A core
    class P policy
    class H state
```

</details>

## 4. OpenCode：服务图、多模型与多客户端

![OpenCode：服务图、多模型与多客户端](diagrams/opencode.svg)

图按职责分层，不把 Hooks 与权限画成严格时序。虚线表示服务端向客户端提供状态。

为什么这样实现：用 Effect 服务依赖和生命周期管理，复用模型、会话与权限能力。代价是抽象学习成本和模型差异适配成本。

[放大 SVG](diagrams/opencode.svg) · [Mermaid 源文件](diagrams/opencode.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["TUI / Web / Desktop"] --> S["独立服务端与 API"]
    S --> X["Effect 服务图 / 依赖与生命周期"]
    X --> Q["Session Inbox / Runner"]
    Q --> C["历史 / 指令 / 上下文压缩"]
    C --> M["AI SDK 适配层 / 模型提供方"]
    M --> T["工具运行时"]
    T --> P["权限服务 / 插件 Hooks"]
    P --> E["文件 / Shell / MCP 等能力"]
    E --> Q
    Q --> H["SQLite 会话状态"]
    H --> C
    Q -.-> S
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U,S input
    class X,Q,C,M,T,E core
    class P policy
    class H state
```

</details>

## 5. Gemini CLI：回合、工具调度与策略分离

![Gemini CLI：回合、工具调度与策略分离](diagrams/gemini-cli.svg)

实线表示主路径；虚线表示辅助服务。策略拒绝、错误恢复等分支为保持可读性未展开。

为什么这样实现：让等待确认、取消和并发工具都成为可管理的状态；策略单独配置。代价是状态机与 Hook 交互更复杂。

[放大 SVG](diagrams/gemini-cli.svg) · [Mermaid 源文件](diagrams/gemini-cli.mmd)

<details>
<summary>查看 Mermaid 源码</summary>

```mermaid
flowchart TB
    U["CLI / 非交互消费者"] --> C["GeminiClient：回合与上下文"]
    C --> M["Turn / 模型流事件"]
    M --> Q["Scheduler：工具作业状态机"]
    Q --> P["PolicyEngine：分层规则"]
    P --> A["必要时等待用户确认"]
    A --> E["ToolExecutor / 执行环境"]
    E --> R["完成 / 错误 / 取消结果"]
    R --> C
    C -.-> G["压缩 / 输出遮蔽 / 循环检测"]
    G -.-> C
    C -.-> H["Chat Recording / 会话记录"]
    classDef input fill:#eff6ff,stroke:#2563eb,color:#172554
    classDef core fill:#f0fdf4,stroke:#16a34a,color:#14532d
    classDef policy fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    classDef state fill:#faf5ff,stroke:#9333ea,color:#581c87
    class U input
    class C,M,Q,E,R,G core
    class P,A policy
    class H state
```

</details>

## 证据与版本

各图固定源码提交及直接证据见[完整报告](../ARCHITECTURE_ANALYSIS.zh-CN.md)。Claude Code 的历史核心图与报告中的较新官方 mods 分开处理，不将不同版本拼接为一套当前实现。

重新生成 SVG：使用 `@mermaid-js/mermaid-cli` 对 `docs/diagrams/*.mmd` 逐一渲染。本次工具版本记录于 [渲染记录](diagrams/RENDERING.md)。
