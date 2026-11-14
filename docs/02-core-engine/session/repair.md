# `repair.ts` — 中断尾部补记
> 中断尾部修复：为悬空的工具调用补上诚实的错误结果


> 源码：`packages/core/session/src/repair.ts`（211 行，模块注释："Pending tool-result recovery shared by failed live steps, interrupted logs, and fork seeds."）

## 文件概览

三种场景会留下"开着的尾部"：

1. **活 step 失败**：assistant 已请求工具、但 step 异常结束，工具调用没有结果；
2. **崩溃后的持久日志**：进程在 turn 中途死亡，日志尾部有 `turn/start` 没有 `turn/end`；
3. **fork seed**：源会话在 fork 切点处 turn 仍开着（见 `fork.ts`）。

Provider 拒绝悬空的 assistant 工具调用——每个请求都必须有配对的 result。`repair.ts` 提供共享的恢复机制：`ToolCallRecovery` 跟踪未应答的调用，`openTurnClosers` 生成**合成收尾**（错误 tool result + `step/end` + 原因特定的 `turn/end`），`interruptedTurnClosers` 是崩溃恢复的入口便捷函数。

修复的边界（模块 JSDoc 原话）："Tail repair preserves closed steps and supplies only missing tool results and lifecycle boundaries, with cause-specific retry guidance."——**只补缺失的结果与生命周期边界，已闭合的 step 原样保留**。

## 关键代码走读

### 1. 两种原因、四段文案

```typescript
export type OpenTurnCloseCause = { readonly kind: 'interrupted' } | { readonly kind: 'forked' }

/** Model-visible wording of the synthetic error tool results, keyed by cause. */
const CLOSER_TEXT = {
  interrupted: {
    started: 'The tool call was interrupted after it was recorded, but no result was durably recorded. Its outcome is unknown. Decide whether to retry from the tool semantics: retry only if the operation is read-only or idempotent; if it may have side effects, first verify external state or ask the user. Do not retry blindly.',
    notStarted: 'The tool call was interrupted before the Harness recorded it as started. Retry it if it is still needed.',
  },
  forked: {
    started: 'The history inherited by this branch records this tool call starting but does not include its result. The parent session may have completed it after the fork point. Decide whether to retry from the tool semantics: retry only if the operation is read-only or idempotent; if it may have side effects, first verify external state or ask the user. Do not retry blindly.',
    notStarted: 'The history inherited by this branch has no record of this tool call starting. The parent session may have executed it after the fork point. Decide whether to retry from the tool semantics: retry only if the operation is read-only or idempotent; if it may have side effects, first verify external state or ask the user. Do not retry blindly.',
  },
} as const
```

cause 决定三件事：合成 `turn/end` 的 reason（`interrupted`/`forked`）、错误结果的**模型可见措辞**、合成 message-id 的前缀。文案设计有两个诚实性原则：

- **区分"没开始"与"开始了但结果未知"**：前者只需要"需要的话重试"；后者明确"结果未知，仅当只读/幂等才可盲重试，可能有副作用就先核实外部状态或问用户"。
- **fork 措辞只描述继承前缀里的事实**：不猜测父会话现状，用"may have"——父会话在 fork 点之后可能完成了、也可能没有。

错误码（`TOOL_NOT_STARTED`/`TOOL_OUTCOME_UNKNOWN`）**跨原因共享**——两种原因对"调用的记录周期"陈述同一个事实。

### 2. `openTurnClosers()` — 扫描日志、定位开口、生成收尾

```typescript
export function openTurnClosers(events: readonly SessionEvent[], cause: OpenTurnCloseCause): SessionEvent[] {
  let openTurn: number | null = null
  let openStep: number | null = null
  const recovery = new ToolCallRecovery(cause)
  for (const event of events) {
    recovery.observe(event)
    switch (event.type) {
      case 'turn/start':
        openTurn = event.data.turn
        openStep = null
        break
      case 'turn/end':
        openTurn = null
        openStep = null
        break
      case 'step/start':
        openStep = event.data.step
        break
      case 'step/end':
        openStep = null
        break
      default:
        break
    }
  }

  const last = events.at(-1)
  if (openTurn === null || last === undefined) return []

  const closers: SessionEvent[] = recovery.results()
  let seq = last.seq + closers.length + 1
  const time = last.time

  if (openStep !== null) {
    closers.push({ type: 'step/end', seq: SessionSeq(seq++), time, data: { turn: openTurn, step: openStep } })
  }
  closers.push({ type: 'turn/end', seq: SessionSeq(seq++), time, data: { turn: openTurn, reason: { kind: cause.kind } } })
  return closers
}
```

