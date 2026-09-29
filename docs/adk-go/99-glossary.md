# 术语表（Glossary）

> ADK Go（`google.golang.org/adk/v2`）里会用到的术语：英文原词、中文解释、以及在本地代码中首次定义的文件。

## 核心概念

- **Agent** —— 可运行的基本单元。接口要求 `Name()`、`Description()`、`Run(InvocationContext) iter.Seq2[*session.Event, error]`、`SubAgents()`、`FindAgent`、`FindSubAgent`。构造用 `agent.New`（自定义 Run）或 `llmagent.New`（LLM 驱动）。 —— `agent/agent.go`
- **Invocation** —— 一次调用：从收到一条用户消息开始，到产出最终响应结束；由 `runner.Run()` 处理，其间可包含一次或多次 agent call。 —— `agent/context.go`（`InvocationContext` 的文档注释）
- **InvocationContext** —— 传给 `Agent.Run` 的顶层上下文，暴露 `Session()`、`Artifacts()`、`Memory()`、`InvocationID()`、`Branch()`、`IsolationScope()`、`RunConfig()`、`EndInvocation()`。实现同时在 `agent` 与 `internal/context`。 —— `agent/context.go`
- **Agent Call** —— 一次 `agent.Run()` 的执行，结束时 agent call 结束；可包含一个或多个 step。 —— `agent/context.go`
- **Step** —— agent call 中的一步：只调一次 LLM 并产出其响应，若有 tool 调用则执行并产出结果；函数响应的总结算作另一个 step。 —— `agent/context.go`
- **Session** —— 一次用户与 agent 交互的持久记录，保存事件历史与当前状态。接口 `session.Session` 有 `ID()`、`AppName()`、`UserID()`、`State()`、`Events()`。 —— `session/session.go`
- **Event** —— 会话历史与通信的原子单位，代表用户消息、模型响应、tool 调用/结果等。关键字段 `Author`、`LLMResponse`、`Actions`、`Branch`、`IsolationScope`、`LongRunningToolIDs`、`Routes`。 —— `session/session.go`
- **EventActions** —— 事件携带的副作用指令：`StateDelta`、`ArtifactDelta`、`RequestedToolConfirmations`、`SkipSummarization`、`TransferToAgent`、`Escalate`、`Compaction`。 —— `session/session.go`
- **State** —— 会话状态，键前缀决定作用域。`session.State` 提供 `Get`/`Set`/`All`，`session.ReadonlyState` 只读。 —— `session/session.go`
- **State Scopes（四种）** —— `app:`（全 app 共享，跨用户跨会话）、`user:`（同一 `user_id` 跨会话）、无前缀（session 级）、`temp:`（仅当前 invocation，`AppendEvent` 时被剥离）。常量 `session.KeyPrefixApp`、`KeyPrefixUser`、`KeyPrefixTemp`。 —— `session/session.go`
- **Artifact** —— 会话级文件/二进制资源。服务接口 `artifact.Service`，上下文侧访问接口 `agent.Artifacts`（`Save`/`List`/`Load`/`LoadVersion`）。 —— `artifact/service.go`、`agent/agent.go`
- **Memory** —— 跨会话的长期记忆，按 `user_id` 作用域。服务接口 `memory.Service`，上下文侧 `agent.Memory`（`AddSessionToMemory`/`SearchMemory`）。 —— `memory/service.go`、`agent/agent.go`
- **Branch** —— 表示在 agent 树中路径的字符串，形如 `agent_1.agent_2`；用于隔离子 agent 的对话历史（并行子 agent 场景）。 —— `agent/context.go`（`Branch()`）
- **IsolationScope** —— 比 branch 更严格的可见性隔离：设值时只看到 `IsolationScope` 完全相等的会话事件；空表示不隔离。 —— `agent/context.go`

## 工具与扩展

