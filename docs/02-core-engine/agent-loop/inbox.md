# `inbox.ts` — 输入收件箱
> 持久化的 agent inbox 投影与驱动命令面


> 源码：`packages/core/agent-loop/src/inbox.ts`（232 行）。导出 `inboxProjectionSchema`、`inboxProjectionDefinition`、`ReactLoopInbox`。

## 文件概览

agent 的待办输入有两条有序列表：`next-turn`（等待独立 turn 的 prompt）与 `next-step`（等待下一个 step 边界的 steering / 注入上下文）。本文件把这个状态做成两件事：

1. **持久化投影**（`inboxProjectionDefinition`）：注册到 `ctx.sessionProjections`（键 `inbox`），从日志中的 `agent/inbox/spliced` 事件同步折叠出当前 pending 状态；折叠器校验 splice 坐标与跨列表 id 唯一性，坏的历史按事件 seq 报错。
2. **命令面**（`ReactLoopInbox`）：实现公开 `Inbox` 契约的结构性命令（append / prepend / replace / remove / splice / clear），所有变更都归一化为一条 `agent/inbox/spliced` 事件提交；另有一个**不属于公开契约**的 loop-only 操作 `claim()`。

`AgentLoop` 在 `Service.init` 里注册投影、再发布工厂——所以冷读在任何 Agent 存在前、全部 Agent 卸载后都可用；`ReactLoopAgent` 构造时通过 `readerFor(session)` 拿到读器传入。

## 关键代码走读

### 投影折叠：从 durable splice 重建 pending 状态

```typescript
/** Wire validation for pending agent input reconstructed from durable inbox splices. */
export const inboxProjectionSchema = z.object({
  'next-turn': z.array(z.custom<UserMessage>()).readonly(),
  'next-step': z.array(z.custom<UserMessage>()).readonly(),
}).readonly()

/** Standard fold that reconstructs pending input and rejects invalid durable splice history. */
export const inboxProjectionDefinition = {
  key: 'inbox',
  stateSchema: inboxProjectionSchema,
  init: (): InboxState => ({ 'next-turn': [], 'next-step': [] }),
  apply(state: InboxState, event) {
    if (event.type !== 'agent/inbox/spliced') return state
    const splice = event.data
    try {
      const inbox = state[splice.target]
      const removedCount = splice.removedCount ?? 0
      if (!Number.isSafeInteger(splice.start) || splice.start < 0 || splice.start > inbox.length
        || !Number.isSafeInteger(removedCount) || removedCount < 0
        || splice.start + removedCount > inbox.length) {
        throw new Error('invalid inbox splice')
      }
      const next = inbox.toSpliced(splice.start, removedCount, ...splice.inserted)
      const ids = new Set<string>()
      for (const message of splice.target === 'next-turn'
        ? [...next, ...state['next-step']]
        : [...state['next-turn'], ...next]) {
        if (ids.has(message.id)) throw new Error(`message "${message.id}" is already pending`)
        ids.add(message.id)
      }
      return splice.target === 'next-turn'
        ? { 'next-turn': next, 'next-step': state['next-step'] }
        : { 'next-turn': state['next-turn'], 'next-step': next }
    } catch (error: unknown) {
      throw new Error(`invalid persisted inbox splice at session seq ${event.seq}`, { cause: error })
    }
  },
  // ...
  stateVersion: 1,
} satisfies ProjectionDefinition<'inbox', InboxState>
```

折叠器的防线有三层：

- **坐标校验**：`start` / `removedCount` 必须是安全整数且落在列表范围内，越界即拒绝——把畸形持久历史定位到具体事件 seq。
- **跨列表 id 唯一性**：变更后合并两个列表检查 `MessageId` 重复。`MessageId` 在两条 pending 列表间全局唯一，这是 `agent/inbox/*` 通知可以被单条消息消费者可靠跟随的前提。
- **不可变更新**：`toSpliced` 产出新数组，投影状态是纯值。

wire 视图就是折叠状态本身：

```typescript
wire: {
  // The wire value is the fold state itself: every pending message already
  // round-trips the session log as lossless JSON. Only the static type
  // narrows to the JSON-safe projection table entry.
  viewSchema: inboxProjectionSchema as unknown as z.ZodType<InboxWireState>,
  view: (state: InboxState) => state as unknown as InboxWireState,
},
```

