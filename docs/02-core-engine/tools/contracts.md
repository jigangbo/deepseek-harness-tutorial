# `contracts.ts`

> 工具体系的全部核心类型契约


`packages/core/tools/src/contracts.ts` 是整个工具体系的"类型词典"：注册表（`index.ts`）、执行管线（`pipeline.ts`）、批次调度（`batch.ts`）、执行器（`executor.ts`）之间交换的每一个值都在这里定义。文件本身几乎没有运行时逻辑 —— 只有四个错误类、两个取消码常量和保留名 `run_code` —— 其余全部是 interface / type。读懂这个文件，后面每个文件的代码都是这些类型的运转。

## 文件概览

类型按职责分四组：

| 组 | 类型 | 问答 |
|---|---|---|
| 工具定义 | `ToolOutputDefinition`、`ToolDefinition`、`ToolResult`、`ToolActivationContext`、`ToolRegistrationOptions`、`ResolvedToolSet` | "一个工具长什么样、怎么条件启用" |
| 执行对象 | `ToolExecutionToken`、`ToolExecutionInput`、`ToolExecution`、`ToolDispatchExecution`、`ToolRunContext`、`ToolExecutionMode` | "一次调用在管线里被谁拿着、能看见什么" |
| 结果与错误 | `ToolExecutionSuccess`/`Failure`/`Result`、`ToolFailure`、`ToolErrorInfo`、`ToolNotFoundError`、`ToolOutputError`、`ToolActivationError`、`TOOL_ABORTED*` | "一次调用如何收场" |
| 策略 | `PreToolDecision`、`PostToolDecision`、`ToolGuard`、`ToolRestriction` | "策略插件能表达什么决定" |

## 关键代码走读

### 1. 保留名与不透明 token

```ts
/** Reserved model-facing program transport name. */
export const RUN_CODE_NAME = 'run_code'
declare const toolExecutionTokenBrand: unique symbol

/** Opaque call identity that permits correlation without exposing mutable execution state. */
export type ToolExecutionToken = symbol & { readonly [toolExecutionTokenBrand]: true }
```

`run_code` 是 PTC 模式的保留传输名 —— 全局注册表无条件拒绝把它注册或遮蔽（否则部署默认态下能注册的名字，一挂 PTC preset 就变成冲突）。`ToolExecutionToken` 是品牌化 symbol（符合仓库"跨边界不透明 id 必须 branded"的约定）：它的**值即身份**，只做同一性比较。执行器把它发给嵌套调用作为 `parent`，外部世界永远拿不到活的执行对象。

### 2. `ToolOutputDefinition`：规范输出的声明

```ts
/** Tool-owned canonical output contract used after the body returns a JSON value. */
export interface ToolOutputDefinition {
  /** Raw supported JSON Schema enforced against every successful canonical value. */
  readonly schema: JsonSchemaNode
  /** Pure projection from validated arguments and value to Native/model content. */
  render(args: unknown, value: JsonValue): ContentBlock[]
  /** Pure replayable presentation projection for native and nested calls. */
  presentationMeta?(args: unknown, value: JsonValue): JsonValue
}
```

工具正文返回的不是模型文本，而是**规范值（canonical value）**：一个通过 `output.schema` 校验的无损 JSON 值。`render` 把它投影成模型可见/落账的 `ContentBlock[]`；`presentationMeta` 再投影一份给 UI 卡片用的 JSON 元数据（会在 `tool/result` / `tool/ptc-dispatch` 事件里持久化，让重放时 UI 仍能渲染结构化卡片）。三个都必须是纯函数。

### 3. `ToolDefinition`：注册表持有的完整定义

```ts
/** A registered tool: its schema plus the execution function. */
export interface ToolDefinition extends ToolSchema {
  /** Mandatory canonical output declaration. */
  readonly output: ToolOutputDefinition
  /**
   * Run one accepted call and return only its canonical lossless-JSON value.
   * ...
   */
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>
  projectContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  finalizeContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  timeoutMs?: number
  isConcurrencySafe?(args: unknown): boolean
  presentCall?(args: unknown): ToolCallView | undefined
  presentResult?(args: unknown, result: ToolResult): ToolResultView | undefined
}
```

继承自 `ToolSchema`（模型可见三件套：`name`/`description`/`parameters`），再叠加执行侧字段。注意几个签名细节：

