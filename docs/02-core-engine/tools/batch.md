# `batch.ts` — 批次调度：屏障、滚动池与有序落账

`packages/core/tools/src/batch.ts` 解决"一个模型 step 发出 N 个工具调用怎么跑"的问题。它把 [pipeline.md](pipeline.md) 的三阶段 API（`prepare`/`dispatch`/`complete`）编排成一条调度规则：**`exclusive` 调用独占成屏障；`parallel` 调用进有界滚动池**；策略、结果与结果上下文始终按模型顺序落定；abort 或调度器内部失败都停止补充新调用并**排空**已启动的调用。文件顶部的模块 JSDoc 就是它的完整契约说明。

## 文件概览

| 成员 | 职责 |
|---|---|
| `executeToolBatch()` | 批次入口：参数校验 → 计划调用 → 分组循环 → 汇总结果 |
| `runGroup()` | 一组的执行：滚动池 + 有序提交 + 取消/失败排空 |
| `BatchSettlement` | 结果收集器：Session `tool/call`/`tool/result` 事件 + 结果/上下文聚合 |
| `parseArguments()` | 宽容解析模型参数 JSON |
| `PlannedCall` / `Slot` / `GroupOutcome` | 内部流转结构 |

## 关键代码走读

### 1. 批次入口 `executeToolBatch()`

```ts
export async function executeToolBatch(
  input: ToolBatchInput,
  pipeline: ToolExecutionPipeline,
  maxParallel: () => number,
): Promise<ToolBatchResult> {
  const { calls: toolCalls, signal, agent, toolSet, maxParallelToolCalls } = input
  if (maxParallelToolCalls !== undefined && (!Number.isInteger(maxParallelToolCalls) || maxParallelToolCalls < 1)) {
    throw new RangeError('maxParallelToolCalls must be a positive integer')
  }
  const settlement = new BatchSettlement(input)
  try {
    // Inputs are distinct because tools/execute wrappers may replace `exec.signal`.
    const planned: PlannedCall[] = toolCalls.map(block => ({
      block,
      exec: {
        callId: block.id,
        name: block.name,
        arguments: parseArguments(block.arguments),
        ...agent === undefined ? {} : { agent },
        ...toolSet === undefined ? {} : { toolSet },
        signal,
      },
    }))

    let next = 0
    let concluded = false
    while (next < planned.length) {
      // Commit before classifying again so registry changes affect unstarted calls.
      const first = planned[next]!
      const mode = pipeline.executionMode(first.exec).kind
      const group = mode === 'parallel' ? planned.slice(next) : [first]
      const outcome = await runGroup(
        pipeline, settlement, maxParallelToolCalls ?? maxParallel(), group, mode, signal,
      )
      next += outcome.consumed
      concluded ||= outcome.concluded
      if (outcome.aborted) {
        for (const call of planned.slice(next)) settlement.skip(call.block)
        return settlement.result(concluded)
      }
    }
    return settlement.result(concluded)
  } catch (error: unknown) {
    throw new ToolBatchExecutionError(error, settlement.result())
  }
}
```

要点逐条：

- 显式并发覆盖必须是正整数，否则同步抛 `RangeError`（misconfiguration fails loud）；否则用 `maxParallel()` —— 一个惰性读取执行器当前配置的函数（`() => number`，这样执行器配置在批次中途变化也能被看到）。
- 每个 `PlannedCall` 携带独立的 `exec` 对象（相同 signal 引用没关系，因为 `tools/execute` 包装者可能替换 `exec.signal`，共享对象会互相污染 —— 注释原话 "Inputs are distinct because..."）。
- **分组循环每次迭代重新分类**：先提交上一组的已就绪结果，再用当前注册表给下一组分类 —— 注册表在批次中途变化（dispose、新 restriction）会影响还没启动的调用。
- 组被 abort 时，剩余未启动调用逐个 `settlement.skip()` 记成合成结果，保证"每个输入调用恰一条结果"。
- 任何内部异常都包成 `ToolBatchExecutionError`，`partial` 里带着已落账的结果。

### 2. `parseArguments()` — 宽容解析

```ts
/** Parse model arguments, preserving invalid JSON as text and mapping empty input to `{}`. */
function parseArguments(raw: string): unknown {
  try {
    return raw ? JSON.parse(raw) : {}
  } catch (_error: unknown) {
    // Preserve malformed model input for tool-owned argument diagnostics.
    return raw
  }
}
```

空串映射成 `{}`（模型常发 `{}` 的空参调用）；**畸形 JSON 不抛错，原样以字符串传下去** —— 让工具自己的参数校验（`defineTool` 的 `ToolArgsError`）去产出对模型最有诊断价值的错误消息，而不是在调度层就变成不可读的失败。注意管线随后会对它做 lossless 快照检查，字符串天然合法。

### 3. `runGroup()` — 滚动池与有序提交

