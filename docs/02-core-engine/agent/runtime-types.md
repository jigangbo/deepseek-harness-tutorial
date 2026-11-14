# `runtime-types.ts` — 运行时类型与事件
> Agent 运行时面与全部 `agent/*` 事件声明


## 文件概览

`packages/core/agent/src/runtime-types.ts`（模块标注 `@module @deepseek-ai/dsh-agent`）是本包的**运行时类型面**，也是整个 `agent/*` 事件词汇的权威声明处。内容分四块：

1. **Agent 选项与取消**：`AgentOptions`、`CancelOptions`、`AgentCancelCause` 再导出。
2. **Agent 拥有的访问面**：`Model`（模型选择）、`Inbox`（待处理工作）、`AgentStatus`、完整的 `Agent` 成员合并。
3. **事件负载类型**：`PreStepDecision`、`RequestErrorAction`、`SessionStartSource`、`AssistantStreamFrame`。
4. **`agent/*` 事件声明**：在 `declare module '@deepseek-ai/cordis'` 中合并进 Cordis `Events`，覆盖生命周期、inbox 通知、扩展点瀑布与错误通知。

文件头就划定了与 `dsh-session` 的边界：

```typescript
// packages/core/agent/src/runtime-types.ts
/**
 * Public agent types and live-runtime events. Durable transcript facts and
 * turn/step boundaries remain `@deepseek-ai/dsh-session` events.
 */
```

即：`agent/*` 是**活体**事件（进程内、scope 过滤、不落日志）；turn/step 边界、消息内容等持久事实是 `dsh-session` 事件（落日志）。

## 关键代码走读

### AgentOptions：路由回退与输出上限

```typescript
// packages/core/agent/src/runtime-types.ts
/**
 * Merge-extensible agent options. Route fields may be live accessors until the
 * Session records a selection or request; step preparation captures their value.
 * Persona belongs to system-prompt sections.
 */
export interface AgentOptions {
  /** Provider route (must have a registered adapter at call time). */
  provider?: string
  /** Model id interpreted by the selected provider adapter. */
  model?: string
  /** Adapter-owned reasoning effort for the selected provider/model route. */
  reasoningEffort?: ReasoningEffortId | undefined
  /** Maximum output tokens for each conversation-model request. */
  maxTokens?: number
}
```

四个字段都是**回退值**：一旦 Session 记录了选择（`model/selection`）或请求头，记录值优先，live options 只在没有记录时供路由。`provider`/`model` 是 step 准备（prompt 组装前）的硬要求；`maxTokens` 必须是正的安全整数，封顶每次对话模型请求。Persona 不在这里——它属于 system-prompt 段落。

### Agent 完整成员：声明合并补齐运行时面

```typescript
// packages/core/agent/src/runtime-types.ts
declare module './types.ts' {
  interface Agent {
    /** Route fallback and output controls; live accessors apply until the Session records a selection or request. */
    readonly options: AgentOptions
    /** The live session this agent drives; its log is the durable source of truth. */
    readonly session: Session
    /** Agent-owned access to durable model selection for future steps. */
    readonly model: Model
    /** Agent-owned access to durable pending work. */
    readonly inbox: Inbox
    /** The current lifecycle state, mirrored on every `agent/status` transition. */
    readonly status: AgentStatus
    /** Agent-scoped context; its contributions are agent-local, unwind on disposal, and reject registration afterward. */
    readonly ctx: Context
    // ...方法成员见下
  }
}
```

合并的目标正是 [types.md](types.md) 里的类型-only `Agent`。六个属性与方法构成消费者能对活体 Agent 做的一切：

**输入投递（四个入口，一条统一路由）：**

```typescript
// packages/core/agent/src/runtime-types.ts
/**
 * Route identified input to an inbox boundary and optionally wake the driver.
 * ...
 */
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void

/** Queue an ordinary follow-up turn and wake the driver. */
followup(message: UserMessage): void

/** Submit steering for the nearest step. */
steer(message: UserMessage): void

/** Queue model-facing context for the next pre-step without waking the driver. */
inject(message: UserMessage): void
```

`send` 是统一形式，三个固定预设是它的别名。投递路径语义（target/wakeup 组合）：

| 入口 | target | wakeup | 典型效果 |
|---|---|---|---|
| `followup()` | `next-turn` | `true` | 排队一个普通后续 turn 并唤醒驱动；该项成为其 turn 的唯一普通消息 |
| `steer()` | `next-step` | `true` | 空闲驱动开新 turn；运行中驱动在下一个 step 边界消费；被拒绝的 step 会把 steering 留在 inbox 直到下次唤醒 |
| `inject()` | `next-step` | `false` | 加入模型可见上下文但**不唤醒**驱动；运行中驱动在最近的后续 step 边界领取，空闲驱动保持 pending 直到 followup/steer 唤醒 |
| `send()` | 显式 | 显式 | 直接暴露 target 与 wakeup 的组合 |