- `execute(args: unknown, ...)` —— 原始定义自己校验参数；第一方工具走 `defineTool`，得到类型化的 `args` 并在进正文前抛 `ToolArgsError`（见 [schema.md](schema.md)）。
- `projectContent` 在 post-execute **之前**安装"执行期预备内容"；`finalizeContent` 是所有策略之后的**最后一英里**内容变换，且刻意收 `ToolExecution`（而非类型化 args）—— 因为非法参数、管线失败这些没有合法 args 的结果也会流经它。
- `timeoutMs` 只是声明：registry 从不强制，需要 `dsh-tool-call-timeout-policy` 包装插件真正执行；声明它等于承诺工具正文会观察 `exec.signal` 达到 quiescence。
- `isConcurrencySafe` 是"是否可与兄弟调用重叠"的纯分类器，**只有精确的 `true` 才并行**，未声明/抛错/返回非 true 一律 exclusive（fail-closed）。
- `presentCall`/`presentResult` 是纯展示回调，可能在**日志重放**时被调用，所以只许依赖 `args`（和落账的 `content/isError/meta`）。

### 4. 执行对象家族：谁能看见什么

```ts
export interface ToolExecutionInput {
  readonly callId: ToolCallId
  readonly rootCallId?: ToolCallId
  readonly name: string
  /** Binding-time tool schema for a PTC inner call; frozen by its producer and never logged. */
  readonly schema?: ToolSchema
  readonly arguments: unknown
  readonly agent?: Agent
  readonly toolSet?: ResolvedToolSet
  readonly parent?: ToolExecutionToken
  /** Required caller-owned cancellation for this invocation. */
  readonly signal: AbortSignal
}

export interface ToolExecution extends ToolExecutionInput {
  readonly rootCallId: ToolCallId
  readonly token: ToolExecutionToken
}

export interface ToolDispatchExecution extends Omit<ToolExecution, 'signal'> {
  signal: AbortSignal
}

export interface ToolRunContext extends ToolExecution {
  deferContext(context: UserMessage): void
  concludeTurn(): void
}
```

这是一个**逐层加权限**的四级结构：

- `ToolExecutionInput` 是调度器提交的原料：调用 id、名字、解析后的参数、可选 agent、step 准入 `toolSet`、可选 `parent`（PTC 子调用标记 —— 在 PTC 执行器选中时，没有 parent 的模型直连调用不许执行 native 工具名，会在策略管线前被拒成 `UNKNOWN_TOOL`）、以及**必填的调用方 signal**。
- `ToolExecution` = input + 管线分配的 `token` + 解析好的 `rootCallId`（根调用 id，嵌套调用继承外层）。所有 waterfall listener 拿到的都是它，字段全 readonly。
- `ToolDispatchExecution` 是 `tools/execute` 包装者看到的视图：唯一可变字段是 `signal` —— 包装者可以给下游换一个信号，但管线会在进正文前把它与**原始调用方 signal 融合**，所以替换无法甩掉调用方取消。
- `ToolRunContext` 是工具正文拿到的运行时扩展：`deferContext()` 把一条 `UserMessage` 上下文挂到**自己这次执行的结果**上（复合工具回传嵌套分发上下文、叶子工具注入插件来源指令的通道，loop 会在 `tool/result` 之后追加）；`concludeTurn()` 把本次成功结果标记为终结当前 turn。

还有一个调度分类：

```ts
export type ToolExecutionMode =
  | { kind: 'parallel' }
  | { kind: 'exclusive' }
```

### 5. 结果联合与错误细节

```ts
/** Successful canonical tool execution, including its Native/model projection. */
export interface ToolExecutionSuccess {
  readonly isError: false
  /** Execution-local canonical value; deliberately omitted from durable events. */
  readonly value: JsonValue
  readonly content: ContentBlock[]
  readonly error?: never
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  /** The agent loop stops after committing this successful result batch. */
  readonly concludesTurn?: true
}

/** Failed canonical tool execution; failures never carry a successful value. */
export interface ToolExecutionFailure {
  readonly isError: true
  readonly error: ToolFailure
  readonly value?: never
  readonly content: ContentBlock[]
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  readonly concludesTurn?: never
}

export type ToolExecutionResult = ToolExecutionSuccess | ToolExecutionFailure
```

判别式联合，`isError` 收窄，两侧用 `?: never` 互斥。三个关键点：`value` 是**执行期本地**的 —— loop 落账只持久化 `content`/`error`/`meta`，重放能还原 UI 但不能还原规范值；失败侧永不携带成功值；`concludesTurn` 只存在于成功侧。

