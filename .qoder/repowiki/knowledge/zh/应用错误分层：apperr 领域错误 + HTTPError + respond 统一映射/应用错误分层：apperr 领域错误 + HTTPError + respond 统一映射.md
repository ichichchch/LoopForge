---
kind: error_handling
name: 应用错误分层：apperr 领域错误 + HTTPError + respond 统一映射
category: error_handling
scope:
    - '**'
source_files:
    - internal/apperr/apperr.go
    - internal/httpapi/gin_helpers.go
    - internal/httpapi/gin_helpers_test.go
    - internal/engine/engine.go
    - internal/knowledge/service.go
---

## 1. 整体方案

LoopForge AgentFlow 后端采用**两层错误模型**，将“业务语义”与“传输状态码”解耦：

- **领域层**（`internal/apperr`）定义与应用无关的错误种类 `Kind`（`Invalid / NotFound / Conflict / Forbidden / Unavailable / Upstream`），通过 `apperr.Error` 携带用户消息和可选的扩展字段（`Fields map[string]any`）。
- **HTTP 层**（`internal/httpapi/gin_helpers.go`）提供 `HTTPError`（直接带 `Status` + `Message`）以及统一的响应入口 `respond()`，负责把任意 error 链解析为 Gin JSON 响应。

这种设计让上层服务（engine、knowledge、repo 等）只关心“发生了什么”，不依赖 HTTP；transport 层在 `respond()` 中集中完成映射。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `internal/apperr/apperr.go` | 定义 `Kind` 枚举、`Error` 结构体、`New/Newf/With/As` API |
| `internal/httpapi/gin_helpers.go` | 定义 `HTTPError`、`respond()`、`appErrStatus()`、JSON 解码中间件 |
| `internal/engine/engine.go` | 导出哨兵错误 `ErrExecutionDisabled`（被 `respond` 特殊处理为 403） |
| `internal/knowledge/service.go` | 典型使用 `apperr.New(apperr.Invalid/NotFound/Conflict/Forbidden/Unavailable/Upstream)` 的业务示例 |
| `internal/httpapi/gin_helpers_test.go` | 断言每种 `Kind` → HTTP 状态码的映射关系 |

## 3. 架构与约定

### 3.1 错误类型层次

```text
error (interface)
├── apperr.Error          // 领域错误，含 Kind + Message + Fields
├── httpapi.HTTPError     // 透传精确 HTTP 状态码（如 404/409）
└── sentinel errors       // 如 engine.ErrExecutionDisabled
```

`respond()` 按以下优先级解析 error 链：

1. `errors.As(&HTTPError{})` → 直接使用 `HTTPError.Status` 与 `Message`。
2. `apperr.As(err)` → 根据 `Kind` 查表映射到 HTTP 状态码，并把 `Fields` 展开进响应体。
3. `errors.Is(err, engine.ErrExecutionDisabled)` → 固定返回 403。
4. 兜底 → 400 Bad Request，body 为 `{"error": err.Error()}`。

### 3.2 `apperr.Kind` → HTTP 状态码映射

| Kind | 状态码 | 含义 |
|---|---:|---|
| `Invalid` | 400 | 请求参数或输入无效 |
| `NotFound` | 404 | 资源不存在 |
| `Conflict` | 409 | 资源冲突（常配合 `.With("id", ...)` 返回冲突项 ID） |
| `Forbidden` | 403 | 权限不足 |
| `Unavailable` | 503 | 依赖未配置（如知识库未启用） |
| `Upstream` | 502 | 上游调用失败（如外部 MCP 不可达） |
| 其他/默认 | 400 | 兜底 |

该映射由 `appErrStatus()` 硬编码并在 `gin_helpers_test.go` 中以表格用例覆盖。

### 3.3 领域错误构造模式

业务代码通过 `apperr.New(kind, message)` 或 `apperr.Newf(kind, format, args...)` 创建错误，并通过链式 `.With(key, value)` 附加结构化字段。例如 `internal/knowledge/service.go` 中：

```go
return item, apperr.New(apperr.Conflict, "该产物已加入沉淀队列").With("id", existing)
return nil, apperr.New(apperr.Upstream, "知识库不可达："+err.Error())
```

### 3.4 非 apperr 错误的处理

- 普通 `fmt.Errorf` 包装的错误最终落入 `respond()` 的 400 分支，仅透传 `err.Error()` 文本。
- 自定义错误类型在各子包内实现（如 `botinteraction.ReplyError`、`notification.ProviderError`、`runner.workspaceBusyError`、`httpapi.skillhubError` 等），但它们没有统一的跨包错误基类——这些类型主要用于日志/测试可读性，而非被 `respond` 识别。

### 3.5 panic 策略

仓库中存在少量 `panic(...)`，集中在：

- `cmd/agentflow/main.go`：启动期参数校验失败时 panic（进程级初始化失败）。
- `internal/auth/client.go`：配置加载失败时 panic。
- `internal/httpapi/mcp_market.go`：MCP catalog 拉取失败时 panic。
- 测试文件中大量 `panic("boom")` / `panic(err)` 用于模拟异常路径。

未见全局 recover 中间件；panic 被视为“不可恢复的启动/配置错误”，由进程退出承载。

## 4. 约定与约束

- **领域错误必须走 `apperr`**：所有面向用户的业务错误应使用 `apperr.New` / `Newf`，以便自动获得正确的 HTTP 状态码和结构化字段。
- **新增 Kind 需三处同步**：`apperr.Kind` 常量、`appErrStatus()` switch、`gin_helpers_test.go` 中的映射用例——否则测试会暴露缺失的映射分支。
- **HTTP 层唯一响应入口是 `respond()`**：handler 不应直接写 `c.JSON(...)` 表达业务错误，而应返回 error 交由 `respond` 渲染。
- **需要精确状态码时使用 `HTTPError`**：当某个错误无法归类到 `apperr.Kind` 时，返回 `httpapi.HTTPError{Status, Message}`。
- **引擎禁用场景使用哨兵错误**：`engine.ErrExecutionDisabled` 作为 `errors.Is` 判断的专用哨兵，被 `respond` 特殊映射为 403，避免把它误归为 `Forbidden`。
- **错误消息面向用户**：`apperr.Error.Message` 是对外展示文案，不应包含内部堆栈；内部细节应放入 `Fields` 或通过日志记录。
- **JSON 请求体大小限制**：`decode` / `decodeOptional` 统一用 `MaxBytesReader(2 MiB)` 截断请求体，非法 JSON 统一返回 400，避免 handler 重复处理。
- **CORS 与 CSP 在中间件中拒绝非法来源**：跨域请求直接返回 403 并 `c.Abort()`，不进入业务逻辑。