- **Tool** —— LLM 可调用的代码单元。公开接口 `tool.Tool`（`Name`/`Description`/`IsLongRunning`）；可执行工具额外实现 `Declaration()` + `Run`，流式工具实现 `Declaration()` + `RunStream`（内部接口）。 —— `tool/tool.go`
- **Toolset** —— 一组 tool 的集合，`Tools(ctx agent.ReadonlyContext) ([]Tool, error)` 可按调用动态决定返回哪些。 —— `tool/tool.go`
- **FunctionTool** —— 内部的可执行工具接口（`Declaration()` + `Run(ctx, args)`），`functiontool.New` 生成的就是它；流式对应 `StreamingFunctionTool`。 —— `internal/toolinternal/tool.go`
- **Function Call** —— 模型请求执行某个 tool 的 `genai.FunctionCall`；对应结果叫 function response。工具/历史里的 call 与 response 用 `ID` 关联。解析工具见 `utils.FunctionCalls` / `utils.FunctionResponses`。 —— `internal/utils/utils.go`
- **Callback** —— 在 agent/model/tool 生命周期的特定点插入的函数，如 `agent.BeforeAgentCallback`、`llmagent.BeforeModelCallback`、`llmagent.AfterToolCallback`。`Before*` 返回非 nil 结果或错误可短路底层调用。 —— `agent/agent.go`、`agent/llmagent/llmagent.go`
- **Plugin** —— 把 run/agent/model/tool 各阶段的回调打成一包，注册到 `runner.Config.PluginConfig`；跨切面行为的首选扩展点。 —— `plugin/plugin.go`
- **Transfer** —— 把控制权移交给另一个 agent。由内部 `TransferToAgentTool`（`transfer_to_agent`）发起，通过 `EventActions.TransferToAgent` 表达；能否向上/向兄弟转移受 `DisallowTransferToParent`、`DisallowTransferToPeers` 约束。 —— `internal/llminternal/agent_transfer.go`
- **Sub-agent** —— agent 的子节点，由 `agent.Config.SubAgents` 声明；ADK 据此自动建立父子关系以支持转移。 —— `agent/agent.go`

## 执行与流程

- **Runner** —— 运行引擎。`runner.New(Config)` 构造，`Run` 驱动一次调用并持久化事件，`RunLive` 启动双向流式会话，`RunLive` 之外还用 `parentmap` 校验 agent 树。 —— `runner/runner.go`
- **Flow** —— LLM agent 的执行流程实现（内部）：持有 `Model`、`Tools`、`RequestProcessors`、`ResponseProcessors` 与各阶段回调，循环「组装请求→调用模型→执行工具」。 —— `internal/llminternal/base_flow.go`
- **Request/Response Processor** —— 请求/响应处理器：前者在 `model.LLMRequest` 到达模型前修改它，后者在 `model.LLMResponse` 返回后处理。默认流水线 `DefaultRequestProcessors` / `DefaultResponseProcessors`。 —— `internal/llminternal/base_flow.go`
- **StreamingMode** —— 流式模式：`agent.StreamingModeNone`、`agent.StreamingModeSSE`（公开）；内部另有 `runconfig.StreamingModeBidi`（live）。 —— `agent/run_config.go`、`internal/agent/runconfig/run_config.go`
- **Live / Bidi** —— 双向流式（bidirectional streaming）：通过长连接持续收发音频/文本，入口 `Runner.RunLive` 与 `agent.LiveSession`（`Send`/`Close`），输入用 `agent.LiveRequest`，配置用 `agent.LiveRunConfig`。 —— `agent/live.go`、`runner/runner.go`
- **Launcher** —— 应用启动器接口：`Execute(ctx, *Config, args)` 解析命令行并运行。 —— `cmd/launcher/launcher.go`
- **SubLauncher** —— 可被父 launcher（如 universal）组合的子启动器接口：`Keyword`/`Parse`/`Run`。各 `cmd/launcher/*` 子包提供 `NewLauncher()`。 —— `cmd/launcher/launcher.go`
- **Loader** —— 按名加载 agent 的接口：`ListAgents`、`LoadAgent(name)`、`RootAgent()`；实现由 `NewSingleLoader` / `NewMultiLoader` 提供。 —— `agent/loader.go`

## 工作流

