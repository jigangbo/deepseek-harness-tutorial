# `assistant-stream.ts` — 流式帧协议
> 单次模型尝试的帧发布与持久流累积


> 源码：`packages/core/agent-loop/src/assistant-stream.ts`（140 行）。导出 `AssistantStreamAttempt`。

## 文件概览

每次模型请求（attempt）需要一个对象同时做三件事：

1. **进程内发布**：把 `start` → `chunk` → `end` 三类 `AssistantStreamFrame` 发到 `agent/assistant-stream`（emit，agent-scoped），供 UI 实时渲染；
2. **持久累积**：把流 chunk 压缩成一份可写入日志的 `AssistantStreamRecord` 快照，并把文本 delta 组装成规范 `ContentBlock[]`；
3. **结算协议**：终结 `end` 帧只能在匹配的持久事件（`assistant/message` 或 `assistant/attempt`）提交之后发出。

`AssistantStreamAttempt` 把三件事折进一个实例，生命周期恰好覆盖一次模型尝试：`agent.ts` 的 `step()` 为每次尝试构造它，`live.start()` → `live.push(chunk)` → `live.settle(...)` 或 `live.abandon()`。

帧协议（类型声明于 `packages/core/agent/src/types.ts`）：

- `start`：`attemptId` + `revision` + `turn`/`step`，在首个交付 chunk 之前发布；
- `chunk`：`index`（尝试内稠密零基位置）+ `time` + `chunk`（`StreamChunk`）；
- `end`：`index`（本尝试的 chunk 帧数）+ `outcome`——`committed`（带 `eventType` 与持久 `seq`）或 `abandoned`（无持久事件可提交时的活体放弃）。

## 关键代码走读

### 构造与身份

```typescript
/** Folds one model attempt into one compact stream plus ordered transient frames. */
export class AssistantStreamAttempt {
  private readonly accumulator = new AssistantStreamAccumulator()
  private readonly assembler = new BlockAssembler()
  private index = 0
  private terminal = false
  /** Attempt identity unique within this Agent lifecycle. */
  readonly attemptId: LlmAttemptId

  constructor(
    sessionId: SessionId,
    attempt: number,
    private readonly nextRevision: () => number,
    readonly turn: number,
    readonly step: number,
    private readonly emit: (frame: AssistantStreamFrame) => void,
  ) {
    this.attemptId = LlmAttemptId(`${sessionId}:${attempt}`)
  }
```

- **`attemptId`** = `` `${sessionId}:${attempt}` ``，在单个 Agent 生命周期内唯一；`attempt` 是 attached-Session 局部计数器（`agent.ts` 的 `assistantAttemptCounter`）。
- **`nextRevision`** 由调用方注入（`() => ++this.assistantStreamRevision`）：revision 是 attached-Session 进程内单调号，替换重启（resume）后从 1 重新开始。注入而不是自持，让"每帧一个 revision"的分配集中在一处。
- **两个内部累积器**分工：`AssistantStreamAccumulator`（`@deepseek-ai/dsh-llm`）做**持久压缩**——决定哪些 chunk 进入最终写入日志的流记录；`BlockAssembler` 做**消息组装**——从同一批 chunk 产出规范内容块。

### start / push：发布与快照一次成型

```typescript
/** Publish the opening marker before the first delivered chunk. */
start(): void {
  this.emit({
    type: 'start',
    attemptId: this.attemptId,
    revision: this.nextRevision(),
    turn: this.turn,
    step: this.step,
  })
}

/** Snapshot one chunk once, then feed durable compaction, assembly, and live publication. */
push(chunk: StreamChunk): void {
  const timed = this.accumulator.push({ time: Date.now(), chunk })
  this.assembler.push(timed.chunk)
  this.emit({
    type: 'chunk',
    attemptId: this.attemptId,
    revision: this.nextRevision(),
    index: this.index++,
    time: timed.time,
    chunk: timed.chunk,
  })
}
```

`push()` 是唯一可能重复消费 chunk 的地方，所以它把"打时间戳"收敛为一次：`accumulator.push()` 返回带时间的 chunk（时间戳被持久流记录复用），随后同一对象喂给 `assembler` 与 `emit`——持久压缩、消息组装、实时发布三个消费者共享同一份快照，无二次拷贝。

`agent.ts` 的调用侧在每轮迭代之间检查取消信号：

```typescript
for await (const chunk of stream) {
  signal.throwIfAborted()
  live.push(chunk)
}
```

取消后循环终止，`push` 不再被调用——已 push 的 chunk 都已发布且已进入累积器。

### settle：持久事件先行，终结帧随后

```typescript
/**
 * Publish terminal settlement after the matching durable event commits.
 * @param eventType - durable settlement type.
 * @param append - synchronous durable append returning its committed seq.
 */
settle(
  eventType: 'assistant/message' | 'assistant/attempt',
  append: () => SessionSeq,
): void {
  let seq: SessionSeq
  try {
    seq = append()
  } catch (error: unknown) {
    this.abandon()
    throw error
  }
  this.terminal = true
  this.emit({
    type: 'end',
    attemptId: this.attemptId,
    revision: this.nextRevision(),
    index: this.index,
    outcome: { kind: 'committed', eventType, seq },
  })
}

/** Publish abandonment when no durable attempt event can be committed. */
abandon(): void {
  this.terminal = true
  this.emit({
    type: 'end',
    attemptId: this.attemptId,
    revision: this.nextRevision(),
    index: this.index,
    outcome: { kind: 'abandoned' },
  })
}
```

