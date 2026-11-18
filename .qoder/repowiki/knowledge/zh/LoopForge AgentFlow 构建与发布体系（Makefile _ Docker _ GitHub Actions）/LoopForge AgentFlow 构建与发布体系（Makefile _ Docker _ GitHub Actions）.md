---
kind: build_system
name: LoopForge AgentFlow 构建与发布体系（Makefile / Docker / GitHub Actions）
category: build_system
scope:
    - '**'
source_files:
    - Makefile
    - Dockerfile
    - scripts/build-binaries.sh
    - scripts/build-chatbot.sh
    - scripts/image-commit.sh
    - .github/workflows/platform.yml
    - .github/workflows/release.yml
    - compose.yaml
    - start.sh
---

## 1. 使用的工具与整体方法

- Go 后端：`go.mod` 指定 `go >= 1.26.11`，通过 `CGO_ENABLED=0` 纯 Go 静态编译；二进制由 `scripts/build-binaries.sh` 统一驱动，支持按 CPU/内存自动估算并行度。
- Web 前端（平台控制台）：Go embed 嵌入 `web/app/`、`web/vscode-web/`、`web/mcp-icons/` 等静态资源，无独立打包步骤。
- 原生聊天 UI（chatbot）：React + Vite，位于 `chatbot/`，构建产物输出到 `web/native-chat/`，被 Go 服务端 embed。
- 容器镜像：单 `Dockerfile`，使用多阶段构建（Node 22 构建 chatbot → golang:1.26.11 交叉编译 Go → node:22 运行期），并通过 QEMU 实现 `linux/amd64, linux/arm64` 双架构发布。
- CI/CD：GitHub Actions `.github/workflows/` 下四个工作流——`platform.yml`（PR/main 上的测试与覆盖率门禁）、`release.yml`（tag 触发镜像发布与 GitHub Release）、`e2e.yml`、`validate.yml`。
- 版本与发行：遵循 `vMAJOR.MINOR.PATCH[-PRERELEASE]` 语义化标签；`compose.yaml` 中必须预写 `loopforge:${VERSION}` 才能打 tag。

## 2. 关键文件

- `Makefile`：本地开发入口（`run` / `build` / `test` / `coverage` / `test-mysql` / `build-chatbot` / `build-gate-mcp`）。
- `Dockerfile`：多阶段构建、Python standalone 内嵌、依赖固定版本（codex/codebuddy/dsh）。
- `scripts/build-binaries.sh`：Go 二进制构建核心，负责 `--helpers` 与 `--platform` 两阶段、四目标 OS/ARCH 交叉编译。
- `scripts/build-chatbot.sh`：Vite 构建 + Vitest 测试并行执行，要求 Node ≥ 22。
- `scripts/image-commit.sh`：从 `LOOPFORGE_COMMIT` 或 `.git/HEAD` 解析 commit SHA 写入 `/app/loopforge-commit`。
- `.github/workflows/platform.yml`：单元测试、真实浏览器测试、覆盖率门禁、route coverage ≥ 85%、govulncheck、gitleaks。
- `.github/workflows/release.yml`：tag 校验、compose.yaml 预检、QEMU + buildx 双架构镜像推送、digest 冒烟测试、gh release 创建。
- `compose.yaml` / `compose.dev.yaml`：服务编排与镜像 pin。
- `start.sh`：安装器/启动器（CI 通过它拉起服务做冒烟）。

## 3. 架构与约定

### 3.1 构建流水线分层

1. **chatbot 阶段**（`Dockerfile` 第 6–12 行）：在 `node:22-bookworm-slim` 上 `npm ci --include=dev` 并 `npm run build`，产出 `web/native-chat/index.html`。此阶段与 Go 编译解耦，可先于 Go 阶段完成。
2. **helpers 阶段**（`Dockerfile` 第 27 行）：`GOOS=${TARGETOS} GOARCH=${TARGETARCH} bash scripts/build-binaries.sh --helpers /out`，编译不 embed web UI 的辅助二进制（`agentflow-mcp`、`loopforge-account`、`agentflow-runner`、`agentflow-gate-mcp`），可在 chatbot 构建的同时并行进行。
3. **platform 阶段**（`Dockerfile` 第 29 行）：复制 chatbot 产物后，再次调用 `build-binaries.sh --platform /out` 构建主服务 `agentflow`，此时 `web/native-chat/index.html` 已就绪。
4. **运行期阶段**（`Dockerfile` 第 30–67 行）：基于 `node:22`，安装 pinned 的 CLI（`@openai/codex@0.162.1`、`@tencent-ai/codebuddy-code@2.167.0`、`@deepseek-ai/dsh@0.2.2-rc.1`），下载并校验 SHA256 的 Python standalone 二进制，拷贝 Go 二进制与 LICENSE/NOTICES，暴露 8080 端口，ENTRYPOINT 为 `/app/bin/agentflow`。

### 3.2 交叉编译策略

- `scripts/build-binaries.sh` 对 `agentflow-runner` 和 `agentflow-gate-mcp` 显式遍历 `linux amd64 arm64` × `darwin amd64 arm64` 四种组合，命名形如 `<binary>-<os>-<arch>`。
- `Makefile` 的 `build-gate-mcp` 目标同样以 `CGO_ENABLED=0` 分别构建 darwin-amd64/darwin-arm64/linux-amd64/linux-arm64。
- 所有 Go 构建均带 `-trimpath -ldflags=-s -w`，禁用 CGO，确保可移植性。

### 3.3 并发构建自适应

