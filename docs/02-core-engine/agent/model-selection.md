# `model-selection.ts` — 模型选择投影
> AgentModel、durable 模型选择投影与切换通知


## 文件概览

`packages/core/agent/src/model-selection.ts`（模块标注 `@module @deepseek-ai/dsh-agent/model-selection`）实现 `agent.model`（`Model` 接口）背后的三件事：

1. **`AgentModel`**：读取并记录一个 Agent 的模型选择，不解析 adapter 默认；
2. **`modelSelectionProjectionDefinition`**：把 `model/selection` 与 `request/header` 事件折叠成持久状态 `lastUsed`/`pending`（客户端视图 `lastUsed`/`next`）；
3. **`installModelSelection(agentCtx)`**：可选的作用域监听器，在被选 Agent 的 step 里追加 `[model changed: ...]` 切换通知。

文件头一句话给出全局角色：

```typescript
// packages/core/agent/src/model-selection.ts
/** Agent-owned model selection, its durable projection, and scoped switch notices. */
```

## 关键代码走读

### 投影状态与比较原语

```typescript
// packages/core/agent/src/model-selection.ts
export interface ModelSelectionProjectionState {
  /** Selection used by the latest recorded model request. */
  readonly lastUsed: ModelSelection | null
  /** Later recorded selection not yet used by a matching model request. */
  readonly pending: ModelSelection | null
}
```

内部状态是一对：`lastUsed`（最近一次模型请求实际用的选择）与 `pending`（已记录但尚未被匹配请求使用的后续选择）。注意客户端视图（`ModelSelectionProjection`，见 [types.md](types.md)）把这对折成 `lastUsed`/`next`——`next = pending ?? lastUsed`。

三个比较/构造辅助：

```typescript
// packages/core/agent/src/model-selection.ts
function toSelection(provider: string, model: string, reasoningEffort: ReasoningEffortId | undefined): ModelSelection {
  return { provider, model, ...reasoningEffort === undefined ? {} : { reasoningEffort } }
}

function sameRoute(left: ModelSelection, right: ModelSelection): boolean {
  return left.provider === right.provider && left.model === right.model
}

function sameModelSelection(left: ModelSelection | null, right: ModelSelection | null): boolean {
  return left === right || (left !== null && right !== null
    && sameRoute(left, right) && left.reasoningEffort === right.reasoningEffort)
}
```

- `toSelection` 构造 detached 选择，缺省 effort 时**整个字段缺席**（不是 `undefined` 值）。
- `sameModelSelection` 的关键语义：`reasoningEffort` 字段的缺席与显式 `undefined` 都按**值相等**比较（`left.reasoningEffort === right.reasoningEffort`）——"缺席"是一个独立取值，不等于任何显式 effort，这与 `ModelSelection` 的契约一致（省略 effort = 跟随 adapter 默认）。

```typescript
// packages/core/agent/src/model-selection.ts
/** Selection a recorded request used, omitting an effort the adapter defaulted. */
function selectionOfHeader({ config, adapterDefaults }: EpochHeader): ModelSelection {
  return toSelection(config.provider, config.model, adapterDefaults?.reasoningEffort === true ? undefined : config.reasoningEffort)
}
```

从请求头提取"这次请求实际用的选择"：当 adapter 标记 `reasoningEffort: true`（表示默认被采用了）时省略 effort。这让**adapter 默认掉的 effort 不会被钉死成选择**——后续请求会重新解析默认，而不是把某次的默认值固化。这是文件头接口 JSDoc 的原话承诺。

### 投影定义：选择如何被请求"退休"

```typescript
// packages/core/agent/src/model-selection.ts
const selectionSchema = z.object({
  provider: z.string().min(1),
  model: z.string().min(1),
  reasoningEffort: z.string().min(1).transform(ReasoningEffortId).exactOptional(),
}).nullable()

export const modelSelectionProjectionDefinition: Required<ProjectionDefinition<'modelSelection', ModelSelectionProjectionState>> = {
  key: 'modelSelection',
  stateVersion: 3,
  stateSchema: z.object({ lastUsed: selectionSchema, pending: selectionSchema }),
  init: () => ({ lastUsed: null, pending: null }),
  apply: (state, event) => {
    if (event.type === 'model/selection') {
      return sameModelSelection(state.pending, event.data) ? state : { ...state, pending: event.data }
    }
    if (event.type !== 'request/header') return state
    const lastUsed = selectionOfHeader(event.data.header)
    const pending = sameModelSelection(state.pending, lastUsed) ? null : state.pending
    return sameModelSelection(state.lastUsed, lastUsed) && pending === state.pending ? state : { lastUsed, pending }
  },
  wire: {
    viewSchema: z.object({ lastUsed: selectionSchema, next: selectionSchema }),
    view: state => ({ lastUsed: state.lastUsed, next: state.pending ?? state.lastUsed }),
  },
}
```

