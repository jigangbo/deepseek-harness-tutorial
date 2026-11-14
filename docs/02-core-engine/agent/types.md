# `types.ts` — Agent 静态类型
> 类型-only 面：Agent 身份、durable 事件词汇与投影类型


## 文件概览

`packages/core/agent/src/types.ts` 是本包的**类型-only 子入口**（模块标注 `@module @deepseek-ai/dsh-agent/types`，对应 `package.json` 的 `./types` 导出）。它的职责是把"可持久化、可跨类型-only 消费者共享"的词汇集中在一个不引入运行时代码的模块里：

- `Agent` 的基础面（只有 `id`；运行时成员由 `runtime-types.ts` 通过声明合并补上）；
- inbox 投影词汇：`InboxTarget`、`InboxState`、`InboxWireState`，以及 `SessionProjectionMap.inbox` 注册；
- `TurnBoundaryProjection`：Session 里已记录的 turn/step 位置；
- `ModelSelection` 与 `ModelSelectionProjection`，以及 `SessionProjectionMap.modelSelection` 注册；
- 两个 durable `SessionEventMap` 成员：`agent/inbox/spliced` 与 `model/selection`；
- 三组声明合并：Workspace 的 `SessionActivityKindMap.turn`、typert 的 `agent` lookup/context。

文件头一行 import 就点明了它的角色：

```typescript
// packages/core/agent/src/types.ts
import type {} from '@deepseek-ai/dsh-workspace/types'
// Type-only: the Workspace registry's archive-admission family map this registry merges `turn` into.
```

## 关键代码走读

### Agent 基础面

```typescript
// packages/core/agent/src/types.ts
/** Public live-agent handle; the runtime face augments its live capabilities. */
export interface Agent {
  /** Session-backed Agent identity. */
  readonly id: SessionId
}
```

类型-only 面上的 `Agent` 只有一个成员：`SessionId` 品牌化的 id——也就是 Session 日志的身份。运行时成员（`session`、`inbox`、`model`、`cancel` 等）在 `runtime-types.ts` 里用 `declare module './types.ts'` 合并进同一个接口（见 [runtime-types.md](runtime-types.md)）。这样纯类型消费者导入 `@deepseek-ai/dsh-agent/types` 时不拖入任何运行时依赖。

### 声明合并：把 `turn` 并入 Workspace 活动家族

```typescript
// packages/core/agent/src/types.ts
declare module '@deepseek-ai/dsh-workspace/types' {
  interface SessionActivityKindMap {
    /** The session's own Agent is inside a turn, including one waiting for an approval or an answer. */
    turn: true
  }
}
```

Workspace 注册表在隐藏/归档 Session 前会询问"还有什么在为这个 Session 运行"（归档准入 seam）。`SessionActivityKindMap` 是 merge-extensible 家族映射；本包把自己的 `turn` 家族并进去——含义是"该 Session 自己的 Agent 正在一个 turn 里，包括等审批或等回答的 turn"。实际回答这个问题的监听器在 [archive-admission.md](archive-admission.md)。

### typert 注册

```typescript
// packages/core/agent/src/types.ts
declare module '@deepseek-ai/dsh-typert-protocol' {
  interface TypertLookupMap {
    agent: TypertLookup<Agent, SessionId>
  }

  interface TypertContextMap {
    /** Agent Context identity shared by Host and Client adapters. */
    agent: TypertContext<SessionId>
  }
}
```

这两条让 typert 类型图知道：跨 wire 时 `Agent` 参数携带 `agentId`（wire 类型是 `SessionId`，资源类型是 session），`agent` Context 同理。对应的运行时注册在 `AgentRegistry` 构造函数里完成（`typert.lookups.register('agent', ...)` / `contexts.registerHost('agent', ...)`），把 `SessionId` 解析回活体 Agent 或其 scoped Context。

### inbox 投影词汇

