# Qoder 适配器

仅在 Qoder Desktop 当前会话实际提供 `Agent` 工具时使用。自定义 Agent 不继承主会话历史，也不能继续派发子 Agent，因此协调者必须提供完整阶段提示并负责所有 helper 派发。

## 安装

Qoder 没有 Classic 宿主包，只支持 Portable Edition：

```bash
loopforge skills install qoder
```

安装结果写入项目的 `.qoder/skills/` 与 `.qoder/agents/`。需要跨项目使用时，把同一份 `devflow`、`devflow-clarify-requirements` 复制到用户资源目录 `~/.qoder/skills/`，并把两个执行器 Agent 放进 `~/.qoder/agents/`。

安装或修改 Agent 定义后需新开 Qoder 会话，Skill 与 Agent 类型列表才会刷新。宿主脚本调用使用 `<PYTHON>`，取法见 [references/workflow-contract.md](../../references/workflow-contract.md)。

## 执行步骤

1. medium/large 使用 `--execution-mode isolated --host-adapter qoder` 初始化。
2. REQUIREMENT 执行 `prepare --emit-prompt → start`，由主 Agent 调用 `devflow-clarify-requirements` 技能并填写需求报告，再 `finish`；禁止启动需求分析子 Agent。
3. route 中其他 Agent 阶段用 `prepare --emit-prompt` 生成提示后，以 `Agent` 工具选择 `devflow-stage-executor` 创建新的子 Agent，把完整阶段提示作为该 Agent 的任务内容传入。
4. 用宿主在创建 Agent 时返回的真实 ID 登记执行者：`assign --role <逻辑角色> --executor-type devflow-stage-executor --executor-id <宿主返回的 ID> --dispatch-tool Agent`。只有宿主真实创建了 Agent 之后才能 `assign`，不得手写、猜测或跨阶段复用 ID。
5. 只读调研选择 `devflow-research-helper`，以 `helper --executor-type devflow-research-helper --dispatch-tool Agent` 登记。SUMMARY 由主 Agent 执行，不创建子 Agent。

不同阶段不得复用同一个 Agent 实例。不得依赖自动 delegation 猜测角色，也不得让阶段子 Agent 嵌套创建 helper。
