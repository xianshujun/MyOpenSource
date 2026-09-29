# 包地图与 API 速查（API Reference）

> `/workspace/github/adk-go`（模块 `google.golang.org/adk/v2`）下运行时、工具、服务、扩展、服务端与命令行公开包的职责与代表符号，附「常见任务 → 该看哪个包」速查表。`examples/` 下另有 45 个示例包，见 [13-examples.md](./13-examples.md)；`internal/` 不是公开 API。

## 涉及代码

- 模块根 `go.mod`（模块路径 `google.golang.org/adk/v2`）。
- 各包目录本身；符号均以 `rg -n '^func [A-Z]|^type [A-Z]' <pkg>` 在本地核对。
- `internal/` 为私有包，不属公开 API。

## 核心运行时

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `agent` | agent 接口、公开上下文、自定义 agent 构造、Loader | `Agent`、`New(Config)`、`InvocationContext`、`Context`、`Loader`、`NewSingleLoader`、`RunConfig`、`LiveSession` |
| `agent/llmagent` | LLM 驱动的 agent 与 LLM 回调 | `New(Config)`、`BeforeModelCallback`、`AfterToolCallback`、`Mode`、`RunLLMAgentAsNode` |
| `agent/workflowagents/sequentialagent` | 顺序执行子 agent | `New(Config)` |
| `agent/workflowagents/parallelagent` | 并行执行子 agent | `New(Config)` |
| `agent/workflowagents/loopagent` | 按轮次循环执行子 agent | `New(Config)` |
| `agent/workflowagent` | 基于 `workflow` 图的自定义 agent 适配 | `New(Config)` |
| `agent/remoteagent` | 通过 A2A 调用远端 agent（v1 兼容面） | `NewA2A(A2AConfig)`、`A2AConfig`、`A2AEventConverter` |
| `agent/remoteagent/v2` | A2A 远端 agent 的 v2 实现 | `NewA2A(A2AConfig)`、`A2AClientProvider` |
| `runner` | 运行引擎：驱动 run 循环、持久化事件、live 入口 | `New(Config)`、`NewInMemory`、`Runner.Run`、`Runner.RunLive`、`WithStateDelta` |
| `workflow` | 图式工作流引擎：节点、边、路由、HITL、重试 | `Workflow`、`New`、`Node`、`Edge`、`Route`、`Start`、`FunctionNode`、`AgentNode`、`ToolNode`、`Resume` |

## 智能与工具

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `model` | LLM 抽象与按名注册的工厂 | `LLM`、`LLMRequest`、`LLMResponse`、`Register`、`NewLLM` |
| `model/gemini` | Gemini 后端 | `NewModel(ctx, name, *genai.ClientConfig)` |
| `model/openaimodel` | OpenAI 兼容后端（走 Responses API） | `NewModel(ctx, name, *ClientConfig)`、`ClientConfig`、`FinishMessageKey` |
| `model/apigee` | 经 Apigee 代理访问 Vertex 模型 | `NewModel(ctx, name, opts...)`、`WithProxyURL`、`WithCustomHeaders`、`WithHTTPClient` |
| `tool` | Tool/Toolset 接口、过滤与 HITL 包装 | `Tool`、`Toolset`、`Predicate`、`FilterToolset`、`WithConfirmation`、`ErrConfirmationRequired` |
| `tool/functiontool` | 把 Go 函数/流式函数包成 tool | `New[TArgs,TResults](Config, Func)`、`NewStreaming[TArgs]` |
| `tool/agenttool` | 把另一个 agent 包成 tool | `New(agent, *Config)` |
| `tool/mcptoolset` | MCP 服务器工具集 | `New(Config)`、`MCPClient`、`Config.Auth` |
| `tool/geminitool` | 模型侧内置工具（如 Google Search） | `GoogleSearch`、`New(name, description, *genai.Tool)` |
| `tool/exitlooptool` / `tool/loadartifactstool` / `tool/loadmemorytool` / `tool/preloadmemorytool` / `tool/exampletool` | 单一用途内置 tool | 各包 `New()` |
| `tool/skilltoolset` | 把 skill 目录暴露成 toolset | `New(ctx, Config)`、`SkillToolset` |
| `tool/skilltoolset/skill` | skill frontmatter 解析与来源 | `Source`、`NewFileSystemSource`、`Parse`、`Validate`、`Build` |
| `tool/toolconfirmation` | HITL 确认的载荷与还原 | `FunctionCallName`（`"adk_request_confirmation"`）、`ToolConfirmation`、`OriginalCallFrom` |
| `tool/toolutils` | 把 tool 打包进 `model.LLMRequest` | `PackTool` |