- **Workflow** —— 基于图的编排引擎。`workflow.New(name, edges, opts...)` 构造，支持并发上限、state schema、HITL、重试。 —— `workflow/workflow.go`
- **Node** —— 工作流中的一个执行单元。接口含 `Run(ctx, input)`、`InputSchema()`/`OutputSchema()`、`ValidateInput`/`ValidateOutput`；实现有 `FunctionNode`、`AgentNode`、`ToolNode`、`JoinNode`、`DynamicNode`，并可用 `NewBaseNode` 自定义。 —— `workflow/workflow.go`
- **Edge / Route** —— Edge 是两个节点间的有向连接（`From`/`To`/`Route`）；Route 判断某个事件是否走这条边（`StringRoute`、`IntRoute`、`BoolRoute`、`MultiRoute`，以及兜底的 `Default`）。入口哨兵是 `Start`。 —— `workflow/workflow.go`
- **NodeStatus / RunState** —— 工作流节点的运行状态（`NodePending`、`NodeRunning`、`NodeCompleted`、`NodeWaiting`、`NodeFailed`）与可持久化的运行态 `RunState`（`NewRunState`）。 —— `workflow/state.go`
- **HITL** —— Human-in-the-Loop：执行暂停等待人工输入/批准。tool 侧用 `ctx.RequestConfirmation` 与 `tool.ErrConfirmationRequired` / `tool.ErrConfirmationRejected`；工作流侧用 `session.RequestInput` 与 `workflow.NewRequestInputEvent`。 —— `tool/tool.go`、`session/session.go`
- **Resume** —— 从暂停处继续：工作流用 `Workflow.Resume(ctx, state, responses)`，以 `RequestInput.InterruptID` 匹配用户响应，错误哨兵 `ErrInvalidResumeResponse`、`ErrNothingToResume`；runner 侧也会从会话历史恢复未完成的调用。 —— `workflow/resume.go`
- **Compaction** —— 上下文压缩：把较早事件替换成摘要以控制 prompt 大小。配置 `session/compaction.Config`（`CompactionInterval`、`TokenThreshold`、`EventRetentionSize` 等），默认摘要器 `LLMSummarizer`。 —— `session/compaction/compaction.go`

## 分布式与协议

- **A2A（Agent-to-Agent）** —— 跨进程 agent 通信协议。客户端侧把远端 agent 当本地 agent 用（`agent/remoteagent/v2` 的 `NewA2A`），服务端把本地 agent 暴露成 A2A 服务（`server/adka2a/v2`）。 —— `agent/remoteagent/v2/`、`server/adka2a/v2/`
- **Agent Card** —— A2A 的 agent 元数据卡（`a2a.AgentCard`），声明名称、能力、skill 与连接接口。ADK 用 `server/adka2a/v2` 的 `BuildAgentSkills` 从 agent 生成技能列表。 —— `server/adka2a/v2/agent_card.go`
- **Agent Registry** —— Google Cloud Agent Registry：发现并装配注册在册的 A2A agent、MCP server 与 model endpoint。客户端 `agentregistry.New`，装配方法 `RemoteAgent`、`MCPToolset`。 —— `agentregistry/registry.go`
- **MCP（Model Context Protocol）** —— 通过 MCP 服务器暴露外部工具。`tool/mcptoolset.New(Config)` 连接并把远端工具转成 `tool.Tool`；`Config.Auth` 可注入 `auth.CredentialProvider`。 —— `tool/mcptoolset/set.go`
- **Skill** —— 以目录（含 frontmatter 的 markdown）描述的能力包，由 `tool/skilltoolset` 加载并暴露成工具；解析与来源抽象在 `skill.Source`、`skill.Parse`。 —— `tool/skilltoolset/skill/source.go`、`tool/skilltoolset/skill/frontmatter.go`

## 可观测与测试