要点：

- **一次扫描两用**：同一个循环既维护 turn/step 边界游标，又把每个事件喂给 `ToolCallRecovery.observe`。
- **平衡日志零开销**：没有开口 turn 就返回 `[]`——完整的日志什么都不加。
- **seq 预算**：`seq = last.seq + closers.length + 1` 先为错误结果预留编号，再给 step/turn 收尾连续编号；**时间戳全部复用最后一个真实事件的时间**（确定性，且永不发明"未来时间"）。
- **收尾顺序**：先错误结果（provider 要求 result 与 assistant 调用配对），再 `step/end`，最后 `turn/end`。

### 3. `ToolCallRecovery` — 只记身份，不留历史

```typescript
export class ToolCallRecovery {
  private readonly pendingCalls = new Map<ToolCallId, { turn: number; step: number; callSeq?: SessionSeqType }>()
  private last: Pick<SessionEvent, 'seq' | 'time'> | undefined

  constructor(private readonly cause: OpenTurnCloseCause = { kind: 'interrupted' }) {}

  observe(event: SessionEvent): void {
    this.last = { seq: event.seq, time: event.time }
    switch (event.type) {
      case 'turn/start':
      case 'turn/end':
      case 'step/end':
        this.pendingCalls.clear()
        break
      case 'assistant/message':
        for (const block of event.data.message.content) {
          if (block.type === 'tool-call') {
            this.pendingCalls.set(block.id, { turn: event.data.turn, step: event.data.step })
          }
        }
        break
      case 'tool/call': {
        const entry = this.pendingCalls.get(event.data.callId)
        if (entry) entry.callSeq = event.seq
        break
      }
      case 'tool/result': {
        const callId = event.data.message.source.callId
        const entry = this.pendingCalls.get(callId)
        if (event.surfaceOp === 'append' && entry !== undefined
          && entry.turn === event.data.turn && entry.step === event.data.step) {
          this.pendingCalls.delete(callId)
        }
        break
      }
      default:
        break
    }
  }
```

状态机的四个事实：

- **跟踪的起点是 `assistant/message` 里的 `tool-call` 块**（assistant 的请求），`tool/call` 事件随后登记 `callSeq`——**有 `callSeq` 就是"已记录开始"**，决定走 `started` 还是 `notStarted` 文案。
- **`turn/start`/`turn/end`/`step/end` 清空 pending**：跨 step/turn 的请求不再是本 step 的待答项（保留它们反而会生成错位的结果）。
- **`tool/result` 抵销需条件全配**：surfaceOp 为 append、turn/step 匹配才从 pending 删除；`default` 分支不清理——union 可扩展，无关事件保留待答状态（"SessionEvent is merge-extensible; unrelated events retain pending requests"）。
- **只存身份不存事件**：模块 JSDoc："This state retains pending identities, not event history."——内存占用与日志长度无关。

### 4. `results()` — 按转录顺序生成错误结果

