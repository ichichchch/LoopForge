---
kind: frontend_style
name: AgentFlow 前端样式体系：双设计系统 + CSS 变量主题
category: frontend_style
scope:
    - '**'
source_files:
    - web/styles.css
    - web/product-v3.css
    - web/product-v4.css
    - web/boot-theme.js
    - chatbot/src/theme.js
    - chatbot/src/styles.css
    - chatbot/src/agent-browser.css
    - chatbot/src/hui-tokens.css
    - chatbot/src/model-provider-icon.css
    - web/workbench.css
    - web/platform.css
    - web/sops.css
    - web/assets.css
    - web/observability.css
    - web/chat-codex.css
---

## 1. 总体方案

仓库包含两个独立的前端应用，各自维护一套原生 CSS 样式体系，没有使用 Tailwind、Sass、CSS Modules 或任何 UI 组件库。

- **web/** — AgentFlow 主 Web SPA（Go embed 嵌入），采用「AgentFlow 设计系统 v2」与「Product UI v3」两套 CSS 主题层。
- **chatbot/** — 基于 React + Vite 的原生聊天前端，使用纯 CSS + CSS 自定义属性（`--var`）实现亮/暗主题。

两者均通过 `data-theme="light|dark"` 切换主题，不依赖任何 CSS-in-JS 框架。样式组织遵循「全局基础层 + 页面/组件级 CSS 文件」的扁平结构，每个 `.jsx` 组件可附带同名 `.css` 文件（如 `agent-browser.jsx` + `agent-browser.css`）。

## 2. 关键文件

- `web/styles.css` — 设计系统 v2 基础层：品牌色、几何令牌、按钮/表单/卡片/状态点/模态框/toast 等通用原子类。
- `web/product-v3.css` — Product UI v3 主题层：将 `--v3-*` 变量映射到 `--bg/--panel/--text/--primary` 等共享语义变量，定义 app-shell、sidebar、topbar、stat-card、flow-workbench、modal 等页面骨架。
- `web/product-v4.css` — 第四代产品主题（存在但未在本文展开）。
- `web/boot-theme.js` — 首屏同步注入 `document.documentElement.dataset.theme`，避免 CSP 下内联脚本问题，并兼容旧 key `loopforge.theme.v3` / `agentflow.native-chat.theme`。
- `chatbot/src/theme.js` — 聊天前端的主题偏好读写（key: `agentflow.theme.v3`），支持 `light/dark/system` 三态及 legacy key 迁移。
- `chatbot/src/styles.css` — 聊天应用的基础布局与 Markdown/code-block/message 等样式。
- `chatbot/src/*.css` — 各组件级样式（`agent-browser.css`、`artifact-panel.css`、`message-info.css`、`model-picker.css`、`home.css`、`hui-tokens.css`、`harness-icons.css` 等）。
- `web/workbench.css`、`web/platform.css`、`web/sops.css`、`web/assets.css`、`web/observability.css`、`web/chat-codex.css`、`web/bots.css`、`web/preferences.css`、`web/sync.css`、`web/artifacts.css`、`web/assistant_models.css`、`web/vscode.css` — 按功能域划分的页面级样式。

## 3. 架构与约定

### 3.1 设计令牌（Design Tokens）

两层 token 体系并存：

- **v2 基础层** (`web/styles.css`)：定义 `--blue/--green/--amber/--red/--purple/--cyan` 语义色，`--bg/--bg-soft/--panel/--panel-2/--panel-3` 背景层级，`--line/--line-strong` 分割线，`--text/--text-strong/--muted/--muted-2` 文字层级，`--radius/--radius-sm/--radius-xs/--radius-pill` 圆角，`--sp-1..7` 间距阶梯，`--fs-xs..fs-2xl` 字号阶梯，`--ease/--ease-spring` 动效曲线，`--accent-gradient` 品牌渐变。
- **v3 产品层** (`web/product-v3.css`)：以 `--v3-bg/--v3-surface/--v3-ink/--v3-accent/--v3-positive/--v3-warning/--v3-danger/--v3-info` 为源，再映射回 `--bg/--panel/--text/--primary/--success/--warning/--danger` 等共享名，使 v2 基础类可直接复用。

所有颜色、尺寸、字体、阴影、圆角都通过 CSS 自定义属性消费，而非硬编码值。

### 3.2 主题切换机制

- 根元素通过 `html[data-theme="light"]` 或 `html[data-theme="dark"]` 选择器覆盖变量。
- `boot-theme.js` 在 `<head>` 中同步执行，从 `localStorage` 读取 `agentflow.theme.v3`（优先）、`agentflow.native-chat.theme`（兼容）、`loopforge.theme.v3`（更旧），若值为 `system` 则回退到 `prefers-color-scheme`。
- chatbot 侧的 `theme.js` 提供 `readThemePreference` / `resolveTheme` / `writeThemePreference` API，同样处理 legacy key 迁移。
- 部分组件级 CSS 使用 `[data-theme="dark"]` 覆盖特定元素（如 `chatbot/src/model-provider-icon.css` 中根据主题切换 SVG 图标）。

### 3.3 响应式策略

- 主要使用 CSS Grid 的 `minmax(0, 1fr)` + 固定侧栏宽度（如 `app-shell{grid-template-columns:248px minmax(0,1fr)}`）实现自适应。
- 断点集中在 `web/product-v3.css` 末尾：`@media(max-width:1280px)` 缩小侧栏至 218px，`@media(max-width:960px)` 折叠为 72px 图标模式，并让 properties 面板变为 fixed 浮层。
- chatbot 使用 `--sidebar-width:275px; --panel-width:480px; --chat-column-width:832px` 等 CSS 变量控制列宽，配合 `@media(hover:none)` 适配触屏。

### 3.4 组件样式命名

- 使用 BEM 风格但无预处理器：`.sh-card`、`.stat-card`、`.button.primary`、`.conversation-project`、`.message.user`、`.run-list-item.active` 等，通过修饰类表达变体。
- 状态通过 data 属性驱动：`[data-conversation-status="running"]`、`[data-theme="dark"]`、`[data-status="loaded"]` 等。
- 动画统一走 `transition: all var(--ease)` 或 `cubic-bezier(.4,0,.2,1)`，并通过 `@media(prefers-reduced-motion:reduce)` 关闭。

### 3.5 构建与集成

- web/ 由 Go 后端通过 `embed.go` 嵌入 HTML/JS/CSS，无需 Node 构建步骤。
- chatbot/ 使用 Vite 6 + React 19，`package.json` 中仅引入 `dompurify`、`mermaid`、`pptxgenjs`、`react-markdown`、`remark-gfm` 作为运行时依赖；样式全部手写，无第三方 UI 库。

## 4. 约定与约束

- **主题键名**：新代码应使用 `agentflow.theme.v3` 作为 localStorage key（见 `chatbot/src/theme.js` 的 `THEME_KEY`），旧 key 会由 `boot-theme.js` 和 `theme.js` 自动迁移。
- **主题取值**：只允许 `light`、`dark`、`system` 三个值（`chatbot/src/theme.js` 的 `THEME_PREFERENCES` 校验）。
- **主题选择器**：样式必须通过 `html[data-theme="light"]` / `html[data-theme="dark"]` 覆盖变量，不应硬编码颜色值。
- **设计令牌**：新增视觉常量应加入 `web/styles.css` 的 `:root` 或 `web/product-v3.css` 的 `--v3-*` 命名空间，并在两处保持语义一致。
- **响应式**：优先使用 CSS Grid + `minmax` + 断点，不使用 JS 计算布局。
- **无障碍**：焦点可见性使用 `:focus-visible` + 蓝色描边；动画需尊重 `prefers-reduced-motion: reduce`。
- **CSP 限制**：首屏主题脚本必须以独立文件加载（`boot-theme.js` 注释明确说明 inline script 被 CSP 禁止）。