## 会话、状态与存储

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `session` | Session/Event/State/EventActions 与内存服务 | `Session`、`Event`、`EventActions`、`State`、`Service`、`InMemoryService`、`NewEvent`、`KeyPrefixApp`/`KeyPrefixTemp`/`KeyPrefixUser` |
| `session/compaction` | 上下文压缩与 LLM 摘要器 | `Config`、`Summarizer`、`LLMSummarizer`、`NewLLMSummarizer`、`ErrCompaction`、`ConversationHistoryPlaceholder` |
| `session/database` | GORM 关系型后端 | `NewSessionService(dialector, opts...)`、`NewSessionServiceFromDB`、`AutoMigrate` |
| `session/vertexai` | Vertex AI 会话后端 | `NewSessionService(ctx, VertexAIServiceConfig, opts...)` |
| `session/sessiontestsuite` | 会话服务一致性测试套件 | `RunServiceTests`、`SuiteOptions`、`Snapshot`、`ExpectedSession` |
| `artifact` | artifact 服务接口与内存实现 | `Service`、`InMemoryService`、`SaveRequest`、`LoadResponse` |
| `artifact/gcsartifact` | GCS artifact 后端 | `NewService(ctx, bucketName, opts...)`、`ErrVersionConflict` |
| `memory` | 长期记忆服务与内存实现 | `Service`、`InMemoryService`、`SearchRequest`、`Entry` |
| `memory/vertexai` | Vertex AI Memory Bank 后端 | `NewService(ctx, *ServiceConfig)` |

## 扩展、认证与可观测

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `plugin` | 跨阶段生命周期插件 | `Config`、`New`、`Plugin`、`BeforeRunCallback`、`OnEventCallback` |
| `plugin/loggingplugin` | 控制台日志插件 | `New(name)`、`MustNew(name)` |
| `plugin/retryandreflect` | tool 失败自纠重试插件 | `New(opts...)`、`WithMaxRetries`、`WithTrackingScope` |
| `plugin/functioncallmodifier` | 动态修改 tool declaration/参数 | `NewPlugin(FunctionCallModifierConfig)` |
| `auth` | 出站请求凭据与 provider | `Credential`、`CredentialProvider`、`StaticToken`、`ADC`、`ServiceAccount`、`Transport`、`InMemoryCredentialStore` |
| `auth/gcp` | GCP Agent Identity / IAM Connector 凭据 | `NewProvider(ctx, ProviderConfig)`、`NewClient(ctx, *Config)`、`ProviderScheme` |
| `agentregistry` | Google Cloud Agent Registry 客户端与装配 | `New(ctx, Config)`、`Client.RemoteAgent`、`Client.MCPToolset`、`WithA2AHeaders` |
| `telemetry` | OpenTelemetry providers 初始化 | `New(ctx, opts...)`、`Providers`、`WithOtelToCloud`、`WithTracerProvider` |
| `platform` | time/UUID/task runner 可替换接缝（确定性测试） | `WithTimeProvider`、`Now`、`WithUUIDProvider`、`NewUUID`、`RunTasks` |
| `util/instructionutil` | 指令模板里的 state 注入 | `InjectSessionState` |
| `util/aiplatform` | Vertex AI 主机名拼装 | `HostURL`、`HostPortURL` |
| `util/vertexai` | Agent Engine 资源名拼装 | `AgentEngineData`、`AgentEngineResource`、`SessionResource` |

## 服务端与部署

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `server` | 仅含包文档（doc-only harness） | 无导出符号 |
| `server/adkrest` | 主 REST/SSE 服务器 | `NewServer(ServerConfig)`、`ServerConfig`、`MaxBytesMiddleware`、`DefaultMaxPayloadSize` |
| `server/adkrest/controllers` | REST 控制器（runtime/apps/artifacts 等） | `NewRuntimeAPIController`、`NewAppsAPIController`、`NewArtifactsAPIController` |
| `server/adkrest/controllers/triggers` | Pub/Sub、Eventarc 触发式入口 | `NewPubSubController`、`NewEventarcController`、`TriggerConfig` |
| `server/adka2a` | A2A 服务端（薄壳，指向 v2） | `Executor`、`Runner`、`RunnerProvider`、`A2AEventConverter` |
| `server/adka2a/v2` | A2A 服务端 v2 实现 | `NewExecutor`、`ExecutorConfig`、`Runner`、`OutputMode`、`BuildAgentSkills` |
| `server/agentengine` | Vertex AI Agent Engine 服务端 | `NewHandler`、`ListClassMethods` |
| `server/agentengine/controllers` | Agent Engine API 控制器 | `NewAgentEngineAPIController`、`MethodHandler` |
| `server/agentengine/controllers/method` | 单个 Agent Engine 方法处理器 | `NewStreamQueryHandler`、`NewCreateSessionHandler` |
| `server/authn` | 入站认证（IAP/OIDC/header） | `Authenticator`、`NewGoogleOIDC`、`NewIdentityAwareProxy`、`NewHeader`、`Middleware` |
| `server/authz` | 入站授权 | `Authorizer`、`NewNoop`、`NewStrict`、`WriteHTTPStatusForAuthError` |

## 命令与启动器

