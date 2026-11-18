# CodeBuddy 执行器

<cite>
**本文引用的文件**   
- [README.md](file://README.md)
- [deployment.md](file://docs/deployment.md)
- [codebuddy_auth.go](file://internal/engine/codebuddy_auth.go)
- [provider_codebuddy.go](file://internal/runtime/provider_codebuddy.go)
- [stream.go](file://internal/httpapi/stream.go)
- [assistant.go](file://internal/httpapi/assistant.go)
- [runtime_test.go](file://internal/runtime/runtime_test.go)
- [provider_test.go](file://internal/runtime/provider_test.go)
- [agent.go](file://internal/domain/agent.go)
- [codebuddy_sponsor.go](file://internal/repo/codebuddy_sponsor.go)
- [platform_settings.go](file://internal/httpapi/platform_settings.go)
- [oauth.go](file://internal/gongfeng/oauth.go)
- [auth.go](file://internal/gitwork/auth.go)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构总览](#架构总览)
5. [详细组件分析](#详细组件分析)
6. [依赖关系分析](#依赖关系分析)
7. [性能与行为特性](#性能与行为特性)
8. [故障排查指南](#故障排查指南)
9. [结论](#结论)
10. [附录：企业部署与最佳实践](#附录企业部署与最佳实践)

## 简介
本文件面向需要在企业环境中集成并运行 CodeBuddy 执行器的平台管理员、运维工程师和高级用户。文档聚焦以下目标：
- 解释 LoopForge 如何启动、配置和调用 CodeBuddy CLI，包括工作区准备、权限隔离、MCP 工具与安全参数注入。
- 说明企业级认证机制：个人 API Key、平台共享 Key、网络环境（国际站/国内站/iOA）与模型白名单控制。
- 说明工作区权限管理、团队协作与项目访问控制：Git SSH 密钥、OAuth 令牌、仓库绑定与分支策略。
- 解释与 Claude Code 类似的 --effort 参数支持，以及 CodeBuddy 特有的安全与沙箱要求。
- 提供 OAuth 流程、SSH 密钥管理与项目访问控制的端到端说明。
- 给出企业部署配置与最佳实践，帮助在受控网络中稳定运行。

## 项目结构
围绕 CodeBuddy 执行器的关键代码分布在以下模块：
- HTTP API 层：负责解析请求、定位 CodeBuddy 二进制、组装命令参数、绑定凭据。
- 运行时提供者：负责为 CodeBuddy 准备工作区、钩子桥接、MCP 配置、权限隔离与 effort 参数注入。
- 引擎认证：负责选择用户个人 Key 或平台共享 Key，并对共享 Key 的模型进行限制。
- 仓库与 Git 集成：负责 GitHub/GitLab 等 OAuth 与 SSH 密钥管理。
- 部署文档：说明容器化安装、环境变量、CodeBuddy 网络环境与托管 CLI 清单。

```mermaid
graph TB
Client["客户端/浏览器"] --> API["HTTP API<br/>assistant.go / stream.go"]
API --> Runtime["运行时提供者<br/>provider_codebuddy.go"]
Runtime --> Engine["引擎认证<br/>codebuddy_auth.go"]
Runtime --> Repo["仓库与凭据<br/>gitwork/auth.go / gongfeng/oauth.go"]
API --> Repo
Runtime --> Sandbox["沙箱与工作区<br/>hooks/MCP/settings"]
Runtime --> CLI["CodeBuddy CLI 进程"]
```

图示来源
- [assistant.go:1730-1814](file://internal/httpapi/assistant.go#L1730-L1814)
- [stream.go:190-232](file://internal/httpapi/stream.go#L190-L232)
- [provider_codebuddy.go:15-89](file://internal/runtime/provider_codebuddy.go#L15-L89)
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [auth.go:23-38](file://internal/gitwork/auth.go#L23-L38)
- [oauth.go:88-115](file://internal/gongfeng/oauth.go#L88-L115)

章节来源
- [README.md:40-133](file://README.md#L40-L133)
- [deployment.md:7-18](file://docs/deployment.md#L7-L18)

## 核心组件
- CodeBuddy 二进制定位与托管清单
  - 优先使用显式环境变量覆盖；其次读取托管 CLI 清单；再次查找 PATH；最后探测常见安装路径；若均未找到则返回缺失错误。
- 运行时提供者
  - 准备 CodeBuddy 的工作区、钩子桥接、settings 合并、MCP 配置、插件目录、会话恢复、严格 MCP 模式与 effort 参数注入。
  - 对 CodeBuddy 启用私有 Home 与 BunFS 沙箱需求，确保并行角色互不干扰。
- 引擎认证与企业特性
  - 优先使用用户个人 CodeBuddy API Key；若无则回退到平台共享 Key，并限制可用模型集合。
  - 当使用共享 Key 时，记录“模型赞助”遥测事件，并在运行状态中提示用户。
- Git 与 OAuth
  - 工蜂 OAuth 用于获取仓库访问令牌，支持授权码交换与刷新令牌。
  - Git 操作将 OAuth 令牌以 Basic Auth 形式注入远程地址，用户名固定为 oauth2。
- 部署与环境
  - 支持通过 AGENTFLOW_CODEBUDDY_BIN 指定二进制；AGENTFLOW_MANAGED_CLI_ROOT 指向托管清单目录。
  - CodeBuddy 网络环境分为国际站、国内站 internal、iOA ioa，影响模型目录与 Key 校验。

章节来源
- [stream.go:190-232](file://internal/httpapi/stream.go#L190-L232)
- [provider_codebuddy.go:15-89](file://internal/runtime/provider_codebuddy.go#L15-L89)
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [oauth.go:88-115](file://internal/gongfeng/oauth.go#L88-L115)
- [auth.go:23-38](file://internal/gitwork/auth.go#L23-L38)
- [deployment.md:7-18](file://docs/deployment.md#L7-L18)

## 架构总览
下图展示一次由 HTTP API 触发的 CodeBuddy 执行过程，涵盖二进制定位、凭据绑定、工作区准备、CLI 启动与结果流式返回。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "HTTP API<br/>assistant.go / stream.go"
participant Runtime as "运行时提供者<br/>provider_codebuddy.go"
participant Engine as "引擎认证<br/>codebuddy_auth.go"
participant Repo as "仓库/OAuth/SSH<br/>gitwork/auth.go / gongfeng/oauth.go"
participant CLI as "CodeBuddy CLI"
Client->>API : 发起助手/执行请求
API->>API : 解析模型、设置、会话ID、effort
API->>Runtime : PrepareProviderCommand(ExecutorDefinition)
Runtime->>Engine : resolveCodeBuddyCredential(userID)
Engine-->>Runtime : 个人Key或共享Key+允许模型
Runtime->>Repo : 绑定Git凭据/MCP/CodeBuddy
Runtime->>Runtime : 准备工作区、钩子、settings、MCP、strict-mcp-config
Runtime->>CLI : 启动 codebuddy --effort ... -- --prompt
CLI-->>Runtime : 流式输出/工具调用
Runtime-->>API : 标准化事件/产物
API-->>Client : 返回消息、日志、制品
```

图示来源
- [assistant.go:1730-1814](file://internal/httpapi/assistant.go#L1730-L1814)
- [provider_codebuddy.go:54-89](file://internal/runtime/provider_codebuddy.go#L54-L89)
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [auth.go:23-38](file://internal/gitwork/auth.go#L23-L38)
- [oauth.go:88-115](file://internal/gongfeng/oauth.go#L88-L115)

## 详细组件分析

### CodeBuddy 二进制定位与托管清单
- 优先级顺序：
  1. 显式环境变量 AGENTFLOW_CODEBUDDY_BIN。
  2. 托管 CLI 清单 AGENTFLOW_MANAGED_CLI_ROOT 下的 inventory。
  3. PATH 中的 codebuddy。
  4. 常见路径 ~/.local/bin/codebuddy、/usr/local/bin/codebuddy、/usr/bin/codebuddy。
  5. 按需 CLI 路径 OnDemandCLIPath("codebuddy")。
- 若均不可用，返回 assistantCLIMissingError，提示缺少 CodeBuddy 二进制。

```mermaid
flowchart TD
Start(["开始"]) --> CheckEnv["检查 AGENTFLOW_CODEBUDDY_BIN"]
CheckEnv --> |存在且可执行| UseEnv["使用环境变量路径"]
CheckEnv --> |不存在| CheckManaged["读取托管清单 AGENTFLOW_MANAGED_CLI_ROOT"]
CheckManaged --> |找到| UseManaged["使用托管二进制"]
CheckManaged --> |未找到| CheckPATH["PATH 查找 codebuddy"]
CheckPATH --> |找到| UsePATH["使用 PATH 中的二进制"]
CheckPATH --> |未找到| CheckCommon["探测常见安装路径"]
CheckCommon --> |找到| UseCommon["使用常见路径"]
CheckCommon --> |未找到| CheckOnDemand["查询 OnDemandCLIPath('codebuddy')"]
CheckOnDemand --> |找到| UseOnDemand["使用按需安装路径"]
CheckOnDemand --> |未找到| Error["返回缺失错误"]
```

图示来源
- [stream.go:190-232](file://internal/httpapi/stream.go#L190-L232)

章节来源
- [stream.go:190-232](file://internal/httpapi/stream.go#L190-L232)

### 运行时提供者与参数注入
- 工作区材料化：
  - 复用 hooks/codebuddy 钩子桥接目录，避免并行角色互相覆盖。
  - 生成 per-agent settings 文件，合并 Agent 自定义 hooks。
  - 创建相对符号链接，使 hooks 与 settings 指向 .agentflow 下受控位置。
- 参数处理：
  - 移除 resume 参数，统一通过 InvocationOptions.ResumeSessionID 传递。
  - 注入 --effort 参数（与 Claude Code 类似）。
  - 当存在 MCP 时，移除内置 --tools 白名单，避免隐藏 HTTP MCP 服务。
  - 强制 --strict-mcp-config，除非已显式传入。
  - 当启用 Skills 时，追加 --plugin-dir。
  - 最终以 -- 分隔选项与位置参数 prompt，防止被误解析为插件目录。
- 沙箱需求：
  - PrivateHome=true，BunFS=true，保证每个 CodeBuddy 实例拥有独立 Home 与文件系统视图。

```mermaid
classDiagram
class Provider {
+Kind() string
+Materialize(ctx) error
+Prepare(in) CommandSpec
+Protocol(ex, sink) protocolAdapter
+SandboxRequirements() SandboxRequirements
}
class CommandSpec {
+Argv []string
+Provider string
+CredentialBindings []CredentialBinding
+PrivateHome bool
+HookOutput HookOutput
+Telemetry any
}
Provider --> CommandSpec : "生成"
```

图示来源
- [provider_codebuddy.go:15-89](file://internal/runtime/provider_codebuddy.go#L15-L89)

章节来源
- [provider_codebuddy.go:15-89](file://internal/runtime/provider_codebuddy.go#L15-L89)
- [provider_test.go:203-280](file://internal/runtime/provider_test.go#L203-L280)

### 引擎认证与企业特性
- 凭据选择逻辑：
  - 若用户设置了个人 CodeBuddy API Key，则使用该 Key 及其网络环境。
  - 否则尝试平台共享 Key；若存在且 SponsorID 不为当前用户，则使用共享 Key，并标记 Sponsored。
  - 若 SponsorID 等于当前用户，视为个人 Key 场景，仅携带网络环境。
- 共享 Key 模型限制：
  - 通过 SharedCodeBuddyKey.Model(requested) 判断请求模型是否在允许列表内。
  - 不在允许列表时，回退到默认模型，并记录 model.sponsored 遥测事件。
  - 若运行尚未包含该事件，则在 Run 状态中追加提示信息，告知用户使用共享 Key 与受限模型。

```mermaid
flowchart TD
A["开始: 解析凭据"] --> B{"用户是否配置个人 Key?"}
B --> |是| C["返回个人 Key + 网络环境"]
B --> |否| D{"是否存在平台共享 Key?"}
D --> |否| E["返回空凭据"]
D --> |是| F{"SponsorID == 当前用户?"}
F --> |是| G["返回仅含网络环境的凭据"]
F --> |否| H["返回共享 Key + 允许模型 + Sponsored=true"]
```

图示来源
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [codebuddy_sponsor.go:33-60](file://internal/repo/codebuddy_sponsor.go#L33-L60)

章节来源
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [codebuddy_sponsor.go:33-60](file://internal/repo/codebuddy_sponsor.go#L33-L60)

### Git 与 OAuth 集成
- 工蜂 OAuth：
  - 支持 AuthorizeURL 构造、Exchange 授权码换令牌、Refresh 刷新令牌。
  - Username 接口用于从 access_token 获取登录名。
  - 所有外部调用使用带超时的 http.Client，响应体限制大小，错误信息脱敏。
- Git 凭据重写：
  - 支持 oauth2:token 前缀，将用户名规范化为 oauth2，密码为 token。
  - 对于 HTTPS 远程地址，将 git@host:path 改写为 https://oauth2:token@host/path。
  - 对于非 github.com 的 SSH 主机，需要配置 known_hosts 与专用 SSH 密钥。

```mermaid
sequenceDiagram
participant User as "用户"
participant UI as "LoopForge 前端"
participant OAuth as "工蜂 OAuth"
participant Git as "Git 操作"
User->>UI : 连接仓库/授权
UI->>OAuth : 打开授权页面(AuthorizeURL)
OAuth-->>UI : 回调授权码(code)
UI->>OAuth : Exchange(code) -> access_token
OAuth-->>UI : 返回 access_token/refresh_token
UI->>Git : 使用 oauth2 : access_token 访问远程仓库
Git-->>UI : 成功克隆/推送
```

图示来源
- [oauth.go:88-115](file://internal/gongfeng/oauth.go#L88-L115)
- [auth.go:23-38](file://internal/gitwork/auth.go#L23-L38)

章节来源
- [oauth.go:88-115](file://internal/gongfeng/oauth.go#L88-L115)
- [auth.go:23-38](file://internal/gitwork/auth.go#L23-L38)

### --effort 参数支持与行为
- 支持的 effort 级别由 ExecutorEffortLevels 定义，codebuddy 与 claude-code 均支持 --effort。
- 运行时会为 CodeBuddy 注入 --effort 参数，同时确保不会重复注入。
- 测试验证了 effort 参数注入位置与去重行为，以及与 resume 参数的相对顺序。

```mermaid
flowchart TD
Start(["构建命令"]) --> AddEffort["注入 --effort (若未显式提供)"]
AddEffort --> Dedup["去重已有 --effort"]
Dedup --> Place["确保位于 resume 之前"]
Place --> End(["完成"])
```

图示来源
- [provider_test.go:203-280](file://internal/runtime/provider_test.go#L203-L280)
- [runtime_test.go:1987-1993](file://internal/runtime/runtime_test.go#L1987-L1993)
- [agent.go:10-36](file://internal/domain/agent.go#L10-L36)

章节来源
- [provider_test.go:203-280](file://internal/runtime/provider_test.go#L203-L280)
- [runtime_test.go:1987-1993](file://internal/runtime/runtime_test.go#L1987-L1993)
- [agent.go:10-36](file://internal/domain/agent.go#L10-L36)

### 工作区权限管理与 MCP 安全
- 权限隔离：
  - 通过 isolateCodeBuddyFlagsWithSource 移除 --setting-sources、--permission-mode、--plugin-dir，并强制 -y 与 bypassPermissions，避免无交互模式下卡住。
  - 强制 --strict-mcp-config，确保 MCP 配置严格生效。
- 工具白名单：
  - 当存在 MCP 时，移除内置 --tools 白名单，避免隐藏 HTTP MCP 服务器。
- 会话持久化：
  - 审阅模式禁用会话持久化，报告模式限制最大轮次与工具集。

章节来源
- [provider_codebuddy.go:54-89](file://internal/runtime/provider_codebuddy.go#L54-L89)
- [assistant.go:1757-1774](file://internal/httpapi/assistant.go#L1757-L1774)

## 依赖关系分析
- HTTP API 依赖运行时提供者来准备命令与凭据绑定。
- 运行时提供者依赖引擎认证来决定使用个人 Key 还是共享 Key。
- Git 操作依赖 OAuth 与 SSH 密钥管理，分别用于 HTTPS 与 SSH 传输。
- 部署文档定义了环境变量与托管 CLI 清单的使用方式。

```mermaid
graph LR
API["HTTP API"] --> Runtime["运行时提供者"]
Runtime --> Engine["引擎认证"]
Runtime --> Repo["Git/OAuth/SSH"]
Deployment["部署文档"] --> API
Deployment --> Runtime
```

图示来源
- [assistant.go:1730-1814](file://internal/httpapi/assistant.go#L1730-L1814)
- [provider_codebuddy.go:54-89](file://internal/runtime/provider_codebuddy.go#L54-L89)
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [deployment.md:7-18](file://docs/deployment.md#L7-L18)

章节来源
- [assistant.go:1730-1814](file://internal/httpapi/assistant.go#L1730-L1814)
- [provider_codebuddy.go:54-89](file://internal/runtime/provider_codebuddy.go#L54-L89)
- [codebuddy_auth.go:17-65](file://internal/engine/codebuddy_auth.go#L17-L65)
- [deployment.md:7-18](file://docs/deployment.md#L7-L18)

## 性能与行为特性
- 二进制定位失败会快速返回错误，避免后续无效等待。
- 工作区钩子与 settings 采用共享目录与 per-agent 文件合并，减少并行冲突。
- --effort 参数注入经过去重与位置校验，避免重复与顺序错误。
- 共享 Key 模型限制通过 Model() 方法 O(n) 匹配，n 为允许模型数量，上限为 32。
- MCP 严格模式与工具白名单调整可降低不必要的工具暴露风险。

[本节为通用性能讨论，不直接分析具体文件]

## 故障排查指南
- CodeBuddy 二进制缺失
  - 现象：返回 assistantCLIMissingError，提示缺少 codebuddy。
  - 排查：检查 AGENTFLOW_CODEBUDDY_BIN、AGENTFLOW_MANAGED_CLI_ROOT、PATH 与常见安装路径。
- 托管 CLI 未安装
  - 现象：读取托管清单失败或未找到 codebuddy。
  - 排查：确认 AGENTFLOW_MANAGED_CLI_ROOT 指向有效清单目录，worker 已安装 CLI。
- 共享 Key 模型不被允许
  - 现象：运行提示使用共享 Key，角色配置的模型不生效。
  - 排查：在平台管理中配置允许的模型列表，并确保请求模型在白名单内。
- Git SSH 密钥缺失
  - 现象：仓库拉取失败，提示需要 SSH 密钥。
  - 排查：在个人设置中添加对应主机的 SSH 私钥，并配置 known_hosts。
- OAuth 令牌问题
  - 现象：无法克隆/推送，或用户信息解析失败。
  - 排查：检查工蜂 OAuth 配置、授权范围、回调地址与令牌有效期。

章节来源
- [stream.go:190-232](file://internal/httpapi/stream.go#L190-L232)
- [codebuddy_auth.go:42-65](file://internal/engine/codebuddy_auth.go#L42-L65)
- [deployment.md:97-107](file://docs/deployment.md#L97-L107)
- [oauth.go:158-195](file://internal/gongfeng/oauth.go#L158-L195)

## 结论
LoopForge 对 CodeBuddy 的执行器集成提供了完整的企业级能力：
- 通过运行时提供者实现工作区隔离、MCP 安全与 effort 参数注入。
- 通过引擎认证实现个人 Key 与平台共享 Key 的自动切换与模型白名单控制。
- 通过 Git OAuth 与 SSH 密钥管理实现安全的仓库访问与协作。
- 通过部署文档与环境变量支持托管 CLI、网络环境与镜像源。
在企业环境中，建议优先使用托管 CLI 清单与平台共享 Key，结合严格的 MCP 配置与 Git SSH 密钥管理，确保稳定、可控与可审计的执行体验。

[本节为总结性内容，不直接分析具体文件]

## 附录：企业部署与最佳实践
- 二进制与托管清单
  - 使用 AGENTFLOW_CODEBUDDY_BIN 指定二进制，或在 AGENTFLOW_MANAGED_CLI_ROOT 下维护托管清单。
  - 生产镜像已包含 CodeBuddy；其他 CLI 按需安装。
- CodeBuddy 网络环境
  - 在个人设置中选择与 Key 匹配的网络环境：国际站、国内站 internal、iOA ioa。
  - 平台共享 Key 需配置网络环境与允许模型列表。
- 共享 Key 管理
  - 平台管理员在“平台共享 CodeBuddy Key”中配置 API Key、网络环境与模型白名单。
  - 未配置个人 Key 的用户将使用共享 Key，并被限制在允许模型内。
- Git 访问
  - 推荐使用 Git SSH 访问私有仓库，并为每个主机配置专用 SSH 密钥与 known_hosts。
  - 使用工蜂 OAuth 获取仓库访问令牌，支持授权码与刷新令牌。
- 安全与隔离
  - 启用 --strict-mcp-config，避免 MCP 配置被绕过。
  - 使用 PrivateHome 与 BunFS 沙箱，确保每个 CodeBuddy 实例隔离。
  - 审阅模式与报告模式限制工具集与轮次，降低风险。

章节来源
- [deployment.md:7-18](file://docs/deployment.md#L7-L18)
- [deployment.md:186-198](file://docs/deployment.md#L186-L198)
- [platform_settings.go:198-272](file://internal/httpapi/platform_settings.go#L198-L272)
- [codebuddy_sponsor.go:135-179](file://internal/repo/codebuddy_sponsor.go#L135-L179)
- [provider_codebuddy.go:54-89](file://internal/runtime/provider_codebuddy.go#L54-L89)