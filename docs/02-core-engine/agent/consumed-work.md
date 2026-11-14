# `consumed-work.ts`

> 从日志回答"被消费的工作去哪了"


## 文件概览

`packages/core/agent/src/consumed-work.ts`（模块标注 `@module @deepseek-ai/dsh-agent/consumed-work`）导出一个纯折叠函数 `foldConsumedWork(events)`：把一个 Agent 日志（或其被拥有的后缀）单遍折叠成 `ConsumedWork`——"最新一个为被消费工作负责的关闭 turn"加上"之后是否有工作被取消未运行"。

它存在的原因写在模块头：turn/step 词汇本身回答不了这个问题。

```typescript
// packages/core/agent/src/consumed-work.ts
/**
 * The turn and step vocabulary alone cannot answer this. A turn that stops
 * before its first step leaves a `turn/end` shaped exactly like the balanced
 * no-op turns a rejection or an empty claim produces, so reading turns in
 * isolation either credits cut-short work as finished or convicts every no-op.
 * The missing fact is the inbox's own record: {@link Inbox} logs each mutation
 * with `removedCount` and marks a cancellation `outcome: 'canceled'`, which
 * separates a turn claiming its input from work being dropped unrun.
 */
```

关键洞察：`turn/end` 的形状无法区分"turn 领了输入但没跑完"与"turn 本来就无事可做"。缺失的事实由 inbox 的持久记录（`agent/inbox/spliced` 的 `removedCount` 与 `outcome: 'canceled'`）补上。这两个字段正是 [types.md](types.md) 里 `agent/inbox/spliced` 事件声明的组成部分。

## 关键代码走读

### ConsumedWork：两个问题，两个答案

```typescript
// packages/core/agent/src/consumed-work.ts
/** How one agent log accounts for the work it consumed. */
export interface ConsumedWork {
  /**
   * The latest closed turn that accounts for consumed work: one that entered a
   * model step, or one that claimed inbox input and then failed, was stopped,
   * or was rejected. Absent when no turn closed over any work.
   */
  readonly end?: SessionEvent<'turn/end'>
  /**
   * Whether accepted work was cancelled out of the inbox, unrun, after that
   * turn. This is the only account of input a cancellation took before any turn
   * could open over it — no `turn/end` describes it.
   */
  readonly droppedUnrun: boolean
}
```

- `end`：最新的"负责的"关闭 turn——要么进入过 model step（`stepped`），要么领取过 inbox 输入且以失败/停止/拒绝收场（`claimed` + 非完成结尾）。没有任何 turn 关闭过工作时不出现。
- `droppedUnrun`：那个 turn 之后，是否有已接受的工作被取消出 inbox 且未运行。这也是**取消发生在任何 turn 打开之前**时输入的唯一去向记录——那种情况下根本没有 `turn/end` 可以描述它。

### accountsForClaim：什么结尾算"对领取负责"

```typescript
// packages/core/agent/src/consumed-work.ts
/**
 * Whether a turn that consumed input but never reached a step ends in a way
 * that accounts for that input. Only a `completed` end does not: it had
 * nothing left to run once its claim was rewritten away. A `blocked` end is
 * that input's ending too — the pre-step rejection that produced it discarded
 * the claimed messages, so the work it took will never run.
 */
function accountsForClaim(reason: TurnEndReason): boolean {
  switch (reason.kind) {
    case 'completed':
      return false
    case 'blocked':
    case 'aborted':
    case 'interrupted':
    case 'error':
      return true
    /* v8 ignore next 5 -- unreachable: `max-tokens` requires a step and short-circuits
     * before this call; `forked` exists only in constructor seed history, while production
     * callers fold an operation-owned suffix. `TurnEndReasonMap` is merge-extensible, so a
     * backend-added variant cannot be listed; an unnameable ending over consumed input must
     * not read as success. */
    default:
      return true
  }
}
```

逐条解释：

- **`completed` → false**：turn 以完成收尾说明它领取后无事可剩（claim 已被重写掉），输入没有悬空——不需要 `end` 来"负责"它。
- **`blocked` → true**：pre-step 拒绝产生的结尾；被领的消息随拒绝被丢弃，工作永远不会运行——这是输入的**终结**，`end` 要为此负责。
- **`aborted`/`interrupted`/`error` → true**：取消或失败的结尾同样终结了领取的输入。
- **default → true**：`TurnEndReasonMap` 是 merge-extensible 的（见 core 教程的 Map→union 模式），后端可以添加列不出的变体。策略是保守的：**无法命名的结尾叠加已消费输入，绝不读成成功**。注释还解释了两个已存在变体为何走 default 分支：`max-tokens` 必然有 step、在此之前短路；`forked` 只出现在构造函数种子历史里，而生产调用者折叠的是操作拥有的后缀。

这个 `default: return true` 是"closed union ends in assertNever; merge-extensible unions fall through a documented default"仓库规范的典型实现——merge-extensible 一侧的 default 必须有文档化的理由。

### foldConsumedWork：单遍折叠

