# Pi 开发者入门指南

依据 [Understand-Anything 的 understand-onboard 技能](https://github.com/Egonex-AI/Understand-Anything/blob/532a2e97dc007104f9a1b8d11b6cc16ae7c8cd3a/understand-anything-plugin/skills/understand-onboard/SKILL.md)整理，生成日期：2026-11-19。

## 项目概览

Pi 是可扩展的智能体工具集，以 TypeScript monorepo 组织多模型 LLM 接口、通用 Agent 循环、编码 CLI、会话管理与终端界面，并提供 Chord 跨进程对象运行时、Durable 持久会话运行时、远程执行环境及 P2 应用。

| 项目项 | 内容 |
| --- | --- |
| 名称 | pi-monorepo |
| 主要语言 | TypeScript、JavaScript；另含 Rust、Swift、Python、C、Objective-C 等代码 |
| 文档与资源格式 | Markdown、JSON、YAML、TOML、HTML、CSS 等；图谱中的语言列表也包含资源格式 |
| 框架与工程工具 | Docker、GitHub Actions、React、Vitest |
| 运行要求 | Node.js ≥ 22.19.0；安装和开发命令见仓库 README |
| 图谱范围 | 4,395 个扫描文件，14,309 个节点，40,452 条关系 |
| 图谱分析时间 | 2026-11-12T10:50:52.569Z |
| 对应源码 | [6b339afc92e1](https://github.com/AManHasNoName12138/pi/commit/6b339afc92e1dc6574f3234e1850dff36c77294b) |

生成指南前已核对图谱提交、当前 HEAD、暂存区、工作区及未跟踪文件；排除 `.ua/` 分析产物后，未发现源码漂移。指南描述上述源码版本，本次新增的学习文档不计入原图谱覆盖范围。

**分析深度：** 原图谱的 606 个文件有语义说明，另外 3,789 个文件使用标注为“结构摘要”的说明。119 个不支持符号提取的文件仍保留文件级节点；复杂度标签和静态关系用于导航，不代表完整语义审查、运行时调用链或测试覆盖率。详情见[分析报告](../.ua/analysis-report.json)。

首次接触项目，可先读[根 README](../README.md)，再沿下面的 15 步路线阅读。准备修改代码时，再读 [AGENTS.md](../AGENTS.md) 和 [CONTRIBUTING.md](../CONTRIBUTING.md)；根目录的开发命令、P2 的独立检查以及具体模块规范以这些文件为准。交互图谱的启动方法见[图谱说明](../.ua/README.md)。

## 架构分层

图谱按包职责划分 10 层，各包内的测试、示例、文档与配置跟随所属包。这里的“层”是学习分组，不表示所有依赖都严格单向；4,407 个文件级节点包含 4,395 个整文件节点和 12 个部署或构建子节点。

[查看架构总览与导览预览](../.ua/preview.png)

| 层 | 职责 | 关键文件 |
| --- | --- | --- |
| 数据契约与遥测基础层 | 以 pi-schema 的 JSON Schema 构造、类型推导与无 eval 校验，以及 pi-telemetry 的追踪、指标和适配器契约，为模型接口与跨进程运行时提供共同基础。 | [packages/schema/src/index.ts](../packages/schema/src/index.ts)<br>[packages/telemetry/src/index.ts](../packages/telemetry/src/index.ts) |
| 模型接入与智能体循环层 | 以 pi-ai 统一模型目录、提供商鉴权和流式调用，并由 pi-agent-core 管理消息、状态与工具调用循环，供 CLI、Durable 和应用组合使用。 | [packages/ai/src/index.ts](../packages/ai/src/index.ts)<br>[packages/ai/src/types.ts](../packages/ai/src/types.ts)<br>[packages/ai/src/models.ts](../packages/ai/src/models.ts) |
| 工具协议与受控执行层 | 通过 pi-mcp 的独立客户端、传输与工具适配，以及 pi-codemode 仅能调用注入工具的 JavaScript 沙箱，为智能体提供可组合的工具接入与执行能力。 | [packages/mcp/src/client.ts](../packages/mcp/src/client.ts)<br>[packages/codemode/src/declarations.ts](../packages/codemode/src/declarations.ts)<br>[packages/codemode/src/runtime/host.ts](../packages/codemode/src/runtime/host.ts) |
| 终端界面层 | 由 pi-tui 提供差量渲染、输入编辑、组件与原生终端辅助能力，支撑编码代理和 p2 的终端交互。 | [packages/tui/src/tui.ts](../packages/tui/src/tui.ts)<br>[packages/tui/src/terminal.ts](../packages/tui/src/terminal.ts)<br>[packages/tui/src/components/editor.ts](../packages/tui/src/components/editor.ts) |
| Chord 跨进程对象运行时层 | 以 Chord 的契约、主机、对象引用、文档、存储和链路连接进程，并提供插件热更新、HTTP、Worker 以及 Web 和终端客户端的运行基础。 | [packages/chord/src/contracts/index.ts](../packages/chord/src/contracts/index.ts)<br>[packages/chord/src/host/index.ts](../packages/chord/src/host/index.ts)<br>[packages/chord/src/context/index.ts](../packages/chord/src/context/index.ts) |
| Durable 持久会话与任务层 | 在 Chord、模型接口与工具执行能力之上管理可恢复的会话、任务、文档和持久化状态，向 p2 等应用提供 Durable 代理运行时。 | [packages/durable/src/index.ts](../packages/durable/src/index.ts)<br>[packages/durable/src/session/session.ts](../packages/durable/src/session/session.ts)<br>[packages/durable/src/harness/harness.ts](../packages/durable/src/harness/harness.ts) |
| 远程执行环境层 | 由 pi-env 的 SSH 部署守护进程与 ExecutionEnv 客户端承接 Durable 的远程执行，并提供进程、文件和终端操作。 | [packages/env/src/ssh.ts](../packages/env/src/ssh.ts)<br>[packages/env/src/connection.ts](../packages/env/src/connection.ts)<br>[packages/env/src/remote-env.ts](../packages/env/src/remote-env.ts) |
| 编码代理命令行层 | 将模型与智能体循环、终端组件、MCP 和脚本工具组合为 pi CLI，提供交互、打印和 RPC 模式、会话管理、内置编码工具及扩展机制。 | [packages/coding-agent/src/cli.ts](../packages/coding-agent/src/cli.ts)<br>[packages/coding-agent/src/main.ts](../packages/coding-agent/src/main.ts)<br>[packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts) |
| 产品应用与宿主层 | 承载组合 Chord、Durable、模型与执行环境的 p2 服务、插件和界面，以及 p2-mac 的 macOS 宿主和 p3 的设计与探索文件。 | [apps/p2/src/hosts/server.ts](../apps/p2/src/hosts/server.ts)<br>[apps/p2/src/hosts/worker.ts](../apps/p2/src/hosts/worker.ts)<br>[apps/p2/src/api/v1/index.ts](../apps/p2/src/api/v1/index.ts) |
| 工程自动化与评测层 | 汇集 monorepo 根配置与贡献文档、构建发布脚本、GitHub CI/CD、Nix 和开发扩展，并通过 pi-evals 与容器环境执行代理行为及文档评测。 | [README.md](../README.md)<br>[package.json](../package.json)<br>[.github/workflows/ci.yml](../.github/workflows/ci.yml) |

整体可分成两条阅读路线：一条由模型与 Agent 循环进入 CLI、工具、扩展和会话树；另一条由 Chord 进入 Durable、远程执行环境和 P2 应用。两条路线共同使用数据契约、模型接口和工具能力，下面的学习顺序先铺设这些基础。

## 关键概念

### 统一模型接口与提供商实现

pi-ai 用共享类型表达消息、工具和使用量，Models 负责目录、凭据和请求分派。先读类型，再读运行时，可以区分上层依赖的接口和具体提供商接入行为。

阅读：[packages/ai/src/types.ts](../packages/ai/src/types.ts) · [packages/ai/src/models.ts](../packages/ai/src/models.ts) · [packages/ai/src/index.ts](../packages/ai/src/index.ts)。

### Agent 循环与编码会话的职责

agent-loop 连接流式模型输出、工具执行和后续消息，Agent 在其上维护状态与消息队列。编码代理的 AgentSession 再加入历史持久化、模型选择、上下文压缩和运行模式协调；阅读时沿这些职责逐层推进。

阅读：[packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts) · [packages/agent/src/agent.ts](../packages/agent/src/agent.ts) · [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

### 工具接入与执行能力

编码代理的 read、write、edit、bash 工具执行文件和进程操作；MCP 提供外部工具的协议接入。codemode 将工具 schema 转为脚本声明，并把注入工具暴露给隔离的 JavaScript 运行时，适合把多个工具调用组合为一个脚本。

阅读：[packages/coding-agent/src/core/tools/bash.ts](../packages/coding-agent/src/core/tools/bash.ts) · [packages/mcp/src/client.ts](../packages/mcp/src/client.ts) · [packages/codemode/src/declarations.ts](../packages/codemode/src/declarations.ts) · [packages/codemode/src/runtime/host.ts](../packages/codemode/src/runtime/host.ts)。

### 会话树、上下文投影与摘要

session-manager 保存带父子关系的历史条目，并把当前分支投影为模型上下文。compaction 与 branch-summarization 分别为长历史和分支切换准备摘要；理解持久历史与发送给模型的上下文之间的关系，是阅读会话逻辑的关键。

阅读：[packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) · [packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) · [packages/coding-agent/src/core/compaction/branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts)。

### 资源发现、扩展契约与事件执行

resource-loader 发现扩展、技能、提示模板、主题和项目指令；扩展类型约束可注册的接口与事件，runner 执行处理器和命令。把资源来源、允许的行为和事件发生时机分别定位，能帮助理解用户定制怎样接入宿主。

阅读：[packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) · [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts) · [packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)。

### Chord 的对象契约、宿主与状态

Chord 用契约描述对象参数、成员、版本和主机归属，用 host 与 facet 连接插件声明、对象实例和生命周期。store 提供存储接口，delta 表达事务式 JSON 变更，watch 处理监听生命周期；这些概念共同构成跨进程应用基础。

阅读：[packages/chord/src/contracts/index.ts](../packages/chord/src/contracts/index.ts) · [packages/chord/src/host/index.ts](../packages/chord/src/host/index.ts) · [packages/chord/src/store/index.ts](../packages/chord/src/store/index.ts) · [packages/chord/src/delta/index.ts](../packages/chord/src/delta/index.ts) · [packages/chord/src/watch.ts](../packages/chord/src/watch.ts)。

### Durable 的持久会话与任务调度

Durable 将会话、文档和任务放到可恢复的运行时中：session 管理提交与存储，harness 提供恢复、取消和分叉等操作，scheduler 协调可运行工作和等待条件。它使用独立的会话与存储 API，学习时应结合其公共入口和适配器理解状态边界。

阅读：[packages/durable/src/index.ts](../packages/durable/src/index.ts) · [packages/durable/src/session/session.ts](../packages/durable/src/session/session.ts) · [packages/durable/src/harness/harness.ts](../packages/durable/src/harness/harness.ts) · [packages/durable/src/harness/scheduler.ts](../packages/durable/src/harness/scheduler.ts)。

### 远程执行与应用进程的组合

pi-env 通过 SSH 部署 daemon，并用 ExecutionEnv 客户端承接远端文件和进程操作。P2 再把服务 host、会话 worker、Chord 存储与 Durable Harness 组合起来；沿 server 到 worker 的路径阅读，有助于定位服务路由和会话执行各自发生的位置。

阅读：[packages/env/src/ssh.ts](../packages/env/src/ssh.ts) · [packages/env/src/remote-env.ts](../packages/env/src/remote-env.ts) · [apps/p2/src/hosts/server.ts](../apps/p2/src/hosts/server.ts) · [apps/p2/src/hosts/worker.ts](../apps/p2/src/hosts/worker.ts)。

## 引导式学习路线

沿用知识图谱的 15 步导览。每一步先理解职责与相邻模块的关系，再沿链接进入文件；先完成 CLI 路线，再阅读 Chord 与 Durable，有助于区分两套会话运行基础。

### 1. 项目概览与学习地图

先读 README，建立 Pi 作为可扩展智能体框架的整体认识，并区分交互式 CLI、打印模式、RPC 与 TypeScript SDK 的使用入口。它介绍的扩展、技能、提示模板和主题说明了项目如何适应不同工作流。后续先学习模型与工具基础，再追踪编码代理，最后进入 Chord、Durable 和 p2 的应用运行时。

阅读：[README.md](../README.md)。

### 2. 数据契约与遥测基础

从概览中的共享库开始，阅读 pi-schema 与 pi-telemetry 的公共入口，认识工具参数、跨进程契约和运行观测所依赖的基础接口。pi-schema 汇总 JSON Schema 构造、类型推导与校验能力，pi-telemetry 提供与供应商无关的追踪和指标契约。先建立这些概念，有助于理解后续模型接口和应用运行时如何共享数据定义与观测方式。

阅读：[packages/schema/src/index.ts](../packages/schema/src/index.ts) · [packages/telemetry/src/index.ts](../packages/telemetry/src/index.ts)。

### 3. 统一模型与提供商接口

在数据契约基础上，阅读 pi-ai 的核心类型与 Models 运行时，理解模型、消息、工具、提供商选项和使用量如何表达。Models 将模型目录、凭据、鉴权与请求分派集中到统一接口，是上层选择和调用模型的重要连接点。公共入口保留无副作用的核心类型与工具函数，而提供商工厂、实现和兼容入口通过独立子路径提供。

阅读：[packages/ai/src/index.ts](../packages/ai/src/index.ts) · [packages/ai/src/types.ts](../packages/ai/src/types.ts) · [packages/ai/src/models.ts](../packages/ai/src/models.ts)。

### 4. 消息驱动的智能体循环

理解模型接口后，沿 Agent 类型、底层循环和有状态 Agent 封装，观察模型输出如何进入工具执行并继续生成消息。底层循环处理流式输出、工具声明变化、顺序或并行执行、中断与事件发布，Agent 则维护消息、队列和运行生命周期。这个分工为后续 CLI 会话提供通用的执行基础。

阅读：[packages/agent/src/types.ts](../packages/agent/src/types.ts) · [packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts) · [packages/agent/src/agent.ts](../packages/agent/src/agent.ts)。

### 5. MCP 工具与脚本执行

有了智能体循环，再看工具能力如何从外部接入，并如何组合成脚本。MCP 客户端管理 JSON-RPC 请求与响应及工具调用，codemode 将工具 schema 转成脚本声明，并由宿主向隔离运行时注入可用工具。最后阅读 MCP 与 codemode 的组合示例，将协议接入、脚本执行和 Agent 工具循环联系起来。

阅读：[packages/mcp/src/client.ts](../packages/mcp/src/client.ts) · [packages/codemode/src/declarations.ts](../packages/codemode/src/declarations.ts) · [packages/codemode/src/runtime/host.ts](../packages/codemode/src/runtime/host.ts) · [packages/agent/examples/mcp-codemode/main.ts](../packages/agent/examples/mcp-codemode/main.ts)。

### 6. 终端输入与界面渲染

掌握模型和工具执行后，补上 CLI 的交互基础：终端适配层管理输入输出与生命周期，TUI 运行基础组织焦点、输入、覆盖层和重绘。多行编辑器把光标、补全、历史和撤销转换为可渲染状态。理解这组职责后，就能在下一步区分界面交互与代理会话的运行逻辑。

阅读：[packages/tui/src/tui.ts](../packages/tui/src/tui.ts) · [packages/tui/src/terminal.ts](../packages/tui/src/terminal.ts) · [packages/tui/src/components/editor.ts](../packages/tui/src/components/editor.ts)。

### 7. CLI 与 SDK 的组装入口

现在沿 cli 到 main，再到 SDK 的依赖路径，把前面认识的模型、工具和终端能力拼成可运行的编码代理。main 解析启动配置并选择 interactive、print 或 RPC 模式，createAgentSession 则组装模型运行时、资源、工具与会话管理器，供 CLI 和外部应用使用。结合 model-runtime 与 interactive-mode，观察模型服务和终端交互分别如何接入会话。

阅读：[packages/coding-agent/src/cli.ts](../packages/coding-agent/src/cli.ts) · [packages/coding-agent/src/main.ts](../packages/coding-agent/src/main.ts) · [packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts) · [packages/coding-agent/src/core/model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) · [packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)。

### 8. 内置编码工具的执行边界

从 SDK 组装的工具集合继续下钻，阅读 read、write、edit 与 bash，理解模型请求怎样变成文件和进程操作。read 将文本或图像整理为模型可读结果，write 与 edit 分别处理内容写入和匹配文本修改，bash 连接执行环境、取消信号与输出收集。这一步把前面的通用工具循环落实为编码代理实际改变工作区的能力。

阅读：[packages/coding-agent/src/core/tools/read.ts](../packages/coding-agent/src/core/tools/read.ts) · [packages/coding-agent/src/core/tools/write.ts](../packages/coding-agent/src/core/tools/write.ts) · [packages/coding-agent/src/core/tools/edit.ts](../packages/coding-agent/src/core/tools/edit.ts) · [packages/coding-agent/src/core/tools/bash.ts](../packages/coding-agent/src/core/tools/bash.ts)。

### 9. 扩展契约与资源加载

了解内置能力后，再看用户如何给同一会话增加行为。resource-loader 统一发现扩展、技能、提示模板、主题和项目指令文件，扩展类型定义宿主允许注册的 API 与事件，而 runner 负责执行事件处理器和命令。将资源发现、扩展契约与事件执行分开阅读，可以理解 README 中的可扩展性如何进入运行流程。

阅读：[packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) · [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts) · [packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)。

### 10. 会话树与上下文压缩

在工具和扩展之上，AgentSession 协调各运行模式共享的会话生命周期，把事件持久化、消息排队、工具执行与会话导航连接起来。session-manager 保存带父子关系的历史条目，并将当前分支投影为模型上下文；压缩与分支摘要则为长历史和分支切换准备后续请求所需的上下文。这组文件说明编码代理如何把一次次模型调用组织成可恢复、可分支的工作过程。

阅读：[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) · [packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) · [packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) · [packages/coding-agent/src/core/compaction/branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts)。

### 11. Chord 对象与宿主契约

完成编码代理路线后，转向 p2 所依赖的跨进程应用基础 Chord，并重新使用第二步的数据契约概念。contracts 定义对象参数、成员、版本与主机归属，host 连接插件声明、对象实例、引用和生命周期，store 则提供存储、表与查询接口。配合被广泛引用的 context 入口，先认识对象、宿主、共享上下文和存储这些概念，为 Durable 的持久运行时建立词汇。

阅读：[packages/chord/src/contracts/index.ts](../packages/chord/src/contracts/index.ts) · [packages/chord/src/host/index.ts](../packages/chord/src/host/index.ts) · [packages/chord/src/context/index.ts](../packages/chord/src/context/index.ts) · [packages/chord/src/store/index.ts](../packages/chord/src/store/index.ts)。

### 12. Durable 持久会话与调度

在 Chord、模型接口和工具能力之上，Durable 提供可恢复的会话、文档与任务运行时。先看公共入口，再对照 session 的提交和事务边界、harness 的恢复与分叉操作，以及 scheduler 的工作调度，理解持久状态如何参与代理执行。SQLite 的 Node 适配入口展示了这些会话能力连接本地数据库的位置。

阅读：[packages/durable/src/index.ts](../packages/durable/src/index.ts) · [packages/durable/src/session/session.ts](../packages/durable/src/session/session.ts) · [packages/durable/src/harness/harness.ts](../packages/durable/src/harness/harness.ts) · [packages/durable/src/harness/scheduler.ts](../packages/durable/src/harness/scheduler.ts) · [packages/durable/src/storage/sqlite/node.ts](../packages/durable/src/storage/sqlite/node.ts)。

### 13. 远程执行环境与守护进程

理解 Durable 的运行方式后，阅读 pi-env 如何把文件与命令操作接到远程机器。SSH 入口负责探测、信任和 daemon 部署启动，connection 关联协议请求与响应，remote-env 将这些能力适配为 Durable 的 ExecutionEnv。再对照 Rust daemon 的请求分派入口，就能理解代理逻辑与凭据留在本地主机时，工具操作如何在远端执行。

阅读：[packages/env/src/ssh.ts](../packages/env/src/ssh.ts) · [packages/env/src/connection.ts](../packages/env/src/connection.ts) · [packages/env/src/remote-env.ts](../packages/env/src/remote-env.ts) · [packages/env/daemon/src/main.rs](../packages/env/daemon/src/main.rs)。

### 14. p2 服务与会话宿主

带着 Chord、Durable 和执行环境的概念，最后进入 p2 的应用组装层。server 接受 HTTP 与 WebSocket 请求并把会话地址路由到对应 worker，worker 通过 IPC 连接服务器，并在会话的 Chord 存储上加载 Durable Harness 与插件 facet。结合公共 API 入口和 sessions 插件契约，观察共享库如何被组织为服务、会话进程与应用接口。

阅读：[apps/p2/src/hosts/server.ts](../apps/p2/src/hosts/server.ts) · [apps/p2/src/hosts/worker.ts](../apps/p2/src/hosts/worker.ts) · [apps/p2/src/api/v1/index.ts](../apps/p2/src/api/v1/index.ts) · [apps/p2/plugins/sessions/contract.ts](../apps/p2/plugins/sessions/contract.ts)。

### 15. 工程配置与行为评测

完成两条运行路线后，回到根 package 配置和 CI 流程，理解 monorepo 的工作区、构建顺序、检查与交付如何支撑这些包协同演进。再看评测 harness 与 Dockerfile，认识编码代理评测入口及其依赖安装、资源准备和容器运行环境。把这些非代码配置与前面的运行模块对照，可以区分应用执行路径和工程验证路径，并找到继续学习或验证改动的位置。

阅读：[package.json](../package.json) · [.github/workflows/ci.yml](../.github/workflows/ci.yml) · [packages/evals/src/harness.ts](../packages/evals/src/harness.ts) · [packages/evals/docker/Dockerfile](../packages/evals/docker/Dockerfile)。

## 关键文件地图

本地图选取导览文件与理解主要机制需要的补充文件，并按架构层组织。这里只列文件级节点；“结构摘要”表示仍需结合源码确认具体行为，复杂度沿用原图谱标签。完整清单见[知识图谱](../.ua/knowledge-graph.json)。

### 数据契约与遥测基础层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/schema/src/index.ts](../packages/schema/src/index.ts) | 汇总 JSON Schema 构造、类型推导与校验接口，作为模型工具参数和跨进程契约的共同基础。 | 简单 | 语义说明 |
| [packages/telemetry/src/index.ts](../packages/telemetry/src/index.ts) | 公开与供应商无关的遥测契约、类型化 schema 和参考适配器，支撑 Pi 各包的追踪与指标记录。 | 复杂 | 语义说明 |

### 模型接入与智能体循环层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/ai/src/index.ts](../packages/ai/src/index.ts) | 导出 pi-ai 的无副作用核心类型与工具函数；provider 工厂、实现和兼容入口通过独立子路径提供。 | 简单 | 语义说明 |
| [packages/ai/src/types.ts](../packages/ai/src/types.ts) | 定义模型、消息、工具、provider 选项和使用量等 pi-ai 核心类型。 | 复杂 | 语义说明 |
| [packages/ai/src/models.ts](../packages/ai/src/models.ts) | 实现统一 Models 运行时：管理各类模型和提供商、目录刷新与凭据、鉴权及 OAuth 登录，并分派聊天、图像、分类和延迟请求，计算费用与思考级别。 | 复杂 | 语义说明 |
| [packages/agent/src/types.ts](../packages/agent/src/types.ts) | 定义 Agent 消息、工具、事件、状态和循环配置契约，约束模型流与工具执行策略。 | 复杂 | 语义说明 |
| [packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts) | 实现 Agent 的流式模型与工具执行循环，处理工具声明变化、顺序或并行执行、中断和事件发布。 | 复杂 | 语义说明 |
| [packages/agent/src/agent.ts](../packages/agent/src/agent.ts) | 以有状态 Agent 封装底层循环，维护会话消息、工具状态、引导与后续消息队列和运行生命周期。 | 复杂 | 语义说明 |
| [packages/agent/examples/mcp-codemode/main.ts](../packages/agent/examples/mcp-codemode/main.ts) | 演示连接 stdio MCP 服务，将服务工具和可调用它们的 JavaScript codemode 工具交给 Agent。 | 简单 | 语义说明 |
| [packages/ai/src/providers/faux.ts](../packages/ai/src/providers/faux.ts) | 提供可编排的模拟模型提供商，以文本、思考和工具调用块构造回复、估计用量并模拟流式增量及延迟消息，供离线测试使用。 | 复杂 | 语义说明 |
| [packages/ai/src/utils/transcript.ts](../packages/ai/src/utils/transcript.ts) | 规范化系统消息和工具声明，计算工具集变化，并把完整 transcript 解析为当前模型上下文。 | 复杂 | 语义说明 |

### 工具协议与受控执行层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/mcp/src/client.ts](../packages/mcp/src/client.ts) | 实现与传输无关的 MCP 客户端，关联 JSON-RPC 请求和响应，处理初始化、超时、取消、通知及工具调用。 | 复杂 | 语义说明 |
| [packages/codemode/src/declarations.ts](../packages/codemode/src/declarations.ts) | 把工具 JSON Schema 转换为脚本使用的 TypeScript 声明、签名和示例，处理局部引用与声明长度限制。 | 复杂 | 语义说明 |
| [packages/codemode/src/runtime/host.ts](../packages/codemode/src/runtime/host.ts) | 实现宿主侧 CodemodeSandbox，向隔离的脚本运行时提供已注入工具并管理执行生命周期。 | 中等 | 语义说明 |

### 终端界面层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/tui/src/tui.ts](../packages/tui/src/tui.ts) | 实现终端 UI 的容器与公共运行基础，管理焦点、输入、覆盖层合成和重绘调度，为主屏及备用屏渲染提供共享能力。 | 复杂 | 语义说明 |
| [packages/tui/src/terminal.ts](../packages/tui/src/terminal.ts) | 抽象终端输入输出和生命周期，将系统终端行为适配到 TUI 运行时。 | 复杂 | 语义说明 |
| [packages/tui/src/components/editor.ts](../packages/tui/src/components/editor.ts) | 实现多行终端编辑器，将文本编辑、光标移动、补全、历史及撤销操作转换为可渲染组件状态。 | 复杂 | 语义说明 |

### Chord 跨进程对象运行时层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/chord/src/contracts/index.ts](../packages/chord/src/contracts/index.ts) | 公开对象契约与 schema 构造器，定义本地和远端对象的参数、成员、版本与主机归属。 | 简单 | 语义说明 |
| [packages/chord/src/host/index.ts](../packages/chord/src/host/index.ts) | 公开 Chord host 与 facet 接口，连接插件声明、对象实例、引用及主机生命周期。 | 简单 | 语义说明 |
| [packages/chord/src/context/index.ts](../packages/chord/src/context/index.ts) | 结构摘要：实现跨进程对象与插件运行时中的「索引」职责，对外提供 「BACKGROUND_CONTEXT」、「TODO_CONTEXT」、「createContextKey」、「withContextValue」。 | 中等 | 结构摘要 |
| [packages/chord/src/store/index.ts](../packages/chord/src/store/index.ts) | 汇总 Chord 存储、表与查询接口，供对象文档和持久任务使用。 | 中等 | 语义说明 |
| [packages/chord/src/delta/index.ts](../packages/chord/src/delta/index.ts) | 公开事务式 JSON 变更跟踪、操作批次编码与不可变应用接口，用于状态提交和远端副本同步。 | 复杂 | 语义说明 |
| [packages/chord/src/watch.ts](../packages/chord/src/watch.ts) | 实现普通 JSON 值与原生值 watch 的共同生命周期，处理启动、取消、结束、失败及监听者繁忙时的更新交付。 | 复杂 | 语义说明 |
| [packages/chord/src/sqlite/node.ts](../packages/chord/src/sqlite/node.ts) | 基于 node:sqlite 提供串行执行、预编译语句缓存和事务数据库适配器。 | 复杂 | 语义说明 |

### Durable 持久会话与任务层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/durable/src/index.ts](../packages/durable/src/index.ts) | 汇总 Durable 的会话、文档、任务、持久化和 harness API，作为构建可恢复代理应用的主要入口。 | 复杂 | 语义说明 |
| [packages/durable/src/session/session.ts](../packages/durable/src/session/session.ts) | 实现持久会话与存储访问，管理提交、文档快照和监听、关闭与失败状态，并用事务边界协调写入。 | 复杂 | 语义说明 |
| [packages/durable/src/harness/harness.ts](../packages/durable/src/harness/harness.ts) | 将持久会话与代理调度连接起来，提供对话提交、恢复、取消、分叉、任务等待及状态观察等运行时操作。 | 复杂 | 语义说明 |
| [packages/durable/src/harness/scheduler.ts](../packages/durable/src/harness/scheduler.ts) | 调度 Durable 中的对话与任务工作，协调可运行状态、等待条件及生命周期变化。 | 复杂 | 语义说明 |
| [packages/durable/src/storage/sqlite/node.ts](../packages/durable/src/storage/sqlite/node.ts) | 将 Durable SQLite 存储接入 Node 数据库适配器，为持久会话提供本地数据库入口。 | 简单 | 语义说明 |

### 远程执行环境层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/env/src/ssh.ts](../packages/env/src/ssh.ts) | 负责 SSH 主机探测、密钥信任与 daemon 部署和启动，为远程执行提供连接入口。 | 复杂 | 语义说明 |
| [packages/env/src/connection.ts](../packages/env/src/connection.ts) | 管理与 pi-env daemon 的连接及协议请求，将异步操作、响应和连接关闭关联起来。 | 复杂 | 语义说明 |
| [packages/env/src/remote-env.ts](../packages/env/src/remote-env.ts) | 把 daemon 的文件与命令能力适配为 Durable ExecutionEnv，使代理逻辑和凭据可以留在本地主机。 | 复杂 | 语义说明 |
| [packages/env/daemon/src/main.rs](../packages/env/daemon/src/main.rs) | 启动远端 Rust daemon，接收带帧的协议请求并分派文件与进程操作，协调执行顺序和响应发送。 | 复杂 | 语义说明 |

### 编码代理命令行层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [packages/coding-agent/src/cli.ts](../packages/coding-agent/src/cli.ts) | 启动编码代理 CLI，并把命令行入口连接到主运行流程。 | 简单 | 语义说明 |
| [packages/coding-agent/src/main.ts](../packages/coding-agent/src/main.ts) | 解析命令行参数、工作目录、鉴权和模型选择，将配置组装为会话服务并启动 interactive、print 或 RPC 运行模式。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts) | 提供 createAgentSession 等 SDK 接口，组装模型运行时、资源、工具和会话管理器，供 CLI 和外部应用嵌入编码代理。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) | 将 pi-ai 模型目录、提供商、认证与请求执行接入编码代理，为模型选择和流式调用提供运行时服务。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) | 以终端交互界面驱动 AgentSession，协调输入编辑、消息与工具渲染、命令、对话框以及会话操作。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/tools/read.ts](../packages/coding-agent/src/core/tools/read.ts) | 实现文件读取工具，将文本或图像整理为模型可读结果，并处理读取范围及输出限制。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/tools/write.ts](../packages/coding-agent/src/core/tools/write.ts) | 实现文件写入工具，将模型提供的内容写到目标路径，并返回写入结果。 | 中等 | 语义说明 |
| [packages/coding-agent/src/core/tools/edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | 实现基于匹配文本的文件编辑工具，准备修改与差异信息后更新文件并返回结果。 | 中等 | 语义说明 |
| [packages/coding-agent/src/core/tools/bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | 实现 shell 命令工具，将执行环境、取消信号和输出收集连接到代理工具调用。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) | 统一发现和加载扩展、技能、提示模板、主题与 AGENTS 文件，合并来源并报告名称冲突，支持资源重新加载。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts) | 定义扩展 API、事件、上下文和工具契约，约束扩展能够注册的行为及与宿主的交互方式。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) | 执行扩展事件处理器与命令，将模型、工具、会话和界面事件交给已注册扩展。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) | 为 interactive、print 与 RPC 模式共享代理会话生命周期，连接事件持久化、模型选择、上下文压缩、工具执行、消息排队和会话树导航。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | 读写并迁移会话条目，维护父子关系与当前分支，将持久历史投影为模型上下文，并支持发现、恢复和派生会话。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | 选择需要压缩的会话上下文并生成摘要，将长历史转换为后续模型请求可用的较短上下文。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/compaction/branch-summarization.ts](../packages/coding-agent/src/core/compaction/branch-summarization.ts) | 为会话树分支切换准备历史摘要，保留转移到新分支所需的上下文。 | 复杂 | 语义说明 |
| [packages/coding-agent/src/core/settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | 管理用户与项目层的代理设置，提供读取、更新、覆盖合并及诊断接口。 | 复杂 | 语义说明 |

### 产品应用与宿主层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [apps/p2/src/hosts/server.ts](../apps/p2/src/hosts/server.ts) | 启动 p2 服务器 host，接受 HTTP 与 WebSocket 请求，加载 server facet，并将 session 地址路由到对应会话 worker。 | 复杂 | 语义说明 |
| [apps/p2/src/hosts/worker.ts](../apps/p2/src/hosts/worker.ts) | 为运行中的会话启动独立 worker host，通过 IPC 连接服务器，在会话 Chord 存储上加载 Durable Harness 及 worker facet。 | 中等 | 语义说明 |
| [apps/p2/src/api/v1/index.ts](../apps/p2/src/api/v1/index.ts) | 结构摘要：汇总p2 应用在 「apps/p2/src/api/v1」 下的模块接口，导出 「AcceptsTable」、「Address」、「columnBool」、「broken」、「ChordError」 等符号。 | 中等 | 结构摘要 |
| [apps/p2/plugins/sessions/contract.ts](../apps/p2/plugins/sessions/contract.ts) | 结构摘要：为p2 的 会话 插件定义「契约」的共享类型、常量或导出接口，包括 「isThinkingLevel」、「ModelRefSchema」、「THINKING_LEVELS」、「ThinkingLevel」、「ThinkingLevelSchema」。 | 复杂 | 结构摘要 |

### 工程自动化与评测层

| 文件 | 职责 | 复杂度 | 摘要来源 |
| --- | --- | --- | --- |
| [README.md](../README.md) | 介绍 Pi 作为可扩展智能体框架的用途、CLI 安装与运行方式，以及 monorepo 中各库的职责、开发与交付流程。 | 中等 | 语义说明 |
| [package.json](../package.json) | 声明 Pi monorepo 的 npm workspaces、各包构建顺序、检查与发布命令，以及共享开发依赖和 Node 版本要求。 | 中等 | 语义说明 |
| [.github/workflows/ci.yml](../.github/workflows/ci.yml) | 结构摘要：定义GitHub 协作与 CI/CD的「ci」自动化流程，组织构建、检查或交付步骤。 | 中等 | 结构摘要 |
| [packages/evals/src/harness.ts](../packages/evals/src/harness.ts) | 结构摘要：实现代理行为评测中的「harness」职责，对外提供 「resolveModelSelection」、「applyIsolatedEnvironment」、「verifySystemPrompt」、「createPiCodingAgentHarness」。 | 复杂 | 结构摘要 |
| [packages/evals/docker/Dockerfile](../packages/evals/docker/Dockerfile) | 结构摘要：定义代理行为评测的容器构建与启动环境，包括依赖安装、资源准备和运行入口。 | 简单 | 结构摘要 |

## 复杂度热点

原图谱将 1,205 个整文件标记为 complex。下表从这些文件中选出与学习路线相关的区域；它是阅读顺序建议，不是缺陷或性能问题清单。对于导出入口，较高复杂度也可能来自接口数量，应该继续定位到实际实现。

| 区域 | 文件 | 建议阅读方式 |
| --- | --- | --- |
| 共享模型契约 | [packages/ai/src/types.ts](../packages/ai/src/types.ts)<br>[packages/ai/src/models.ts](../packages/ai/src/models.ts) | 先区分消息与工具类型、模型目录和请求分派；接口被多个上层包引用，阅读时要同时看调用方。 |
| 消息循环与中断 | [packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)<br>[packages/agent/src/agent.ts](../packages/agent/src/agent.ts) | 沿模型输出、工具执行、排队消息和取消过程梳理事件顺序，再看顺序与并行工具执行。 |
| 会话历史与上下文 | [packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)<br>[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)<br>[packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | 先画出历史条目的父子关系和当前分支，再追踪上下文压缩与恢复入口。 |
| 扩展协议与调用时机 | [packages/coding-agent/src/core/extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts)<br>[packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) | 先阅读事件与上下文契约，再看处理器如何接入会话，避免把接口声明和执行时机混在一起。 |
| 交互状态与终端渲染 | [packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)<br>[packages/tui/src/tui.ts](../packages/tui/src/tui.ts)<br>[packages/tui/src/components/editor.ts](../packages/tui/src/components/editor.ts) | 把输入编辑、焦点与重绘、AgentSession 事件分开追踪，最后再组合成一次用户交互。 |
| Chord 变更与监听 | [packages/chord/src/delta/index.ts](../packages/chord/src/delta/index.ts)<br>[packages/chord/src/watch.ts](../packages/chord/src/watch.ts)<br>[packages/chord/src/sqlite/node.ts](../packages/chord/src/sqlite/node.ts) | 分别理解变更批次、监听启动与取消、数据库事务边界，再查看它们的组合点。 |
| 持久状态与调度 | [packages/durable/src/session/session.ts](../packages/durable/src/session/session.ts)<br>[packages/durable/src/harness/harness.ts](../packages/durable/src/harness/harness.ts)<br>[packages/durable/src/harness/scheduler.ts](../packages/durable/src/harness/scheduler.ts) | 按提交、调度、等待、恢复的顺序阅读，明确每一层保存什么状态以及何时推进工作。 |
| 协议连接与远端操作 | [packages/mcp/src/client.ts](../packages/mcp/src/client.ts)<br>[packages/env/src/connection.ts](../packages/env/src/connection.ts)<br>[packages/env/daemon/src/main.rs](../packages/env/daemon/src/main.rs) | 重点跟踪请求和响应的对应关系、取消及连接关闭，以及客户端和 daemon 的职责边界。 |
| P2 服务与插件契约 | [apps/p2/src/hosts/server.ts](../apps/p2/src/hosts/server.ts)<br>[apps/p2/plugins/sessions/contract.ts](../apps/p2/plugins/sessions/contract.ts) | 先区分 server 路由和会话 worker，再阅读插件契约。sessions/contract.ts 仅有结构摘要，具体语义需要回到源码确认。 |

修改这些区域前，先理解相关契约、状态变化与对应测试。做可控的模型行为实验时，可从已有的模拟提供商 [packages/ai/src/providers/faux.ts](../packages/ai/src/providers/faux.ts) 入手；具体测试命令和运行约束仍以 [AGENTS.md](../AGENTS.md) 为准。

---

本指南由 Understand-Anything v2.9.8 的 `understand-onboard` 流程生成，输入为[知识图谱](../.ua/knowledge-graph.json)中的项目元数据、分层、导览、文件级摘要与复杂度。图谱生成流程、覆盖范围及已知限制见[图谱说明](../.ua/README.md)与[分析报告](../.ua/analysis-report.json)。
