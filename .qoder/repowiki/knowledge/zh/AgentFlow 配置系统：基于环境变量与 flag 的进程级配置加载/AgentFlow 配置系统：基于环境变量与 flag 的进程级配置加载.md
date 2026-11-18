---
kind: configuration_system
name: AgentFlow 配置系统：基于环境变量与 flag 的进程级配置加载
category: configuration_system
scope:
    - '**'
source_files:
    - cmd/agentflow/main.go
    - internal/config/endpoints.go
    - .env.example
    - compose.yaml
    - compose.dev.yaml
    - cmd/agentflow-runner/main.go
    - cmd/agentflow-mcp/main.go
    - cmd/agentflow-gate-mcp/main.go
---

## 1. 总体方案

LoopForge AgentFlow 后端不使用任何第三方配置库（如 viper、koanf），而是采用 **Go `flag` + `os.Getenv`** 的原生方式，在进程启动时一次性解析所有运行时配置，随后通过包级单例 `internal/config.Endpoints` 向其余模块暴露。配置来源按优先级从高到低为：

1. CLI `-flag`（仅主入口 `cmd/agentflow/main.go`）
2. 进程环境变量（统一以 `AGENTFLOW_` 前缀命名）
3. `.env` 文件（由内置 `loadDotEnv()` 自动发现并填充到 `os.Environ`）
4. Go 代码中的硬编码默认值

前端与部署侧则通过 `compose.yaml` / `compose.dev.yaml` 将环境变量注入容器，并通过 `.env.example` 提供模板。

## 2. 关键文件

- `cmd/agentflow/main.go` — 唯一的主入口，负责 flag 解析、`.env` 加载、校验、角色分发以及调用 `config.LoadEndpoints`。
- `internal/config/endpoints.go` — 唯一的配置解析中心，定义 `Endpoints` 结构体并提供 `LoadEndpoints` / `SetCurrent` / `Current`。
- `.env.example` — 开发环境所需的环境变量清单与注释。
- `compose.yaml` / `compose.dev.yaml` — Docker Compose 中实际注入的 `AGENTFLOW_*` 变量集合。
- `scripts/configure.sh`（被 `.env.example` 引用）— 生成带随机密钥的 `.env`。
- `cmd/agentflow-runner/main.go` — Runner 子进程同样使用 `flag.Parse()` + `os.Getenv("AGENTFLOW_RUNNER_*")` 读取自身配置。
- `cmd/agentflow-mcp/main.go`、`cmd/agentflow-gate-mcp/main.go`、`cmd/loopforge-account/main.go` — 其他可执行入口各自直接读 `os.Getenv`。

## 3. 架构与约定

### 3.1 单一 Endpoints 聚合
`internal/config/endpoints.go` 将进程监听地址、API 基址、平台 MCP 地址、控制面 URL、Worker 回退地址、AgentTool 网关、OTLP Collector 基址等全部收敛到 `Endpoints` 结构体。`LoadEndpoints(role, addrFlag)` 根据 role（`api` / `worker` / `all`）返回不同的组合，例如 worker 不暴露 APIBase，而 api 端把 PlatformBase 默认指向自身。

### 3.2 进程级全局发布
`SetCurrent(ep)` 写入包级 `current *Endpoints`（带 `sync.RWMutex`），`Current()` 供无法接收参数的辅助函数读取；若 main 从未调用 `SetCurrent`（测试或 helper 二进制），则回退到 `LoadEndpoints("all", "")` 的默认值。

### 3.3 `.env` 自动发现顺序
`loadDotEnv()` 会依次查找：
1. 当前工作目录及最多上溯三层父目录下的 `.env`
2. 可执行文件同目录下的 `.env`

每条 `KEY=VALUE` 行支持 `export KEY=VALUE` 语法，且 **已存在的环境变量不会被覆盖**（`if k != "" && os.Getenv(k) == ""`），保证 shell 环境与容器注入始终优先。

### 3.4 配置校验策略
- `AGENTFLOW_SESSION_SECRET` 缺失时直接 `log.Fatal`（`main.go:107`）。
- `AGENTFLOW_AUTH_REQUIRED` 未设置时默认设为 `true`（`main.go:110`）。
- `AGENTFLOW_HTTP_PORT` 必须为 1–65535 的整数，否则 `LoadEndpoints` 报错（`endpoints.go:76`）。
- `AGENTFLOW_ROLE` 仅接受 `api` / `worker` / `all`，非法值返回错误（`resolveRole`）。
- `AGENTFLOW_RUN_LEASE_TTL`、`AGENTFLOW_EXECUTE_RUNS`、`AGENTFLOW_DRAIN_TIMEOUT` 等数值/布尔型变量在解析失败时 `log.Fatalf`。