```ts
/** Structured error metadata for a failed tool call (alongside the model-facing text). */
export interface ToolErrorInfo {
  name: string
  code: string
  /** Optional raw user-facing detail; durable projections preserve it but model-facing content does not include it. */
  reason?: string
}

/** Canonical failure detail; internal routing information remains optional. */
export interface ToolFailure {
  message: string
  info?: ToolErrorInfo
}
```

模型看到的失败永远只是 `Error: <message>` 一行文本；`ToolErrorInfo.reason` 是给**持久化诊断和 UI** 看的原始细节，不进模型内容 —— 这是"模型可见 ⟺ 已落账"与"用户可见可以更多"的分界。

### 6. 决策类型与单调 guard

```ts
export type PreToolDecision =
  | { kind: 'allow' }
  | { kind: 'deny'; reason: string; info?: ToolErrorInfo }
  | { kind: 'cancel' }
  | { kind: 'ask'; reason?: string; displayReason?: { readonly en: string; readonly [locale: string]: string } }

export type PostToolDecision =
  | { kind: 'accept'; content?: ContentBlock[]; value?: never; additionalContexts?: UserMessage[] }
  | { kind: 'accept'; value: JsonValue; content?: never; additionalContexts?: UserMessage[] }
  | { kind: 'block'; feedback: ContentBlock[]; additionalContexts?: UserMessage[] }

export type ToolGuard = (execution: Readonly<ToolExecution>) => string | undefined
```

`PreToolDecision` 的四种：allow 放行；deny 带模型可见 reason（可附结构化 `info`）；cancel 选择"规范的前分发取消结果"而不呈现策略拒绝；ask 交给审批服务，只有 `allowed-once` 才放行，其余情况一律 deny。**没有 rewrite** —— 参数已被记录与呈现，改写会让日志、审计、UI 和实际执行四者失配。`PostToolDecision` 的 accept 有两个互斥形态：换 `content`（保住规范值）或换 `value`（会被重新校验并重算 content/meta），block 则把纠错反馈变成 isError。

`ToolGuard` 返回 `string | undefined`：刻意**没有 allow 分支** —— `undefined` 表示弃权保留上游决定，返回 reason 只能降低权限。这就是"单调 deny"的类型层保证：无论 guard 以什么顺序注册，后注册的都翻不了先注册的拒绝。

### 7. 条件激活与 restriction

```ts
export interface ToolActivationContext {
  readonly agent: Agent
  readonly modelSelection: Readonly<ModelSelection>
  readonly signal: AbortSignal
}

export interface ToolRegistrationOptions {
  readonly when?: (context: ToolActivationContext) => boolean | Promise<boolean>
}

export interface ResolvedToolSet {
  dispose(): void
  view(scope: ScopeKey | undefined): ReadonlyMap<string, ToolDefinition>
}

export interface ToolRestriction {
  readonly allow?: readonly string[]
  readonly deny?: readonly string[]
}
```

`when` 收到 step 准备时捕获的 `modelSelection`（provider/model/reasoning effort），同一个 step 的所有请求尝试共用它。`ResolvedToolSet` 是一次 `resolve()` 的产物：`dispose()` 关闭准入，`view(scope)` 按当前注册表复核后给出 admitted 定义。`ToolRestriction` 只作用于**继承的全局工具**，allow 是白名单、deny 是黑名单，多条 restriction 取交集。

### 8. 错误类与取消码

```ts
/** Canonical error code for cancellation after a tool body was invoked. */
export const TOOL_ABORTED = 'ABORTED'

/** Canonical error code for cancellation before a tool body was invoked. */
export const TOOL_ABORTED_BEFORE_DISPATCH = 'ABORTED_BEFORE_DISPATCH'

export class ToolNotFoundError extends HarnessError {
  constructor(toolName: string, reachableFrom?: string) {
    super(
      reachableFrom === undefined
        ? `unknown tool "${toolName}"`
        : `unknown tool "${toolName}": ${reachableFrom}`,
      'UNKNOWN_TOOL',
    )
    this.name = 'ToolNotFoundError'
  }
}

export class ToolOutputError extends HarnessError {
  readonly violations: string[]
  constructor(toolName: string, violations: string[]) {
    super(`tool "${toolName}" returned invalid output: ${violations.join('; ')}`, 'INVALID_TOOL_OUTPUT')
    ...
  }
}
```

两个取消码区分"正文还没开始就被取消"与"正文已启动后取消替换成功结果"—— 调度器和重放逻辑靠它分辨该不该把未启动调用记成合成结果。`ToolNotFoundError` 继承 `HarnessError` 使 `UNKNOWN_TOOL` 与工具自身错误一样可路由（重试/沙箱/重放代码可以区分）；`reachableFrom` 参数表达一种特殊情形：名字其实可见，只是呈现层禁止模型直连（比如 PTC 模式下 native 工具只能被子程序调用）。`ToolActivationError`（`TOOL_ACTIVATION_FAILED`）在前面第 3 节的条件激活中已经出现。

