---
kind: dependency_management
name: 多语言依赖管理：Go modules + npm + pip + Dependabot
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
    - .github/dependabot.yml
    - Makefile
    - scripts/build-chatbot.sh
    - chatbot/package.json
    - chatbot/package-lock.json
    - e2e/package.json
    - webtest/package.json
    - local/pyproject.toml
---

## 1. 使用的系统与方法

仓库是一个多语言聚合项目，采用三种包管理器分别管理不同子系统的依赖，并通过 GitHub Dependabot 统一进行版本更新：

- **Go 后端**：使用 Go Modules（`go.mod` + `go.sum`），模块名为 `agentflow`，要求 Go 1.26.11。依赖声明在根级 `go.mod` 的 `require` 块中，直接依赖包括 gin、gorm、sqlite、mysql driver、websocket、compress、crypto/net/sys 等。
- **Node.js 前端/测试**：使用 npm（`package.json` + `package-lock.json`），分布在 `chatbot/`（React/Vite 原生聊天 UI）、`e2e/`（Playwright 端到端测试）、`webtest/`（jsdom 前端单元测试）三个独立 npm 工作区，每个都有独立的 `package-lock.json`。
- **Python CLI**：使用 setuptools / `pyproject.toml`，位于 `local/`，发布为 `devflow-workflow` 包，`requires-python = ">=3.8"`，构建后端根据 Python 版本条件选择 setuptools 75.3.4 或 84.2.1。
- **自动化更新**：通过 `.github/dependabot.yml` 对 gomod、npm、pip、github-actions 四类生态配置自动拉取 PR。

没有使用私有 registry、GOPRIVATE、GONOSUMCHECK/GONOSUMDB、vendor 目录或 GOPROXY 镜像；所有依赖均从公共源解析。