```ts
async function runGroup(
  pipeline: ToolExecutionPipeline,
  settlement: BatchSettlement,
  maxParallelToolCalls: number,
  group: PlannedCall[],
  mode: ToolExecutionMode['kind'],
  signal: AbortSignal,
): Promise<GroupOutcome> {
  const slots: (Slot | undefined)[] = group.map(() => undefined)
  // Started slots retain their `tool/call` seq so the result can cite it.
  const callSeqs: Array<SessionSeq | undefined> = group.map(() => undefined)
  let nextToStart = 0
  let committed = 0
  let started = 0
  let aborted: boolean = signal.aborted
  let concluded = false
```

四个游标 + 两条状态数组：`slots[i]` 是第 i 个调用落定的 dispatch 结果，`callSeqs[i]` 是它 `tool/call` 事件的 seq（结果事件用 `sourceEventSeqs` 引用它，形成日志内的因果配对）。三个关键闭包：

```ts
  // `committed` advances only across contiguous model-order slots.
  const commitReady = async (): Promise<void> => {
    while (committed < group.length) {
      const slot = slots[committed]
      if (slot === undefined) break
      const call = group[committed]
      const result = await pipeline.complete(slot.exec, slot)
      settlement.commit(call!.block, result, callSeqs[committed])
      concluded ||= result.concludesTurn === true
      committed++
    }
  }
```

**`committed` 只沿连续前缀推进**：就算第 3 个调用先落定，也要等第 1、2 个落定并完成 `pipeline.complete`（后置策略 + finalizeContent + 广播）后才提交 —— 结果顺序永远等于模型顺序。

```ts
  const startCall = async (index: number): Promise<void> => {
    const call = group[index]!
    callSeqs[index] = settlement.start(call.block)   // 追加 tool/call 事件
    started++
    const prepared = await pipeline.prepare(call.exec)   // 顺序的 pre-execute 策略
    throwSchedulerFailure()
    switch (prepared.kind) {
      case 'dispatch': {
        const promise = pipeline.dispatch(prepared.exec).then(
          (outcome) => { slots[index] = { ...outcome, exec: prepared.exec }; return index },
          (error: unknown) => { schedulerFailure ??= { error }; return index },
        )
        inFlight.set(index, promise)
        break
      }
      case 'post-result':
      case 'final-result':
        slots[index] = prepared   // 拒绝/取消/管线失败：无需 dispatch
        break
```

**prepare 顺序、dispatch 并发**：pre-execute 策略（含审批提示）一次只处理一个调用，避免审批弹窗乱序；真正跑正文的 `dispatch` 进 `inFlight` 池。dispatch 的失败不抛出，而是记进 `schedulerFailure`（第一个错误定案），让排空逻辑接管。

```ts
  const fillPool = async (): Promise<void> => {
    while (!aborted && nextToStart < group.length && inFlight.size < maxParallelToolCalls) {
      // Re-read later modes after ordered commits so registry changes can create a barrier.
      const nextCall = group[nextToStart]!
      if (nextToStart > 0 && mode === 'parallel'
        && pipeline.executionMode(nextCall.exec).kind !== 'parallel') break
      await startCall(nextToStart)
      nextToStart++
      throwSchedulerFailure()
      await commitReady()
      throwSchedulerFailure()
      if (signal.aborted) aborted = true
    }
  }
```

滚动池补位前**重新分类下一个调用**：parallel 组里如果后面的调用此刻已被注册表变化（或分类器输出）改成 exclusive，就停止补位 —— 剩余调用留给外层循环在池排空后作为新屏障运行。主循环：

```ts
  try {
    await fillPool()
    while (inFlight.size > 0) {
      const settledIndex = await Promise.race(inFlight.values())
      inFlight.delete(settledIndex)
      throwSchedulerFailure()
      await commitReady()
      throwSchedulerFailure()
      if (signal.aborted) aborted = true
      await fillPool()
    }
  } catch (error: unknown) {
    schedulerFailure ??= { error }
    await Promise.allSettled(inFlight.values())
    throw schedulerFailure.error
  }

  if (aborted) {
    // Started calls and accepted context settle first; every remaining model
    // call then receives an ordered synthetic result before the turn aborts.
    for (const call of group.slice(started)) settlement.skip(call.block)
    return { consumed: group.length, aborted: true, concluded }
  }
```

`Promise.race(inFlight.values())` 等任意一个 dispatch 落定 → 有序提交 → 补位 → 循环。**abort 不抛出**：已启动的调用排空并正常落账（正文 quiescence 由管线保证），未启动的记 `ABORTED_BEFORE_DISPATCH` 合成结果，返回 `aborted: true` 让外层跳过后续组。调度器失败才抛出（先 `Promise.allSettled` 排空），由 `executeToolBatch` 包成 `ToolBatchExecutionError`。

### 4. `BatchSettlement` — Session 日志与结果聚合

