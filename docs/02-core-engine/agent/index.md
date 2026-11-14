# 第 10 章 Agent：接口与活体注册表

`packages/core/agent`（包名 `@deepseek-ai/dsh-agent`）定义了 harness 中"活体 Agent"的**契约面**：`ctx.agents` 注册表服务、`Agent` 接口、`AgentHandle` 所有权句柄、`agent/*` 事件词汇，以及把发起 Agent 携带过异步驱动链的进程内 initiator scope。它本身**不创建也不驱动**模型请求——具体的创建与驱动由注册进来的工厂（默认是 `packages/core/agent-loop`）完成。

## 定位：契约与实现分离

这个包建立在一条核心分离之上：

- **本包（`dsh-agent`）**：公开 `Agent` 表面、活体注册表 `AgentRegistry`、工厂插槽 `AgentFactory`、`agent/*` 事件词汇、投影类型。
- **`dsh-agent-loop`**：`Agent` 契约的唯一具体实现。它在构造时通过 `ctx.agents.setFactory()` 把工厂注册进注册表，由它负责 step 准入、turn/step 状态机、模型请求与工具执行编排。

由此推出一条依赖纪律：**扩展插件（UI、hooks、orchestrator、团队协作等）依赖 `dsh-agent`，永远不直接依赖 `dsh-agent-loop`**。消费者针对 `ctx.agents` 编程（比如 ACP bridge），驱动实现保持可替换。`ctx.agents.get(id)` 返回裸 `Agent`；`AgentHandle`（含 `dispose()`）只暴露给创建它的所有者——句柄是拆除能力，不是查询接口。

## 与 Session 的关系

每个 Agent 绑定一个 `Session`，且**共享同一个身份**：`Agent.id === agent.session.id`（`enter()` 会在发布前强制校验这一点）。`Session` 的 append-only 事件日志是唯一持久真相；Agent 的 inbox（两条待处理消息列表）、模型选择、turn/step 位置全部以 `SessionEvent` 的形式写入日志，再由 session-projection 折叠重建。换句话说：

- Agent 是"活着的那一侧"——注册表里的一个条目，携带 scoped `ctx`、inbox、model、status。
- Session 是"记录的那一侧"——日志 + 存储。Agent 被拆除（detach）后，Session 事件依然存在，可被投影/回放消费。

## ASCII 总览图

```text
                        ┌──────────────────────────────────────────────────────┐
                        │            AgentRegistry (ctx.agents 服务)           │
                        │   store: Map<SessionId, AgentEntry>                  │
                        │   factory slot ◀── setFactory() ◀── dsh-agent-loop   │
                        │   initiator scope (AsyncLocalStorage)                │
                        │   turnBoundaryProjectionHandle                       │
                        └──────┬───────────────────────────┬───────────────────┘
              create()/resume()│                           │ get(id) / list()
                （返回句柄）     │                           │（裸 Agent 查询）
                        ▼       ▼                           ▼
                 ┌─────────────────────┐            ┌─────────────────────┐
                 │ AgentHandle         │            │ 插件 / UI / 编排器    │
                 │  agent + dispose()  │            │ （无法拆除他人 agent）│
                 └──────────┬──────────┘            └──────────┬──────────┘
                            ▼                                  │ 监听
   ┌───────────────────────────────────────────────┐           │
   │              Agent（运行时面）                  │           │
   │  id / options / session / model / inbox       │           │
   │  status / ctx（agent-scoped Context）          │           │
   │  followup / steer / inject / send             │           │
   │  cancel / whenIdle / runMaintenance           │           │
   └──────┬───────────────────────────┬────────────┘           │
          │ inbox 变更、model 选择、    │ agentEvents 融合派发     │
          │ turn/step 边界 → append    │（subject 与 scope 合一）  │
          ▼                           ▼                        ▼
   ┌──────────────────┐    ┌─────────────────────────────────────────┐
   │  Session 日志     │    │  agent/* 事件（Cordis Events，scope 过滤）│
   │  （持久真相）      │    │  created / disposed / status            │
   │  turn/start(end) │    │  inbox/inserted · claimed · discarded   │
   │  step/start(end) │    │  pre-step ▸ request ▸ request-error     │
   │  inbox/spliced   │    │  assistant-stream / turn-stopping / error│
   │  model/selection │    └─────────────────────────────────────────┘
   └────────┬─────────┘
            │ 投影折叠（session-projections）
            ▼
   ┌──────────────────────────────────────┐
   │ turnBoundary / inbox / modelSelection│──▶ 冷消费者（无活体 Agent 也能读）
   └──────────────────────────────────────┘
```