```typescript
// packages/core/agent/src/types.ts
/** One of the two ordered pending-message lists owned by an agent. */
export type InboxTarget = 'next-turn' | 'next-step'

/** Complete pending Inbox value reconstructed from durable splices. */
export interface InboxState {
  readonly 'next-turn': readonly UserMessage[]
  readonly 'next-step': readonly UserMessage[]
}

/**
 * Wire-JSON pending Inbox value. Each message round-trips the session log
 * losslessly, but the fold state's full `UserMessage` type cannot cross a
 * typert Remote boundary (its source union carries an `unknown` replay
 * field), so the typed projection table keeps this JSON-safe form.
 */
export interface InboxWireState {
  readonly 'next-turn': readonly JsonValue[]
  readonly 'next-step': readonly JsonValue[]
}

declare module '@deepseek-ai/dsh-session-projection/types' {
  interface SessionProjectionMap {
    /** Pending agent input reconstructed from durable inbox splices. */
    inbox: InboxWireState
  }
}
```

Agent 拥有两条有序待处理消息列表：`next-turn`（等待各自 turn 的普通 prompt）与 `next-step`（等待下一个 step 边界的 steering/注入）。注意 `InboxState` 与 `InboxWireState` 的刻意的**对称但不同形**：内存态用完整 `UserMessage`，投影表用 JSON-safe 的 `JsonValue[]`——因为 `UserMessage` 的 source union 带 `unknown` replay 字段，过不了 typert Remote 边界；每条消息本身在日志里可无损往返。`SessionProjectionMap.inbox` 注册的是 wire 形态，投影 key 为 `inbox`，标准注册者是 dsh-agent-loop（见 [agent-loop 教程](../agent-loop/index.md)）。

### TurnBoundaryProjection

```typescript
// packages/core/agent/src/types.ts
/** Turn and step positions recorded in a Session. */
export interface TurnBoundaryProjection {
  /** Seq of the open turn's `turn/start`, or null between turns. */
  readonly openTurnStartSeq: OptionalSessionSeq
  /** Seq of the latest `step/start` event, or null before the first step. */
  readonly lastStepStartSeq: OptionalSessionSeq
  /** The latest step boundary (`step/start` or `step/end`) and its seq, or null before the first step boundary. */
  readonly lastStepBoundary: { readonly kind: 'start' | 'end'; readonly seq: SessionSeq } | null
  /** Turn number of the latest `turn/start`; 0 before the first turn. */
  readonly lastTurn: number
}
```

四个字段回答"日志走到哪了"：未关闭 turn 的 `turn/start` seq、最近一次 `step/start` seq、最近的 step 边界（含方向 start/end）与最新 turn 编号。消费入口是 `ctx.agents.turnBoundaryOf(session)`（注册表拥有投影注册，见 [projection.md](projection.md)）。

### ModelSelection 与客户端视图

```typescript
// packages/core/agent/src/types.ts
/** Complete provider, model, and optional reasoning effort chosen for one Session's model steps. */
export interface ModelSelection {
  /** Registered provider route. */
  provider: string
  /** Provider-owned model id. */
  model: string
  /** Adapter-owned reasoning effort, or provider/default behavior when absent. */
  reasoningEffort?: ReasoningEffortId
}

/** Client view of the durable model-selection fold. */
export interface ModelSelectionProjection {
  /** Selection used by the latest recorded model request. */
  readonly lastUsed: ModelSelection | null
  /** Selection the next model step uses; null while the Agent's initial options supply the route. */
  readonly next: ModelSelection | null
}

declare module '@deepseek-ai/dsh-session-projection/types' {
  interface SessionProjectionMap {
    /** Recorded model selection for the next model step and the latest request. */
    modelSelection: ModelSelectionProjection
  }
}
```

`ModelSelection` 是"一个 Session 的模型步骤所选的完整 provider/model/effort 三元组"。`reasoningEffort` 缺席是**一个值**（表示跟随 adapter 默认），不是"未设置"——比较语义见 [model-selection.md](model-selection.md) 的 `sameModelSelection()`。投影的客户端视图把内部 `(lastUsed, pending)` 对折成 `(lastUsed, next)`：`next = pending ?? lastUsed`。

### durable 事件词汇

```typescript
// packages/core/agent/src/types.ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    /**
     * One normalized mutation of an agent's durable pending-message lists.
     * The session-projection registry applies the committed event before
     * `Session.append()` returns; Inbox live notifications follow that commit.
     */
    'agent/inbox/spliced': {
      target: InboxTarget
      start: number
      removedCount?: number
      inserted: UserMessage[]
      outcome?: 'canceled'
    }
    /**
     * Complete model selection that applies from the next model step until a
     * later selection. Log-only: it never enters derived model history.
     */
    'model/selection': ModelSelection
  }
}
```

