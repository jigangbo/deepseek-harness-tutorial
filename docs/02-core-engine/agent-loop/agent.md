# `agent.ts`

> 具体 Agent 驱动：turn/step 状态机与请求构建


> 源码：`packages/core/agent-loop/src/agent.ts`（745 行）。类 `ReactLoopAgent` 是公开 `Agent` 契约的唯一具体实现，包外不可见。

## 文件概览

`agent.ts` 把一个 session 驱动为连续的 turn（回合）与 step（步）：领取 inbox 输入 → `agent/pre-step` 裁决 → 组装与准入 → 构建请求 → 流式消费 → 工具执行 → 回到 step 边界或关闭 turn。文件分四层：

1. **模块级工具**（`requestProposal`、`abortedCancelCause`）：请求提议的适配器默认值剥离、取消原因的日志安全拷贝。
2. **相位机**（`Phase` / `setPhase` / `wakeDriver` / `kick` / `whenIdle` / `runMaintenance`）：`idle` / `maintenance` / `running` 三相与唤醒闩。
3. **turn/step 循环**（`preStep` / `turn` / `step`）：领取、瀑布裁决、准入、流式、工具分发、失败补记。
4. **请求构建**（`prepareRequest` / `buildRequest`）：`agent/request` 瀑布 + `ctx.llm.prepareCall()` 的能力绑定，以及 header/context 记账与消息冻结。

## 关键代码走读

### 相位机：Phase 与唤醒闩

```typescript
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | {
    kind: 'maintenance'
    abort: AbortController
    lastTurn: number
    wakeRequested: boolean
  }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

每个相位携带自己的 `AbortController`；`wakeRequested` 是"无法立即消费的唤醒"闩。`status` getter 把 `maintenance` 对外呈现为 `idle`——公开状态只有 `idle | running` 两值，维护任务不改变 `agent/status` 的观察面。

```typescript
private wakeDriver(wakeAfterAbort = false): void {
  if (this.phase.kind !== 'idle') {
    // Maintenance and aborted drivers cannot deliver the wake: latch it for
    // replay at convergence. Live drivers claim queued work themselves;
    // disposal never latches, so teardown waits on no model turn.
    const reason = abortedCancelCause(this.phase.abort.signal)
    if (reason?.kind !== 'disposed' && (this.phase.kind === 'maintenance' || wakeAfterAbort)) {
      this.phase.wakeRequested = true
    }
    return
  }
  const driver = Promise.withResolvers<void>()
  this.activityDone = driver.promise
  this.setPhase({
    kind: 'running',
    abort: new AbortController(),
    turn: this.phase.lastTurn,
    step: 0,
    wakeRequested: false,
  })
  this.loopCtx.agents.withInitiator(this, () => this.kick()).then(driver.resolve, driver.reject)
}
```

三个关键点：

- **存活驱动不需要闩**：运行中的驱动会在 step 边界自己领取队列；只有 maintenance 与已取消的活动才需要把唤醒闩住，收敛时重放。
- **disposal 永不闩**：拆除不等待模型 turn。
- **initiator scope**：驱动在 `ctx.agents.withInitiator(this, ...)` 内运行，链路上的 `requireInitiator()` 能做因果归因。

```typescript
async whenIdle(): Promise<void> {
  let activity: Promise<void>
  do {
    await (activity = this.activityDone)
  } while (activity !== this.activityDone)
}
```

`whenIdle()` 用"等待后再核对"循环跟随被唤醒输入启动的**替代活动**：await 期间新驱动替换了 `activityDone`，就继续等下一个。

### 投递三别名与取消

```typescript
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void {
  // Waking input cannot join an aborted activity, so it starts the next turn.
  // Captured before the insertion so a reentrant cancel from a splice observer cannot reclassify it.
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}

followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)
}

steer(input: UserMessage): void {
  this.send(input, 'next-step', true)
}