三条流值得记住：

1. **输入流**：`followup`/`steer`/`inject`/`send` → `Inbox` 结构性命令 → `agent/inbox/spliced` 持久事件 → 步边界由循环 claim（纯删除 splice）→ 进入 step。
2. **事件流**：循环驱动器用 `agentEvents()` 融合派发器发出 `agent/*` 事件；监听者按 scope 过滤只收到自己关心的那个 Agent。
3. **投影流**：日志被 `turnBoundary`、`inbox`、`modelSelection` 三个投影折叠，UI 与冷消费者从投影读状态，不需要活体 Agent。

## 源文件地图

| 文件 | 角色 | 教程 |
|---|---|---|
| `packages/core/agent/src/index.ts` | 插件入口：`AgentRegistry`、工厂插槽、initiator scope、创建/恢复选项（并入本篇） | 本篇 |
| `packages/core/agent/src/types.ts` | 类型-only 面：`Agent` 基础接口、inbox 投影词汇、`ModelSelection`、durable 事件词汇 | [types.md](types.md) |
| `packages/core/agent/src/runtime-types.ts` | 运行时面：`Agent` 完整成员、`AgentOptions`、`Inbox`/`Model`、`agent/*` 事件声明 | [runtime-types.md](runtime-types.md) |
| `packages/core/agent/src/dispatch.ts` | 融合派发器 `agentEvents()` 与 prompt 组装上下文 `assembleContextFor()` | [dispatch.md](dispatch.md) |
| `packages/core/agent/src/model-selection.ts` | `AgentModel`、model-selection 持久投影、作用域模型切换通知 | [model-selection.md](model-selection.md) |
| `packages/core/agent/src/projection.ts` | `turnBoundary` 投影：Session 的 turn/step 位置 | [projection.md](projection.md) |
| `packages/core/agent/src/consumed-work.ts` | `foldConsumedWork(events)`：日志里被消费的工作去向 | [consumed-work.md](consumed-work.md) |
| `packages/core/agent/src/archive-admission.ts` | Workspace 归档准入的 `turn` 家族 | [archive-admission.md](archive-admission.md) |

包导出两个入口：`.`（运行时面）与 `./types`（类型-only 面）。只需投影词汇的消费者（如渲染 `SessionActivityKindMap` 家族的 Client）可以从 `@deepseek-ai/dsh-agent/types` 导入而不拉入运行时代码。

## 插件入口 index.ts

### AgentRegistry 服务

`AgentRegistry extends Service`，`static inject = ['sessionProjections']`。通过声明合并挂到 `ctx.agents`：

```typescript
// packages/core/agent/src/index.ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    agents: AgentRegistry
  }
}
```

构造函数做四件事：注册 typert 的 `agent` lookup/host context（跨 wire 边界把 `SessionId` 解析回活体 Agent 或其 scoped Context）；监听 `internal/status`，当拥有本服务的 fiber 卸载时关闭 initiator 边界；安装 initiator 生命周期的组合 effect（先等 drain 再禁用底层存储）；以及调用 `installTurnArchiveAdmission()`，让注册表为它发布的每个 Agent 回答 Workspace 的归档准入。

`[Service.init]` 中注册 `turnBoundary` 投影并保存句柄：

```typescript
// packages/core/agent/src/index.ts
protected async [Service.init](): Promise<void> {
  this.turnBoundaryProjectionHandle = await this.ctx.sessionProjections.register(turnBoundaryProjectionDefinition)
}

/**
 * Read the turn and step positions recorded in a Session.
 */
turnBoundaryOf(session: Session): TurnBoundaryProjection {
  return this.turnBoundaryProjectionHandle.stateOf(session)
}
```

`turnBoundaryOf(session)` 让任何消费者（包括没有活体 Agent 的冷消费者）读取 Session 里已记录的 turn/step 位置——投影定义由注册表拥有，消费者不要自己注册。

### 创建与恢复：工厂委托

`create()` / `resume()` 不自己构造 Agent，而是转发给已注册的工厂：

