# `executor.ts`

> 执行器契约与原始批次输入/输出


`packages/core/tools/src/executor.ts` 定义"谁来执行工具"的契约。`ToolsService`（注册表）只回答"有哪些工具、谁有资格调用"；真正把一批模型 tool-call 变成一批结果的是 `ToolExecutor` —— 一个由独立插件实现的接口，内置实现是 `dsh-tool-executor-native`（原生 function-calling）和 `dsh-tool-executor-ptc`（`run_code` 程序模式），两者都把单次调用委托回本模块下层的共享管线。本文件还定义批次级数据结构（`ToolBatchInput`/`ToolBatchResult`）和两个批次级错误。

## 文件概览

| 类型/类 | 职责 |
|---|---|
| `ToolModelInput` | 执行器对提示词组装的产出：schemas + knownNames |
| `ToolPromptSection` | 执行器自带的提示词段落（随执行器注册/卸载） |
| `ToolBatchInput` | 原始批次：calls、signal、并发覆盖、agent、step 准入、日志 step |
| `ToolCallResult` / `ToolBatchResult` | 按 callId 关联的结果 + 延迟上下文 + 是否终结 turn |
| `ToolExecutor` | 执行器接口：`describe()` + `execute()`，仅两个方法 |
| `ToolBatchExecutionError` | 批次内部失败：拒绝前先排空，携带已落账的部分结果 |
| `ToolExecutorUnavailableError` | 选中执行器缺失/已卸载的结构化错误 |

## 关键代码走读

### 1. `ToolModelInput` 与 `ToolPromptSection`

```ts
/** Model-call schemas produced by an executor. */
export type ToolModelInput = Pick<ToolProviderResult, 'schemas' | 'knownNames'>

/** Executor-owned guidance registered through the system-prompt section API. */
export interface ToolPromptSection extends Pick<PromptSection, 'name' | 'order' | 'interpolate'> {
  /** Static guidance or text generated from this step's visible capability definitions. */
  readonly text: string | ((tools: readonly ToolDefinition[]) => string)
}
```

`describe(tools)` 的返回值就是这两样：模型请求里的 `schemas`（native 模式是每个工具一条，PTC 模式只有 `run_code` 一条）和 `knownNames`（提示词里工具顺序校验用的名字集合）。`ToolPromptSection` 允许执行器往 system prompt 贡献自己的使用指南：`text` 可以是静态字符串，也可以是根据**本 step 可见工具定义**动态生成的函数 —— 例如 PTC 执行器用它生成 SDK API 文档。这些段落随 `setExecutor()` 注册、随执行器卸载销毁，且渲染时会先检查"当前 scope 选中的执行器还是不是我"，不是则渲染空串。

### 2. `ToolBatchInput` — 一批原始调用

```ts
/** Raw model calls, with optional Agent ownership and model-step logging. */
export interface ToolBatchInput {
  readonly calls: readonly ToolCallBlock[]
  readonly signal: AbortSignal
  /**
   * Positive integer overriding the executor's concurrency limit for this batch.
   * Applies to concurrency-safe root calls; program subcall limits remain separate.
   * Undefined uses the executor's configured limit.
   */
  readonly maxParallelToolCalls?: number
  readonly agent?: Agent
  readonly toolSet?: ResolvedToolSet
  readonly log?: {
    readonly turn: number
    readonly step: number
  }
}
```

五个字段的语义边界值得逐一记牢：

- `calls` 是模型消息里的原始 `ToolCallBlock[]`（含未解析的 JSON 字符串参数）—— 参数解析是执行器的职责（共享批次入口会宽容解析，见 [batch.md](batch.md)）。
- `agent` 缺省 = 服务级独立调用（无 scope、无 Session）；有 `agent` 但无 `log` = 程序化调用（初始化 hook 等）：仍然供给工具 scope 与执行上下文，但**不产生根 tool/call / tool/result 事件**。
- `log` 指向已提交的模型 step（turn/step 编号），要求 `agent` 同时存在；有它才写根调用/结果事件。
- `toolSet` 是本 step 的 `ResolvedToolSet` 准入，覆盖批次内每一个调用（含 PTC 程序子调用）。
- `maxParallelToolCalls` 是**批次数覆盖**，只作用于并发安全的根调用；PTC 程序的子调用并发是执行器自己的另一套配置。

### 3. `ToolBatchResult` — 模型序的收场

```ts
/** One settled result correlated with its original model call. */
export interface ToolCallResult {
  readonly callId: ToolCallId
  readonly result: ToolExecutionResult
}

/** Model-ordered outcomes and context accepted for the next step. */
export interface ToolBatchResult {
  readonly results: readonly ToolCallResult[]
  readonly additionalContexts: readonly UserMessage[]
  readonly concluded: boolean
}
```

三条铁律写在 `ToolExecutor.execute` 的 JSDoc 里：**结果必须按输入顺序、每个输入调用恰一条（abort 时被跳过的调用也要有合成结果）；已启动的工作未 quiescence 前绝不落定**；`additionalContexts` 是本批次收集到的延迟上下文（工具正文 `deferContext`、post-execute 决定注入的），agent loop 会把它们按 FIFO 追加到已记录的工具结果之后。`concluded` 在任一成功结果携带 `concludesTurn` 时为 true，loop 据此结束当前 turn。

