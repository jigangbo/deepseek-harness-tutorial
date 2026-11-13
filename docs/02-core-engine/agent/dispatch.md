# `dispatch.ts` — 融合派发器：agent subject 与 scope carrier 合一

## 文件概览

`packages/core/agent/src/dispatch.ts`（模块标注 `@module @deepseek-ai/dsh-agent/dispatch`）解决一个具体问题：**`agent/*` 事件的 subject（payload 里的 `agent` 字段）与 scope carrier（事件的 `this` 上的过滤键）必须永远一致**。如果由调用方分别传两者，迟早会出现 subject 是 Agent A、scope 键是 Agent B 的派发。

文件提供四个导出：

- `agentCarrier(agent)` —— 构造 scope 载体；
- `agentEvents(ctx, agent, carrier?)` —— 融合派发器（`emit`/`serial`/`waterfall` 三方法）；
- `emitAgentEvent(ctx, agent, name, payload)` —— 一次性包含式通知；
- `assembleContextFor(agent, signal?)` —— prompt 组装上下文构造。

模块头注明了使用模式：

```typescript
// packages/core/agent/src/dispatch.ts
/**
 * The fused dispatcher {@link agentEvents} couples the agent subject to its
 * scope carrier, so the scope key and the payload's `agent` cannot diverge;
 * repeat dispatchers (the loop driver) build it once in the agent's
 * constructor and reuse it.
 */
```

## 关键代码走读

### AgentSubjectEvent：类型层筛出"subject 是 Agent 的事件"

```typescript
// packages/core/agent/src/dispatch.ts
export type AgentSubjectEvent = {
  [K in keyof Events]: Events[K] extends (this: Scoped<Agent>, ...args: infer P) => unknown
    ? P extends [infer Payload, ...unknown[]]
      ? Payload extends { agent: Agent } ? K : never
      : never
    : never
}[keyof Events]
```

这个映射类型遍历 Cordis `Events` 的全部事件名，用**两个条件**同时卡住：

1. 处理器签名声明了 `this: Scoped<Agent>`（scope-carrier 契约）；
2. 第一个参数（payload）携带 `agent: Agent` 字段。

`this` 检查是刻意的防御：只检查 payload 的形态，会让"payload 碰巧带了个 Agent 字段"的事件（或零参事件——其参数元组通过可调用性满足裸 rest-tuple 检查）混进融合派发面。运行时 `agent/*` 事件全都满足这两个条件，所以 `AgentSubjectEvent` 正好是它们的联合键。

### agentCarrier：无状态路由对象

```typescript
// packages/core/agent/src/dispatch.ts
export function agentCarrier(agent: Agent): Scoped<Agent> {
  return scopeTarget(agent, agent)
}
```

`scopeTarget(subject, key)` 来自 `dsh-scope`——这里 subject 与 key 传同一个 Agent，从根上消除了分离的可能。载体是无状态的，`agentEvents` 接受现成 carrier 参数，让重复派发者（loop 驱动器）在 Agent 构造函数里建一次、热路径复用，派发零分配。

### agentEvents：三个派发模式

```typescript
// packages/core/agent/src/dispatch.ts
export function agentEvents(ctx: Context, agent: Agent, carrier: Scoped<Agent> = agentCarrier(agent)): AgentEventDispatch {
  const fused = <K extends AgentSubjectEvent>(payload: PayloadRest<K>): PayloadOf<K> =>
    ({ ...payload, agent } as PayloadOf<K>)
  return {
    emit(name, payload) { /* 见下 */ },
    async serial(name, payload) {
      const serial = ctx.serial as (thisArg: Scoped<Agent>, name: string, ...args: unknown[]) => Promise<never>
      return await serial(carrier, name, fused(payload))
    },
    waterfall(name, payload, ...rest) {
      const waterfall = ctx.waterfall as (thisArg: Scoped<Agent>, name: string, ...args: unknown[]) => never
      return waterfall(carrier, name, fused(payload), ...rest)
    },
  }
}
```

`fused()` 是 subject 注入点，两处细节：

- **payload 类型是 `PayloadRest<K>`**：完整 payload 减去 `agent` 字段（`Omit<PayloadOf<K> & object, 'agent'>`）。调用方根本无法提供 agent 字段——类型上就被拿走了。
- **spread 在前**：`{ ...payload, agent }` 把注入放在展开之后，即使某个结构上可接受的 payload 碰巧带 `agent` 字段，也永远覆盖不了注入的 subject。

`serial`/`waterfall` 直接转发 Cordis 的事件 mixin（访问器返回的是预绑定函数，用一条保留形状的受控 cast 让泛型 `Tail<K>` 通过）。`waterfall` 的 `rest` 就是事件 payload 之后的参数——末位是内层 `next` 回调（监听链要包装的默认实现）。这与 AGENTS.md 的瀑布规则呼应：**瀑布监听者必须调用 `next()`**，返回而不委托会短路链条。

### emit：逐监听者包含两种失败