```typescript
// packages/core/agent/src/index.ts
async create(options: CreateAgentOptions): Promise<AgentHandle> {
  const ownerCtx = this.ctx
  const { target } = this.requireFactory()
  const receiver = getTraceable(ownerCtx, target)
  return Reflect.apply(target.createAgent, receiver, [ownerCtx, options])
}
```

三个要点：

- **ownerCtx 绑定**：注册表显式传入调用者的 Context，工厂实现把未发布事务与 resulting lifecycle 挂到这个 owner 上——不从 factory 对象自己的注册上下文推断所有权。`getTraceable()` + `Reflect.apply` 把 Service 形态的工厂重新 trace 到访问者上下文，避免堆叠两层 Cordis 代理。
- **工厂插槽是普通 holder**：`FactorySlot { readonly target }` 防止 Cordis 在调用者上下文未知前 trace 工厂字段。`setFactory()` 用 `ctx.effect()` 注册，重复注册抛错；effect-scoped 意味着 loop 插件卸载时插槽自动清空。
- **未注册工厂时快速失败**：`requireFactory()` 抛出 `no agent factory registered (load an agent-loop plugin)`——配置错误在最早可解析点响亮失败。

### AgentHandle：拆除是一种能力

```typescript
// packages/core/agent/src/index.ts
export interface AgentHandle {
  agent: Agent
  dispose(): Promise<void>
}
```

`dispose()` 的完整拆除序列：停止并 drain 循环 → 注销 Agent → 从存储移除 Session → 展开 scoped world。句柄只在 `create()`/`resume()` 返回值里出现；`ctx.agents.get(id)` 永远返回裸 `Agent`。工厂的 provider 是结构共同所有者（scoped agent 依赖其服务 API），provider 卸载会停止并 drain 它创建的所有活句柄。

### 创建/恢复选项

`CreateAgentOptions` 携带共享身份与发布前需要的一切：`sessionId`（活体 Agent 与 Session 日志共用的唯一 id）、`meta`（验证过的 `cwd`、fork 血缘 `parentSession`、`isSeeded` 标记、`origin: 'subagent'`、`delegationDepth`、`agentPreset`）、`inheritedEventCount`（fork 切点）、`seed`（从 seq 0 连续的重放/继承前缀）、`agentOptions`、创建期 `signal`，以及 `setup`。`ResumeAgentOptions` 是持久化对应物：`resumeSessionId` + `agentOptions`/`signal`/`setup`。

`setup` 回调的类型是本包自己导出的：

```typescript
// packages/core/agent/src/index.ts
export type AgentSetup = (
  agentCtx: Context,
  agent: Agent,
) => AgentSetupCommit | Promise<AgentSetupCommit | void> | void
```

工厂在铸造 `agentCtx`、进入 SessionStore、完成 `session/created` 之后 await setup；Agent 在 setup 完成前不进注册表。setup 里通过 `agentCtx` 注册的一切（scoped 工具、prompt 段落、变量、监听器、restrict）都在 `agent/created` 与第一次 prompt 组装之前存在。setup 可以返回 `AgentSetupCommit`——一个同步 `commit()`，在所有 setup await 结束后、注册表发布前一刻被调用，让可变供给在精确的发布边界重新校验。setup 抛错、commit 抛错或 owner dispose 都会回滚事务（已提交的 Session 事件保留）。**setup 只做组合，不做驱动**：这是受信任的同进程代码上的契约，运行时不强制。

### 注册表生命周期：enter / announce / register

`AgentEntry` 记录每个条目的全部可变生命周期状态（`announced`、`announcing`、`detachRequested`）。高级原语拆成两半，供异步工厂使用：

```typescript
// packages/core/agent/src/index.ts
enter(agent: Agent): () => void {
  const id = agent.id
  if (id !== agent.session.id) {
    throw new Error(`agent id "${id}" does not match session id "${agent.session.id}"`)
  }
  // ...（构造 entry，校验 id 冲突——这里是权威的碰撞边界）
  this.store.set(id, entry)
  // ...（返回 detach 闭包：若正在 announcing 则只置 detachRequested，
  //     延迟到 serial 创建派发 settle 后再真正移除）
}

async announce(agent: Agent, source: SessionStartSource, signal?: AbortSignal): Promise<void> {
  // ...（校验 exact live entry、未宣布过）
  entry.announcing = true
  entry.announced = true   // 先标记再派发，监听器无法递归制造第二条生命周期边
  try {
    await this.ctx.serial(entry.carrier, 'agent/created', { ... })
  } finally {
    entry.announcing = false
    if (entry.detachRequested) this.detachEntered(entry)
  }
}
```