pending 消息早已以无损 JSON 在日志中往返，wire 层无需再变换；唯一的不匹配是静态类型，这里以显式的窄化注释说明理由。

### ReactLoopInbox：命令面与 loop-only claim

```typescript
export class ReactLoopInbox implements InboxContract {
  constructor(
    private readonly readState: () => InboxState,
    private readonly session: Session,
    private readonly dispatch: AgentEventDispatch,
  ) {}

  get nextTurn(): readonly UserMessage[] { return this.readState()['next-turn'] }
  get nextStep(): readonly UserMessage[] { return this.readState()['next-step'] }

  get hasPending(): boolean {
    const state = this.readState()
    return state['next-turn'].length > 0 || state['next-step'].length > 0
  }

  /** Durably cancel all pending input, clearing next-step before next-turn. */
  clear(): void {
    this.splice('next-step', 0, this.nextStep.length, [])
    this.splice('next-turn', 0, this.nextTurn.length, [])
  }
```

实现不持有任何可变状态：读操作全部走 `readState()`（投影读器），写操作全部走 `mutate()` 提交事件。`clear()` 先清 `next-step` 再清 `next-turn`，每条移除都以 `outcome: 'canceled'` 记录并发 `agent/inbox/discarded`——普通移除与 `clear()` 都是"取消"语义。

`claim()` 是唯一的例外路径：

```typescript
/**
 * Remove and return the complete batch proposed for one step.
 * @param target - whether this boundary also consumes one queued turn.
 * @param turn - turn that will own the claimed batch.
 * @returns next-step input followed by the queued turn, when requested.
 */
claim(target: InboxTarget, turn: number): UserMessage[] {
  const claimed = this.mutate('next-step', 0, this.nextStep.length, [], false)
  if (target === 'next-turn') claimed.push(...this.mutate('next-turn', 0, 1, [], false))
  for (const message of claimed) this.dispatch.emit('agent/inbox/claimed', { message, turn })
  return claimed
}
```

claim 与普通移除的区别在 `mutate` 的第五个参数 `discardRemoved`：

- **claim 传 `false`**：纯删除 splice——事件不带 `outcome`，不发 `agent/inbox/discarded`，只逐条发 `agent/inbox/claimed { message, turn }`。
- **turn 边界再领 1 条 `next-turn`**：`target === 'next-turn'` 时在 step 批次后追加队列头部的第一条 prompt；step 边界只领 `next-step` 全部。
- claim 不属于公开 `Agent.inbox`——公开消费者观察 `agent/inbox/claimed` 通知，而不是调用它。

其余结构性命令是 `mutate` 的薄包装：`append` / `prepend` / `remove` / 公开 `splice` 传 `discardRemoved = true`；`replace` 定位后以"删 1 插 1"实现，因此发出旧消息 discarded、新消息 inserted 两条通知（替换可能改变身份）。

### mutate：归一化、唯一性守卫与事件提交

```typescript
/** Commit one normalized mutation and publish its live events. */
private mutate(
  target: InboxTarget,
  start: number,
  deleteCount: number,
  inserted: UserMessage[],
  discardRemoved: boolean,
): UserMessage[] {
  const state = this.readState()
  const inbox = state[target]
  const truncatedStart = Math.trunc(start)
  const offset = Number.isNaN(truncatedStart) ? 0 : truncatedStart
  const actualStart = offset < 0
    ? Math.max(inbox.length + offset, 0)
    : Math.min(offset, inbox.length)
  const truncatedDeleteCount = Math.trunc(deleteCount)
  const actualDeleteCount = Math.min(
    Math.max(Number.isNaN(truncatedDeleteCount) ? 0 : truncatedDeleteCount, 0),
    inbox.length - actualStart,
  )
  if (actualDeleteCount === 0 && inserted.length === 0) return []
  const candidate = inbox.toSpliced(actualStart, actualDeleteCount, ...inserted)
  const ids = new Set<string>()
  for (const message of target === 'next-turn'
    ? [...candidate, ...state['next-step']]
    : [...state['next-turn'], ...candidate]) {
    if (ids.has(message.id)) throw new Error(`message "${message.id}" is already pending`)
    ids.add(message.id)
  }
  const outcome = discardRemoved && actualDeleteCount > 0 ? 'canceled' as const : undefined
  const splice: SessionEventMap['agent/inbox/spliced'] = {
    target,
    start: actualStart,
    ...(actualDeleteCount === 0 ? {} : { removedCount: actualDeleteCount }),
    inserted,
    ...(outcome === undefined ? {} : { outcome }),
  }
  const removed = inbox.slice(actualStart, actualStart + actualDeleteCount)
  const event = this.session.append('agent/inbox/spliced', splice)
  if (discardRemoved) {
    for (const message of removed) this.dispatch.emit('agent/inbox/discarded', { message })
  }
  for (const message of event.data.inserted) {
    this.dispatch.emit('agent/inbox/inserted', { message })
  }
  return removed
}
```