三个预设共同的语义：每条 `UserMessage` 携带内容与 `MessageSource`（供应者归属）；取消后提交的唤醒输入排队等下一个 turn；对空闲 Agent 的唤醒**总是**打开 turn 边界（即使消息随后被清掉——驱动唤醒收敛契约）。

**取消与静默：**

```typescript
// packages/core/agent/src/runtime-types.ts
cancel(cause: AgentCancelCause, options?: CancelOptions): void
whenIdle(): Promise<void>
runMaintenance<T>(task: (signal: AbortSignal) => Promise<T>): Promise<T>
```

- `cancel(cause)`：清空排队与 steering 工作（除非 `keepInbox: true`）并中止活动 turn 或 turn 间任务。第一个 cause 赢得该活动；没有活动活动时取消是 no-op，不会"预武装备"后来的工作。`cause` 是稳定调用者意图（`user`/`parent`/`hook`/`disposed`，定义在 `dsh-session`），由本文件再导出：
  ```typescript
  export type { AgentCancelCause } from '@deepseek-ai/dsh-session'
  ```
  `CancelOptions.keepInbox` 保留排队项：活动 turn 仍被中止，但未开始的与 pending 的输入活下来等以后的 turn，且不写 canceled inbox splice。
- `whenIdle()`：整个 Agent 达到静默后 resolve；跟随被观察驱动退休前开始的替代工作，但不对应任何特定 message 的 settle。
- `runMaintenance(task)`：从真空闲相运行一个非 turn 维护任务；唤醒输入留在 inbox 直到任务结束，公开 status 保持 `idle`。turn 驱动或另一维护任务已占用时同步抛错。

### Inbox：待处理工作的结构性接口

```typescript
// packages/core/agent/src/runtime-types.ts
/** Agent-owned access to pending work; concrete storage belongs to the driver. */
export interface Inbox {
  /** Prompts awaiting individual turns. */
  readonly nextTurn: readonly UserMessage[]
  /** Input awaiting the next step boundary. */
  readonly nextStep: readonly UserMessage[]

  /** Durably cancel all pending input, clearing next-step before next-turn. */
  clear(): void
  append(target: InboxTarget, message: UserMessage): void
  prepend(target: InboxTarget, message: UserMessage): void
  replace(messageId: MessageId, newMessage: UserMessage): boolean
  remove(messageId: MessageId): boolean
  splice(target: InboxTarget, start: number, deleteCount: number, inserted: UserMessage[]): UserMessage[]
}
```

结构化接口 + 持久化命令：每个方法对应一条规范化 `agent/inbox/spliced` 事件。普通移除与 `clear()` 是**持久取消**；`replace`/`remove` 跨两条列表按 `MessageId` 定位，替换可换身份并先发旧消息 discarded 再发新消息 inserted。具体存储属于驱动器（dsh-agent-loop 的包内 `ReactLoopInbox`）；step 边界的 claim 是循环内部的纯删除 splice，**不在**这个公开接口里。

### Model：未来步骤的选择权

```typescript
// packages/core/agent/src/runtime-types.ts
/** Agent-owned access to the model choice for future steps. */
export interface Model {
  current(): Readonly<ModelSelection> | undefined
  select(selection: Readonly<ModelSelection>): void
}
```

`current()` 解析"下一个将开始准备的步骤"的选择：记录选择 > 最近请求 > live options；没有任何来源同时给出 provider 与 model 时返回 `undefined`。读取既不启动步骤也不改变已准备的步骤。`select()` 整体替换下一次完整选择：省略 effort 跟随 adapter 默认（不继承旧 effort、不在此解析默认）；与已记录选择相同是 no-op；与初始 options 相同也会被记录（成为显式 `model/selection` 事件）。模型可用性与能力校验归调用者。具体实现在 [model-selection.md](model-selection.md) 的 `AgentModel`。

### AgentStatus 与 PreStepDecision

```typescript
// packages/core/agent/src/runtime-types.ts
export type AgentStatus = 'idle' | 'running'

/** Whether and with which messages the loop enters a proposed step. */
export type PreStepDecision =
  | { kind: 'reject' }
  | {
    kind: 'enter'
    messages: UserMessage[]
    /** Start a distinct model-message series before this step's admitted messages. */
    startsRequestSeries?: true
  }
```