`register(agent)` 是普通调用者的入口（登记一个已构造好的 Agent，source 为 `startup`）：组合 effect 先 `yield this.enter(agent)` 再 `await this.announce(...)`，所以 teardown 嵌套在正确顺序里。 detach 闭包绑定**精确的 entry 身份**——`detachEntered()` 里 `store.get(entry.id) !== entry` 的检查保证过期能力永远删不掉后来的同 id 替代生命周期。未 announce 的插入被回滚时不发 `agent/disposed`（不存在"从未对外创建过"的销毁边）；已 announce 的才通过 `emitDisposed()` 发出包含式（per-listener try/catch + promise rejection 日志）的销毁通知。

### initiator scope：进程内因果归属

每个驱动器在 `ctx.agents.withInitiator(agent, ...)` 里跑完整生命周期，其继承异步链都能观察到该 Agent：

```typescript
// packages/core/agent/src/index.ts
withInitiator<T>(agent: Agent, operation: () => T): T   // 建立 initiator 边界
withoutInitiator<T>(operation: () => T): T              // 建立隐藏继承 initiator 的边界
currentInitiator(): Agent | undefined                   // 可选读取（日志/追踪/指标）
requireInitiator(): Agent                               // 严格读取（契约禁止无 agent 调用的私有助手）
```

内部用两个 `AsyncLocalStorage`：`initiators` 存 Agent 本身，`initiatorRuns` 存跟踪记录 `InitiatorRun { active, parent }` 链。`runWithInitiator()` 同步返回值直接释放跟踪；Promise 返回值在 settle（fulfilled 或 rejected）后释放，兜底 catch 处理 branded Promise 暴露失败 `@@species` 的极端情况。拆除顺序：`closeInitiators()` 拒绝新边界 → 等 returned-Promise 边界 drain（排除发起本次 teardown 的嵌套链本身，`releaseReentrantInitiatorRuns()`）→ `initiatorState = 'disposed'` 并 `disable()` 底层存储。

**边界只有进程内归属一个语义**：环境性存在既不是存活证明也不是授权；worker、进程、持久化、wire 边界上的身份必须显式携带。队列或 wire 接收方只有在验证了显式身份、解析出精确活体 Agent 之后才能建立这个边界——`withInitiator` 本身不做任何验证。

## 推荐阅读顺序

1. **本篇** —— 建立注册表、句柄、工厂委托、initiator scope 的全景。
2. [types.md](types.md) —— 类型-only 面：`Agent` 基础接口、durable 事件词汇、投影类型。
3. [runtime-types.md](runtime-types.md) —— `Agent` 完整成员与全部 `agent/*` 事件语义（本模块的核心）。
4. [dispatch.md](dispatch.md) —— 事件怎么发出去：融合派发器与 scope 载体。
5. [model-selection.md](model-selection.md) —— `agent.model` 背后的持久选择与切换通知。
6. [projection.md](projection.md) —— `turnBoundaryOf()` 背后的折叠。
7. [consumed-work.md](consumed-work.md) —— 从日志回答"被消费的工作去哪了"。
8. [archive-admission.md](archive-admission.md) —— 一个小而完整的注册表监听器样例（Workspace 归档准入）。

想看契约的默认实现（inbox claim、step 状态机、取消收敛），转到同目录的 [agent-loop](../agent-loop/index.md) 教程。

## 小结

- `dsh-agent` 定义契约与注册表；`dsh-agent-loop` 注册工厂并实现驱动；扩展插件只依赖前者。
- `Agent` 与 `Session` 共享 `SessionId` 身份；日志是持久真相，inbox/model 选择/turn 位置都以事件形式落日志再投影。
- `AgentHandle` 是所有者专属的拆除能力；`ctx.agents.get(id)` 只给裸 `Agent`。
- `agent/*` 事件是插件作用于活体工作的扩展点，全部经融合派发器按 scope 过滤。
- initiator scope 只做进程内因果归属，跨进程的身份必须显式传递。