要点：

- **`Array.prototype.splice` 的语义归一化**：`send()` 传入的 `Infinity` 起点、负偏移（从尾部数）、NaN 全部收敛为安全坐标；零删除零插入是 no-op，不提交事件。折叠器校验的坐标域与这里生成的坐标域一致。
- **事件先提交，通知随后**：`session.append()` 同步落日志，返回的 `event.data.inserted` 是提交后的规范数据；`inserted` 通知从提交事件读取，保证通知与日志一致。投影注册表同步折叠该事件，所以 `Session.append()` 返回时活跃投影已反映拼接结果。
- **幂等守卫双侧对称**：`mutate` 在提交前检查候选 id 集合，与折叠器对持久历史的检查使用同一规则——写入侧非法直接抛错，读取侧非法按坏历史报错。

## 核心类与函数（表格）

| 名称 | 类别 | 职责 |
|---|---|---|
| `inboxProjectionSchema` | zod schema | 两条 pending 列表的 wire 校验 |
| `inboxProjectionDefinition` | `ProjectionDefinition<'inbox'>` | 键 `inbox` 的标准折叠：重建 pending、拒绝畸形 splice 历史；`stateVersion: 1` |
| `ReactLoopInbox` | class | 实现 `InboxContract`；读走投影、写走 `mutate()` |
| `.nextTurn / .nextStep / .hasPending` | getter | 投影状态的只读访问 |
| `.clear()` | method | 持久化取消全部 pending（先 next-step 后 next-turn，`outcome: 'canceled'`） |
| `.claim(target, turn)` | method | loop-only：纯删除领取 step 批次（+ turn 边界的 1 条 next-turn），发 `claimed` 通知 |
| `.append / .prepend / .replace / .remove / .splice` | method | 结构性命令，统一经 `mutate()` 提交 |
| `.locate(messageId)` | private | 跨两列表定位一条 pending 身份 |
| `.mutate(...)` | private | 坐标归一化、id 唯一性守卫、提交 `agent/inbox/spliced`、发 discarded/inserted 通知 |

## 与其他模块的关系

| 对象 | 关系 |
|---|---|
| `packages/core/session-projection` | `ProjectionDefinition` 类型与 `ctx.sessionProjections.register()`；`readerFor(session)` 产出驱动用的 `readState` |
| `packages/core/session` | `Session.append('agent/inbox/spliced', ...)` 是唯一存储通道；折叠器消费该事件类型 |
| `packages/core/agent` | `InboxContract` / `InboxState` / `InboxTarget` 类型与 `agent/inbox/*` 事件词汇；claim 不属于公开契约 |
| `packages/llm` | `MessageId` 品牌类型 |
| `agent.ts` | `ReactLoopAgent` 构造 `ReactLoopInbox`；`send()` 经 `splice` 插入、`cancel()` 经 `clear()` 清空、`preStep()` 经 `claim()` 领取、`step()` 经 `splice` 追加 `additionalContexts` |
| `index.ts` | `AgentLoop` 注册投影、构造时传读器 |

## 小结

`inbox.ts` 把"待办输入"从驱动内存搬进日志：每一条插入、编辑、移除、领取、取消都是一条归一化的 `agent/inbox/spliced` 事件，折叠器从事件流重建状态并拒绝畸形历史。公开 `Inbox` 契约覆盖结构性命令；claim 是驱动私有的领取通道，以纯删除提交、以 `agent/inbox/claimed` 通知公开。写入侧与读取侧共享同一套坐标与 id 唯一性规则，通知与日志永远一致——这是"消息跟随者"（UI、编排器）可以只依赖三种 inbox 通知的基础。
