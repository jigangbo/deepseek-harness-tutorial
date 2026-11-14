# `index.ts`

> PTC 执行器：程序化工具调用


> 源文件：`packages/core/tool-executor-ptc/src/index.ts`（114 行）。PTC 模式下模型只看见一个 `run_code` 工具 + 一段生成的 SDK 文档；其余工具全部经程序内嵌套调用进入同一条守卫管线。

## 文件概览

`PtcExecutor` 与 native 实现同一 `ToolExecutor` 接口，但做了三件本质不同的事：

1. **描述面收窄**：`describe` 只投影 `run_code` 的 schema（`knownNames: [RUN_CODE_NAME]`）；
2. **提示词注入两段**：`tools:ptc-only`（"只有 run_code 可直调"的指令）与 `tools:sdk`（按运行时语言渲染的工具 SDK 文档）；
3. **管线接管解析**：用 `get`/`reject` 覆写让嵌套调用走同一管线，而外层直调非 `run_code` 名称会被拒绝。

## 关键代码走读

### 指令与提示词段

```ts
const PTC_ONLY_INSTRUCTION = `\`${RUN_CODE_NAME}\` is the only tool you can call directly — a tool call naming any other tool fails. Reach every tool the SDK declares below from inside the program.`

this.sections = [
  { name: 'tools:ptc-only', order: ctx.systemPrompt.getSectionOrder('PTC_ONLY'), text: PTC_ONLY_INSTRUCTION, interpolate: false },
  { name: 'tools:sdk', order: ctx.systemPrompt.getSectionOrder('TOOLS_SDK'), text: tools => this.renderSdk(tools), interpolate: false },
]
```

段的排序值从 `ctx.systemPrompt.getSectionOrder('PTC_ONLY' / 'TOOLS_SDK')` 读取——**顺序常量归 system-prompt 拥有**，执行器只声明名字与内容（见 system-prompt 章）。

```ts
private renderSdk(tools: readonly ToolDefinition[]): string {
  const runtime = requirePtcRuntime(this.ctx)
  const render = SDK_RENDERERS[runtime.language]   // typescript | python
  return render(tools.map(definition => ({
    ...projectToolSchema(definition, false),
    output: definition.output.schema,
  })), runtime.programFormat)
}
```

SDK 文档按 **`ctx.ptcRuntime.language`** 选择渲染器（TS 或 Python），并把每个工具的 `output.schema` 一并声明——程序里的类型化返回契约（能力缝篇 PTC 章）由此生成。

### 管线覆写：一个入口，两种调用

```ts
this.pipeline = new ToolExecutionPipeline(ctx, {
  get: (name, scope, toolSet) => name === RUN_CODE_NAME
    ? this.requirePtcTransport()
    : tools.executable(name, scope, toolSet),
  reject: (name, nested) => !nested && name !== RUN_CODE_NAME
    ? new ToolNotFoundError(name, `only \`${RUN_CODE_NAME}\` is callable directly — call \`${name}\` from inside a \`${RUN_CODE_NAME}\` program instead`)
    : undefined,
})
```

- **`get`**：解析名字为可执行对象——`run_code` 命中保留的传输工具定义；其他名字走注册表的常规可执行解析（**嵌套调用因此与外层调用同权**：审批、guard、超时全适用）。
- **`reject`**：只拦"外层直调非 run_code"（`nested` 为 false 时）；程序内部的嵌套调用放行。

### 环境重置

```ts
async reset(agent: Agent): Promise<void> {
  const runtime = requirePtcRuntime(this.ctx)
  if (!runtime.retainsState) throw new CodeResetFailedError('The PTC runtime does not retain an environment')
  await this.calls.reset(agent.session.id,
    () => runtime.release(agent),
    () => { agent.inject(createUserMessage({
      content: [{ type: 'text', text: 'Program environment reset; retained variables cleared. Completed tool effects remain.' }],
      source: { kind: 'ptc-mode' },
    })) })
}
```

- 保留状态的运行时才可重置（否则 `CodeResetFailedError`，不改准入）；
- 重置 = `runtime.release(agent)` + 一条**模型可见的注入消息**（`source: { kind: 'ptc-mode' }`）——状态丢失这件事本身进日志，符合"模型可见 ⟺ 已记录"；
- 失败时阻止后续程序直到重试成功——**不静默吞掉不确定状态**。

### 插件装配

```ts
export const name = 'ptc-tool-executor'
export const inject = ['tools', 'systemPrompt', 'ptcRuntime']

export function apply(ctx: Context, config: Config): void {
  ctx.tools.setExecutor(new PtcExecutor(ctx, config))
}
```

三重注入：注册表、提示词服务、PTC 运行时——缺一不激活（能力缺失在组合期暴露）。`Config` 提供"按超时的子调用允许量与并发上限"（`config.ts`）。

## 核心类与函数

| 名称 | 职责 |
|------|------|
| `PtcExecutor` | PTC 执行器：单工具描述面 + SDK 段 + 覆写管线 |
| `describe` | 只投影 `run_code` schema，`knownNames` 锁定 |
| `execute` | 同 native：`executeToolBatch` 跑批次 |
| `reset` | 清空某 agent 的保留程序环境，注入模型可见通知 |
| `requirePtcTransport` | 惰性创建保留的 `run_code` 工具定义（不进注册能力表） |
| `renderSdk` | 按运行时语言渲染工具 SDK 文档段 |

## 与其他模块的关系

- **ptc-runtime 缝**（能力缝篇）：`requirePtcRuntime` 解析运行时；`language`/`programFormat`/`retainsState` 决定渲染与重置行为。
- **tools 包**：`RUN_CODE_NAME`、`ToolNotFoundError`、管线与批次全部复用。
- **system-prompt**：两个 section 按注册的 order 插入组装。
- **PtcCalls**（`ptc-calls.ts`）：嵌套调用的记账与 reset 实现。
- **agent-loop**：外层取消照常生效；程序内嵌套调用的取消语义由运行时与管线信号融合决定。

## 小结

PTC 执行器把"模型看见什么"从 N 个 schema 压缩成 1 个 + 文档，把"怎么调用"从协议往返变成程序控制流——而守卫管线通过 `get`/`reject` 覆写保持**唯一的执行语义**。这是"接口不变、形态可换"的教科书实现。