inject(input: UserMessage): void {
  this.send(input, 'next-step', false)
}
```

`followup` / `steer` / `inject` 是 `send` 的固定预设：turn 队列唤醒、step 边界唤醒、step 边界不唤醒。**`agent.inject()` 的语义由此落地**：注入的模型可见上下文被追加到 `next-step` 队列但不唤醒驱动——空闲驱动保持 pending，运行驱动在最近的 step 边界领取；领到的消息 `source.kind === 'runtime-context'`，在 `step()` 中会被"原位替换"为新鲜快照（见下文准入一节）。

`wakingAfterAbort` 在插入**之前**捕获：inbox 的 splice 观察者可能重入 `cancel()`，先捕获防改写分类。

```typescript
cancel(cause: AgentCancelCause, options: CancelOptions = {}): void {
  if (!options.keepInbox) {
    this.inbox.clear()
    if (this.phase.kind !== 'idle') this.phase.wakeRequested = false
  }
  if (this.phase.kind !== 'idle') this.phase.abort.abort(cause)
}
```

无活动时 `cancel()` 是 no-op（除清空 inbox），不会武装未来的工作——不预先取消下一 turn。空闲相位下 `keepInbox` 未设时仍会清空两个待处理列表。

`abortedCancelCause()` 是取消原因进入日志前的守门：

```typescript
function abortedCancelCause(signal: AbortSignal): AgentCancelCause | undefined {
  if (!signal.aborted) return undefined
  // `cancel()` 是该循环拥有的信号唯一的 abort 发起者。
  const cause = signal.reason as AgentCancelCause
  switch (cause.kind) {
    case 'user':
    case 'parent':
    case 'disposed':
      return { kind: cause.kind }
    case 'hook':
      return { kind: 'hook', reason: cause.reason }
    /* v8 ignore next -- cancel accepts the closed AgentCancelCause union */
    default:
      return assertNever(cause)
  }
}
```

只拷贝 `turn/end` 会记录的字段。进程内的 `AbortSignal.reason` 仍是调用者原对象——Node 的 fetch 会往上挂 `stack`，而 `Session.append` 拒绝或无法序列化这种附加数据。拷贝把痕迹挡在日志外，同时保证收尾追加可提交。

### preStep：领取与 accept/rewrite/reject 裁决

```typescript
private async preStep(target: InboxTarget, position: { turn: number; step: number }): Promise<PreparedStep> {
  if (this.phase.kind !== 'running') throw new Error(`agent "${this.id}": pre-step outside running phase`)
  const signal = this.phase.abort.signal
  const claimed = this.inbox.claim(target, position.turn)
  const selection = this.model.current()
  if (selection === undefined) {
    throw new Error(`agent "${this.id}" has no provider/model: set AgentOptions.provider and AgentOptions.model or record a model selection`)
  }
  const toolSet = await this.loopCtx.tools.resolve({ agent: this, modelSelection: selection, signal })
  try {
    const assembly = await this.loopCtx.systemPrompt.assemble({ ...assembleContextFor(this, signal), modelSelection: selection, toolSet })
    signal.throwIfAborted()
    const decision = await this.dispatch.waterfall(
      'agent/pre-step', { messages: claimed, ...position, signal, modelSelection: selection },
      (): Promise<PreStepDecision> => {
        const sections = renderContextSections(assembly)
        const context = this.runtimeContext.project(joinContextSections(sections), sections)
        return Promise.resolve<PreStepDecision>({
          kind: 'enter',
          messages: context === undefined ? claimed : [...claimed, context],
        })
      },
    )
    signal.throwIfAborted()
    if (decision.kind === 'reject') {
      toolSet.dispose()
      return decision
    }
    return { ...decision, assembly, selection, toolSet }
  } catch (error: unknown) {
    toolSet.dispose()
    throw error
  }
}
```

执行顺序是固定的：**先 claim，再准备**。claim 是"纯删除 splice + 逐条 `agent/inbox/claimed` 通知"（见 [inbox.md](inbox.md)），被改写掉的消息不会回流队列。随后解析工具集与 prompt 装配——`assembleContextFor()`（`packages/core/agent/src/dispatch.ts`）同时带 `agent` 与 `scope: agent`，agent 局部的 prompt/tool 贡献不会漏掉。

`agent/pre-step` 是请求派生前唯一的瀑布监听链，三种决策：

| 决策 | 效果 |
|---|---|
| 调用 `next()`（默认终点） | accept：进入 `claimed`（可能追加运行时上下文候选）。**不设置 `startsRequestSeries`**——只有监听器能声明新系列 |
| 返回 `{ kind: 'enter', messages, startsRequestSeries? }` | rewrite：用返回的批次进入 step；被省略的 claimed 消息保持移除；claim 之后插入的输入留在队列 |
| 返回 `{ kind: 'reject' }` | reject：不开 step，turn 以 `{ kind: 'blocked' }` 关闭；已 claim 的消息停在 claimed 状态，既不丢弃也不重新提交 |

reject 与异常路径都 `toolSet.dispose()`——准入（admission）在 step 的工具批次结算或不开 step 时关闭。

### turn()：turn 边界与失败补记

```typescript
private async turn(): Promise<boolean> {
  // ...
  const { signal } = phase.abort
  signal.throwIfAborted()
  const turn = phase.turn + 1
  try {
    this.session.append('turn/start', { turn })
  } catch (error: unknown) {
    this.throwError(error)
  }
  phase.turn = turn
  let turnEnds: TurnEndReason | null = null
  let target: InboxTarget = 'next-turn'
```

turn 编号来自 `phase.turn + 1`，而 `phase.turn` 初始值是构造时传入的 `lastTurn`——由 `AgentLoop` 经 `ctx.agents.turnBoundaryOf(session).lastTurn` 读出，因此**恢复后 turn 计数延续持久化历史**，不会从 0 重来。

step 循环里最有信息量的一段是失败补记：

```typescript
try {
  signal.throwIfAborted()
  this.session.append('step/start', { turn, step })
  phase.step = step
  const toolRecovery = new ToolCallRecovery()
  const stopRecovery = this.ctx.on('session/event', (session, event) => {
    if (session === this.session) toolRecovery.observe(event)
  })
  try {
    // max-tokens is sticky: once any step hits the ceiling, later steps
    // that complete normally must not downgrade the turn outcome.
    const stepEnd = await this.step(decision)
    if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd
  } catch (error: unknown) {
    try {
      for (const event of toolRecovery.results()) {
        this.session.append('tool/result', event.data, {
          surfaceOp: 'append',
          ...event.sourceEventSeqs === undefined ? {} : { sourceEventSeqs: event.sourceEventSeqs },
        })
      }
    } catch (recoveryError: unknown) {
      throw new AggregateError([error, recoveryError], 'Step failed and its pending tool results could not be recorded', { cause: error })
    }
    throw error
  } finally {
    stopRecovery()
    this.session.append('step/end', { turn, step })
  }
} finally {
  // Admission closes once the step's tool batch settles or no step opens.
  decision.toolSet.dispose()
}
```

`ToolCallRecovery`（`packages/core/session/src/repair.ts`）观察本 session 的已提交事件，跟踪未应答的 assistant tool call。step 抛错时在 `step/end` **之前**补记合成 `tool/result`：已记录 `tool/call` 的得 `TOOL_OUTCOME_UNKNOWN`（"Its outcome is unknown."），连开始记录都没有的得 `TOOL_NOT_STARTED`。补记失败升级为 `AggregateError`（原始错误 + 恢复错误）——原始失败保持为 cause。

turn 收尾与续转：

```typescript
if (turnEnds && this.inbox.nextStep.length === 0) {
  await this.dispatch.serial('agent/turn-stopping', { turn, signal })
  signal.throwIfAborted()
}
if (turnEnds && this.inbox.nextStep.length === 0) break
target = 'next-step'
```

turn 只在"step 有了结论（completed / max-tokens / blocked）且 next-step 已空"后运行 serial 的 `agent/turn-stopping`——监听器通过 `agent.steer(...)` 反对即注入下一步输入，机器重读 inbox，数据决定是否继续。异常路径把取消原因转成 `{ kind: 'aborted', reason: cause }`，普通错误转成 `LlmError.failure` 或 `errorChain` 文本（`ToolActivationError` 保留其 code，其余 `UNKNOWN`），经 `throwError()` 先发 `agent/error` 再重抛。`finally` 无条件追加 `turn/end`。

```typescript
if (!this.inbox.hasPending) return false
phase.abort = new AbortController()
// A fresh controller makes a latch set on the old one stale: the live driver claims the queue itself.
phase.wakeRequested = false
phase.step = 0
return true
```

换新 `AbortController` 是精妙的一笔：旧信号上挂起的唤醒闩随之失效，存活驱动自己领取队列，不再需要重放。

首步空批的特例（就在循环开头）：

```typescript
if (turnEnds && decision.messages.length === 0) break
// A removed waking message or an enter decision rewritten to empty
// still owns the initial turn boundary, but it spends no model call.
if (phase.step === 0 && decision.messages.length === 0) {
  turnEnds = { kind: 'completed' }
  return false
}
```

被移除的唤醒消息或被改写为空的 enter 决策仍拥有 turn 边界（turn/start 已提交），但不花模型调用。

### step()：准入时序与流式消费

`step()` 是整个文件的核心。先看准入段：

```typescript
const { assembly } = decision
const renderedPrompt = renderPrompt(assembly)
let firstAttempt = true
while (true) {
  const { config, preparedCall } = await this.prepareRequest(turn, step, decision.selection, signal)
  const currentContext = this.loopCtx.systemPrompt.refreshContext(assembly, {
    ...assembleContextFor(this, signal), modelSelection: decision.selection,
  })
  const sections = renderContextSections(currentContext)
  const context = this.runtimeContext.project(joinContextSections(sections), sections)
  signal.throwIfAborted()
  const startsRequestSeries = firstAttempt && decision.startsRequestSeries === true
  const commits = this.systemPrompt.project(renderedPrompt, {
    inHistory: preparedCall?.systemPromptUpdate === 'in-history',
    startsSeries: startsRequestSeries
      || this.requestSurfaceGeneration !== this.session.surface.contentGeneration
      || (preparedCall?.toolUpdate === undefined && this.toolsChanged(assembly.tools)),
  })
  for (const { message, intent } of commits) {
    this.session.append('system/message', { turn, step, message }, intent)
  }
```

**取消承诺的落点就在这一行 `signal.throwIfAborted()`**：`agent/request` 瀑布与 `refreshContext` 都在它之前，两个异步阶段都看不到待提交的 system prompt 或已接受的 user 消息；取消发生在其间时 **system 与 user 均不提交**。`throwIfAborted()` 之后到 user 准入之间没有 await，这一段提交在 JS 语义上是原子的。

`startsRequestSeries` 三个来源在 `startsSeries` 参数里汇聚：

1. pre-step 决策声明的 `decision.startsRequestSeries === true`（仅首次尝试生效）；
2. surface 替换或图像卸载让 `session.surface.contentGeneration` 发生变化；
3. 装配出的工具 schema 有变，而 prepared route 不支持 `toolUpdate`。

**`startsRequestSeries` 与 `request/header` 的联动**在 `buildRequest()` 里落地（见下节）：新系列让 header 以 `series` 记录或携带 `startsSeries: true`，且 system prompt 调和会重新基线化。

user 准入段：

```typescript
let contextAdmitted = false
if (firstAttempt) {
  for (const message of decision.messages) {
    if (message.source.kind === 'runtime-context') {
      if (context !== undefined && !contextAdmitted) {
        this.session.append('user/message', context, { surfaceOp: 'append' })
        contextAdmitted = true
      }
    } else {
      this.session.append('user/message', message, { surfaceOp: 'append' })
    }
  }
}
if (context !== undefined && !contextAdmitted) {
  this.session.append('user/message', context, { surfaceOp: 'append' })
}
firstAttempt = false
```

三条规则：

- **user 消息只在首次尝试提交**：重试复用同一份装配，不重复准入。
- **运行时上下文原位替换**：claim 到的 `runtime-context` 消息（`agent.inject()` 的产物）不直接提交，而是在其队列位置上换成新鲜的 `context` 候选；候选为 `undefined`（快照未变）时连声称的消息也不重复提交。
- **重试恢复运行时上下文**：循环尾部的兜底 `if (context !== undefined && !contextAdmitted)` 每次迭代都会执行——重试间快照变化会补一条新快照。

流式段（骨架）：

```typescript
const request = this.buildRequest(config, preparedCall, assembly.tools, { turn, step }, startsRequestSeries, signal)
const live = new AssistantStreamAttempt(
  this.session.id,
  ++this.assistantAttemptCounter,
  () => ++this.assistantStreamRevision,
  turn,
  step,
  (frame) => { this.dispatch.emit('agent/assistant-stream', { frame }) },
)
let started = false
try {
  const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
  signal.throwIfAborted()
  live.start()
  started = true
  for await (const chunk of stream) {
    signal.throwIfAborted()
    live.push(chunk)
  }
  signal.throwIfAborted()
} catch (error: unknown) {
  if (!started) throw error
  try {
    if (signal.aborted) {
      const content = live.interruptedBlocks()
      if (content.length > 0) {
        live.settle('assistant/message', () => this.session.append('assistant/message', {
          turn,
          step,
          message: createAssistantMessage({
            content,
            source: { provider: request.provider, model: request.model, /* replayState? */ },
          }),
          interrupted: true,
          ...live.usage === undefined ? {} : { usage: live.usage },
          stream: live.stream,
        }, { surfaceOp: 'append' }).seq)
      } else {
        live.settle('assistant/attempt', () => this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq)
      }
    } else {
      live.settle('assistant/attempt', () => this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq)
    }
  } catch (settlementError: unknown) {
    throw new AggregateError([error, settlementError], 'Assistant stream failed and its durable settlement was rejected', { cause: error })
  }
  throw error
}
```

绑定的 `preparedCall` 优先于 `ctx.llm.stream()`——同一 adapter 贯穿解析、header 记账与分发。取消时**已交付前缀不丢**：`interruptedBlocks()` 非空则提交 `interrupted: true` 的 `assistant/message` 携带用户看到的文本，下一次请求就包含它；可见前缀为空（或错误流）落 `assistant/attempt`。

终结处理与重试：

```typescript
const finish = live.finish
if (finish.kind === 'error' || finish.kind === 'aborted') {
  live.settle('assistant/attempt', () => this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq)
  const action = await this.dispatch.waterfall(
    'agent/request-error', { turn, step, provider: request.provider, failure: finish.failure, retryPolicy: preparedCall?.retryPolicy, signal },
    () => Promise.resolve<RequestErrorAction>(undefined),
  )
  signal.throwIfAborted()
  if (action?.kind !== 'retry') {
    throw new LlmError(finish.failure.message, finish.failure.code, finish.failure)
  }
  continue
}
```

最终 adapter 选择、分发与迭代失败以终结 finish 到达，进入 `agent/request-error` 瀑布：持有恢复的监听器返回 `{ kind: 'retry' }` 且**不调用 `next()`**，默认 `undefined` 保持失败终结。`continue` 回到循环头——同一份装配直接重试，不重跑 `agent/pre-step` 或 user 准入（不变量"重试不重复组装"）。

成功路径提交 `assistant/message` 后处理工具调用：

```typescript
const toolCalls = message.content.filter(block => block.type === 'tool-call')
if (toolCalls.length === 0) return { kind: 'completed' }
const maxParallelToolCalls = this.loopCtx.agentLoop.config.maxParallelToolCalls.get()
try {
  const result = await this.loopCtx.tools.execute({
    calls: toolCalls, signal, agent: this, log: { turn, step },
    maxParallelToolCalls, toolSet: decision.toolSet,
  })
  for (const context of result.additionalContexts) {
    this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context])
  }
  return result.concluded ? { kind: 'completed' } : null
} catch (error: unknown) {
  if (!(error instanceof ToolBatchExecutionError)) throw error
  for (const context of error.partial.additionalContexts) {
    this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context])
  }
  throw error.cause
}
```

循环把**完整原始批次**交给 `ctx.tools.execute()`，并传入当前 volatile 配置 `maxParallelToolCalls`（覆盖执行器自身配置）与 step 准入解析的 `toolSet`。返回 `null` 表示工具延续：外层 turn 循环把 `target` 切到 `next-step`，领取 steering / `additionalContexts` / `inject` 的输入开下一步。`ToolBatchExecutionError` 拆包：先落部分 `additionalContexts`，再抛 `cause`。

### prepareRequest：agent/request 瀑布与 prepareCall

```typescript
private async prepareRequest(
  turn: number,
  step: number,
  selection: Readonly<ModelSelection>,
  signal: AbortSignal,
): Promise<{ config: LlmCallConfig; preparedCall?: PreparedLlmCall }> {
  const { session } = this

  // A loop instance seeds output controls from its options until it logs its
  // own header; later steps re-resolve marked defaults.
  const maxTokens = this.options.maxTokens
  const persisted: Partial<LlmCallConfig> = this.requestHeaderLogged
    ? requestProposal(session.requestHeader()!)
    : maxTokens === undefined ? {} : { maxTokens }
  const { provider: _provider, model: _model, reasoningEffort: _effort, ...controls } = persisted
  const seedConfig = deepFreeze(structuredClone({ ...controls, ...selection }))
  const proposedConfig = await this.dispatch.waterfall(
    'agent/request', { turn, step, signal },
    () => Promise.resolve(seedConfig),
  )
```

种子配置的组装规则：实例记录过自己的 header 之后，从 `session.requestHeader()` 折叠持久控制并用 `requestProposal()` 剥离 adapter 标记的默认字段；此前则从 `AgentOptions.maxTokens` 播种。route 字段（provider/model/effort）始终来自 step 准备时捕获的 `selection`——**`agent/pre-step.modelSelection` 与 `AssembleContext.modelSelection` 携带同一份捕获，贯穿本 step 的全部重试**。

瀑布之后是路线守卫：

```typescript
if (proposedConfig.provider !== selection.provider || proposedConfig.model !== selection.model) {
  throw new Error(`agent "${this.id}": agent/request proposed "${proposedConfig.provider}/${proposedConfig.model}" after step preparation fixed "${selection.provider}/${selection.model}"; use agent.model.select() before step preparation`)
}
if (proposedConfig.reasoningEffort !== selection.reasoningEffort) {
  throw new Error(`agent "${this.id}": agent/request proposed reasoning effort "${proposedConfig.reasoningEffort ?? 'adapter default'}" after step preparation fixed "${selection.reasoningEffort ?? 'adapter default'}"; use agent.model.select() before step preparation`)
}
```

已弃用的 `agent/request` 钩子可以改采样等控制，但换 provider/model/effort 一律拒绝——route 变更必须走 `agent.model.select()` 且在 step 准备之前。之后 `ctx.llm.prepareCall()` 校验 adapter 拥有的字段、在活跃 turn 信号下解析 effort 与输出 token 默认值：

```typescript
try {
  preparedCall = await this.loopCtx.llm.prepareCall(proposedConfig, signal)
  config = preparedCall.config
} catch (error: unknown) {
  // Middleware may serve an unregistered route; terminal dispatch still requires an adapter.
  if (!(error instanceof LlmError) || error.code !== 'NO_ADAPTER') throw error
  config = proposedConfig
}
```

`NO_ADAPTER` 不立即失败：中间件可能自己服务未注册路线，`preparedCall` 缺席，终结分发仍要求 adapter 存在（否则照旧 `NO_ADAPTER` 失败）。循环在解析、`request/header` 记账与分发之间保留**同一个** prepared adapter。

### buildRequest：header 记账、series 标记与请求冻结

```typescript
const header = canonicalHeader({
  config,
  ...preparedCall === undefined ? {} : { adapterDefaults: preparedCall.adapterDefaults },
  ...tools.length > 0 ? { tools } : {},
})
const baseline = this.session.requestHeader()
const startsSeries = startsRequestSeries
  || this.requestSurfaceGeneration !== surfaceGeneration
let headerSeq: SessionSeq | undefined
if (!this.requestHeaderLogged) {
  // Compaction during the first resumed pre-step must still mark a new series.
  headerSeq = this.session.append('request/header', {
    header,
    reason: baseline === undefined ? 'initial' : 'resume',
    ...startsSeries ? { startsSeries: true } : {},
  }).seq
  this.requestHeaderLogged = true
} else if (baseline === undefined || !headerEquals(baseline, header)) {
  headerSeq = this.session.append('request/header', {
    header,
    reason: 'change',
    ...startsSeries ? { startsSeries: true } : {},
  }).seq
} else if (startsSeries) {
  this.session.append('request/header', { header, reason: 'series' })
}
```

`request/header` 的四种记录时机：

| 情形 | reason | 说明 |
|---|---|---|
| 本实例首请求 | `initial`（日志无 header）或 `resume`（有） | 恢复的会话在此续上 header 链；恢复期间 pre-step 里发生的 compaction 仍会标新系列 |
| header 与基线不等 | `change` | config 或 tools 变化（prompt 不属于 header） |
| header 相等但开新系列 | `series` | `startsRequestSeries` 声明或 surface generation 变化 |
| 相同且不开系列 | 不记录 | 未变的 step、重试、同系列后续 turn 继承最新 header |

**`startsRequestSeries` 就在这里落进 `request/header`**：以 `startsSeries: true` 标记或独立的 `series` 记录，请求系列由此定界。

header 之后还有两条记账：工具增删生成一条 `developer/message`（与 headerSeq 一起提交，与模型能力无关——只比较前后 header 的工具名集合）；`request/context`（provider、model、`contextWindow`、route 的 `systemPromptUpdate` 模式）仅在四元组相对上一快照任一不同时追加。

最后是派生与冻结：

```typescript
// canonicalHeader is shallow; append logs a detached snapshot, not these local values.
deepFreeze(header)
const boundaryMessages = session.deriveMessages()
for (const message of boundaryMessages) {
  if (this.frozenMessages.has(message)) continue
  deepFreeze(message)
  this.frozenMessages.add(message)
}
Object.freeze(boundaryMessages)
const request = markAgentLoopRequest(Object.freeze({
  ...header.config,
  messages: boundaryMessages,
  toolHistory: session.toolHistory(),
  ...header.tools !== undefined ? { tools: header.tools } : {},
  sessionId: this.session.id,
  signal,
}))
return request
```

冻结的三个精确边界：

- **每条派生消息身份在它的首个请求上深冻结**，`WeakSet` 保证弱引用不滞留被替换的历史；冻结证明只在本 agent 内复用，恢复的消息保持原身份。
- **冻结请求自身三件套**：本地 canonical header、新消息数组（`Object.freeze(boundaryMessages)`）、信封——**取消信号保持活跃**（`signal` 原样进入请求）。
- **请求不携带 `system` 字段**：请求 = `header.config` + `deriveMessages()` + `header.tools`，system prompt 以 `system/message` 节点出现在派生历史里。

`markAgentLoopRequest()`（`packages/llm/llm/src/call-config.ts`）打上"来自 agent loop 的请求"标记，供 LLM 层识别调用来源。

## 核心类与函数（表格）

| 名称 | 类别 | 职责 |
|---|---|---|
| `ReactLoopAgent` | class | 具体 `Agent` 实现；一个实例驱动一个 session 的全部 turn |
| `Phase` | type | `idle` / `maintenance` / `running` 三相判别联合 |
| `PreparedStep` | type | pre-step 裁决结果：`reject` 或 `enter` + 装配/选择/工具集 |
| `requestProposal(header)` | function | 剥离 adapter 标记的默认字段，让插件重新提议 |
| `abortedCancelCause(signal)` | function | 取消原因的日志安全拷贝（只保留 `turn/end` 记录的字段） |
| `setPhase(next)` | method | 提交相位并发布 `agent/status` 转换（仅状态变化时） |
| `send / followup / steer / inject` | method | 投递与唤醒预设；`inject` 不唤醒 |
| `cancel(cause, options)` | method | 清 inbox（除非 `keepInbox`）并 abort 当前活动 |
| `runMaintenance(job)` | method | 空闲相运行维护任务；异常时同步抛出 |
| `wakeDriver(wakeAfterAbort)` | private | 启动驱动或闩住唤醒待收敛重放 |
| `whenIdle()` | method | 跟随全部活动的收敛（含替代活动） |
| `throwError(error)` | private | 在活跃边界发 `agent/error` 后重抛 |
| `kick()` | private | `while (await this.turn()) {}` 的驱动循环与包含边界 |
| `preStep(target, position)` | private | claim、选择捕获、工具解析、装配、`agent/pre-step` 瀑布 |
| `toolsChanged(tools)` | private | 装配工具 schema 与日志 header 的比较 |
| `turn()` | private | turn 边界：开 turn、step 循环、失败补记、`turn/end` 与续转 |
| `step(decision)` | private | 单步：准入、请求构建、流式、工具分发、重试 |
| `prepareRequest(...)` | private | `agent/request` 瀑布 + route 守卫 + `ctx.llm.prepareCall()` |
| `buildRequest(...)` | private | header/context 记账、series 标记、派生与冻结 |

## 与其他模块的关系

| 依赖 | 用途 |
|---|---|
| `@deepseek-ai/dsh-agent` | `Agent` 契约、`AgentModel`、`agentEvents()` 分发器、`assembleContextFor()` |
| `packages/core/session` | `Session.append()`（全部记账）、`deriveMessages()`、`requestHeader()`、`canonicalHeader`/`headerEquals`、`ToolCallRecovery` |
| `packages/core/system-prompt` | `assemble()`/`refreshContext()`、`renderPrompt()`、`renderContextSections()`/`joinContextSections()` |
| `packages/core/tools` | `resolve()`（step 准入）、`execute()`（完整批次 + `toolSet`）、`ToolActivationError`/`ToolBatchExecutionError` |
| `packages/llm` | `prepareCall()`/`stream()`、`LlmError`、`createAssistantMessage()`、`markAgentLoopRequest()` |
| 包内 `./inbox.ts` | `ReactLoopInbox`：claim 与 splice |
| 包内 `./runtime-context.ts` | `SystemPromptProjection`（system 三路由）、`RuntimeContextProjection`（上下文快照） |
| 包内 `./assistant-stream.ts` | `AssistantStreamAttempt`：帧发布与持久结算 |
| `ctx.agents` | `withInitiator()`、`turnBoundaryOf()` |

被谁消费：仅 `packages/core/agent-loop/src/index.ts` 的 `initializeAgent()` 构造它；扩展插件通过 `agent/*` 事件间接交互，不 import 本文件。

## 小结

`agent.ts` 的全部复杂性都围绕一条纪律展开：**请求从日志派生，模型可见之前先落日志，落日志之前先过取消点**。claim 与裁决在 turn/step 边界先行；`prepareRequest` 把 route 捕获、`agent/request` 瀑布与 adapter 能力绑定成一回事；准入段在 `throwIfAborted()` 之后原子地提交 system（三种路由）与 user（仅首次、运行时上下文原位替换）；`buildRequest` 记 header/context、标 series、冻结派生请求；流式消费先结算持久事件再发终结帧。取消、重试、失败补记都不是特例，而是同一不变量集合在不同边界的投影。