### 4. `ToolExecutor` — 只有两个方法的插件契约

```ts
/** A plugin owns argument parsing, dispatch, scheduling, cancellation, and logging. */
export interface ToolExecutor {
  readonly sections?: readonly ToolPromptSection[]
  describe(tools: readonly ToolDefinition[]): ToolModelInput
  execute(input: ToolBatchInput): Promise<ToolBatchResult>
}
```

注释第一行就是分工宣言：**一个执行器插件全权拥有参数解析、分发、调度、取消与日志**。registry 对执行器的最低要求只有两个方法 —— 自定义执行器可以实现完全自己的执行方式（甚至不用共享管线）。内置执行器选择复用 `ToolExecutionPipeline` + `executeToolBatch`，所以它们天然获得三段 waterfall、guard、协作取消与统一日志（这正是 [pipeline.md](pipeline.md) 与 [batch.md](batch.md) 的内容）。

失败契约同样在 JSDoc 里：内部失败必须先**排空**所有已启动的工作，然后以 `ToolBatchExecutionError` 拒绝并保留已落账的结果与上下文 —— 调用方（agent loop）据此停下依赖工作、补记缺失的日志结果。

### 5. 两个批次级错误

```ts
/** A drained batch failed internally; only the included results are committed. */
export class ToolBatchExecutionError extends Error {
  constructor(cause: unknown, readonly partial: ToolBatchResult) {
    super('Tool batch execution failed', { cause })
    this.name = 'ToolBatchExecutionError'
  }
}

/** The selected provider is missing or unloaded; another provider must not take its calls. */
export class ToolExecutorUnavailableError extends HarnessError {
  constructor() {
    super('The selected tool executor is unavailable. Load an executor in this composition before running the agent.', 'TOOL_EXECUTOR_UNAVAILABLE')
    this.name = 'ToolExecutorUnavailableError'
  }
}
```

前者是普通 `Error` 子类（批次内部失败属于实现 bug，不是可路由的业务失败），关键是 `partial`：排空后已确定的那部分结果不会丢。后者继承 `HarnessError`、code `TOOL_EXECUTOR_UNAVAILABLE`：`selectExecutor()` 找不到可用执行器（没挂执行器插件，或挂了但已卸载）时抛出 —— 错误消息明确指向修复方式（在当前组合里加载一个执行器）。registry 卸载执行器时**保留空记录**，正是为了让这个错误而不是"静默换执行器"发生。

## 核心类型与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ToolModelInput` | `Pick<ToolProviderResult, 'schemas' \| 'knownNames'>` | `describe()` 产出，进入提示词组装 |
| `ToolPromptSection` | `PromptSection 字段 + text` | 执行器自有的提示词段落，可按可见工具动态生成 |
| `ToolBatchInput` | `{ calls, signal, maxParallelToolCalls?, agent?, toolSet?, log? }` | 原始批次输入；agent/log/toolSet 三级可选语义 |
| `ToolCallResult` | `{ callId, result }` | 单条结果与原始调用的关联 |
| `ToolBatchResult` | `{ results, additionalContexts, concluded }` | 模型序收场 + 延迟上下文 + 终结标记 |
| `ToolExecutor` | `{ sections?, describe(), execute() }` | 执行器契约：解析/分发/调度/取消/日志全归插件 |
| `ToolBatchExecutionError` | `Error & { partial: ToolBatchResult }` | 排空后拒绝，保留部分已落账结果 |
| `ToolExecutorUnavailableError` | `HarnessError('TOOL_EXECUTOR_UNAVAILABLE')` | 选中执行器缺失/卸载，新批次拒绝 |

## 与其他模块的关系

- `index.ts` 的 `ToolsService.setExecutor()` 存 `ToolExecutor`、`selectExecutor()` 找它、`execute()` 把 `ToolBatchInput` 转发给它；`ToolExecutorUnavailableError` 由 `selectExecutor` 抛出。
- `pipeline.ts` 与 `batch.ts` 是内置执行器的"内部零件"：`ToolExecutor` 的 JSDoc 逐条对应 `executeToolBatch(input, pipeline, maxParallel)` 的行为（顺序结果、abort 合成跳过、排空后拒绝、`maxParallelToolCalls` 优先于执行器配置）。
- `contracts.ts` 提供 `ToolDefinition`（describe 的输入）与 `ToolExecutionResult`（每条结果的类型）。
- 上游包：`dsh-tool-executor-native`、`dsh-tool-executor-ptc` 是两个内置实现；`dsh-system-prompt` 的 `PromptSection` 被 `ToolPromptSection` 复用；`dsh-llm` 提供 `ToolCallBlock`/`ToolCallId`。执行器选择与两种内置形态的对照见 [executors 章节](../executors/index.md)。

## 小结

`executor.ts` 把"注册表知道什么"与"执行器做什么"切成一条清晰的缝：注册表交付**原始批次**（模型给什么就传什么，外加 scope、准入、日志位置与并发覆盖），执行器交回**模型序的完整结果**（一个调用都不能少，跳过的给合成结果，排空才落定）。契约只有两个方法，因此换一种模型交互形态（原生 function calling、PTC 程序、甚至完全自定义）不需要动注册表一行代码；而内置执行器复用共享管线，换来的则是统一的三段策略、取消与日志语义。