## 核心类型与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `RUN_CODE_NAME` | `const 'run_code'` | PTC 程序传输保留名，注册与 restriction 均不可点名 |
| `ToolOutputDefinition` | `{ schema, render, presentationMeta? }` | 规范输出声明：值 schema + 内容渲染 + UI 元数据投影 |
| `ToolDefinition` | `ToolSchema & { output, execute, … }` | 注册表持有的完整工具定义 |
| `ToolExecutionToken` | branded symbol | 调用身份 token，值即身份，仅作同一性比较 |
| `ToolExecutionInput` | `{ callId, name, arguments, signal, … }` | 管线输入：身份 + 参数 + 必填调用方取消信号 |
| `ToolExecution` | `Input & { rootCallId, token }` | 管线内只读执行对象，waterfall 共享 |
| `ToolDispatchExecution` | `Omit<ToolExecution,'signal'> & { signal }` | `tools/execute` 包装者视图，仅 signal 可换 |
| `ToolRunContext` | `ToolExecution & { deferContext, concludeTurn }` | 工具正文运行时上下文：延迟上下文与终结 turn |
| `ToolExecutionMode` | `'parallel' \| 'exclusive'` 判别联合 | 调度分类：重叠 or 屏障 |
| `ToolExecutionResult` | `Success \| Failure` 判别联合 | 一次调用的最终收场；`value` 只在执行期存在 |
| `ToolFailure` / `ToolErrorInfo` | `{ message, info? }` / `{ name, code, reason? }` | 模型可见消息 + 结构化诊断（reason 不进模型内容） |
| `PreToolDecision` | allow/deny/cancel/ask 联合 | pre-execute 的四种决定，无 rewrite |
| `PostToolDecision` | accept(content 或 value)/block 联合 | post-execute 的三种决定，content 与 value 互斥 |
| `ToolGuard` | `(exec) => string \| undefined` | 单调 guard：只能 deny，不能 allow |
| `ToolRestriction` | `{ allow?, deny? }` | 继承全局工具的 per-scope 过滤，多条取交集 |
| `ToolActivationContext` / `ToolRegistrationOptions` / `ResolvedToolSet` | 条件激活三件套 | step 级工具准入 |
| `ToolNotFoundError` | `HarnessError('UNKNOWN_TOOL')` | 未知/不可直连工具，可路由的结构化失败 |
| `ToolOutputError` | `HarnessError('INVALID_TOOL_OUTPUT')` | 正文值或渲染违反 output schema |
| `ToolActivationError` | `HarnessError('TOOL_ACTIVATION_FAILED')` | `when` 条件抛错（非普通不可用） |
| `TOOL_ABORTED` / `TOOL_ABORTED_BEFORE_DISPATCH` | 错误码常量 | 正文后取消 / 正文前取消 |

## 与其他模块的关系

- `index.ts` 的 `ToolsService` 消费 `ToolDefinition`/`ToolRestriction`/`ToolGuard`/`ResolvedToolSet`，是这些类型的唯一持有者。
- `pipeline.ts` 消费全部执行对象与决策类型，并负责把任意抛出值规范成 `ToolExecutionResult`。
- `executor.ts` 的批次类型引用 `ToolExecutionResult`；`batch.ts` 调度 `ToolExecutionInput` 并消费 `TOOL_ABORTED_BEFORE_DISPATCH` 合成跳过结果。
- `schema.ts` 的 `defineTool` 产出 `ToolDefinition`；`presentation.ts` 的 `ToolCallView`/`ToolResultView` 是 `presentCall`/`presentResult` 的返回类型。
- `events.ts` 把决策类型作为三段 waterfall 的返回契约。

## 小结

`contracts.ts` 用一个文件锁定了工具体系的全部词汇：**定义**（模型可见 schema + 规范输出 + 执行/策略/呈现回调）、**执行对象**（四级只读递进，唯一可变点是 around 包装的 signal）、**结果**（成功携带执行期本地规范值、失败携带结构化但不进模型内容的错误细节）、**决定**（pre 无 rewrite、post 内容与值互斥、guard 单调）。所有后续文件都是把这套类型在运行时安全地兑现 —— 管线负责"每个抛出都变成合法的 `ToolExecutionResult`"，类型负责"每个越权操作在编译期就写不出来"。
