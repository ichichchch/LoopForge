---
kind: logging_system
name: 可观测性日志系统：TelemetryEvent/LogRecord 统一事件模型与 Hook/OTLP 双通道采集
category: logging_system
scope:
    - '**'
source_files:
    - internal/domain/observability.go
    - internal/observability/hooks.go
    - internal/observability/otlp.go
    - internal/engine/engine_observe.go
    - cmd/agentflow/main.go
    - cmd/agentflow-runner/main.go
    - cmd/loopforge-account/main.go
---

## 1. 使用的系统与总体思路

AgentFlow 后端没有采用通用的第三方日志框架（如 logrus、zap、slog），而是自研了一套**以结构化事件为中心的遥测/日志体系**，核心由 `internal/domain` 中的领域类型定义，并由 `internal/observability` 提供多种采集适配器。所有业务模块通过构造 `domain.TelemetryEvent` / `domain.LogRecord` 并调用引擎的 `observe()` / `enqueueObservation()` 写入，最终汇聚到 Run 对象的 `r.Telemetry` 与 `r.Logs` 两个字段。

- **事件模型**：`domain.TelemlemetryEvent`（带 Category/Name/Phase/Level/Message/Attributes）用于“指标型”观察；`domain.LogRecord`（带 Stream/Body/Sequence）用于“诊断型”日志流，二者共享 `TelemetrySource` 与 `TelemetryContext` 做关联。
- **统一入口**：`WorkerEnvelope` + `ObservabilitySpecVersion = "agentflow.observability/v1"` 是 CLI、远程 worker、SDK、容器 runner 等外部执行后端的 vendor-neutral ingress 契约。
- **采集适配器**：
  - **Hook 适配器**（`AdapterHooks = "agent-hooks-v1"`、`AdapterLegacyHooks = "codebuddy-hooks-v1"`）：向宿主（CodeBuddy/Claude Code 等）注入一个 Python hook 脚本，将 stdout/stderr 追加到 `.agentflow/observability/hooks.ndjson`，平台侧通过 `WatchHookFile` 轮询解析为 TelemetryEvent。
  - **OTLP 适配器**（`AdapterOTEL = "codebuddy-otel-v1"`）：接收 JSON 或 Protobuf 编码的 OTel traces，内置轻量 proto 解析器（`decodeOTLPProtobuf` / `walkResourceSpans`），无需引入 `google.golang.org/protobuf`。
  - **Live 适配器**（`AdapterLive = "anthropic-stream-v1"`）：用于 Anthropic 原生流式输出。
- **路由与过滤**：`MatchAdapter(event, view)` 根据 `event.Source.Adapter` 把事件分派到 hooks / otel / live / all 视图。
- **环境变量注入**：`InjectCodeBuddyEnv` 在子进程环境中设置 `AGENTFLOW_RUN_ID`、`AGENTFLOW_NODE_ID`、`CODEBUDDY_CODE_ENABLE_TELEMETRY=1`、`CLAUDE_CODE_ENABLE_TELEMETRY=1`、`OTEL_TRACES_EXPORTER=otlp`、`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`、`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=<CollectorTracesURL()>`、`OTEL_RESOURCE_ATTRIBUTES=agentflow.run_id=...,agentflow.node_id=...,service.name=agentflow-codebuddy`。

## 2. 关键文件与包

| 路径 | 作用 |
|---|---|
| `internal/domain/observability.go` | 定义 `TelemetrySource`、`TelemetryContext`、`TelemetryEvent`、`LogRecord`、`WorkerEnvelope`、`ObservabilitySpec`、`AdapterDescriptor` 等核心领域类型及常量 |
| `internal/observability/hooks.go` | 生成并写入宿主 hook 脚本、解析 `hooks.ndjson`、`WatchHookFile` 轮询、`MatchAdapter`、`CollectorTracesURL`、`InjectCodeBuddyEnv` |
| `internal/observability/otlp.go` | 解析 JSON/Protobuf OTLP traces 为 `[]TelemetryEvent`，含内嵌 proto 扫描器 |
| `internal/engine/engine_observe.go` | 引擎侧 `enqueueObservation` / `applyObservation`，把 observation 归一化后写入 `r.Telemetry`，并把 message 作为 `LogRecord` 写入 `r.Logs` |
| `cmd/agentflow/main.go` 等 cmd 入口 | 使用标准库 `log.Fatal` / `log.Printf` 打印启动信息（非结构化） |

## 3. 架构与约定

### 3.1 事件模型

