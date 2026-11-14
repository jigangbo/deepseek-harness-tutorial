# `pipeline.ts` — 执行管线
> 单次工具调用的共享执行管线


`packages/core/tools/src/pipeline.ts`（772 行，本模块最大的文件）实现单次工具调用的完整生命周期：参数快照与冻结 → pre-execute 策略 + 审批 + guard → around 包装 + 工具正文 → projectContent → post-execute → finalizeContent → 冻结并广播最终结果。`ToolExecutionPipeline` 类对外暴露**三个阶段方法**（`prepare` / `dispatch` / `complete`），让批次调度器（`batch.ts`）可以在"顺序的策略与落账"之间重叠"并发的正文执行"；`batch.ts` 底部还有一次性封装 `executeToolBatch`。内置 native 与 PTC 执行器都委托给它。

## 文件概览

| 成员 | 职责 |
|---|---|
| `ToolExecutionSource` | 执行器注入的查找/拒绝钩子（PTC 用它拦截模型直连的 native 名字） |
| `ScheduledToolPreparation` / `ScheduledToolDispatch` | 阶段间流转的"下一站"类型：dispatch / post-result / final-result |
| `ToolExecutionPipeline.prepare()` | 快照冻结 + token + pre-execute waterfall + ask 审批 + guard |
| `ToolExecutionPipeline.dispatch()` | tools/execute waterfall + 工具正文 + 结果规范化 + 延迟上下文 |
| `ToolExecutionPipeline.complete()` | projectContent + post-execute waterfall + finish（finalizeContent + 广播） |
| `executionMode()` | 并行/独占分类（fail-closed） |
| 内部机制 | WeakMap 状态（取消/延迟上下文/finalizer/规范结果标记）、信号融合、规范化函数 |

## 关键代码走读

### 1. 构造与执行器来源钩子

```ts
/** Executor-owned lookup and restrictions applied before extensible execution policy. */
export interface ToolExecutionSource {
  /** Override registry lookup for executor-owned capabilities; omitted uses the retained registry's executable lookup... */
  get?(name: string, scope?: ScopeKey, toolSet?: ResolvedToolSet): ToolDefinition | undefined
  /** Reject an otherwise visible invocation before policy; omitted permits it. */
  reject?(name: string, nested: boolean): Error | undefined
}

export class ToolExecutionPipeline {
  private readonly tools: ToolsService

  constructor(private readonly ctx: Context, private readonly source: ToolExecutionSource = {}) {
    // Accepted calls retain their event subject while the service is withdrawn during shutdown.
    this.tools = ctx.tools
  }
```

`source.reject()` 允许执行器在**策略管线之前**拒绝一个本来看得见的调用（`nested` 参数区分 PTC 子调用）—— 例如 PTC 模式下模型直连 native 工具名会被拒成 `UNKNOWN_TOOL`。构造函数把 `ctx.tools` **保留**成字段而不是每次从 ctx 取：已接受的调用在服务被撤下时仍保留事件主题（Shutdown 期间不炸）。

### 2. 三阶段流转类型

```ts
/** Prepared invocation; immediate results carry their post-execute requirement. */
export type ScheduledToolPreparation =
  | { kind: 'dispatch'; exec: ToolRunContext }
  | ({ exec: ToolRunContext } & ScheduledToolDispatch)

/** A `post-result` receives post-execute; a `final-result` bypasses it. */
export type ScheduledToolDispatch =
  | { kind: 'post-result'; result: ToolExecutionResult }
  | { kind: 'final-result'; result: ToolExecutionResult }
```

`prepare` 返回"下一站"：`dispatch` = 正常进入正文；`post-result` = 已经有结果（拒绝/取消），**但仍要走 post-execute**（策略插件必须看到每一次调用，包括被拒的）；`final-result` = **跳过 post-execute** 直接收尾（管线自身失败、执行器拒绝等）。

