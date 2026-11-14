# `projection.ts` — turn 边界投影
> turnBoundary 投影：Session 的 turn/step 位置


## 文件概览

`packages/core/agent/src/projection.ts` 是本包最小的一个源文件（60 行），导出一个纯折叠定义 `turnBoundaryProjectionDefinition`：把 Session 日志里的四个边界事件（`turn/start`、`turn/end`、`step/start`、`step/end`）折叠成 `TurnBoundaryProjection`——"这个 Session 的对话推进到哪了"。

它不定义新的词汇（状态类型 `TurnBoundaryProjection` 在 [types.md](types.md)），只负责 `apply` 纯函数与 schema。文件头即其定位：

```typescript
// packages/core/agent/src/projection.ts
/** Shared pure fold of the Session's turn and step events. */
```

## 关键代码走读

### schema：把裸数字品牌化回 SessionSeq

```typescript
// packages/core/agent/src/projection.ts
const sessionSeqSchema = zod.number().int().nonnegative().transform(brandNumber<SessionSeq>)

const turnBoundaryProjectionSchema: zod.ZodType<TurnBoundaryProjection> = zod.object({
  openTurnStartSeq: sessionSeqSchema.nullable(),
  lastStepStartSeq: sessionSeqSchema.nullable(),
  lastStepBoundary: zod.object({
    kind: zod.union([zod.literal('start'), zod.literal('end')]),
    seq: sessionSeqSchema,
  }).nullable(),
  lastTurn: zod.number().int().nonnegative(),
})
```

两层结构：

- `sessionSeqSchema`：整数、非负，然后 `transform(brandNumber<SessionSeq>)`——从持久化读回的裸 `number` 在验证点恢复成品牌化的 `SessionSeq`。这是仓库"在 parser/持久边界做验证"原则的落实：品牌在 wire 上就是 number，进入内存即恢复类型身份（branded id 模式，见 core 教程的"Branded IDs"一节）。
- 状态的四个字段与 `TurnBoundaryProjection` 一一对应，两个位置字段可空（turn 之间、首 step 之前），`lastStepBoundary` 记录最近的 step 边界**及其方向**。

### 投影定义本体

```typescript
// packages/core/agent/src/projection.ts
/** Host projection of agent turn and step boundaries. */
export const turnBoundaryProjectionDefinition = {
  key: 'turnBoundary',
  stateVersion: 2,
  stateSchema: turnBoundaryProjectionSchema,
  init: () => ({
    openTurnStartSeq: null,
    lastStepStartSeq: null,
    lastStepBoundary: null,
    lastTurn: 0,
  }),
  apply: (state, event) => {
    switch (event.type) {
      case 'turn/start':
        return {
          ...state,
          openTurnStartSeq: event.seq,
          lastTurn: event.data.turn,
        }
      case 'turn/end':
        return {
          ...state,
          openTurnStartSeq: null,
        }
      case 'step/start':
        return {
          ...state,
          lastStepStartSeq: event.seq,
          lastStepBoundary: { kind: 'start', seq: event.seq },
        }
      case 'step/end':
        return {
          ...state,
          lastStepBoundary: { kind: 'end', seq: event.seq },
        }
      default:
        return state
    }
  },
} satisfies ProjectionDefinition<'turnBoundary', TurnBoundaryProjection>
```

逐个 case：

- **`turn/start`**：记下该事件的 `seq`（日志位置，不是 turn 编号）到 `openTurnStartSeq`，同时更新 `lastTurn = event.data.turn`。
- **`turn/end`**：只清 `openTurnStartSeq`。注意 `lastTurn` **不清零**——它是"最新 turn 编号"的游标，0 表示"还没开过第一个 turn"；turn 关闭后仍要知道最后是第几个 turn。
- **`step/start`**：同时更新两个位置——`lastStepStartSeq`（专记最新的 step 开始）与 `lastStepBoundary`（记最新边界 + 方向 start）。
- **`step/end`**：只更新 `lastStepBoundary`（方向 end）。`lastStepStartSeq` 保持为最近一次 `step/start` 的 seq——即使 step 已结束，"开始位置"仍可追溯。
- **`default`：返回原 state**。`SessionEventMap` 是 merge-extensible 的，日志里会出现大量其他事件类型（`user/message`、`tool/call`、`agent/inbox/spliced`……），全部静默穿过。这是 merge-extensible union 的文档化 default 落穿，与 [consumed-work.md](consumed-work.md) 的保守 default 形成对比：这里"未命名事件"对位置折叠无关紧要，返回原 state 是正确行为。

`apply` 是纯函数：每臂返回新对象或原引用（无可变共享状态），可安全地被投影注册表对同一日志反复重放。

### 注册与消费：注册表拥有它

投影定义**不由消费者注册**——`AgentRegistry` 在 `[Service.init]` 里注册并保存句柄：

```typescript
// packages/core/agent/src/index.ts
protected async [Service.init](): Promise<void> {
  this.turnBoundaryProjectionHandle = await this.ctx.sessionProjections.register(turnBoundaryProjectionDefinition)
}

turnBoundaryOf(session: Session): TurnBoundaryProjection {
  return this.turnBoundaryProjectionHandle.stateOf(session)
}
```

`ctx.agents.turnBoundaryOf(session)` 由此成为唯一入口。这个所有权安排有两个后果：

- 消费者（Web、SDK、检查工具）不需要知道投影 key，也不存在"重复注册同名投影"的冲突面；
- 折叠随日志提交自动前进，**不需要活体 Agent 在场**——冷消费者对已关闭的 Session 一样能读出 turn/step 位置。

## 核心导出速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `turnBoundaryProjectionDefinition` | `satisfies ProjectionDefinition<'turnBoundary', TurnBoundaryProjection>` | 折叠四个边界事件为位置状态；key `turnBoundary`，`stateVersion: 2` |
| `sessionSeqSchema` / `turnBoundaryProjectionSchema` | 模块内 zod schema | 验证并品牌化持久化的位置状态 |

## 与其他模块的关系

- **`types.ts`**：状态类型 `TurnBoundaryProjection` 在此声明；`ProjectionDefinition` 泛型的 `'turnBoundary'` key 与之配对。
- **`index.ts`**：注册点与唯一消费入口 `turnBoundaryOf(session)`。
- **`dsh-session`**：`turn/start`、`turn/end`、`step/start`、`step/end` 是 session 持久事件；`SessionSeq` 品牌经 `brandNumber` 恢复。
- **`dsh-session-projection`**：`ProjectionDefinition` 接口的实现者之一；与 `inbox`（注册者是 agent-loop）、`modelSelection`（定义在 [model-selection.md](model-selection.md)）共享同一机制。
- **消费方**：任何需要"读到哪了"的路径——如恢复/附加/续写/导出工具定位追加点、UI 显示当前 turn 编号——用 `ctx.agents.turnBoundaryOf(session)`。

## 小结

- 60 行的纯折叠：四个事件进，四个位置字段出；`apply` 无副作用，default 落穿吞掉所有无关事件。
- 语义细节：`lastTurn` 是游标不清零；`lastStepStartSeq` 与 `lastStepBoundary` 各记各的（最新 step 开始 vs 最新边界方向）；`openTurnStartSeq` 为 null 表示 turn 之间。
- 所有权在注册表：消费者只认 `ctx.agents.turnBoundaryOf(session)`，投影注册与 key 都不外泄。
- 数字品牌在验证点恢复——持久化的是 number，内存里是 `SessionSeq`。