```ts
/** Collect committed results and optionally append their durable Session events. */
class BatchSettlement {
  private readonly results: ToolCallResult[] = []
  private readonly contexts: UserMessage[] = []

  constructor(private readonly input: ToolBatchInput) {}

  start(block: ToolCallBlock): SessionSeq | undefined {
    if (this.input.log === undefined) return undefined
    const { agent, log: { turn, step } } = this.input
    return agent!.session.append('tool/call', {
      turn, step, callId: block.id, name: block.name, arguments: block.arguments,
    }).seq
  }

  commit(block: ToolCallBlock, result: ToolExecutionResult, callSeq: SessionSeq | undefined): void {
    if (this.input.log !== undefined) {
      const { agent, log: { turn, step } } = this.input
      agent!.session.append('tool/result', {
        turn, step,
        message: createToolResultMessage({ callId: block.id, content: result.content, isError: result.isError }),
        ...result.error?.info ? { error: result.error.info } : {},
        ...result.meta !== undefined ? { meta: result.meta } : {},
      }, {
        surfaceOp: 'append',
        sourceEventSeqs: [callSeq!],
      })
    }
    this.results.push({ callId: block.id, result })
    this.contexts.push(...result.additionalContexts ?? [])
  }

  skip(block: ToolCallBlock): void {
    this.commit(block, {
      content: [{ type: 'text', text: 'Error: tool call aborted before dispatch' }],
      isError: true,
      error: {
        message: 'tool call aborted before dispatch',
        info: { name: 'AbortError', code: TOOL_ABORTED_BEFORE_DISPATCH },
      },
    }, this.start(block))
  }

  result(concluded = ...): ToolBatchResult {
    return { results: this.results, additionalContexts: this.contexts, concluded }
  }
}
```

`log` 缺省时一切照常收集，只是**不写根事件**（工具自己的事件仍可用 Session）。`start`/`commit` 的配对：`tool/call` 在正文启动前落账（重放时 pending 卡片可还原），`tool/result` 用 `sourceEventSeqs` 引用它的 seq。落账的展示字段是 `content`、结构化 `error.info` 和 `meta` —— 注意**没有 `value`**：规范值是执行期本地的（见 [contracts.md](contracts.md)）。`meta` 就是 `presentationMeta` 投影出的卡片元数据，重放 UI 靠它渲染结构化卡片。`skip` 补记合成取消结果（先 `start` 再 `commit`，日志依然配对完整）。`result()` 默认从已收集结果里推导 `concluded`。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `executeToolBatch()` | `(input, pipeline, maxParallel) => Promise<ToolBatchResult>` | 批次入口：校验、计划、分组循环、失败包装 |
| `runGroup()` | `(pipeline, settlement, max, group, mode, signal) => Promise<GroupOutcome>` | 一组的滚动池执行与有序提交 |
| `commitReady()`（runGroup 闭包） | `() => Promise<void>` | 沿连续前缀做 `pipeline.complete` + 落账 |
| `startCall()`（runGroup 闭包） | `(index) => Promise<void>` | 记 tool/call → 顺序 prepare → 并发 dispatch |
| `fillPool()`（runGroup 闭包） | `() => Promise<void>` | 池补位，补位前重分类，可形成新屏障 |
| `BatchSettlement` | `start/commit/skip/result` | Session 事件落账 + 结果/上下文聚合 |
| `parseArguments()` | `(raw: string) => unknown` | 空串→`{}`，畸形 JSON→原样字符串 |
| `PlannedCall` / `Slot` / `GroupOutcome` | 内部结构 | 计划调用 / 落定槽 / 组结果 |

## 与其他模块的关系

- `pipeline.ts`：`executeToolBatch` 的第二参数；`prepare`/`dispatch`/`complete`/`executionMode` 全部来自它；本文件末尾被 `pipeline.ts` re-export（`export { executeToolBatch } from './batch.ts'`）。
- `executor.ts`：`ToolBatchInput`/`ToolBatchResult`/`ToolCallResult` 定义处；`ToolBatchExecutionError` 是内部失败的出口。内置执行器把 `maxParallel` 实现为读取自己当前配置的函数。
- `contracts.ts`：`TOOL_ABORTED_BEFORE_DISPATCH` 用于 `skip` 的合成结果。
- `dsh-llm` 的 `createToolResultMessage`：把 content/isError 装配成 Session 消息。
- `dsh-session` 的 `SessionSeq`：`tool/call` 与 `tool/result` 的因果配对。
- agent loop：消费 `ToolBatchResult`，把 `additionalContexts` 按 FIFO 追加、据 `concluded` 结束 turn。

## 小结

`batch.ts` 的全部复杂度服务于两条不变量：**结果与日志严格按模型顺序**（`committed` 只沿连续前缀推进，`tool/call`→`tool/result` 靠 seq 配对），**每个输入调用都有收场**（abort 给未启动调用记合成结果，内部失败以 `partial` 保留已落账部分）。在这个骨架上，`exclusive` 屏障与 bounded 滚动池实现并发安全工具的重叠执行，而批次中途的注册表变化通过"提交后再分类、补位前重分类"即时生效 —— 并发是工具声明的特权，顺序是调度器保证的义务。