```typescript
// packages/core/agent/src/dispatch.ts
emit(name, payload) {
  // Cordis emit invokes callbacks through Array.map: one synchronous throw
  // starves later listeners, and returned promises are discarded. Agent
  // notifications are non-vetoing, so resolve the same filtered callback
  // set ourselves and contain both failure modes independently.
  const args: unknown[] = [carrier, name, fused(payload)]
  const callbacks = ctx.events.dispatch('emit', args)
  for (const callback of callbacks) {
    try {
      const returned: unknown = callback(...args)
      void Promise.resolve(returned).catch((error: unknown) => {
        ctx.logger.warn(`agent event "${name}" listener rejected: ${String(error)}`)
      })
    } catch (error: unknown) {
      ctx.logger.warn(`agent event "${name}" listener threw: ${String(error)}`)
    }
  }
}
```

这是本文件最值得细读的部分。Cordis 原生 `emit` 通过 `Array.map` 调用回调，带来两个问题：一个同步 throw 会让**后续所有监听者饿死**；返回的 Promise 被**丢弃**（rejection 无人观察）。而 `agent/status`、`agent/disposed` 这类通知是**非否决性**的——任何一个监听者的失败都不应否决生命周期推进或饿死后来的观察者。所以 `emit` 自己解析过滤后的回调集合，对每个监听者独立包含同步 throw 与 promise rejection（都降级为 `ctx.logger.warn`）。

同样的包含式 emit 在 `index.ts` 的 `emitDisposed()` 里手工复刻了一份（见 [index.md](index.md) 的注册表生命周期一节）——注册表发 `agent/disposed` 时没有经过融合派发器，但要求相同的失败语义。

### emitAgentEvent：免保留派发器的一次性通知

```typescript
// packages/core/agent/src/dispatch.ts
export function emitAgentEvent<K extends AgentSubjectEvent>(
  ctx: Context,
  agent: Agent,
  name: K,
  payload: PayloadRest<K>,
): void {
  agentEvents(ctx, agent).emit(name, payload)
}
```

只想发一条通知、不需要保留派发器的调用方用这个便捷函数（内部临时建 carrier + dispatcher）。需要连发多个事件的驱动器则应自己建一次 `agentEvents` 复用。

### assembleContextFor：组装上下文两字段一起设

```typescript
// packages/core/agent/src/dispatch.ts
export function assembleContextFor(agent: Agent, signal?: AbortSignal): AssembleContext {
  return { agent, scope: agent, ...signal === undefined ? {} : { signal } }
}
```

把 `agent` 与 `scope` **一起**设置（同一个 Agent），使 agent-scoped 的 prompt 段落与工具贡献不可能被静默漏掉——这与融合派发器是同一个设计原则在 prompt 组装上的应用。`signal` 是当前 turn 的取消信号，仅当组装属于某个 turn 时携带。产出类型 `AssembleContext` 的 `agent`/`modelSelection` 字段由 [runtime-types.md](runtime-types.md) 的 `dsh-system-prompt` 合并声明补充。

## 核心导出速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `AgentSubjectEvent` | 映射类型（事件名联合） | 类型层筛出"this 是 `Scoped<Agent>` 且 payload 带 `agent`"的事件 |
| `AgentEventDispatch` | `{ emit; serial; waterfall }` | 融合派发器接口；payload 类型为 `PayloadRest<K>`（无 `agent` 字段） |
| `agentCarrier` | `(agent) => Scoped<Agent>` | 构造 subject=key 的无状态 scope 载体 |
| `agentEvents` | `(ctx, agent, carrier?) => AgentEventDispatch` | 建立融合派发器；subject 注入 + scope 一致 + emit 失败包含 |
| `emitAgentEvent` | `(ctx, agent, name, payload) => void` | 一次性包含式 agent 通知 |
| `assembleContextFor` | `(agent, signal?) => AssembleContext` | 组装上下文：`agent` 与 `scope` 一起设 |

## 与其他模块的关系

- **`runtime-types.ts`**：`AgentSubjectEvent` 的形态契约来自那里声明的 `agent/*` 事件（`this: Scoped<Agent>` + `payload.agent`）；`AssembleContext` 的 `agent` 字段也由其合并声明补充。
- **`dsh-scope`**：`scopeTarget`/`Scoped` 是载体构造与 scope 过滤的底层原语（见 [scope 教程](../scope/index.md)）。
- **`dsh-agent-loop`**：主要消费者——驱动器在 Agent 构造函数里建一次 `agentEvents`，复用派发全部 `agent/*` 事件；`assembleContextFor` 用于每次 prompt 组装。
- **`index.ts`（注册表）**：`emitDisposed()` 手工复刻了融合 emit 的包含语义（发 `agent/disposed`）；`enter()`/`announce()` 构造的 `entry.carrier` 同样是 `scopeTarget(agent, agent)`。
- **`model-selection.ts`**：`installModelSelection` 注册的 `agent/pre-step` 监听器就是经这套 scope 机制被过滤派发的。

## 小结

- 一条不变式贯穿全文件：**subject 与 scope key 必须是同一个 Agent**——融合派发器从类型（`PayloadRest` 无法提供 `agent`）和运行时（spread 在前）两端强制。
- `emit` 是自己实现的包含式派发：非否决性通知绝不因单个监听者失败而饿死后续监听者或丢失 rejection。
- `serial`/`waterfall` 忠实转发 Cordis 语义；瀑布监听者必须调用 `next()` 委托。
- 载体无状态可复用；热路径驱动器建一次用到底。
- `assembleContextFor` 把同一原则带到 prompt 组装：`agent` 与 `scope` 成对出现。