这两个事件是本模块"模型可见 ⟺ 已记录"纪律的载体：

- **`agent/inbox/spliced`**：inbox 的每次规范化变更（插入、claim、取消）都以标准 splice 语义写日志。`removedCount` 缺省表示"没有删除任何东西"；`outcome: 'canceled'` 把"取消"与"领取"区分开——这正是 [consumed-work.md](consumed-work.md) 区分"输入被消费"与"输入被丢弃未运行"的关键事实。投影注册表在 `Session.append()` 返回前应用已提交事件；inbox 的每条消息活体通知（`agent/inbox/inserted` 等）跟随该提交之后。
- **`model/selection`**：完整的下一次模型步骤选择。**Log-only**——它进入日志但从不进入派生模型历史（不消耗模型 token）。

## 核心类型与合并点

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `Agent` | `interface { readonly id: SessionId }` | 活体 Agent 的类型-only 基础面；运行时成员由 `runtime-types.ts` 合并 |
| `InboxTarget` | `'next-turn' \| 'next-step'` | 两条待处理消息列表的目标键 |
| `InboxState` | `{ 'next-turn': UserMessage[]; 'next-step': UserMessage[] }` | 从 durable splice 重建的完整待处理 inbox 值（内存态） |
| `InboxWireState` | 同形但 `JsonValue[]` | 投影表的 wire-JSON 形态（typert Remote 安全） |
| `TurnBoundaryProjection` | 4 个位置字段 | Session 里已记录的 turn/step 位置 |
| `ModelSelection` | `{ provider, model, reasoningEffort? }` | 一个 Session 模型步骤的完整选择 |
| `ModelSelectionProjection` | `{ lastUsed, next }` | model-selection 投影的客户端视图 |
| `agent/inbox/spliced` | `SessionEventMap` 成员 | inbox 每次规范化变更的 durable 事件 |
| `model/selection` | `SessionEventMap` 成员 | durable 模型选择事件（log-only） |
| `SessionActivityKindMap.turn` | Workspace 声明合并 | 归档准入的 `turn` 活动家族 |
| `TypertLookupMap.agent` / `TypertContextMap.agent` | typert 声明合并 | `Agent`/`agent` Context 的 wire 映射（`SessionId`） |

## 与其他模块的关系

- **`runtime-types.ts`**：用 `declare module './types.ts'` 把 `options`/`session`/`model`/`inbox`/`status`/`ctx` 与方法成员合并进这里的 `Agent` 接口。两个文件共同构成一个接口的"类型-only 面 + 运行时面"。
- **`model-selection.ts`**：`AgentModel` 实现 `Model`（`Model` 接口本体在 `runtime-types.ts`），其投影定义产出 `ModelSelectionProjection` 视图；`model/selection` 事件在这里声明、在那里写入。
- **`projection.ts`**：`turnBoundary` 投影定义产出 `TurnBoundaryProjection` 状态。
- **`consumed-work.ts`**：直接读 `agent/inbox/spliced` 事件的 `removedCount`/`outcome` 字段做工作记账。
- **`archive-admission.ts`**：运行时回答 `SessionActivityKindMap.turn` 家族的准入问题。
- **`dsh-session` / `dsh-session-projection` / `dsh-workspace` / `dsh-typert-protocol`**：四处声明合并的目标模块。
- **`dsh-agent-loop`**：`inbox` 与 `modelSelection` 投影的标准注册者；`agent/inbox/spliced` 的标准写入者。

## 小结

- 本文件是 `@deepseek-ai/dsh-agent/types` 子入口：不引运行时代码，专供类型-only 消费者。
- `Agent` 基础面只有 `id`；完整成员靠声明合并补齐——接口的所有权在类型-only 面。
- 两条 durable 事件（`agent/inbox/spliced`、`model/selection`）是本包的持久化词汇；`outcome: 'canceled'` 与 log-only 语义是下游折叠（consumed-work、model-selection 投影）依赖的事实。
- 三个投影（`inbox`、`modelSelection`）与一个合并家族（`turn`）通过声明合并注册到各自的映射表，无需改动拥有这些映射的包。