`build_jobs()` 函数读取 `nproc`、`/proc/meminfo` 与 cgroup `memory.max`，取较小值作为并行度上限，默认至少 1、最多等于 CPU 数；可通过 `LOOPFORGE_BUILD_JOBS` 覆盖。注释说明冷构建峰值约 1.3 GiB，4 个 job 时不会 OOM。

### 3.4 版本与提交信息注入

- `image-commit.sh` 优先使用 `LOOPFORGE_COMMIT` 环境变量，否则解析 `.git/HEAD`（支持 ref 与 packed-refs），最终写入 `/app/loopforge-commit`。
- `Dockerfile` 通过 `ARG LOOPFORGE_COMMIT=` 传入，并在构建时调用 `scripts/image-commit.sh /ctx /app/loopforge-commit`。
- `release.yml` 将 `LOOPFORGE_COMMIT=${steps.release.outputs.commit}` 作为 build arg 传给 `docker/build-push-action`，保证镜像内的 commit 与发布 tag 一致。

### 3.5 依赖锁定

- Go：`go.mod` + `go.sum`，CI 通过 `actions/setup-go@v6` 的 `go-version-file` 自动选择 1.26.11。
- Node：`chatbot/package-lock.json`、`webtest/package-lock.json`、`e2e/package-lock.json`，CI 使用 `npm ci`。
- 运行时 CLI：`Dockerfile` 用 `npm install -g @openai/codex@0.162.1 @tencent-ai/codebuddy-code@2.167.0 @deepseek-ai/dsh@0.2.2-rc.1` 精确 pin。
- Python：通过 `astral-sh/python-build-standalone` 按 `TARGETARCH` 下载对应 SHA256 的 tarball，严格校验。

## 4. 约定与约束（规则级）

- **Go 版本约束**：`go.mod` 声明 `go >= 1.26.11`，`Dockerfile` 使用 `golang:1.26.11-bookworm`，CI 通过 `go-version-file` 拉取相同版本。（来源：`go.mod`、`Dockerfile` 第 13 行、`.github/workflows/platform.yml` 第 18 行）
- **Chatbot 需要 Node ≥ 22**：`scripts/build-chatbot.sh` 在启动时检查 `process.versions.node.split(".")[0] < 22` 即退出。（来源：`scripts/build-chatbot.sh` 第 6 行）
- **覆盖率门禁**：Go 语句覆盖率最低阈值来自 `scripts/coverage-go.min`，CI 通过 `bash scripts/coverage-go.sh --min "$min"` 强制；HTTP route 覆盖率最低 85%（`e2e/coverage/route-coverage.mjs --min 85`）。（来源：`.github/workflows/platform.yml` 第 45–49、163 行）
- **MySQL 集成测试隔离**：`make test-mysql` 仅使用 `127.0.0.1:33306` 的临时 MySQL 8.0 容器，DSN 为 `root:root@tcp(127.0.0.1:$(MYSQL_PORT))/agentflow_it`，结束后 `docker rm -f` 清理，绝不触碰外部数据库。（来源：`Makefile` 第 34–55 行）
- **Release tag 格式与前置条件**：必须是 `^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$`；checkout commit 必须与 tag commit 一致且是 main 的祖先；`platform.yml` 在该 commit 上必须成功；`compose.yaml` 必须恰好两处 pin `loopforge:${VERSION}`（platform 镜像与 `AGENTFLOW_ISOLATED_IMAGE`）。（来源：`.github/workflows/release.yml` 第 39–63 行）
- **镜像发布平台**：`docker/build-push-action` 固定 `platforms: linux/amd64,linux/arm64`，通过 `docker/setup-qemu-action@v4` 启用 arm64 emulation。（来源：`.github/workflows/release.yml` 第 78–105 行）
- **发布镜像冒烟测试**：发布后使用 `./start.sh -d` 拉起镜像并通过 `curl -fsS http://127.0.0.1:8090/api/health` 验证健康端点。（来源：`.github/workflows/release.yml` 第 114–119 行）
- **构建缓存**：Go 模块缓存挂载到 `/go/pkg/mod`，Go build cache 挂载到 `/root/.cache/go-build`，Docker BuildKit cache 通过 `type=gha` 复用。（来源：`Dockerfile` 第 21、27、29 行；`.github/workflows/release.yml` 第 111–112 行）
- **安全扫描门禁**：CI 同时运行 `go vet ./...`、`govulncheck`、`scripts/test-scan-secrets.py`、`scripts/scan-secrets.sh`（gitleaks）。（来源：`.github/workflows/platform.yml` 第 59–67 行）
- **Go 构建标志**：所有二进制均使用 `CGO_ENABLED=0 -p=<jobs> -trimpath -ldflags="-s -w"`，确保静态链接、strip 符号表与调试信息。（来源：`scripts/build-binaries.sh` 第 39–40 行）
- **Makefile 的 GOCACHE**：所有 go 命令统一设置 `GOCACHE=$(GOCACHE)`，默认 `/tmp/agentflow-gocache`，便于跨任务共享缓存。（来源：`Makefile` 第 3、7、10、19、50、59–62 行）

## 5. 总结

该仓库采用“Go 后端 + Vite 前端 + Docker 多阶段镜像”的混合构建模型，通过 `Makefile` 提供本地开发体验，通过 `scripts/build-binaries.sh` 统一 Go 交叉编译，通过 `Dockerfile` 将 chatbot 构建、Go 编译与运行期环境组装成单一镜像，并由 GitHub Actions 在 PR/main 上执行单元测试、真实浏览器测试、覆盖率门禁与安全扫描，在 `v*.*.*` tag 上触发双架构镜像发布与 GitHub Release。版本管理完全围绕语义化 tag 与 `compose.yaml` 中的镜像 pin 展开，形成严格的发布契约。