### 3. `executionMode()` — fail-closed 的并行分类

```ts
executionMode(exec: ToolExecutionInput): ToolExecutionMode {
  const tool = this.resolveExecution(exec.name, exec.agent, exec.parent !== undefined, exec.toolSet)
  if (!tool?.isConcurrencySafe) return { kind: 'exclusive' }
  try {
    const concurrencySafe: unknown = tool.isConcurrencySafe(exec.arguments)
    return concurrencySafe === true ? { kind: 'parallel' } : { kind: 'exclusive' }
  } catch {
    return { kind: 'exclusive' }
  }
}
```

只有"工具可见 + 声明了分类器 + 对这组参数精确返回 `true`"才并行；查不到、未声明、抛错、返回别的值全部独占。这里对分类器**不做参数校验**（原始定义的 args 是 `unknown`，`defineTool` 生成的包装版本会先软校验），分类器自己负责容错。

### 4. `createExecution()` — 快照、冻结、token、回调捕获

```ts
private createExecution(exec: ToolExecutionInput): ScheduledToolPreparation | { kind: 'ready'; exec: MutableToolRunContext } {
  const deferredContexts: UserMessage[] = []
  const token = createExecutionToken()
  const rootCallId = exec.rootCallId ?? callId
  ...
  const base = {
    token, callId, rootCallId, name, signal,
    ...agent !== undefined ? { agent } : {},
    ...
    deferContext(context: UserMessage): void { deferredContexts.push(context) },
    concludeTurn(this: ToolExecution): void { concludingExecutions.add(this) },
  }
  // Capture callbacks before argument getters run; rejected calls retain finalization only when aborted.
  const capturedFinalizer = visible?.finalizeContent?.bind(visible)
  const capturedProjector = visible?.projectContent?.bind(visible)
  const finalizerFor = (): ToolDefinition['finalizeContent'] | undefined =>
    rejected && !signal.aborted ? undefined : capturedFinalizer
  try {
    const detached = snapshotJsonValue(exec.arguments)
    if (detached === undefined) {
      throw new TypeError('tool execution arguments must be losslessly JSON-serializable')
    }
    const execution: MutableToolRunContext = { ...base, arguments: deepFreeze(detached) }
    this.deferredContexts.set(execution, deferredContexts)
    this.contentFinalizers.set(execution, finalizerFor())
    if (!rejected) this.contentProjectors.set(execution, capturedProjector)
    this.cancellationStates.set(execution, { callerSignal: signal, bodyInvoked: false })
    ...
```

这一步是"Model-visible ⟺ logged"的物理保障：参数先过 `snapshotJsonValue`（非无损 JSON 直接 TypeError），再 `deepFreeze` —— 之后无论谁拿到 `exec.arguments` 都不可能改写已记录的参数。四个 `WeakMap` 把可变状态挂在执行对象上：延迟上下文数组（`deferContext` 闭包引用它）、**开始策略前就捕获**的 `finalizeContent`/`projectContent` 回调（捕获优先于参数 getter 求值；被拒绝的调用只有在"已被取消"时才保留 finalizer，避免拒绝原因被覆盖）、取消状态 `{ callerSignal, bodyInvoked }`（调用方原始 signal 存在包装视图之外，包装者换 `exec.signal` 动不了它）。`concludeTurn` 是方法调用 `this` 进 `WeakSet` 的技巧。

### 5. `prepare()` — pre-execute + 审批 + guard