`settle()` 的顺序就是不变量"模型可见 ⟺ 已记录"在流协议上的落点：**先 `append()` 拿到持久 `seq`，再发 `end` 帧**——帧的消费者看到 `committed` 时，持久事件已经在日志里。追加失败则转 `abandon()` 再重抛：`agent.ts` 的外层 `catch` 里 `if (!live.ended) live.abandon()` 由此不再触发第二次（`terminal` 已置位）。

`abandon()` 用于"活体放弃"：装配或提交失败、工具分发前的抛错——该尝试没有任何持久 assistant 事件可依附，终结帧以 `abandoned` 收口，消费者知道这条流的 durable 版本不会出现了。

### 数据出口：同一批 chunk 的三种视图

```typescript
/** Exact compact stream for the final durable event. */
get stream(): SessionEventMap['assistant/attempt']['stream'] {
  return [...this.accumulator.snapshot()] as AssistantStreamRecord[]
}

/** Canonical completed-message blocks from the same chunks. */
blocks(): ContentBlock[] {
  return this.assembler.blocks()
}

/** Safe visible prefix when cancellation interrupts the attempt. */
interruptedBlocks(): ContentBlock[] {
  return this.assembler.interruptedBlocks()
}

/** Latest adapter-reported usage in the stream. */
get usage(): TokenUsage | undefined { return this.assembler.usage }

/** Terminal reason, defaulting to stop when the stream omitted one. */
get finish(): FinishReason { return this.assembler.finish }

/** Replay state carried by the terminal finish record. */
get replayState(): ReplayEnvelope | undefined { return this.assembler.replayState }
```

四个读数支撑 `agent.ts` 的四种结算形态：

| 读数 | 消费场景 |
|---|---|
| `stream` | 写入 `assistant/message` 或 `assistant/attempt` 的持久流记录 |
| `blocks()` | 正常完成时组装 `createAssistantMessage()` 的内容 |
| `interruptedBlocks()` | 取消时安全的可见前缀；非空则提交 `interrupted: true` 的 `assistant/message` 携带已交付文本，为空则落 `assistant/attempt` |
| `finish` | `'error' | 'aborted'` 进 `agent/request-error` 重试决策；`'max-tokens'` 直接以 max-tokens 关 step；`'stop'` 为默认终结 |
| `usage` / `replayState` | 随持久消息携带适配器报告的用量与重放信封 |

## 核心类与函数（表格）

| 名称 | 类别 | 职责 |
|---|---|---|
| `AssistantStreamAttempt` | class | 单次模型尝试的帧发布器 + 持久流累积器 + 消息组装器的融合实例 |
| `.attemptId` | 属性 | `` `${sessionId}:${attempt}` ``，Agent 生命周期内唯一 |
| `.ended` | getter | 是否已发出终结帧（`terminal`） |
| `.start()` | method | 在首个交付 chunk 前发布 `start` 帧 |
| `.push(chunk)` | method | 一次打时间戳，喂持久压缩、组装与发布；`index` 稠密递增 |
| `.settle(eventType, append)` | method | 先提交持久事件拿 `seq`，再发 `committed` 终结帧；追加失败转 `abandon` 后重抛 |
| `.abandon()` | method | 无持久事件可依附时发 `abandoned` 终结帧 |
| `.stream` | getter | 最终持久事件的精确压缩流记录 |
| `.blocks() / .interruptedBlocks()` | method | 规范完成块；取消时安全可见前缀 |
| `.usage / .finish / .replayState` | getter | 适配器用量；终结原因（缺省 `stop`）；重放信封 |

## 与其他模块的关系

| 对象 | 关系 |
|---|---|
| `packages/llm` | `AssistantStreamAccumulator`（持久压缩）与 `BlockAssembler`（块组装）是全部累积逻辑的所在；`StreamChunk` / `ContentBlock` / `FinishReason` / `TokenUsage` / `ReplayEnvelope` 类型 |
| `packages/core/agent` | `AssistantStreamFrame` 帧类型与 `agent/assistant-stream` 事件；`LlmAttemptId` 品牌类型 |
| `packages/core/session` | `SessionEventMap['assistant/attempt']['stream']` 的记录形态；`SessionSeq` 由驱动经 `session.append()` 获得 |
| `agent.ts` | 唯一消费者：每次尝试构造一个实例，`emit` 回调接 `dispatch.emit('agent/assistant-stream', { frame })`；取消结算与重试决策读它的四个出口 |

## 小结

`AssistantStreamAttempt` 是流协议的守门员：`start` → `chunk` → `end` 三类帧与持久日志严格对齐——每个 `chunk` 帧只在对应持久 assistant 帧结算后才到达（`settle` 先 `append` 后发帧），终结 `end` 恰好一个，`committed` 与 `abandoned` 明确区分"durable 版本已在日志里"与"durable 版本不会出现"。同一批 chunk 经两个累积器产出持久流记录与规范内容块两种视图，取消时的 `interruptedBlocks()` 保证用户已经看到的文本进入下一次请求。