- **Telemetry / Span** —— OpenTelemetry 集成。公开 `telemetry.New(ctx, opts...)` 返回 `telemetry.Providers`（Tracer/Logger provider）；内部用 `StartTrace`、`StartGenerateContentSpan`、`StartExecuteToolSpan`、`StartNodeSpan` 等创建 span。 —— `telemetry/telemetry.go`、`internal/telemetry/telemetry.go`
- **Platform（time/UUID provider）** —— 可替换的接缝：`platform.WithTimeProvider`/`Now`、`platform.WithUUIDProvider`/`NewUUID`、`platform.WithTaskRunner`/`RunTasks`，用于确定性测试与事件回放。 —— `platform/time.go`、`platform/uuid.go`、`platform/exec.go`
- **httprr** —— 仓库内 vendored 的 HTTP 录制/回放库（`Open(file, rt)` 返回实现 `http.RoundTripper` 的 `RecordReplay`，`Recording(file)` 判断录/放）。测试通过 `internal/testutil` 的 `NewGeminiTransport` 接到 genai 客户端上；**测试不得打真实模型**。 —— `internal/httprr/rr.go`

## 状态作用域速查

| 前缀常量 | 字面值 | 可见范围 | 生命周期 |
| --- | --- | --- | --- |
| `session.KeyPrefixApp` | `app:` | 整个 application | 永久，跨用户跨会话 |
| `session.KeyPrefixUser` | `user:` | 同一 `user_id`（同一 `app_name` 内） | 永久，跨会话 |
| （无前缀） | — | 单个 session | 随 session 删除而消失 |
| `session.KeyPrefixTemp` | `temp:` | 当前 invocation | `AppendEvent` 时被 `trimTempDeltaState` 剥离，不落库 |

## 缩写

| 缩写 | 全称 | 在代码中的落点 |
| --- | --- | --- |
| **ADK** | Agent Development Kit | 模块 `google.golang.org/adk/v2` |
| **A2A** | Agent-to-Agent | `agent/remoteagent/v2`、`server/adka2a/v2` |
| **MCP** | Model Context Protocol | `tool/mcptoolset` |
| **HITL** | Human-in-the-Loop | `tool.ErrConfirmationRequired`、`session.RequestInput` |
| **SSE** | Server-Sent Events | `agent.StreamingModeSSE`、`server/adkrest` |
| **LRO** | Long Running Operation | `tool.Tool.IsLongRunning`、`session.Event.LongRunningToolIDs` |
| **OTel** | OpenTelemetry | `telemetry`、`internal/telemetry` |

## 术语 → 包 快速索引

| 术语 | 先看这个包 |
| --- | --- |
| Agent / Invocation / Callback / Loader | `agent`、`agent/llmagent` |
| Runner / RunLive | `runner` |
| Flow / Request·Response Processor | `internal/llminternal`（非公开） |
| Session / Event / EventActions / State | `session` |
| Artifact | `artifact` |
| Memory | `memory` |
| Tool / Toolset / FunctionTool / HITL | `tool`、`tool/functiontool`、`tool/toolconfirmation` |
| MCP | `tool/mcptoolset` |
| Skill | `tool/skilltoolset`、`tool/skilltoolset/skill` |
| Plugin | `plugin` 及子包 |
| Workflow / Node / Edge / Route / Resume | `workflow` |
| Transfer / Sub-agent | `internal/llminternal/agent_transfer.go`、`agent/agent.go` |
| Launcher / SubLauncher | `cmd/launcher` 及子包 |
| A2A / Agent Card | `server/adka2a/v2`、`agent/remoteagent/v2` |
| Agent Registry | `agentregistry` |
| Compaction | `session/compaction` |
| Telemetry / Span | `telemetry`、`internal/telemetry` |
| Platform（time/UUID） | `platform` |
| httprr | `internal/httprr`（非公开） |

## 相关页面

- [03-core-concepts.md](./03-core-concepts.md) —— Agent/InvocationContext/Session/Event/State 的用法。
- [04-agent-execution.md](./04-agent-execution.md) —— Runner、Flow、处理器与流式。
- [06-tool-system.md](./06-tool-system.md) —— Tool/Toolset/FunctionTool/HITL/MCP。
- [11-graph-workflow-engine.md](./11-graph-workflow-engine.md) —— Workflow/Node/Edge/Route/Resume。
- [12-advanced-topics.md](./12-advanced-topics.md) —— live、state 作用域、测试录制、插件与认证的细节。
- [14-api-reference.md](./14-api-reference.md) —— 每个公开包的职责与代表符号。