```typescript
results(): SessionEvent<'tool/result'>[] {
  if (this.last === undefined) return []
  let seq = this.last.seq + 1
  const time = this.last.time
  const results: SessionEvent<'tool/result'>[] = []

  const text = CLOSER_TEXT[this.cause.kind]
  for (const [callId, { turn, step, callSeq }] of this.pendingCalls) {
    const started = callSeq !== undefined
    const message: ToolResultMessage = deepFreeze({
      id: brandString<MessageId>(`${this.cause.kind}-tool-result-${callId}-${seq}`),
      role: 'tool',
      toolCallId: callId,
      isError: true,
      source: { kind: 'tool', callId },
      content: [{ type: 'text', text: started ? text.started : text.notStarted }],
    })
    results.push({
      type: 'tool/result',
      seq: SessionSeq(seq++),
      time,
      data: {
        turn, step, message,
        error: started
          ? { name: 'ToolOutcomeUnknownError', code: TOOL_OUTCOME_UNKNOWN }
          : { name: 'ToolNotStartedError', code: TOOL_NOT_STARTED },
      },
      surfaceOp: 'append',
      ...started ? { sourceEventSeqs: [callSeq] } : {},
    })
  }
  return results
}
```

- **Map 插入序 = 转录序**：pendingCalls 按请求顺序插入，生成结果顺序与 assistant 请求顺序一致。
- **message-id 确定性**：`${cause}-tool-result-${callId}-${seq}`——同一前缀重放得到同一批 id。
- **完整的表面契约**：`surfaceOp: 'append'`（错误结果以正常方式进入派生历史）；`started` 时引用 `callSeq` 作为 `sourceEventSeqs`（"我的证据是那次 tool/call"）——这些事件能直接通过 `Session.append`/seed 的全部校验。
- **调用方义务**（JSDoc）："Callers commit the results and observe those commits before recovering again."——提交后要把提交事件再 observe 一遍，状态才与日志对齐。

### 5. `interruptedTurnClosers()` — 崩溃恢复入口

```typescript
export function interruptedTurnClosers(events: readonly SessionEvent[]): SessionEvent[] {
  return openTurnClosers(events, { kind: 'interrupted' })
}
```

一个语义命名的便捷包装。fork seed 不走它——`buildForkSeed` 直接以 `forked` cause 调 `openTurnClosers`。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `TOOL_NOT_STARTED` / `TOOL_OUTCOME_UNKNOWN` | `const string` | 两种恢复错误码：请求未达"已记录开始" / 已记录但结果未知 |
| `OpenTurnCloseCause` | union 类型 | `interrupted`（崩溃恢复）或 `forked`（fork seed）——决定 reason、文案、id 前缀 |
| `openTurnClosers` | `(events, cause) => SessionEvent[]` | 扫描日志：开口 turn 则生成 [错误 results…] + step/end + turn/end；平衡日志返回空 |
| `ToolCallRecovery` | `class` | 逐步 observe 事件，跟踪未应答的 assistant 工具请求（身份而非历史） |
| `ToolCallRecovery.observe` | `(event) => void` | 状态机：登记/抵销/清理 pending；无关事件不清状态 |
| `ToolCallRecovery.results` | `() => SessionEvent<'tool/result'>[]` | 按请求序生成确定性错误结果（cause 特定文案与错误码） |
| `interruptedTurnClosers` | `(events) => SessionEvent[]` | 崩溃恢复入口：`interrupted` cause 的 `openTurnClosers` |

## 与其他模块的关系

- **`fork.ts`**：`buildForkSeed` 以 `forked` cause 复用同一机制；`inheritedEventCount` 不含合成收尾。
- **`agent-loop`**（`packages/core/agent-loop/src/agent.ts`）：直接导入 `ToolCallRecovery`——活 step 失败时在线生成结果再关闭 step（默认 `interrupted` cause）；崩溃恢复（resume 路径）用 `interruptedTurnClosers` 补写持久日志的开口尾部。
- **`session-query`**：冷读合成 `interrupted` 的 `turn/end`（不落盘）用于展示。
- **`Session`/校验路径**：生成的合成事件完全满足 append/seed 校验（seq 连续、JSON 可序列化、表面元数据齐全）——修复产出与正常写入走同一条门。

## 小结

`repair.ts` 的原则是**修复最小且诚实**：只补"provider 必需的配对结果 + 缺失的生命周期边界"；错误结果区分"没开始"与"结果未知"，fork 与崩溃各用各的措辞，不发明事实（"may have"），不回溯修复已闭合的历史。确定性（固定文案、确定性 id、复用最后时间戳）保证同一日志前缀的重放得到同一批修复事件。