```ts
async prepare(input: ToolExecutionInput): Promise<ScheduledToolPreparation> {
  const created = this.createExecution(input)
  if (created.kind !== 'ready') return created
  const exec = created.exec
  if (this.callerCancelled(exec)) {
    return { kind: 'final-result', exec, result: toolAbortedBeforeDispatchResult() }
  }
  try {
    const carrier = scopeTarget(this.tools, exec.agent)
    const gate = await this.ctx.waterfall(
      carrier, 'tools/pre-execute', exec,
      () => Promise.resolve<PreToolDecision>({ kind: 'allow' }),
    )
    const askResolution = gate.kind === 'ask'
      ? await this.serviceAsk(exec, gate)
      : { decision: gate, approvalCancelled: false }
    const { decision } = askResolution
    if (this.callerCancelled(exec) && askResolution.approvalCancelled) {
      return { kind: 'post-result', exec, result: toolAbortedBeforeDispatchResult() }
    }
    if (decision.kind === 'cancel') {
      return { kind: 'post-result', exec, result: toolAbortedBeforeDispatchResult() }
    }
    const denialReason = decision.kind === 'allow' ? this.tools.checkGuards(exec) : decision.reason
    const denialInfo = decision.kind === 'deny' ? decision.info : undefined
    if (denialReason !== undefined) {
      return {
        kind: 'post-result',
        exec,
        result: this.materializeFinalResult({
          content: [{ type: 'text', text: `Error: ${denialReason}` }],
          isError: true,
          error: { message: denialReason, ...denialInfo === undefined ? {} : { info: denialInfo } },
        }),
      }
    }
    if (this.callerCancelled(exec)) {
      return { kind: 'post-result', exec, result: toolAbortedBeforeDispatchResult() }
    }
    return { kind: 'dispatch', exec }
  } catch (error: unknown) {
    return { kind: 'final-result', exec, result: toolErrorResult(error) }
  }
}
```

顺序值得背下来：**取消检查 → pre-execute waterfall → ask 审批 → 取消×审批取消联合检查 → guard → 取消检查 → dispatch**。注意几个不对称：

- 拒绝（deny 或 guard）与取消返回 `post-result`（仍过 post-execute，策略插件能看到全部调用），而 pre-execute listener **抛错**返回 `final-result`（管线失败跳过 post-execute，避免坏掉的策略再被策略处理）；
- guard 只在 waterfall 决议为 allow 时才查 —— deny 的 reason 已有主；guard 的 reason 与 deny 的 reason 走同一份材料化结果：模型看到 `Error: <reason>`；
- 每次异步落定后重查取消（"read live abort state across an await"），但从不抛弃 listener 的 promise。

### 6. `serviceAsk()` — 审批 seam 的机会式消费

```ts
private async serviceAsk(exec: ToolExecution, ask: ...): Promise<ToolAskResolution> {
  const approval = this.ctx.get('approval')
  if (approval === undefined) {
    return { decision: { kind: 'deny', reason: ask.reason ?? `tool "${exec.name}" requires approval (not yet supported)` }, approvalCancelled: false }
  }
  if (exec.agent === undefined) {
    return { decision: { kind: 'deny', reason: `tool "${exec.name}" requires approval, but the call has no agent to route it through` }, approvalCancelled: false }
  }
  const outcome = await approval.request({ agent: exec.agent, toolName: exec.name, callId: exec.callId, ..., signal: exec.signal })
  switch (outcome) {
    case 'allowed-once': return { decision: { kind: 'allow' }, approvalCancelled: false }
    case 'rejected': return { decision: { kind: 'deny', reason: `the user rejected tool "${exec.name}"` }, ... }
    case 'cancelled': return { decision: { kind: 'deny', reason: `approval for tool "${exec.name}" was cancelled` }, approvalCancelled: true }
    case 'unavailable': return { decision: { kind: 'deny', reason: `tool "${exec.name}" requires approval, but no approval channel is available` }, ... }
    default: return assertNever(outcome, 'ApprovalOutcome')
  }
}
```

审批服务是**可选 seam**：`ctx.get('approval')` 拿不到就降级为 deny（四种 deny reason 各不相同，让模型能区分"用户说不"和"没有审批通道"）。`approvalCancelled: true` 用于审批等待期间用户取消会话的情形 —— 与调用方取消联合判断后记成前分发取消而不是策略拒绝。

