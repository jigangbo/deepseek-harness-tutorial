# 2.7 工具执行器 — native 与 PTC 双执行器

> 同一套工具注册表、同一套守卫管线，两种**模型可见形态**：native（模型逐个发工具调用）与 PTC（模型写一段程序批量编排）。选择哪一个，是一个作用域级的组合决定。核心源码：`packages/core/tool-executor-native/`、`packages/core/tool-executor-ptc/`。

## 选择模型

`ctx.tools.setExecutor(executor)` 是唯一的执行器选择入口（`ToolsService` 的 resolve 逻辑见核心引擎篇 tools 章）。两个执行器插件挂载后各自调用它：

```ts
// tool-executor-native/src/index.ts
export function apply(ctx: Context): void {
  ctx.tools.setExecutor(new NativeToolExecutor(ctx))
}

// tool-executor-ptc/src/index.ts
export const inject = ['tools', 'systemPrompt', 'ptcRuntime']  // PTC 需要 runtime
export function apply(ctx: Context, config: Config): void {
  ctx.tools.setExecutor(new PtcExecutor(ctx, config))
}
```

作用域语义：执行器跟随挂载它的 scope——部署默认、preset 作用域或单个 agent，都可以不同（这正是 `dsh-scope` 的用武之地）。

```text
┌──────────────────────────────────────────────────────────┐
│                 ctx.tools（注册表 + 守卫管线）            │
│   setExecutor(NativeToolExecutor)  ←── native 插件        │
│   setExecutor(PtcExecutor)          ←── ptc 插件          │
└───────────────┬──────────────────────────────────────────┘
                │ describe(tools) 决定模型看见什么
    ┌───────────▼───────────┐        ┌──────────────────────┐
    │ native:               │        │ ptc:                 │
    │ 全部可见 schema        │        │ 只有 run_code schema │
    │ → 模型逐个 tool/call  │        │ + 生成的 SDK 段      │
    └───────────────────────┘        └──────────────────────┘
```

两者共用同一个 `ToolExecutionPipeline` 与 `executeToolBatch`（prepare/dispatch/complete 三阶段、审批、guard、并发上限默认 10）——**策略层零分叉**。

## 对比速查

| | Native（`native-tool-executor`） | PTC（`ptc-tool-executor`） |
|---|---|---|
| 模型看到 | 每个可见工具的 schema | 仅 `run_code` + SDK 文档段 |
| 直接可调 | 任何可见工具 | 只有 `run_code`；其他工具名直接调会收到 `ToolNotFoundError` |
| 工具调用路径 | tool/call → 管线 | 程序内嵌套调用 → 同一管线（重新进入守卫） |
| 运行时依赖 | 无 | `ctx.ptcRuntime`（沙箱化 Node） |
| 提示词段 | （schema 即说明） | `tools:ptc-only` + `tools:sdk` 两段，order 取自 systemPrompt 的 section order |

## 历史包袱：agent-tool-presentation

`packages/core/agent-tool-presentation` 是**已弃用**的兼容插件（`mode: 'native' | 'ptc'`），`apply` 只是转挂对应执行器包。新组合应直接挂 `dsh-tool-executor-native` 或 `dsh-tool-executor-ptc`——见下一节两篇逐文件精读与 `agent-tool-presentation/index.md`。