```typescript
// packages/core/agent/src/consumed-work.ts
export function foldConsumedWork(events: readonly SessionEvent[]): ConsumedWork {
  const stepped = new Set<number>()
  const claimed = new Set<number>()
  let open: number | undefined
  let end: SessionEvent<'turn/end'> | undefined
  let droppedUnrun = false
  for (const event of events) {
    switch (event.type) {
      case 'turn/start':
        open = event.data.turn
        break
      case 'step/start':
        stepped.add(event.data.turn)
        break
      case 'agent/inbox/spliced': {
        const { removedCount, outcome, inserted } = event.data
        if (removedCount === undefined) break
        // A replacement keeps the work pending under a new identity, so only a
        // cancellation that leaves nothing behind drops it.
        if (outcome === 'canceled') droppedUnrun ||= inserted.length === 0
        // Claims are the loop's own step-boundary reads, always inside a turn.
        else if (open !== undefined) claimed.add(open)
        break
      }
      case 'turn/end': {
        const { turn, reason } = event.data
        open = undefined
        if (stepped.delete(turn) || (claimed.delete(turn) && accountsForClaim(reason))) {
          end = event
          // Anything dropped before this turn closed is what its own ending
          // reports; only a later drop is still unaccounted for.
          droppedUnrun = false
        }
        break
      }
      default:
        break
    }
  }
  return { ...end === undefined ? {} : { end }, droppedUnrun }
}
```

逐段走读：

**状态**：`stepped` 记录进入过 model step 的 turn 编号；`claimed` 记录领取过 inbox 输入的 turn 编号；`open` 是当前打开的 turn；`end` 是最终答案里的负责 turn；`droppedUnrun` 是第二答案。

**`agent/inbox/spliced` 分支**（核心区分逻辑）：

- `removedCount === undefined` → 该 splice 没删任何东西（纯插入），无关记账，跳过。
- `outcome === 'canceled'` → 这是一次**取消**。`droppedUnrun ||= inserted.length === 0`：替换型取消（删除若干条、又插入了替代消息）把工作**换了个身份继续 pending**，不算丢弃；只有删除后什么都没留下的取消才真正把工作丢成 unrun。
- 否则（普通删除，无 canceled 标记）→ 这是循环在 step 边界的**claim 读取**，claim 总是发生在 turn 内部，所以记入 `claimed.add(open)`。

**`turn/end` 分支**（记账判定）：

- `open = undefined` 无论结论如何——turn 关了。
- 判定条件：`stepped.delete(turn)`（进过 step 的 turn 无条件负责——它消费过模型工作）或 `claimed.delete(turn) && accountsForClaim(reason)`（领过输入且结尾负责该输入）。
- 满足则 `end = event`，并且**重置 `droppedUnrun = false`**：这个 turn 关闭之前发生的丢弃已由它自己的结尾报告；只有其后发生的丢弃仍属未记账。注意：一个"领取了输入但什么都没跑"的 no-op turn（rejected）走到这里会命中 `claimed` + `blocked`，成为 `end`——这正是模块头说的"不能把截断工作记成完成，也不能把每个 no-op 都定罪"的平衡点。

**返回**：条件展开 `{ ...end === undefined ? {} : { end }, droppedUnrun }`，让 `end` 字段在无值时彻底缺席，而不是 `end: undefined`。

### 为什么强调"单遍 + 输入即日志"

函数 JSDoc 点明两个设计约束：

```typescript
// packages/core/agent/src/consumed-work.ts（JSDoc 摘录）
/**
 * Fold one agent log, or an owned suffix of one, into its account of consumed
 * work. Single pass, and every input is the log itself: no caller has to sample
 * live state before cancelling, so a cancellation issued by anyone — the
 * owner's teardown, an ancestor's interrupt, an unloading plugin — reads the same.
 */
```

调用者（典型是取消路径：owner 拆除、祖先中断、插件卸载）不需要在取消前采样任何活体状态，直接折叠日志即可得到一致答案——无论谁发起取消，读法相同。

## 核心导出速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ConsumedWork` | `{ end?: SessionEvent<'turn/end'>; droppedUnrun: boolean }` | 日志的被消费工作记账 |
| `accountsForClaim` | `(reason: TurnEndReason) => boolean`（模块私有） | 领取后未进 step 的 turn 何种结尾算负责；merge-extensible default 保守为 true |
| `foldConsumedWork` | `(events: readonly SessionEvent[]) => ConsumedWork` | 单遍折叠；取消读法与发起者无关 |

## 与其他模块的关系

- **`types.ts`**：依赖 `agent/inbox/spliced` 事件的 `removedCount`/`outcome` 字段——它们在该文件的 `SessionEventMap` 合并里声明。
- **`dsh-session`**：`SessionEvent`、`TurnEndReason`（merge-extensible）来自 session 包；`turn/start`/`step/start`/`turn/end` 是持久事件而非 `agent/*` 活体事件。
- **`dsh-agent-loop`**：`agent/inbox/spliced` 的标准写入者；claim 用纯删除 splice（无 canceled 标记），取消用 `outcome: 'canceled'`——本文件的区分正是建立在驱动器的写入纪律上。
- **消费方**：需要回答"这个 Agent 还欠不欠工作"的路径——典型如拆除/中断前的检查——调用本函数读日志即可，不需要活体 Agent 在场。

## 小结

- `foldConsumedWork(events)` 用一遍扫描回答两个问题：最新负责的关闭 turn（`end`）与之后被取消未运行的工作（`droppedUnrun`）。
- 关键事实差在 inbox 自己的持久记录：claim（普通删除）与取消（`outcome: 'canceled'` 且删除后无插入替代）是两种不同的 splice。
- `accountsForClaim` 对 merge-extensible 的 `TurnEndReasonMap` 采用文档化的保守 default：未命名结尾叠加已消费输入绝不读成成功。
- 输入即日志本身：任何取消发起方读到的答案一致，无需采样活体状态。