### 3.5 多进程配置一致性
API、Runner、MCP Gate、Account 四个可执行程序共享同一套 `AGENTFLOW_*` 命名空间，但每个程序只读取自己关心的子集：
- API/Worker 主进程：`AGENTFLOW_HTTP_PORT`、`AGENTFLOW_MYSQL_DSN`、`AGENTFLOW_SESSION_SECRET`、`AGENTFLOW_AUTH_REQUIRED`、`AGENTFLOW_PLATFORM_MCP_URL`、`AGENTFLOW_CONTROL_TOKEN`、`AGENTFLOW_WORKER_URL`、`AGENTFLOW_AGENT_TOOL_GATEWAY_URL`、`AGENTFLOW_INSTANCE_ID`、`AGENTFLOW_RUN_LEASE_TTL`、`AGENTFLOW_EXECUTE_RUNS`、`AGENTFLOW_CLOUD_RUNNER_LABEL`、`AGENTFLOW_CLOUD_RUNNER_PROJECTS`、`AGENTFLOW_LOCAL_EXECUTION`、`AGENTFLOW_DRAIN_TIMEOUT`、`AGENTFLOW_MANAGED_CLI_ROOT` 等。
- Runner：`AGENTFLOW_SERVER_URL`、`AGENTFLOW_RUNNER_BOOTSTRAP_TOKEN`、`AGENTFLOW_RUNNER_STATE_PATH`、`AGENTFLOW_RUNNER_NAME`、`AGENTFLOW_RUNNER_WORKSPACE`、`AGENTFLOW_RUNNER_CONCURRENCY`、`AGENTFLOW_MANAGED_CLI_ROOT`。
- MCP/Gate：`AGENTFLOW_API_URL`、`AGENTFLOW_GATE_RESULT_PATH`。

### 3.6 部署层映射
`compose.yaml` 是生产/本地 Compose 部署的唯一配置源，它将宿主机端口 `${LOOPFORGE_BIND:-0.0.0.0}:${AGENTFLOW_HTTP_PORT:-8090}:8080` 映射到容器内固定 8080 端口，并把数据库 DSN、Session Secret、GitHub Client ID、SSH Known Hosts、Assistant Model、Preview Public URL、Docker Network、Isolated Image、Runner Binary 路径等全部以 `AGENTFLOW_*` 形式注入。`compose.dev.yaml` 在此基础上覆盖 `AGENTFLOW_ISOLATED_IMAGE=loopforge-platform:local` 用于源码构建。

## 4. 约定与约束

- **环境变量命名规范**：所有运行时配置均以 `AGENTFLOW_` 为前缀（Runner 子进程额外使用 `AGENTFLOW_RUNNER_*`）。该约定由 `cmd/agentflow/main.go`、`cmd/agentflow-runner/main.go`、`cmd/agentflow-mcp/main.go`、`cmd/agentflow-gate-mcp/main.go`、`internal/config/endpoints.go` 多处共同体现，并由 `compose.yaml` 作为部署契约固化。
- **`.env` 不可覆盖已有环境变量**：`loadDotEnv()` 显式跳过已存在的 key，这是实现层面的强制行为（`os.Getenv(k) == ""` 守卫）。
- **Session Secret 必填**：非 CLI-admin 模式下缺少 `AGENTFLOW_SESSION_SECRET` 会直接终止进程（`log.Fatal`）。
- **认证默认开启**：未设置 `AGENTFLOW_AUTH_REQUIRED` 时自动设为 `true`。
- **HTTP 端口范围校验**：`AGENTFLOW_HTTP_PORT` 不在 1–65535 范围内会导致启动失败。
- **角色枚举约束**：`AGENTFLOW_ROLE` 仅允许 `api`、`worker`、`all`，其它值返回错误。
- **无外部配置框架**：仓库未引入 viper/koanf/envcfg 等库，所有配置解析均为手写 `os.Getenv` + `flag`。
- **配置集中化**：除各子进程的独立 flag 外，业务相关配置全部集中在 `internal/config/endpoints.go` 的 `Endpoints` 结构体中，避免散落在 HTTP handler 或 engine 内部。
- **Compose 是部署期权威源**：`compose.yaml` 中通过 `${VAR:?Run scripts/configure.sh first}` 语法强制要求先运行 `scripts/configure.sh` 生成 `.env`，否则 Compose 启动即失败。

## 5. 前端与 E2E 侧

- `chatbot/src/platform.js`、`chatbot/src/engines.js`、`web/app/*.js` 等前端模块通过相对路径 `/api/...` 访问后端，不直接读取配置；模型列表、执行器选择等动态能力由运行时 API 返回。
- `e2e/lib/env.mjs` 与 `e2e/playwright.config.mjs` 从 Node 环境读取测试用环境变量，与后端 `AGENTFLOW_*` 命名空间解耦。