### 7. `dispatch()` 与 `dispatchToolBody()` — 信号融合与规范化

```ts
private async dispatchToolBody(exec: MutableToolRunContext): Promise<ToolExecutionResult> {
  const state = this.cancellationStates.get(exec)
  const wrapperSignal = exec.signal
  const fused = fuseToolSignals(state.callerSignal, wrapperSignal)
  const signal = fused.signal

  if (isAborted(signal)) { fused.dispose(); return toolAbortedBeforeDispatchResult() }
  exec.signal = signal
  try {
    const tool = this.resolveExecution(exec.name, exec.agent, exec.parent !== undefined, exec.toolSet)
    if (!tool) throw new ToolNotFoundError(exec.name)
    state.bodyInvoked = true
    const returned = await tool.execute(exec.arguments, exec)
    const result = this.createSuccessResult(exec, tool, returned)
    return isAborted(signal) ? toolAbortedResult(result) : result
  } catch (error: unknown) {
    return toolErrorResult(error)
  } finally {
    fused.dispose()
    exec.signal = wrapperSignal
  }
}
```

这是取消语义的心脏：`tools/execute` 包装者换过的 signal 在这里与**调用方原始 signal 融合**（`fuseToolSignals`：同一信号直接复用；否则 `AbortController` + 两个 `once` listener，body 落定后 `dispose()` 摘除 —— 不嵌套 `AbortSignal.any`，监听器生命周期受控）。`state.bodyInvoked = true` 在正文启动**前**置位，决定取消码是 `ABORTED_BEFORE_DISPATCH` 还是 `ABORTED`。**取消绝不抛弃已启动的正文**：正文 promise 一直等到 quiescence，只有它的成功结果会被取消结果替换（错误结果保留更具体的失败身份）。

`dispatch()` 外层再跑 `tools/execute` waterfall：

```ts
const result = await this.ctx.waterfall(
  carrier, 'tools/execute', mutableExec,
  () => this.dispatchToolBody(mutableExec),
)
const normalized = this.normalizeDispatchResult(exec, result)
const deferredContexts = this.deferredContexts.get(exec)
...
const resultWithDeferredContexts = deferredContexts.length === 0
  ? normalized
  : this.markCanonical(exec, { ...normalized, additionalContexts: [...deferredContexts, ...normalized.additionalContexts ?? []] })
return {
  kind: 'post-result',
  result: this.callerCancelled(exec) && !resultWithDeferredContexts.isError
    ? this.cancellationResult(exec, resultWithDeferredContexts)
    : resultWithDeferredContexts,
}
```

`normalizeDispatchResult()` 依靠 `canonicalResults` WeakMap（结果对象 → 管线 token）识别"这个结果是不是管线自己刚产出的"：是就原样放行；不是（包装者手写的结果）就按**拥有工具的 output 契约重新规范化** —— 错误保留字段重打包，成功值重新走 `createSuccessResult` 的快照+校验+渲染。工具正文 `deferContext` 的上下文被**前置**到 `additionalContexts`。

### 8. `complete()` 与 `finish()` — 后置策略与最终广播

```ts
async complete(exec: ToolRunContext, outcome: ScheduledToolDispatch): Promise<ToolExecutionResult> {
  switch (outcome.kind) {
    case 'final-result': return this.finish(exec, outcome.result)
    case 'post-result': break
    default: return assertNever(outcome, 'scheduled tool outcome')
  }
  const { result } = outcome
  try {
    const project = this.contentProjectors.get(exec)
    this.contentProjectors.delete(exec)
    const content = project?.(exec, result)
    const projected = content === undefined ? result : this.markCanonical(exec, this.materializeFinalResult({ ...result, content }))
    const postResult = await this.postExecute(exec, projected)
    return this.finish(exec, this.callerCancelled(exec) && !postResult.isError ? this.cancellationResult(exec, postResult) : postResult)
  } catch (error: unknown) {
    return this.finish(exec, toolErrorResult(error))
  }
}
```