折叠规则两条：

- **`model/selection` 事件** → 成为 `pending`。与现有 pending 相同则返回原 state（引用不变，避免无意义的状态更新）。
- **`request/header` 事件** → 从头部提取 `lastUsed`；若 `pending` 与刚用的选择相同，则它被"退休"（清空）；两处都无变化时返回原 state。

schema 用 zod 定义（`stateVersion: 3` 是投影自身的持久化版本号，非 session 格式版本）；`ReasoningEffortId` 品牌由 transform 恢复。`wire.view` 产出客户端视图：`next = pending ?? lastUsed`——两者皆空（未记录过选择也未请求过）时 `next` 为 null，此时路由由 Agent 初始 options 提供。

### AgentModel：读取与记录

```typescript
// packages/core/agent/src/model-selection.ts
export class AgentModel implements Model {
  constructor(
    private readonly readState: () => ModelSelectionProjectionState,
    private readonly session: Session,
    private readonly options: AgentOptions,
  ) {}

  current(): Readonly<ModelSelection> | undefined {
    const recorded = this.recordedSelection()
    if (recorded !== null) return { ...recorded }
    const { provider, model, reasoningEffort } = this.options
    return provider && model ? toSelection(provider, model, reasoningEffort) : undefined
  }

  select(selection: Readonly<ModelSelection>): void {
    if (sameModelSelection(this.recordedSelection(), selection)) return
    this.session.append('model/selection', toSelection(selection.provider, selection.model, selection.reasoningEffort))
  }

  private recordedSelection(): ModelSelection | null {
    const state = this.readState()
    return state.pending ?? state.lastUsed
  }
}
```

- 构造参数体现所有权分工：`readState` 是**驱动器拥有**的投影读取器；`session` 承接持久写入；`options` 是记录出现前的路由回退。实例不持有选择的第二份副本——全部真相在日志 + 投影里。
- `current()` 优先级：记录选择（`pending ?? lastUsed`）> live options；options 侧必须同时有 `provider` 与 `model` 才返回，否则 `undefined`。返回 `{ ...recorded }` detached 副本——调用者改不动内部状态。
- `select()` 的两条守则：与已记录选择相同是 **no-op**（注意比较对象是记录值而非 live options——README 说明了推论：与初始 options 相同的选择也会被记录为显式 `model/selection`）；写入直接 `session.append('model/selection', ...)`，沿用 Session 既有的持久化与失败保证。省略 effort 意味着"下次跟随 adapter 默认"，**不继承**上一次的 effort。

### 切换通知：installModelSelection

先看消息构造：

```typescript
// packages/core/agent/src/model-selection.ts
declare module '@deepseek-ai/dsh-llm' {
  interface MessageSourceMap {
    'model-selection': { kind: 'model-selection' } & ContextFormed
  }
}

function routeLabel(route: ModelSelection, other: ModelSelection): string {
  return route.provider === other.provider ? route.model : `${route.provider}/${route.model}`
}

function modelSwitchNotice(previous: ModelSelection, selected: ModelSelection) {
  const from = routeLabel(previous, selected)
  const to = routeLabel(selected, previous)
  return createUserMessage({
    content: [{
      type: 'text',
      text: `[model changed: assistant turns above this point were generated by ${from}; the session continues with ${to}]`,
    }],
    source: {
      kind: 'model-selection',
      form: 'notice',
      summary: boundContextSummary(`${from} → ${to}`),
    },
  })
}
```

通知文本只在跨 provider 时带 provider 前缀（`routeLabel` 对比另一个选择）。source 是新增的 `model-selection` 消息源（merge-extensible `MessageSourceMap` 的又一条扩展），带 `ContextFormed` 元数据（`form: 'notice'` 与 summary）供 UI 归类展示。

监听器本体：