`running` 描述驱动级 drain 区间，可以横跨连续排队的 turn，不证明 turn 仍开着；拆除把 Agent 移出注册表，**不是**第三个可观察状态。`PreStepDecision` 是 `agent/pre-step` 瀑布的返回类型：`reject` 不开 step；`enter` 携带**完整的**已识别、冻结消息批（接受不等于提交——组装与 `step/start` 之后、提交系统 prompt 与用户批之前还有异步相，取消发生在其间则两者都不提交）。`startsRequestSeries: true` 声明一个独立的 model-message 系列；包装型监听者除非有意替换，应保留该声明与消息批。

### SessionStartSource 与 AssistantStreamFrame

```typescript
// packages/core/agent/src/runtime-types.ts
/** Why a session lifecycle began; seeded creates are `startup`, while persisted loads are `resume`. */
export type SessionStartSource = 'startup' | 'resume' | 'clear' | 'compact'

/** One process-local live assistant streaming publication. */
export type AssistantStreamFrame =
  | { readonly type: 'start'; readonly attemptId: LlmAttemptId; readonly revision: number; readonly turn: number; readonly step: number }
  | { readonly type: 'chunk'; readonly attemptId: LlmAttemptId; readonly revision: number; readonly index: number; readonly time: number; readonly chunk: StreamChunk }
  | { readonly type: 'end'; readonly attemptId: LlmAttemptId; readonly revision: number; readonly index: number; readonly outcome: ... }
```

`SessionStartSource` 是 `agent/created` 的来源标记（bridge 按它键控 SessionStart 匹配器）。`AssistantStreamFrame` 是一次进程内 Assistant 尝试的有序帧：`start` 命名 turn/step，`revision` 在同一 Agent 生命周期内单调（替换重启为 1）；`chunk.index` 从 0 稠密递增；`end.index` 是下一 chunk 位置，`outcome` 区分 `committed`（持久 settlement 已在通知前提交，携带 `assistant/message` 或 `assistant/attempt` 事件类型与 seq）与 `abandoned`（没有 settlement 的活体放弃）。**活体流只是呈现数据**：循环在提交 end 帧之前把完整压缩流作为一个持久事件写入，回放的真相在日志里。

### AssembleContext 合并

```typescript
// packages/core/agent/src/runtime-types.ts
declare module '@deepseek-ai/dsh-system-prompt' {
  interface AssembleContext {
    /** Agent for this assembly; absent on diagnostics. When present, `scope` must identify the same agent. */
    agent?: Agent
    /** Captured step selection, retained across retries; omitted effort uses adapter defaults. Absent outside agent steps. */
    modelSelection?: Readonly<ModelSelection>
  }
}
```

prompt 组装上下文带上 Agent 与该步捕获的选择（跨重试保留），使 agent-scoped prompt 段落与工具能在组装时解析。配合 [dispatch.md](dispatch.md) 的 `assembleContextFor()` 一起使用。

### `agent/*` 事件全景

声明合并进 Cordis `Events`，每个都标注 dispatch mode 与 scope 过滤（`this: Scoped<Agent>`，由 [dispatch.md](dispatch.md) 的融合派发器保证 subject 与 scope key 不分离）：

| 事件 | mode | 语义要点 |
|---|---|---|
| `agent/created` | `serial` | 工厂 setup 完成后按序 await；监听器抛错则创建失败并跳过后续监听器；**不得** await `agent.whenIdle()` 或自己 owner 的 dispose |
| `agent/disposed` | `emit` | 驱动静默、scoped 注册展开之后、Session detach 之前发出 |
| `agent/status` | `emit` | `idle` ⇄ `running` 翻转；唤醒投递在预约取消后同步进入 `running` |
| `agent/inbox/inserted` | `emit` | 一条消息进入活体 inbox |
| `agent/inbox/claimed` | `emit` | 一条消息在其打开的 turn 内离开 inbox；step 被拒绝时被领的消息到此为止——不丢弃也不重发为 `user/message`，turn 无 step 关闭 |
| `agent/inbox/discarded` | `emit` | 一条消息被丢弃 |
| `agent/pre-step` | `waterfall` | 拒绝提议的 step 或替换进入它的消息；调用 `next()` 保留当前消息（不调用 `next()` 的瀑布监听会短路链条——必须委托） |
| `agent/request` | `waterfall` | **已弃用**，仅为既有监听者保留；可调控制项但必须保留 provider/model/effort；新代码用 `agent.model.select()` 与 `AgentOptions.maxTokens` |
| `agent/request-error` | `waterfall` | 失败的模型请求尝试在循环重试/关 step 前到达这里；拥有恢复的监听者**不调用 `next()`** 并返回 `{ kind: 'retry' }`，默认 `undefined` 让失败终结 |
| `agent/assistant-stream` | `emit` | 进程内流帧发布；chunk 帧是瞬态的，持久真相是随后的 `assistant/message`/`assistant/attempt` |
| `agent/turn-stopping` | `serial` | turn 即将关闭（无活跃 tool call、无新 steering）时 await；监听者通过 `agent.steer(...)` 反对即保持 turn 打开——由数据决定，监听者顺序无法改变结果 |
| `agent/error` | `emit` | step/turn 出错；即使错误没有 turn 内位置可供持久记录也在这里上报 |