| 包 | 职责 | 代表符号 |
| --- | --- | --- |
| `cmd/adkgo` | CLI 入口（`main`） | `RootCmd`、`DeployCmd`、`Execute` |
| `cmd/launcher` | launcher 接口与 Config | `Launcher`、`SubLauncher`、`Config` |
| `cmd/launcher/full` / `prod` / `universal` / `agentengine` | 组合式 launcher | 各包 `NewLauncher()` |
| `cmd/launcher/console` | 终端交互 launcher | `NewLauncher()` |
| `cmd/launcher/web` | Web 服务 launcher 与路由 | `NewLauncher(sublaunchers...)`、`Sublauncher`、`BuildBaseRouter` |
| `cmd/launcher/web/api` / `webui` / `a2a` / `agentengine` | Web 子启动器 | 各包 `NewLauncher()` |
| `cmd/launcher/web/triggers/pubsub` / `eventarc` | 触发式 Web 启动器 | 各包 `NewLauncher()` |

## internal/ 说明

`internal/` 下的包**不是公开 API**，不承诺向后兼容，也不应被外部导入。它们承载实现细节：`internal/llminternal`（LLM flow、处理器、流式聚合、转账）、`internal/agent/parentmap`、`internal/agent/runconfig`、`internal/agent/compactionctx`、`internal/context`（InvocationContext/CallbackContext/ToolContext 实现）、`internal/toolinternal`、`internal/telemetry`、`internal/testutil`、`internal/httprr`（vendored 录制器）、`internal/workflowinternal`、`internal/compactioninternal`、`internal/configurable`（一致性回放）、`internal/typeutil`、`internal/utils`、`internal/converters`、`internal/plugininternal`、`internal/memory`、`internal/artifact`、`internal/adkcontext`。其中 `internal/llminternal` 与 `internal/utils` 的实现细节被本 wiki 的进阶页大量引用，见 [12-advanced-topics.md](./12-advanced-topics.md)。

## 常见任务 → 该看哪个包

| 想做的事 | 该看 |
| --- | --- |
| 加一个工具 | `tool/functiontool`（`New` / `NewStreaming`）；要完全控制就实现 `tool.Tool` |
| 加一组动态工具 | 实现 `tool.Toolset`（`tool`），或用 `tool/mcptoolset`、`tool/skilltoolset` |
| 换模型后端 | `model`（`Register`/`NewLLM`），实现 `model.LLM`；现成后端在 `model/gemini`、`model/openaimodel`、`model/apigee` |
| 加一个 LLM agent | `agent/llmagent.New`，配 `GenerateContentConfig`、`Instruction`、`Tools`、回调 |
| 多 agent 顺序/并行/循环 | `agent/workflowagents/{sequentialagent,parallelagent,loopagent}` |
| 自定义图工作流 | `workflow`（`New`、`FunctionNode`、`AgentNode`、`ToolNode`、`Edge`/`Route`、`Resume`） |
| 接 A2A | 服务端 `server/adka2a/v2`；客户端 `agent/remoteagent/v2`；从注册表发现 `agentregistry` |
| 从 Agent Registry 装配远端 agent/MCP | `agentregistry`（`Client.RemoteAgent`、`Client.MCPToolset`） |
| 换会话/记忆/artifact 后端 | `session/database`、`session/vertexai`、`memory/vertexai`、`artifact/gcsartifact` |
| 上下文压缩 | `session/compaction`，经 `runner.Config.Compaction` 开启 |
| 跨阶段钩子/日志/重试 | `plugin`、`plugin/loggingplugin`、`plugin/retryandreflect`、`plugin/functioncallmodifier` |
| 出站请求带凭据 | `auth`（`Credential`/`CredentialProvider`/`Transport`）；GCP 用 `auth/gcp` |
| 开可观测 | `telemetry`；服务端集成见 `cmd/launcher/internal/telemetry` |
| 起 HTTP 服务 | `server/adkrest`，经 `cmd/launcher/web` 组合 |
| 部署到 Agent Engine / Cloud Run | `cmd/adkgo` 的 `DeployCmd`、`cmd/launcher/agentengine`、`server/agentengine` |
| 确定性测试（时间/UUID） | `platform`（`WithTimeProvider`、`WithUUIDProvider`、`WithTaskRunner`） |
| 测一个 agent | `runner.NewInMemory` 或 `internal/testutil`（仅仓库内测试可用） |
| 为会话服务写后端测试 | `session/sessiontestsuite` |

## 相关页面

- [01-overview.md](./01-overview.md) —— 模块划分与 v2 结构。
- [03-core-concepts.md](./03-core-concepts.md) —— 核心接口的语义。
- [07-a2a-distributed.md](./07-a2a-distributed.md) —— A2A 协议与 `server/adka2a`。
- [08-service-layer.md](./08-service-layer.md) —— session/artifact/memory 服务与后端。
- [09-launcher-deployment.md](./09-launcher-deployment.md) —— `cmd/adkgo`、launcher、adkrest。
- [12-advanced-topics.md](./12-advanced-topics.md) —— 流式、live、重排、插件、认证等进阶细节。
