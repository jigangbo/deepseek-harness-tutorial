# 第 8 章 Agent Loop：默认驱动实现

`packages/core/agent-loop` 是公开 `Agent` 契约（声明于 `packages/core/agent`）的唯一具体实现，也是 harness 的默认产品循环。它把"调用模型、执行工具、再调用模型"这一标准生命周期落到持久化 session 日志上：每个请求都从日志派生，每个模型可见的事实都先写回日志再做下一步派生。

消费者从不 import 这个包。它把 `AgentFactory` 注册到 `ctx.agents`（服务 `ctx.agentLoop`），上层通过 `ctx.agents.create()` / `ctx.agents.resume()` 使用；扩展插件只依赖 `packages/core/agent` 的 `agent/*` 事件词汇，因此这个驱动可被替换。

## 文件概览

| 文件 | 角色 |
|---|---|
| `packages/core/agent-loop/src/index.ts` | 插件入口：`AgentLoop` 服务、配置 schema、声明式 agent 启动、工厂注册、创建/恢复事务与有序拆除 |
| `packages/core/agent-loop/src/agent.ts` | 具体 `ReactLoopAgent` 驱动：inbox 领取、turn/step 状态机、请求构建、取消 |
| `packages/core/agent-loop/src/inbox.ts` | 包内 `ReactLoopInbox`：持久化 projection、结构性命令、仅供循环使用的 claim |
| `packages/core/agent-loop/src/runtime-context.ts` | 两个循环拥有的 surface 消息的 projection：system prompt 与运行时上下文快照 |
| `packages/core/agent-loop/src/assistant-stream.ts` | 单次模型尝试的进程内帧发布与持久化流累积 |
| `packages/core/agent-loop/src/constants.ts` | 调度默认值：`DEFAULT_MAX_PARALLEL_TOOL_CALLS` |