`agent/turn-stopping` 的"数据决定"值得一读：

```typescript
// packages/core/agent/src/runtime-types.ts（事件 JSDoc 摘录）
/**
 * The turn is about to close: the model owes no response (no live tool
 * calls, no fresh steering). Awaited before the boundary commits — a
 * listener that objects steers (`agent.steer(...)`) and the machine
 * re-reads its inbox: fresh steering runs another step, none closes the
 * turn. Data decides, so listener order cannot change the outcome.
 */
'agent/turn-stopping'(this: Scoped<Agent>, payload: { agent: Agent; turn: number; signal: AbortSignal }): Promise<void> | void
```

监听者不返回"否决值"，而是往 inbox 里投一条 steering；机器重读 inbox——有新 steering 就再跑一个 step，没有就关 turn。反向控制（提前结束 tool 循环）同样是数据：携带 `concludesTurn` 的 tool result 在其 step 结束 turn。

## 核心类型速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `AgentOptions` | `{ provider?; model?; reasoningEffort?; maxTokens? }` | 路由回退与输出上限；可变访问者在记录前生效 |
| `CancelOptions` | `{ keepInbox? }` | 取消时保留排队与 steering 输入 |
| `Model` | `current() / select()` | Agent 拥有的未来步骤模型选择 |
| `Inbox` | `nextTurn/nextStep` + 6 个命令 | Agent 拥有的持久待处理工作访问面 |
| `AgentStatus` | `'idle' \| 'running'` | 驱动级生命周期状态 |
| `PreStepDecision` | `reject \| enter{messages, startsRequestSeries?}` | pre-step 瀑布的准入决定 |
| `RequestErrorAction` | `{ kind: 'retry' } \| undefined` | request-error 瀑布的恢复声明 |
| `SessionStartSource` | `'startup' \| 'resume' \| 'clear' \| 'compact'` | 会话生命周期起点来源 |
| `AssistantStreamFrame` | `start \| chunk \| end` | 进程内流式发布帧 |
| `agent/*` 事件 | 12 个 Cordis `Events` 成员 | 活体事件词汇（见上表） |

## 与其他模块的关系

- **`types.ts`**：本文件合并进其 `Agent` 接口，并消费其 `InboxTarget`/`ModelSelection`。
- **`dispatch.ts`**：`AgentSubjectEvent` 类型过滤器以"`this: Scoped<Agent>` 且 payload 携带 `agent`"为本文件声明的事件的形态契约；融合派发器据此为每个事件生成正确的发射面。
- **`dsh-session`**：`AgentCancelCause`、`Session`、`SessionSeq`、`UserMessage` 来自这里；持久事实（turn/end 原因等）留在 session 侧。
- **`dsh-system-prompt`**：`AssembleContext` 合并让 agent-scoped 组装成为可能。
- **`dsh-agent-loop`**：本文件声明的每个事件、每个接口的**实现者**；驱动器用 `agentEvents()` 派发事件、提供 `Inbox`/`Model` 的具体实现。
- **UI/插件**：按 scope 过滤监听 `agent/*`；`agent/request` 已弃用不得新增监听。

## 小结

- 本文件把 `Agent` 从"只有 id"补齐为完整运行时面，并声明全部 12 个 `agent/*` 事件——扩展插件作用活体工作的全部扩展点。
- 输入投递四入口一条统一路由（`send` 为宗，三个预设为别名）；取消是能力明确的整体操作（`cause` + `keepInbox`）。
- `PreStepDecision.enter` 携带完整冻结消息批；接受 ≠ 提交，中间的异步相都可被取消打断。
- `agent/turn-stopping` 用"数据决定"（steer 注入）而非监听者返回值表达否决；`agent/request` 已弃用。
- 活体流事件只是呈现数据；持久真相永远在 Session 日志。