`projectContent`（工具自己的"执行期预备内容"）在 post-execute **之前**安装 —— 策略随后仍可替换或阻断它。`finish()` 是所有出口的汇聚点：

```ts
private finish(exec: ToolRunContext, result: ToolExecutionResult): ToolExecutionResult {
  let materializedResult: ToolExecutionResult
  try {
    materializedResult = this.materializeFinalResult(result)
  } catch (error: unknown) {
    materializedResult = this.materializeFinalResult(toolErrorResult(error))
  }
  let finalResult: ToolExecutionResult
  try {
    finalResult = this.materializeFinalResult(this.applyFinalContent(exec, materializedResult))
  } catch (error: unknown) {
    finalResult = this.materializeFinalResult(toolErrorResult(error))
  }
  this.notifyResult(exec, finalResult)
  return finalResult
}
```

顺序：**先材料化候选（快照失败→isError）→ 应用定义自有 `finalizeContent`（最后一次内容变换）→ 再材料化 → `notifyResult`**。`finalizeContent` 拿到的是完整规范化结果（连绕过 post-execute 的管线失败也会流经它），只许换 `content`，其余字段归执行器所有，且必须无副作用不抛错 —— 两次 try/catch 保证它自己的失败也变成 isError 而不是逃逸。

```ts
private notifyResult(exec: ToolExecution, result: ToolExecutionResult): void {
  // Freeze the registry's live object before observers receive its readonly WeakMap-keyable view.
  Object.freeze(exec)
  const callbacks = this.ctx.events.dispatch('emit', [scopeTarget(this.tools, exec.agent), 'tools/result', exec, result])
  for (const callback of callbacks) {
    try {
      const returned: unknown = callback(exec, result)
      void Promise.resolve(returned).catch(reportFailure)
    } catch (error: unknown) {
      reportFailure(error)
    }
  }
}
```

`tools/result` 前**冻结执行对象与结果**，观察者的同步抛错与被拒绝的 promise 都被降级为 warn 日志 —— emit 通道对结果零影响。

### 9. 结果工厂与规范化小函数