```text
TelemetryEvent
├── Category: lifecycle / tool / model / sandbox / artifact / ...
├── Name:     具体事件名（如 agent.initialized、tool.call、model.request）
├── Phase:    start / progress / finish
├── Level:    info / error（由 exitCode != 0 或 span status.code == 2 推导）
├── Message:  人类可读摘要（clip 至 180 字符）
├── Attributes: map[string]any（结构化字段）
├── Source:   TelemetrySource{Layer, Adapter, Provider, Protocol}
└── Context:  TelemetryContext{RunID, NodeID, WorkflowID, ...}
```

`LogRecord` 与 `TelemetryEvent` 解耦：前者是 cursor-addressable 的诊断日志流（带 `Stream`、`Body`、`Sequence`），后者用于聚合与可视化。两者都携带相同的 `Source` 与 `Context`，以便按 run/node/workflow 关联。

### 3.2 采集管线

1. 业务代码（engine、httpapi、runner 等）构造 `TelemetryEvent` 并通过 `e.observe(runID, nodeID, event)` 入队。
2. `engine_observe.applyObservation` 将 event 追加到 `r.Telemetry`（保留最近 4000 条），同时把 `observation.Message` 包装成 `LogRecord` 追加到 `r.Logs`（保留最近 8000 条）。
3. 对来自外部 worker 的事件，通过 `IngestWorkerEnvelope` 消费 `WorkerEnvelope`，其 `Type` 区分 `observation` 与 `log`。
4. 对宿主侧产生的数据：
   - Hook 模式：`WriteCodeBuddyHookBridgeAt` 写出 `hook.py` 与 `codebuddy-settings.json`，`WatchHookFile` 每 250ms 扫描 `hooks.ndjson`，用 `HookEvent` 解析为 TelemetryEvent。
   - OTLP 模式：`ParseOTLPTraces` 接收 HTTP body，先尝试 JSON，再回退到内嵌 proto 解析。

### 3.3 适配器识别

`MatchAdapter` 支持以下 view 值：`""` / `"all"`（全部）、`"hooks"`、`"otel"`、`"live"`、自定义 adapter ID。该函数决定 UI 中哪些事件可见。

### 3.4 安全与裁剪

- `clip(s, n)` 截断字符串（消息 180、属性值 2000）。
- Hook 脚本只写 `.agentflow/observability/hooks.ndjson`，且 `writeFileIfChanged` 通过 temp+rename 原子替换，避免并发 CLI 读到截断文件。
- OTLP 解析器不依赖 protobuf 库，仅读取必要字段，失败时静默丢弃。

## 4. 约定与约束

- **规范版本**：所有事件必须声明 `specVersion = ObservabilitySpecVersion = "agentflow.observability/v1"`，由 `WorkerEnvelope.SpecVersion` 承载。
- **Source 不可分支**：`TelemetrySource` 注释明确说明 “Consumers must not branch on native payloads”，下游应基于 `Category/Name/Phase` 而非原始 payload 分支。
- **Level 推导规则**：当 `exitCode != 0` 或 OTLP span `Status.Code == 2` 时，level 强制设为 `"error"`，其余为 `"info"`。
- **消息长度限制**：`Message` 经 `clip(..., 180)` 截断；属性值经 `clip(..., 2000)` 截断。
- **存储上限**：`r.Telemetry` 最多保留 4000 条，`r.Logs` 最多保留 8000 条（见 `engine_observe.go` 切片赋值）。
- **OTLP endpoint 优先级**：`AGENTFLOW_OTEL_TRACES_ENDPOINT` > `AGENTFLOW_OTEL_ENDPOINT` + `/api/otel` > `config.Current().CollectorBase + "/api/otel"`，最终拼接 `/v1/traces`。
- **宿主环境变量**：`InjectCodeBuddyEnv` 固定注入 `AGENTFLOW_RUN_ID`、`AGENTFLOW_NODE_ID`、`CODEBUDDY_CODE_ENABLE_TELEMETRY=1`、`CLAUDE_CODE_ENABLE_TELEMETRY=1`、`OTEL_TRACES_EXPORTER=otlp`、`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`、`OTEL_LOG_TOOL_DETAILS=1` 以及 `OTEL_RESOURCE_ATTRIBUTES=agentflow.run_id=...,agentflow.node_id=...,service.name=agentflow-codebuddy`。
- **cmd 入口日志**：`cmd/agentflow/main.go`、`cmd/agentflow-runner/main.go`、`cmd/loopforge-account/main.go` 等入口仍使用 Go 标准库 `log.Fatal` / `log.Printf` 输出启动与错误信息，未接入结构化 TelemetryEvent 体系。
- **Hook 协议兼容**：`HookEvent` 同时接受新旧字段名（如 `hook_event_name` / `hookEventName`、`session_id` / `sessionId`、`_agentflow_execution_phase` / `_agentflow_exit_code` 等），体现向后兼容约定。