> 本教程把 `index.ts` 的内容并入本篇（见下文 [插件入口 index.ts](#插件入口-indexts)），其余每源文件一篇。

## 定位与职责边界

- **实现 `Agent` 接口**：`ReactLoopAgent` 提供 `send` / `followup` / `steer` / `inject` / `cancel` / `whenIdle` / `runMaintenance`，以及 `model`（`AgentModel`）和 `inbox`（`ReactLoopInbox`）两个 agent 拥有的访问面。
- **拥有 `ctx.agentLoop` 服务**：`AgentLoop extends Service`，声明 `static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']`，是 session 写句柄的生产获取点。
- **不拥有**：工具分发（交给 `ctx.tools.execute()`）、prompt 段落组装（交给 `ctx.systemPrompt.assemble()`）、模型流（交给 `ctx.llm.stream()` / `preparedCall.stream()`）。循环只做编排、准入与记账。

## 驱动状态机

驱动级别的 `Phase` 只有三种（`agent.ts`）：

```text
              send/followup/steer (wakeup=true)
                        │
                        ▼
               ┌─────────────────┐  wakeDriver()   ┌──────────────────────────────┐
        ┌─────▶│  idle           │────────────────▶│  running                     │
        │      │  lastTurn = T   │                 │  kick(): while(turn()){}     │
        │      └─────────────────┘                 └──────────────┬───────────────┘
        │              ▲                                          │
        │              │ kick()/runMaintenance() finally          │ 每次 turn 返回 true：
        │      ┌───────┴───────────┐                              │ 队列仍有 next-turn 输入，
        │      │  maintenance      │                              │ 换新 AbortController
        │      │  runMaintenance() │                              │ 继续 while 循环
        │      └───────────────────┘                              ▼
        │                                                ┌────────────────┐
        └────────────────────────────────────────────────┤  idle (回踢)   │
              wakeRequested 被闩住，收敛时若 inbox.hasPending 重新唤醒
```

turn 内部的 step 循环：

```text
 turn()：append turn/start
   │
   ▼
 ┌────────────────────────────────────────────────────────────────────┐
 │ preStep(target)                                                    │
 │   inbox.claim()            领取全部 next-step；turn 边界再领 1 条    │
 │   model.current()          无 route 直接抛错                        │
 │   ctx.tools.resolve()                                              │
 │   ctx.systemPrompt.assemble()                                      │
 │   agent/pre-step 瀑布      accept / rewrite / reject               │
 └──────┬──────────────────────────┬──────────────────────────────────┘
   reject│                     enter│messages.length > 0
        ▼                          ▼
  turnEnds = 'blocked'      append step/start
  return false（无 step）          │
                            ┌──────▼───────────────────────────────────┐
                            │ step(decision)   ← 重试回到这里，不重装    │
                            │   prepareRequest()  agent/request +      │
                            │                     ctx.llm.prepareCall()│
                            │   system/message 准入（三种路由）          │
                            │   user/message 准入（仅首次尝试）          │
                            │   buildRequest()    header/context/冻结   │
                            │   流式消费 → assistant/message 或          │
                            │              assistant/attempt           │
                            └──────┬───────────────────────────────────┘
                                   │ 有 tool-call
                                   ▼
                          ctx.tools.execute()
                                   │
                    ┌──────────────┴──────────────┐
              concluded                    未 concluded / 有 additionalContexts
                    │                             │
                    ▼                             ▼
          turnEnds = completed/max-tokens   target = 'next-step'，回到 preStep
                    │
                    ▼
          next-step 已空 → agent/turn-stopping（serial）
          仍空 → break；append turn/end
                    │
          inbox.hasPending? ──是──▶ 新 AbortController，return true（下一 turn）
                    │否
                    ▼
                  idle
```

三条关键事实：

1. **turn 边界领取 `next-turn` 的一条消息**；step 边界只领取 `next-step` 的全部消息。
2. **拒绝的 step 不开 step**：`turnEnds = { kind: 'blocked' }`，已 claim 的消息停在 `claimed` 状态（既不丢弃也不重新提交）。
3. **首步批量被改写为空也拥有 turn 边界**，但花不掉模型调用：`turnEnds = { kind: 'completed' }` 直接关 turn。

## 四个关键不变量

### 1. 取消承诺

- `agent.cancel(cause)` 中止当前活动；除非 `keepInbox`，清空两个待处理列表并撤回未消费的唤醒闩。
- 被取消的流**保留已交付的文本**：`interruptedBlocks()` 非空时以 `interrupted: true` 的 `assistant/message` 提交已见前缀，下一次请求包含用户看到的内容。
- **取消时机决定提交与否**：`agent/request` 瀑布与 `prepareCall()` 这两个异步阶段都不曾看到待提交的 system prompt 或已接受的 user 消息；`step()` 在这两个阶段之后先 `signal.throwIfAborted()` 再提交任何 `system/message` 或 `user/message`。因此取消发生在其间时**system 与 user 均不提交**。
- 写入日志的取消原因经过 `abortedCancelCause()` 拷贝：`turn/end` 只记录 `kind` 与 `hook.reason`。进程内 `AbortSignal.reason` 仍是调用者的对象（Node 的 fetch 会往上面挂 `stack`），拷贝让这类附加痕迹既不进日志也不阻断收尾追加。
- 取消后到达的唤醒输入被重分类为 `next-turn`：`send()` 在插入前捕获 `wakingAfterAbort`，防止拼接观察者里的重入 `cancel()` 改写分类。

### 2. 重试不重复组装

`step()` 的 `while (true)` 循环里，`agent/pre-step` 瀑布、prompt 装配、user 消息准入都只发生在进入 `step()` 之前（`preStep()`）或首次尝试（`firstAttempt`）。重试复用同一份渲染装配：只重新执行 `prepareRequest()`、运行时上下文刷新与 system prompt 调和。`agent/request-error` 返回 `{ kind: 'retry' }` 后 `continue` 回到循环头部，不重跑装配。

### 3. 模型可见 ⟺ 已记录

- 每个请求由 `session.deriveMessages()` 从日志派生，请求不携带日志之外的任何消息。
- 每次模型尝试恰好一个进程内 `start` 帧；每个 `chunk` 帧只在对应的持久化 assistant 帧结算之后发出（`settle()` 先 `append()` 再发 `end` 帧）；恰好一个终结 `end` 帧——`committed` 跟随持久的 `assistant/message` / `assistant/attempt`，装配或追加失败则为 `abandoned`。
- 每次成功的模型调用追加一个消息锚点；被取消的流追加 `interrupted: true` 锚点携带已交付前缀。
- 首请求之前提交的 `request/header` 与 `request/context` 让请求可以从日志重建。

### 4. 失败补记工具结果

turn 循环内每个 step 挂一个 `ToolCallRecovery`，通过 `session/event` 观察已提交事件。step 失败时，在 `step/end` 之前把未应答的 assistant tool call 补记为合成 `tool/result`：

- 已记录 `tool/call` 但未提交结果 → `TOOL_OUTCOME_UNKNOWN`（"Its outcome is unknown."，仅只读/幂等操作可重试）；
- 连调用记录都没有 → `TOOL_NOT_STARTED`（"The tool call was interrupted before the Harness recorded it as started."）。

这些补记让后续请求可以使用成对的工具历史，而不自动重试结果不确定的操作。补记自身失败则抛 `AggregateError`（原始失败 + 恢复失败）。

## 插件入口 index.ts { #插件入口-indexts }

`AgentLoop` 服务串起创建/恢复/拆除三条路径。

### 配置与启动

```ts
// packages/core/agent-loop/src/index.ts
export class AgentLoop extends Service implements AgentFactory {
  static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']

  static Config: z<{ agents?: Config['agents']; maxParallelToolCalls?: number }, Config> = z.object({
    maxParallelToolCalls: z.number().step(1).min(1).default(DEFAULT_MAX_PARALLEL_TOOL_CALLS).volatile(),
    agents: z.array(z.object({
      id: z.string().required(),
      sessionId: z.string().min(1),
      provider: z.string(),
      model: z.string(),
      reasoningEffort: z.string().min(1) as z<ReturnType<typeof ReasoningEffortId>>,
      maxTokens: z.number().step(1).min(1).max(Number.MAX_SAFE_INTEGER),
      cwd: z.string(),
      resumeSessionId: z.string(),
    })).default([]),
  }) as z<{ agents?: Config['agents']; maxParallelToolCalls?: number }, Config>
```

- `maxParallelToolCalls` 是 volatile 正整数（默认 10，`1` 为串行），`Service.init` 里注册的两个 prompt 变量把 `provider` / `model` 暴露给 prompt 模板。
- `agents` 是启动期配置：`id` 必填；`sessionId`（存在则恢复、首次则创建）与 `resumeSessionId`（必须已持久化）互斥，`validateConfiguredAgents()` 在任何 agent 启动前拒绝冲突与重复的精确身份。
- `CONFIGURED_AGENT_IDENTITIES_KEY` 允许 launcher 在挂载前用 `ctx.provide()` 固定配置条目的身份，`applyLauncherIdentities()` 会整体替换两个身份键，配置提供的身份不能与 launcher 提供的共存。

### Service.init：投影、工厂与声明式启动

```ts
// packages/core/agent-loop/src/index.ts
protected async [Service.init](): Promise<void> {
  const ctx = this.runtime.ctx
  const ownership = this.ownership
  // Agent projections remain readable until teardown finishes.
  ctx.effect(() => () => ownership.dispose(), 'agentLoop.transactions()')
  this.modelProjectionHandle = await ctx.sessionProjections.register(modelSelectionProjectionDefinition)
  this.inboxProjectionHandle = await ctx.sessionProjections.register(inboxProjectionDefinition)
  ctx.effect(() => ctx.agents.setFactory(this), 'agentLoop.setFactory()')
  ctx.systemPrompt.variable('provider', context => context.modelSelection?.provider ?? context.agent?.options.provider)
  ctx.systemPrompt.variable('model', context => context.modelSelection?.model ?? context.agent?.options.model)
```

循环先注册 `modelSelection` 与 `inbox` 两个 session projection 再发布工厂——projection 由服务生命周期持有，所以冷读在任何 Agent 存在前或全部卸载后都可用。随后每个配置条目走三条路径之一：无 `sessionId` 直接 `create()`；有 `sessionId` 走 `restoreOrCreateConfigured()`（恢复优先，仅 `SessionPersistenceNotFoundError` 才回退首次创建，损坏/占用/后端失败保持响亮）；有 `resumeSessionId` 的条目包装成 effect，等 `sessionPersistence` 服务注入后 `resume`。声明式启动失败通过 `reportConfiguredStartupFailure()` 发出 `agent-loop/config-start-failed`（emit），监听器错误被包含后告警。

恢复路径上 `applyConfiguredRoute()` 把配置声明的 route 记为 Session 的模型选择（配置未声明 effort 且 route 相同时保留此前 effort）。

### initializeAgent：一回事务，多所有者汇聚

`initializeAgent()` 是创建与恢复共用的核心。它在任何资源存在之前就注册三路融合的去激活（调用者取消信号、owner fiber 卸载、工厂拆除），然后构造一个**记忆化的反向拆除**：

```ts
// packages/core/agent-loop/src/index.ts
const teardown = (): Promise<void> => (disposing ??= (async () => {
  abort.abort(new Error(this.ownership.isActive()
    ? `agent "${id}" lifecycle disposed`
    : 'agent loop is not active'))
  // ...等待 publication 与 machineReady...
  if (machine !== undefined) {
    machine.cancel({ kind: 'disposed' }, { keepInbox: !ready })
    await machine.whenIdle()
  }
  // machine.scope.dispose() → stored?.handle.close() → detachAgent/detachSession → untrack
  throwCollectedFailures(failures, `agent "${id}" disposal failed`)
})())
```

`disposing ??=` 让调用者卸载、工厂卸载、句柄 `dispose()` 三个来源 await 同一次收敛。拆除顺序：停止驱动（`cancel({ kind: 'disposed' })`，未 ready 时保留 inbox）→ 展开 agent scope → 关闭写句柄 → 反注册 agent → 反注册 session。初始化失败时 `cancel({ kind: 'disposed' }, { keepInbox: true })`——已恢复的 inbox 输入与已存储事件都不丢。

发布本身包在 `agent.runMaintenance()` 里（空闲相保证发布期间没有驱动竞争）：

```ts
// packages/core/agent-loop/src/index.ts
return await agent.runMaintenance(async () => {
  publication = Promise.withResolvers<void>()
  // ...
  await this.appendUnstoredSuffix(stored, session)     // 构造期历史入库
  detachSession = agent.ctx.sessions.enter(session)
  agent.ctx.sessions.announce(session)
  const setupCommit = await raceAbort(setup?.(agent.ctx, agent), abort.signal, id)
  setupCommit?.commit()
  detachAgent = loopCtx.agents.enter(agent)
  await loopCtx.agents.announce(agent, source, abort.signal)
  ready = true
  return { agent, dispose }
})
```

### create / createAgent / resume

```ts
// packages/core/agent-loop/src/index.ts
async create(id: SessionId, options: AgentOptions = {}, meta: Pick<SessionHeader, 'cwd'> = {}): Promise<Agent> {
  using preparation = SessionPreparation.create(this.runtime.ctx.sessions.prepare(id, { meta }))
  const stored = await this.createStoredSession(preparation.session)
  return (await this.initializeAgent(this.ctx, {
    session: preparation.session,
    stored,
    agentOptions: options,
    source: 'startup',
  })).agent
}
```

- `create`：`SessionPreparation` 用 `using` 保证失败时释放未发布 session；持久化后端挂载时 `persistence.create(header)` 先取写所有权。
- `resumeWith`：**先 `persistence.open(id, 'write')`**（写所有权排除了同 id 并发恢复），读物理有效的日志，把 `interruptedTurnClosers()` 生成的合成收尾事件作为普通批次追加——语义崩溃修复是 agent 层的职责，不是存储入口的。
- `turnBoundaryOf(session).lastTurn`（`packages/core/agent/src/index.ts` 的 `ctx.agents.turnBoundaryOf`）读出已提交的 turn 编号传给驱动，恢复后 turn 计数从上一个 turn 继续，而不是从 0 重来。

### FactoryOwnership

工厂级所有权收集所有 live agent 的拆除闭包与启动期任务。`dispose()` 先停收新工、abort 工厂信号（reason: `agent loop is not active`）、resolve `inactive`，再 `Promise.allSettled` 全部拆除与启动任务，失败收集为一个 `AggregateError` 抛出。`waitWhileActive()` 让配置恢复在"同身份正在排水"时等待，又在工厂拆除开始时立即放弃。

## 关键类型与导出（表格）

| 导出 | 类别 | 说明 |
|---|---|---|
| `AgentLoop` | `Service` + `AgentFactory` | 具体 agent 工厂与驱动服务（`ctx.agentLoop`） |
| `Config` | 配置接口 | `maxParallelToolCalls`（volatile）与声明式 `agents[]` |
| `DEFAULT_MAX_PARALLEL_TOOL_CALLS` | 常量 | `10`，再导出自 `constants.ts` |
| `LauncherAgentIdentity` / `ConfiguredAgentIdentities` | 接口 | launcher 拥有的精确 session 身份 |
| `CONFIGURED_AGENT_IDENTITIES_KEY` | 常量 | launcher 通过 `ctx.provide()` 设置身份的 Context 键 |
| `ReactLoopAgent` | 类（包内） | 具体 `Agent` 实现，导出但属实现面 |
| `inboxProjectionDefinition` / `inboxProjectionSchema` | projection | `inbox` 键的持久化折叠与 wire 校验 |
| `SystemPromptProjection` / `RuntimeContextProjection` | 类（包内） | 两个循环拥有的 surface 消息 projection |
| `AssistantStreamAttempt` | 类（包内） | 单次模型尝试的帧发布与流累积 |
| `'agent-loop/config-start-failed'` | 事件（emit） | 声明式条目启动失败的可观察信号 |

## 扩展点：如何拦截驱动

所有拦截都在 `agent/*` 事件词汇上（声明于 `packages/core/agent/src/runtime-types.ts`），扩展插件依赖 `@deepseek-ai/dsh-agent` 注册监听，scope 过滤分派（agent-scoped 监听器只收到对应 agent）。

| 事件 | 模式 | 拦截能力 |
|---|---|---|
| `agent/pre-step` | waterfall | **accept / rewrite / reject**。返回 `{ kind: 'reject' }` 拒绝（不开 step，turn 以 `blocked` 关闭）；返回 `{ kind: 'enter', messages, startsRequestSeries? }` 改写进入 step 的消息批次或声明新请求系列；调用 `next()` 保留当前消息。载荷含独占 claim 的 `messages`、`turn`/`step` 坐标、当前 turn 的取消 `signal` 与捕获的 `modelSelection` |
| `agent/request` | waterfall（已弃用） | 提议请求控制（如采样参数）。改 `provider`/`model`/`reasoningEffort` 会在 step 准备之后被拒绝抛错——换 route 必须走 `agent.model.select()` |
| `agent/request-error` | waterfall | 失败的模型尝试进入循环重试决策：持有恢复的监听器返回 `{ kind: 'retry' }` **且不调用 `next()`**；默认 `undefined` 保持失败终结 |
| `agent/turn-stopping` | serial | turn 即将关闭且无工具/steering 延续时运行；监听器通过 `agent.steer(...)` 反对即注入下一步输入，机器重读 inbox——数据决定结果 |
| `agent/created` / `agent/disposed` / `agent/status` / `agent/error` / `agent/assistant-stream` / `agent/inbox/*` | emit / serial | 生命周期与可观察性；`agent/inbox/inserted|claimed|discarded` 是跟随单条消息的精确通知 |

瀑布语义提醒：`agent/pre-step` 是请求派生前唯一的瀑布监听链；**水瀑布监听器必须调用 `next()` 委托**（`agent/request-error` 的 retry 分支是唯一例外——持有恢复权时不调 `next()`）。

## 推荐阅读顺序

1. **本篇** — 状态机、不变量与入口事务。
2. [`agent.md`](agent.md) — `ReactLoopAgent`：turn/step 主循环、`prepareRequest`（`agent/request` + `prepareCall`）、请求冻结、system/user 准入时序。
3. [`inbox.md`](inbox.md) — 持久化 inbox 投影、splice 归一化、loop-only claim。
4. [`runtime-context.md`](runtime-context.md) — `SystemPromptProjection`（三种路由）与 `RuntimeContextProjection`（运行时上下文注入）。
5. [`assistant-stream.md`](assistant-stream.md) — 单次尝试的帧发布协议与取消结算。
6. [`constants.md`](constants.md) — 调度默认值。

外围阅读：`packages/core/agent/README.md`（`Agent` 契约）、`docs/subsystems/core.md`（turn 流与拦截决策）、`docs/subsystems/session.md`（循环写入的持久化日志）、`docs/subsystems/tools.md`（循环分发经过的管线）。

## 小结

agent-loop 是"从日志来、回日志去"的编排层：inbox 提供持久化的待办输入，`preStep` 在 claim 后交给 `agent/pre-step` 瀑布裁决，`step` 里 `prepareRequest` 绑定 adapter 能力，system/user 准入在取消点之后提交，`buildRequest` 记 header/context 并冻结派生请求，流式结果经由 `AssistantStreamAttempt` 先落日志再发终结帧。取消、重试、失败恢复全部围绕同一不变量集合：模型可见的必然已记录，已记录的必然可重建。