```typescript
// packages/core/agent/src/model-selection.ts
export function installModelSelection(agentCtx: Context): () => void {
  return agentCtx.on(
    'agent/pre-step',
    async ({ agent, messages, signal, step, modelSelection }, next): Promise<PreStepDecision> => {
      const decision = await next()
      if (decision.kind === 'reject' || signal.aborted) return decision
      // The loop skips an empty first step and an emptied offered continuation.
      if (decision.messages.length === 0 && (step === 1 || messages.length > 0)) return decision
      const previous = agent.session.requestHeader()?.config
      if (
        previous === undefined
        || (modelSelection.provider === previous.provider && modelSelection.model === previous.model)
      ) return decision
      return { ...decision, messages: [...decision.messages, modelSwitchNotice(previous, modelSelection)] }
    },
    { prepend: true },
  )
}
```

逐条守则：

- **包装型监听器**：先 `await next()` 拿到链的最终决定，再决定要不要追加——不改变准入语义，只改进入 step 的消息批。
- **reject 或已取消**：原样返回，通知无从谈起。
- **空 step 短路**：循环会跳过空的第一个 step 与被清空的续接 step（`messages.length === 0 && (step === 1 || messages.length > 0)`）——这些不产生请求，追加通知只会让下一步误报。
- **同路由不加通知**：`previous === undefined`（本会话还没有请求头——初始请求）或 provider/model 都相同（**effort-only 变化不算切换**）时返回原决定。
- **追加而非替换**：`{ ...decision, messages: [...decision.messages, notice] }`——通知作为保留的 user 消息追加到已接受的批里，历史只增不减。
- **`{ prepend: true }`**：挂到链首，保证"读 previous 头 + 决定追加"发生在其他 pre-step 监听者加工之前；scope 归 `agentCtx` 所有，返回 disposer，Agent 作用域拆除时监听器随之注销。

一个诚实的失败模式写在 JSDoc 里：若通知追加后、请求头持久化之前失败，下一步会**重复**通知——因为 `previous` 还是旧路由。这是持久语义的自然结果，不是 bug。

## 核心导出速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ModelSelectionProjectionState` | `{ lastUsed: ModelSelection \| null; pending: ModelSelection \| null }` | 投影内部状态 |
| `modelSelectionProjectionDefinition` | `Required<ProjectionDefinition<'modelSelection', ...>>` | 折叠 `model/selection` 与 `request/header`；wire 视图 `next = pending ?? lastUsed` |
| `AgentModel` | `implements Model`，构造 `(readState, session, options)` | `current()` 优先级读取；`select()` 记录显式选择 |
| `installModelSelection` | `(agentCtx: Context) => () => void` | 作用域 pre-step 监听器：跨路由 step 追加 `[model changed: ...]` 通知 |
| `modelSwitchNotice` / `routeLabel` | 模块私有 | 构造通知消息；跨 provider 才带 provider 前缀 |
| `selectionOfHeader` / `toSelection` / `sameRoute` / `sameModelSelection` | 模块私有 | 头部提取（省略 adapter 默认 effort）、构造、路由/整体比较 |

## 与其他模块的关系

- **`types.ts`**：`ModelSelection`/`ModelSelectionProjection` 与 `model/selection` 事件在此消费；`SessionProjectionMap.modelSelection` 注册视图类型。
- **`runtime-types.ts`**：`Model` 接口在此实现；`agent/pre-step` 事件在此被监听。
- **`dsh-session`**：`model/selection` 事件写入、`request/header` 事件读取、`session.requestHeader()` 查询都走 Session。
- **`dsh-session-projection`**：投影定义的宿主机制；标准注册者是 `dsh-agent-loop`（README：默认驱动器创建每个 `AgentModel` 并拥有投影注册）。
- **`dsh-agent-loop`**：捕获 detached 选择于 step 准备之前并跨重试保留（`AssembleContext.modelSelection` 的来源）；`installModelSelection` 的安装者（Web/ACP 作用域切换入口）。
- **UI**：读 `modelSelection` 投影的 `lastUsed`/`next` 展示当前与下次路由，无需活体 Agent。

## 小结

- 模型选择的真相只存一份：日志里的 `model/selection` 与 `request/header` 事件，经投影折叠成 `lastUsed`/`pending`；`AgentModel` 只是读取器 + 追加器。
- 缺席 effort 是一个值：adapter 默认掉的 effort 不被钉死，缺席与显式值在 `sameModelSelection` 下按位相等比较。
- `select()`：整组替换、同记录 no-op、effort 不继承；比较对象是记录值，所以与初始 options 相同也会显式记录。
- 切换通知是作用域可选件：包装 pre-step、同路由与空 step 不加、追加不替换、prepend 到链首、随 agentCtx 注销。