- `createSuccessResult(exec, tool, candidate)`：值快照 → `validateJsonSchemaValue(tool.output.schema, …)`（违反抛 `ToolOutputError`）→ `deepFreeze` → `render()`（抛错转 `projectionError`）→ 渲染结果快照 → 可选 `presentationMeta` 同样处理 → 附加 `concludesTurn` → `markCanonical` + `materializeFinalResult`。
- `postExecute()`：waterfall 默认 accept；`block` → feedback 变 isError（`failureMessageFromContent` 从 content 块提取消息）；**同时声明 content 与 value 抛 TypeError**；换 `value` 必须在成功结果上、且经 `createSuccessResult` 重新校验渲染；被 block 的调用丢弃正文延迟的上下文，只保留 block 决定自带的。
- `materializeFinalResult()`：只把 `content`/`meta`/`additionalContexts`（展示面）快照+冻结；成功侧在冻结产物旁保留执行期的 `value`。
- `toolErrorMessage(error)`：对任意抛出值**全容错**的消息提取（hostile getter、不可字符串化的值都有兜底 `<unprintable thrown value>`）。
- `toolErrorResult(error)`：`HarnessError` → 附 `{ name, code }` 结构化 info；内容统一 `Error: <message>`。
- `toolAbortedResult` / `toolAbortedBeforeDispatchResult`：两个规范取消结果，保留先前的 `additionalContexts`。
- `fuseToolSignals(caller, wrapper)`：见第 7 节。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ToolExecutionSource` | `{ get?, reject? }` | 执行器注入的查找覆盖与策略前拒绝 |
| `ToolExecutionPipeline` | `constructor(ctx, source?)` | 三阶段管线的持有者，保留 `ctx.tools` 主题 |
| `executionMode()` | `(exec: ToolExecutionInput) => ToolExecutionMode` | 并行/独占分类，一切非精确 true 均独占 |
| `prepare()` | `(input) => Promise<ScheduledToolPreparation>` | 快照/冻结/token → pre-execute → 审批 → guard |
| `dispatch()` | `(exec) => Promise<ScheduledToolDispatch>` | tools/execute waterfall → 正文 → 规范化 + 延迟上下文 |
| `complete()` | `(exec, outcome) => Promise<ToolExecutionResult>` | projectContent → post-execute → finish |
| `finish()`（私有） | `(exec, result) => ToolExecutionResult` | 材料化 → finalizeContent → 再材料化 → 广播 |
| `serviceAsk()`（私有） | `(exec, ask) => Promise<ToolAskResolution>` | 审批 seam 消费，四态映射 + 两种降级 deny |
| `dispatchToolBody()`（私有） | `(exec) => Promise<ToolExecutionResult>` | 信号融合、正文调用、成功结果构造、取消替换 |
| `createSuccessResult()`（私有） | `(exec, tool, candidate) => ToolExecutionSuccess` | 值校验/渲染/meta 投影/规范标记 |
| `normalizeDispatchResult()`（私有） | `(exec, result) => ToolExecutionResult` | 包装者手写结果按 output 契约重新规范化 |
| `postExecute()`（私有） | `(exec, result) => Promise<ToolExecutionResult>` | post-execute waterfall 与三种决定的落实 |
| `notifyResult()`（私有） | `(exec, result) => void` | 冻结 + 广播 tools/result，观察者失败包容 |
| `materializeFinalResult()`（私有） | `(result) => ToolExecutionResult` | 展示面快照 + 冻结；成功侧保留执行期 value |
| `markCanonical()`（私有） | `(exec, result) => result` | WeakMap 标记"管线验证过的结果" |
| `fuseToolSignals()` | `(caller, wrapper) => FusedToolSignal` | 融合调用方与包装者取消，dispatch 作用域内清理 |
| `toolErrorMessage()` | `(error: unknown) => string` | 全容错的消息提取（导出供他用） |
| `executeToolBatch` | re-export 自 `batch.ts` | 批次入口，见 batch.md |

## 与其他模块的关系

- `contracts.ts`：全部执行对象与决策类型的定义处；两个取消码在这里兑现。
- `index.ts`：管线调 `tools.executable()`（查找）、`tools.checkGuards()`（guard）、`ctx.waterfall`（三段事件）；管线保留 `ctx.tools` 主题。
- `batch.ts`：调度器顺序调用 `prepare`/`complete`、并发调 `dispatch`；`executionMode` 决定分组。
- `json-schema.ts`：`validateJsonSchemaValue` 兑现 output schema 校验。
- `dsh-util-values`：`snapshotJsonValue`/`deepFreeze`/`assertNever` 是无损快照与冻结的原语。
- `dsh-user-approval`：`ctx.get('approval')` 的机会式消费（`import type {}` 仅引入类型 side-effect）。
- 上游执行器：`dsh-tool-executor-native` 与 `dsh-tool-executor-ptc` 都构造本管线；PTC 另有自己的子调用日志 waterfall（`tools/ptc-dispatch-log`）。

## 小结

`pipeline.ts` 兑现了 `contracts.ts` 的全部承诺：参数在进策略前变成冻结快照；pre-execute → 审批 → guard → execute → post-execute → finalizeContent 的顺序不可协商；任何环节的抛出（策略、正文、投影、材料化）都收敛成合法的 `ToolExecutionResult`；取消是协作且 quiescent 的 —— 正文前取消记 `ABORTED_BEFORE_DISPATCH`，正文后只替换成功结果，从不抛弃已启动的 promise；包装者唯一能动的 signal 也会被融合回调用方原始信号。最后 `tools/result` 看到的冻结结果可以直接安全地落账。
