# 一次对话的完整生命周期

上一篇我们建立了 DeepSeek Harness 的全局架构图景:一切皆插件、三种事件域、能力接缝。这一篇我们把镜头拉近,跟踪**一次对话从用户敲下回车,到模型回复完毕、会话日志落盘**的每一个事件。读完后,你应当能对着 `packages/core/agent-loop/src/agent.ts` 逐行读懂默认驱动 `ReactLoopAgent`,并知道每个环节该挂在哪个扩展点上。

本章内容全部来自 `docs/architecture.md` 的 Turn flow 一节、`docs/agent-lifecycle.md` 的时序图,以及 `packages/core/agent-loop`、`packages/core/tools` 的真实源码。所有源码路径均相对主仓库根目录。

---

## 1. 三个基本单位:step、turn、inbox

### 1.1 step 与 turn

`docs/architecture.md` 的定义只有两句话,但每个词都有精确含义:

> A **step** is one model request plus the tools it calls. A **turn** is zero or more steps: it opens before its first input is claimed and closes once nothing is owed.

- **step(步)** = 一次模型请求 + 这次请求触发的全部工具调用。一次请求如果调了 5 个工具,它们同属一个 step。
- **turn(轮)** = 0 个或多个 step。它**先于第一条输入被声明(claim)之前开启**,在"没有任何东西欠着"时关闭。模型回复了纯文本、没有工具调用,turn 就闭合;模型调了工具,工具结果"欠"一次新的模型请求,turn 就继续下一个 step。

"0 个 step 的 turn"不是退化情况,而是有意设计:输入被声明后可能被 `agent/pre-step` 拒绝,或改写为空——此时 turn 仍然完整开启又闭合,日志里留下 `turn/start` + `turn/end` 一对边界事件,但**不花一次模型调用**。这让"这次输入确实到达并被处理了(即使结果是拒绝)"成为可回放的事实。

### 1.2 inbox:喂数据给驱动器的唯一入口

驱动器 `ReactLoopAgent`(实现 `Agent` 接口)只有一个数据来源:**inbox(收件箱)**。它内部是两条持久化队列:

| 队列 | 语义 | 谁写入 |
|---|---|---|
| `next-turn` | 排队等待各自新 turn 的普通消息,每条独占一个 turn | `followup()` |
| `next-step` | 等待最近 step 边界的注入内容(转向、上下文) | `steer()`、`inject()` |

四个公开入口(agent 契约定义于 `packages/core/agent/src/runtime-types.ts`):

```ts
followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)
}

steer(input: UserMessage): void {
  this.send(input, 'next-step', true)
}

inject(input: UserMessage): void {
  this.send(input, 'next-step', false)
}
```

摘自 `packages/core/agent-loop/src/agent.ts`。第三个参数 `wakeup` 是关键差异:`followup` 和 `steer` 会唤醒驱动器,`inject` 只入队不唤醒——**注入的上下文要等下一条"叫醒消息"才能被消费**("One inbox feeds the driver; injected context waits for a waking message")。空闲的 agent 收到 `inject()` 后会一直挂着,直到下一次 `followup()` 或 `steer()` 才真正开 turn。

`send()` 本身还有一段防御细节:唤醒分类在插入收件箱**之前**捕获,防止 splice 观察者重入调用 `cancel()` 把"唤醒"错误降级:

```ts
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void {
  // Waking input cannot join an aborted activity, so it starts the next turn.
  // Captured before the insertion so a reentrant cancel from a splice observer cannot reclassify it.
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}
```

### 1.3 inbox 是持久的

收件箱的每一次变异(插入、替换、删除、声明、清空)都提交一条 `agent/inbox/spliced` 会话事件——它是持久事件,注册表按标准 splice 坐标折叠重建,所以**重启或恢复会话后,待处理输入原样还原**,不需要任何活着的 Agent 存在(`AgentLoop` 为其服务生命周期注册 `inbox` projection)。插入消息同时发出活事件 `agent/inbox/inserted`;普通删除发 `agent/inbox/discarded`;而**声明(claim)是纯删除,发 `agent/inbox/claimed`**。

### 1.4 claim:驱动器怎么取走一批输入

声明逻辑在 `packages/core/agent-loop/src/inbox.ts`:

```ts
claim(target: InboxTarget, turn: number): UserMessage[] {
  const claimed = this.mutate('next-step', 0, this.nextStep.length, [], false)
  if (target === 'next-turn') claimed.push(...this.mutate('next-turn', 0, 1, [], false))
  for (const message of claimed) this.dispatch.emit('agent/inbox/claimed', { message, turn })
  return claimed
}
```

读法:

- **turn 的第一个 step**(target 为 `'next-turn'`):一次性取走*整个* `next-step` 队列,再从 `next-turn` 取**一条**排队消息。这批消息合并为同一个 step 的输入——所以"一条 followup + 赶到的 steering"会一起进入首次请求。
- **后续 step**(target 为 `'next-step'`):只取 `next-step` 队列(此时它由 steering / 注入 / 工具附加上下文填充)。

`claim` 是原子操作:清空与取走在同一次持久 splice 中完成,不存在"取了一半被取消"的中间态。

---

## 2. 全景时序图

下面是一次典型 turn(含工具调用)的完整时序,编号 ①–⑮。**所有事件名都是真实的事件标识符**;方括号标注分类:

- `[durable]` = 持久会话事件,经 `session.append()` 写入日志,重载后可回放;
- `[live]` = 活扩展点(emit),仅存在于当前进程;
- `[waterfall]` = 瀑布事件,监听器**必须调用 `next()`** 委托链;
- `[serial]` = 串行事件,逐个 await,无 `next()`。

```text
 caller              Driver (ReactLoopAgent)       hooks (live extension points)      services (prompt/llm/tools)     session log (durable)
    |                       |                              |                               |                          |
    | (1) followup(msg)     |                              |                               |                          |
    |---------------------->| splice('next-turn') ------------------------------------------------------------- ------------>| agent/inbox/spliced     [durable]
    |                       | emit agent/inbox/inserted                                                    | agent/inbox/inserted   [live]
    |                       | wakeDriver(): emit agent/status running                                      | agent/status           [live]
    |                       | (2) open the turn            |                               |                          |
    |                       |-------------------------------------------------------------------------------------------->| turn/start { turn }     [durable]
    |                       |                              |                               |                          |
    |                       | (3) claim first-step input   |                               |                          |
    |                       |  drain next-step + take one queued message                                   |
    |                       |-------------------------------------------------------------------------------------------->| agent/inbox/spliced     [durable]
    |                       |  (pure deletion, no outcome)                                                 | agent/inbox/claimed    [live, per message]
    |                       | (4) snapshot selection + assemble                          |                          |
    |                       | selection = agent.model.current()                                            |
    |                       |------------------------------------------- tools.resolve() ->| tools.resolve()          |
    |                       |--------------------------------------------- ctx.systemPrompt.assemble ->|  (waterfall system-prompt/assemble)
    |                       | (5) waterfall agent/pre-step ------------------------------->|                          |
    |                       |<------ decision: reject | enter(messages, startsRequestSeries?) ---          |
    |                       |   fallback projects runtime context            |             |                          |
    |                       |  reject, or first claim rewritten empty -------------------> ---------------->| turn/end (blocked/completed)
    |                       |  ==> turn closes with NO step; jump to (15)  |               |                          |
    |                       | (6) ------------------------ ------------------------------------------------>| step/start { turn, step }[durable]
    |                       | (7) waterfall agent/request ------------------------------->|                          |
    |                       |<------ proposed config (controls only) -----|                |                          |
    |                       |--------------------------------------------- ctx.llm.prepareCall(config, signal) ->  bind adapter
    |                       |  cancellation during (7) commits neither system nor users    |                          |
    |                       | (8) synchronous admission using the prepared-call capability |                          |
    |                       |--------------------------------------------------------------------------------->| system/message (per-node) [durable]
    |                       |  admit users (first attempt only)                                            | user/message             [durable]
    |                       |--------------------------------------------------------------------------------->| request/header + request/context [durable, as needed]
    |                       | (9) derive + freeze request from the log     |               |                          |
    |                       |-------------------------------------------- stream via llm/stream waterfall ->     |
    |                       | (10) streaming               |                               |                          |
    |                       |<--------------- StreamChunk* --------------------------------|                          |
    |                       | emit agent/assistant-stream: start, chunk*                    |                          | [live, transient]
    |                       |-------------------------------------------------------------------------------->| assistant/message (success)        [durable]
    |                       |   (or assistant/attempt for failed/retried/cancelled)       |                          | assistant/attempt         [durable, no model history]
    |                       | emit agent/assistant-stream: end (committed)                 |                          | [live]
    |                       |  on attempt failure: waterfall agent/request-error -------->| retry | terminal          |
    |                       | (11) for each tool-call block ----------------------------------------------->| tool/call { callId,.. }  [durable, logged BEFORE execution]
    |                       | (12) ctx.tools.execute(batch) ------------> batch scheduler ->|                          |
    |                       |                              |     waterfall tools/pre-execute -> guards / approval     | [waterfall]
    |                       |                              |     waterfall tools/execute   -> around dispatch (body)  | [waterfall]
    |                       |                              |     waterfall tools/post-execute -> accept / block       | [waterfall]
    |                       |<-------- results commit in model order -------------------------------------->| tool/result              [durable]
    |                       |                              |     emit tools/result (frozen outcome)                   | [live]
    |                       | (13) ------------------------ --------------------------------------------->| step/end { turn, step }  [durable]
    |                       | dispose the step tool set   |                               |                          |
    |                       | (14) does anything owe another request?          |           |                          |
    |                       |  tool calls ran (step ended without a final text-only reply),|                          |
    |                       |  or next-step inbox non-empty                                                |
    |                       |    yes: target='next-step', goto (3) with the NEXT step      |                          |
    |                       |    no : serial agent/turn-stopping ------------------------->|                          | [serial]
    |                       |            listener may agent.steer(): re-read inbox,       |                          |
    |                       |            fresh steering -> another step; none -> close     |                          |
    |                       |                              |                               |                          |
    |                       | (15) ------------------------ --------------------------------------------->| turn/end { reason }      [durable]
    |                       | emit agent/status idle       |                               |                          | [live]
    |                       |  inbox still pending? new AbortController, next turn         |                          |```

图注:

1. **垂直双列对照**:右端一列是落进日志的持久事件;中间两列("hooks"、"services")上的交互全部是进程内的活事件或瀑布,不会出现在会话日志里(唯一例外:`agent/inbox/spliced` 虽以 `agent/` 开头,却是持久事件——见 §2.1)。
2. `(5)` 的拒绝出口和 `(14)` 的续步判断是 turn 里最容易走错的两条岔路,§4.2 和 §4.7 分别展开。
3. `(10)` 里"chunk 是活的、settlement 是持久的"这对关系,是 §5 不变量的最佳注脚。

### 2.1 事件分类速查

| 分类 | 事件 |
|---|---|
| 持久会话事件 | `agent/inbox/spliced`、`turn/start`、`step/start`、`system/message`、`user/message`、`developer/message`、`assistant/message`、`assistant/attempt`、`tool/call`、`tool/result`、`request/header`、`request/context`、`turn/end`、`step/end` |
| 活事件(emit) | `agent/status`、`agent/inbox/inserted`、`agent/inbox/claimed`、`agent/inbox/discarded`、`agent/assistant-stream`、`agent/error`、`tools/result` |
| 瀑布(waterfall,必须 `next()`) | `system-prompt/assemble`、`agent/pre-step`、`agent/request`(已弃用)、`agent/request-error`、`llm/stream`、`tools/pre-execute`、`tools/execute`、`tools/post-execute` |
| 串行(serial,无 `next()`) | `agent/created`、`agent/turn-stopping` |

`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*` 是持久事件,其余 `agent/*` 是活的扩展点——这是 architecture.md 的原话。注意两个"例外分子":`agent/inbox/spliced` 是持久事件(收件箱必须跨重启还原),`developer/message`(工具增减的提示)也是持久事件。

---

## 3. 逐行展开 turn flow 伪代码

architecture.md 的 Turn flow 一节给了一份权威伪代码。我们先原样引用,再逐行给出讲解(括号里的编号对应 §2 时序图):

```text
turn/start
  claim next-step input plus one queued message
  snapshot Session model selection; assemble prompt sections + step-admitted tool schemas
  -> agent/pre-step; project runtime context at fallback
                                     reject | enter(messages, startsRequestSeries?)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     agent/request -> prepareCall (cancellation commits neither system nor users)
     reconcile system/message using the prepared call capability
     append entered messages as user/message; reconcile retained runtime context
     log request/header and request/context as needed
     derive and freeze model history from the log
     stream the bound prepared call -> llm/stream -> agent/assistant-stream start
       agent/assistant-stream chunk*
       assistant/message | assistant/attempt -> agent/assistant-stream end
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

| 行 | 讲解 | 编号 |
|---|---|---|
| `turn/start` | 持久事件,turn 边界先于任何输入声明落盘;失败(日志写不进)直接 `agent/error` + 抛出 | ② |
| `claim next-step input plus one queued message` | §1.4 的 `inbox.claim('next-turn', turn)`:清空 next-step + 取一条 next-turn;纯删除 splice + `agent/inbox/claimed` | ③ |
| `snapshot Session model selection; assemble ...` | `model.current()` 快照 provider/model/effort,作为 `AssembleContext.modelSelection` 与 `agent/pre-step.modelSelection` 携带;`ctx.tools.resolve()` 得到本 step 准入的工具集,`ctx.systemPrompt.assemble()` 组装提示词段与工具 schema | ④ |
| `-> agent/pre-step; project runtime context at fallback` | 瀑布事件;**默认实现(无监听器时)在监听器跑完后投影 runtime context**,把注册的运行时事实(如工作目录)作为附加用户上下文拼进消息 | ⑤ |
| `reject | enter(messages, startsRequestSeries?)` | 瀑布返回的决定是**权威的**;包装监听器要 `{ ...decision, messages }` 保留下游结果 | ⑤ |
| `reject, or a first enter rewritten empty -> close the turn with no step` | 首次声明被拒→ `turn/end { kind: 'blocked' }`;首次声明改写为空→ `turn/end { kind: 'completed' }`;两者都不产生 step | ⑤→⑮ |
| `step/start` | 持久事件;此后才进入请求准备 | ⑥ |
| `agent/request -> prepareCall` | 已弃用的瀑布(只能改采样等 controls,不能改路由)+ `ctx.llm.prepareCall()` 绑定适配器;**这两个异步阶段中取消,system 与 users 都不提交** | ⑦ |
| `reconcile system/message ...` | 用 prepared call 的能力(`systemPromptUpdate`/`toolUpdate`)把渲染好的提示词调和进 `system/message` 历史 | ⑧ |
| `append entered messages as user/message; ...` | 仅首次尝试:声明进来的消息逐条落 `user/message`;runtime context 候选原位替换对应消息 | ⑧ |
| `log request/header and request/context as needed` | 信封(config+tools)变了记 `request/header`(reason `change`);provider/model/contextWindow/systemPromptUpdate 变了记 `request/context` | ⑧ |
| `derive and freeze model history from the log` | `session.deriveMessages()` 从日志投影模型历史并逐条 `deepFreeze`——请求不携带任何日志之外的内容 | ⑨ |
| `stream the bound prepared call -> llm/stream -> ...` | 走绑定的适配器经 `llm/stream` 瀑布发起流式请求;进程内发 `agent/assistant-stream` start/chunk 帧 | ⑨⑩ |
| `assistant/message | assistant/attempt -> ... end` | 成功结算为 `assistant/message`(嵌入完整紧凑带时标流);失败/重试/取消结算为 `assistant/attempt`(不进模型历史);之后才发 committed end 帧 | ⑩ |
| `tool/call* -> tools/pre-execute -> ... -> tool/result*` | 工具管线四段瀑布 + 两端持久事件;详见 §4.6 | ⑪⑫ |
| `step/end` | 持久事件,**在 finally 中落盘**——无论 step 成功、失败还是取消 | ⑬ |
| `tools owe another request, or next-step input arrived -> claim -> next step` | step 以工具调用收尾(还欠一次请求)或 next-step 队列有货→ 回到 claim,同一个 turn 的下一个 step | ⑭ |
| `-> agent/turn-stopping` | **串行**检查点(无 `next()`):turn 即将关闭,监听器可通过 `agent.steer()` 再喂一步;数据说了算 | ⑭ |
| `turn/end` | 持久事件,reason ∈ `completed / aborted / blocked / error / max-tokens`(恢复路径另有 `interrupted`、`forked` 标记) | ⑮ |

---

## 4. 分阶段细讲

### 4.1 turn/start 与输入声明(claim)

驱动器的主循环在 `ReactLoopAgent.turn()`(`packages/core/agent-loop/src/agent.ts`)。开场三件事:

```ts
const turn = phase.turn + 1
try {
  this.session.append('turn/start', { turn })
} catch (error: unknown) {
  this.throwError(error)
}
phase.turn = turn
```

turn 号是**日志顺序号之外的独立计数**,随会话持久。之后进入 `while (true)` 循环,第一个动作就是 `preStep(target, { turn, step })`,其中第一个 step 的 target 是 `'next-turn'`。

注意一个语义细节:`wakeDriver()` 注释写道——"A wake sent while idle always opens its turn boundary, even when its message was cleared."空闲时收到唤醒一定会开 turn 边界,哪怕消息随后被清掉。也就是说"边界"与"内容"解耦:边界是日志事实,内容可以被 pre-step 拒绝。被取消的活动无法吸纳唤醒输入,它会重定位为下一 turn(`wakingAfterAbort` 分类)。

### 4.2 agent/pre-step:接受、改写、拒绝

`preStep()` 是 step 的"海关"。完整顺序:

1. `this.inbox.claim(target, position.turn)` —— 取走一批输入(§1.4);
2. `const selection = this.model.current()` —— 没有 provider/model 直接抛错,step 无法准备;
3. `await this.loopCtx.tools.resolve({ agent, modelSelection: selection, signal })` —— 解析本 step 准入的工具集;
4. `await this.loopCtx.systemPrompt.assemble({ ...assembleContextFor(this, signal), modelSelection: selection, toolSet })` —— 组装提示词;
5. `signal.throwIfAborted()` —— 取消检查;
6. 发起 `agent/pre-step` 瀑布:

```ts
const decision = await this.dispatch.waterfall(
  'agent/pre-step', { messages: claimed, ...position, signal, modelSelection: selection },
  (): Promise<PreStepDecision> => {
    const sections = renderContextSections(assembly)
    const context = this.runtimeContext.project(joinContextSections(sections), sections)
    return Promise.resolve<PreStepDecision>({
      kind: 'enter',
      messages: context === undefined ? claimed : [...claimed, context],
    })
  },
)
```

瀑布的 `next()` 默认实现负责把 runtime context(工作目录等注册的运行时事实)追加为附加消息。监听器可以:

- **接受**:调 `next()` 原样返回(包装型监听器必须 `{ ...decision, messages }` 保留下游消息和 `startsRequestSeries`,除非有意替换);
- **改写**:返回 `{ kind: 'enter', messages: [...自己的消息] }`,替换进入 step 的内容——steering 和注入上下文正是借此机制与用户消息合流;
- **拒绝**:返回 `{ kind: 'reject' }`。

决定是**权威的(authoritative)**。拒绝的后果在 `turn()` 里:

```ts
const decision = await this.preStep(target, { turn, step })
if (decision.kind === 'reject') {
  turnEnds = { kind: 'blocked' }
  return false
}
```

被拒绝的消息**留在了"已移除"状态**:既不回炉收件箱,也不会变成 `user/message`,turn 以 `blocked` 关闭、不产生 step。`agent/inbox/claimed` 的 JSDoc 明确记录了这个终点。

"改写为空"的分支同样在 `turn()` 里,有两层判断:

```ts
if (turnEnds && decision.messages.length === 0) break
// A removed waking message or an enter decision rewritten to empty
// still owns the initial turn boundary, but it spends no model call.
if (phase.step === 0 && decision.messages.length === 0) {
  turnEnds = { kind: 'completed' }
  return false
}
```

- 第一层:turn 已经决定要结束(比如前一个 step 以纯文本完成),而这条边界声明又是空的 → 直接 break,不再花费 step;
- 第二层:**首条声明**为空(消息被移除,或 enter 改写为空)→ turn 边界照开(`turn/start` 已落盘),但零 step 以 `completed` 关闭——这就是 §1.1 说的"0 个 step 的 turn"。

### 4.3 提示词组装:system 段 + step 准入的工具 schema

组装发生在 pre-step 瀑布**之前**,由两个能力服务完成:

- `ctx.tools.resolve({ agent, modelSelection, signal })` 返回 `ResolvedToolSet`。它是本 step 的**准入名单**:只有名单内的工具 schema 会进入提示词与请求头,名单也在 step 结束后统一 `dispose()`(许可关闭,"Admission closes once the step's tool batch settles or no step opens")。
- `ctx.systemPrompt.assemble(...)` 是 `system-prompt/assemble` 瀑布:各插件贡献提示词段,最终产出 `PromptAssembly`(sections + tools)。

关键细节:**modelSelection 快照先于一切**。`agent.model.current()` 在组装前捕获 provider、model、显式 reasoning effort,作为 `AssembleContext.modelSelection` 与 `agent/pre-step.modelSelection` 双通道携带;**重试沿用同一快照**——中途改模型选择只影响下一个 step。未指定 effort 时由适配器解析默认值。

提示词本身不在请求时临时拼字符串,而是走"历史调和":渲染好的提示词在每次尝试时与日志中既存的 `system/message` 节点调和(空文本清空、支持 `in-history` 的路由追加更新、其余路由在首个 system 节点合并)。这保证了 §5 的不变量——系统提示词也是日志的一部分。

### 4.4 agent/request 与 prepareCall:配置、取消承诺与冻结

`step/start` 落盘后,进入请求准备。`prepareRequest()` 的骨架:

```ts
const seedConfig = deepFreeze(structuredClone({ ...controls, ...selection }))
const proposedConfig = await this.dispatch.waterfall(
  'agent/request', { turn, step, signal },
  () => Promise.resolve(seedConfig),
)
signal.throwIfAborted()
if (proposedConfig.provider !== selection.provider || proposedConfig.model !== selection.model) {
  throw new Error(`agent "${this.id}": agent/request proposed "${proposedConfig.provider}/${proposedConfig.model}" after step preparation fixed "${selection.provider}/${selection.model}"; use agent.model.select() before step preparation`)
}
...
preparedCall = await this.loopCtx.llm.prepareCall(proposedConfig, signal)
config = preparedCall.config
```

要点:

1. **种子配置**:首次请求由 AgentOptions 的输出上限 + 本次 selection 组成;后续 step 从已记录的 `request/header` 折叠,并用 `requestProposal()` **剔除适配器默认值字段**(标记了 `adapterDefaults` 的 `reasoningEffort`/`maxTokens`),让当前路由重新解析默认值,显式设置则保留。
2. **`agent/request` 已弃用**:只能替换 controls(采样参数等),试图改 provider/model/effort 会直接抛错——路由选择属于 `agent.model.select()`,必须在 step 准备前完成。它不能改消息:"Model-visible content must use logged channels; this waterfall cannot mutate messages."
3. **`ctx.llm.prepareCall()` 绑定适配器**:验证适配器自有字段、解析 effort 与输出上限默认值,返回 `PreparedLlmCall`。此后整个 step(含 `request/header` 记录与派发)都沿用这个绑定的适配器。中间件可能服务未注册的路由,所以 `NO_ADAPTER` 在这里不致命;终端派发时才要求适配器存在。
4. **取消承诺**:`agent/request` 与 `prepareCall()` 都是异步阶段;注释与文档反复强调——"cancellation during either async phase commits neither system nor users"。取消发生在 step/start 之后、输入提交之前,日志里不会留下任何半提交的输入。

#### 从日志冻结(freeze)模型历史

`buildRequest()` 最后做两件事:按需落盘 `request/header`(reason:`initial`/`resume`/`change`/`series`)与 `request/context`,然后冻结请求:

```ts
// canonicalHeader is shallow; append logs a detached snapshot, not these local values.
deepFreeze(header)
const boundaryMessages = session.deriveMessages()
for (const message of boundaryMessages) {
  if (this.frozenMessages.has(message)) continue
  deepFreeze(message)
  this.frozenMessages.add(message)
}
Object.freeze(boundaryMessages)
const request = markAgentLoopRequest(Object.freeze({
  ...header.config,
  messages: boundaryMessages,
  toolHistory: session.toolHistory(),
  ...header.tools !== undefined ? { tools: header.tools } : {},
  sessionId: this.session.id,
  signal,
}))
```

三个细节值得停一停:

- 请求 = `header.config` + `deriveMessages()`(从日志投影)+ `header.tools` + `toolHistory()`。**没有独立的 `system` 字段**——系统提示词就是 `system/message` 历史。这就是"每次请求都可从日志重建"的构造性证明(§5)。
- 冻结身份用 `WeakSet` 复用:同一消息对象在多次请求间只冻结一次,已冻结的直接跳过("reuses freeze evidence for frozen identities");恢复的消息保留原身份。
- `Object.freeze` 的是消息数组与信封,**取消信号保持 live**——冻结不可变性服务于"请求不可被中途改写",而不是提前掐断取消。

### 4.5 流式:llm/stream → agent/assistant-stream → 持久结算

流式请求的循环体(`step()` 内):

```ts
const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
signal.throwIfAborted()
live.start()
started = true
for await (const chunk of stream) {
  signal.throwIfAborted()
  live.push(chunk)
}
signal.throwIfAborted()
```

`AssistantStreamAttempt` 维护本尝试的进程内状态,每帧经 `agent/assistant-stream` 发出。帧类型(`packages/core/agent/src/runtime-types.ts`)是三段封闭联合:

- `{ type: 'start', attemptId, revision, turn, step }` —— 每次尝试一帧;
- `{ type: 'chunk', ..., index, time, chunk }` —— **瞬态帧**,不持久;
- `{ type: 'end', ..., outcome: { kind: 'committed', eventType, seq } | { kind: 'abandoned' } }` —— 终帧,且**保证在对应持久结算提交之后**才发。

#### assistant/message 与 assistant/attempt 的区别

成功路径:完整内容组装成 `assistant/message` 持久化,**内嵌产生它的确切紧凑带时标流(compact timed stream)**,可回放:

```ts
live.settle(
  'assistant/message',
  () => this.session.append('assistant/message', {
    turn,
    step,
    message,
    ...live.usage === undefined ? {} : { usage: live.usage },
    stream: live.stream,
  }, { surfaceOp: 'append' }).seq,
)
```

失败/重试/取消/流错误路径:已到达的部分结算为 `assistant/attempt`——**留痕但不进模型历史**:

```ts
live.settle(
  'assistant/attempt',
  () => this.session.append('assistant/attempt', { turn, step, stream: live.stream }).seq,
)
```

这条不对称设计的动机:失败尝试的内容不可信、不完整,若进入派生历史会污染后续请求;但调试、遥测、UI 都需要看到"那次尝试到底收到了什么"。此外有一条特殊规则:**取消但已交付前缀**时,即使流被中断,只要已有可见内容,仍结算为 `assistant/message` 且带 `interrupted: true` 标记——"下一个请求包含用户已经看到的文本",用户看到的历史才算数。

硬性边界:进程在结算前崩溃,活流就此丢失,不留持久痕迹("a hard process loss before settlement leaves no durable attempt stream")。`agent/assistant-stream` 的唯一远端消费者是 Web 的 Session-follow 适配器。

### 4.6 工具管线:从 tool/call 到 tool/result

模型回复含工具调用时,`step()` 把**完整原始批次**交给 `ctx.tools.execute()`:

```ts
const maxParallelToolCalls = this.loopCtx.agentLoop.config.maxParallelToolCalls.get()
const result = await this.loopCtx.tools.execute({
  calls: toolCalls, signal, agent: this, log: { turn, step },
  maxParallelToolCalls, toolSet: decision.toolSet,
})
```

#### 批调度:barrier 与有界滚动池

调度器在 `packages/core/tools/src/batch.ts`。策略:**独占调用形成 barrier,parallel-safe 调用进有界滚动池,且每个调用开跑前重新分类**:

```ts
const fillPool = async (): Promise<void> => {
  while (!aborted && nextToStart < group.length && inFlight.size < maxParallelToolCalls) {
    // Re-read later modes after ordered commits so registry changes can create a barrier.
    const nextCall = group[nextToStart]!
    if (nextToStart > 0 && mode === 'parallel'
      && pipeline.executionMode(nextCall.exec).kind !== 'parallel') break
    await startCall(nextToStart)
    ...
  }
}
```

- `maxParallelToolCalls` 是可变 Config 字段,默认 10,`1` 即串行;
- 并发只发生在**工具体派发**上;`tools/pre-execute` 按模型顺序执行,结果也按模型顺序提交(`commitReady()` 只推进连续的已就绪槽位);
- 重新分类意味着:池子跑到一半遇到独占调用,会等当前池排空,独占调用自己成为下一个 barrier。

#### 持久事件的时机:`tool/call` 先于执行

`BatchSettlement.start()` 在派发**前**落 `tool/call` 并记住其 seq;`commit()` 落 `tool/result` 并用 `sourceEventSeqs: [callSeq]` 显式引用配对关系:

```ts
start(block: ToolCallBlock): SessionSeq | undefined {
  if (this.input.log === undefined) return undefined
  ...
  return agent!.session.append('tool/call', {
    turn, step, callId: block.id, name: block.name, arguments: block.arguments,
  }).seq
}
```

"先记录后执行"是刻意为之:执行可能崩溃、挂死,而"模型请求了什么"必须先于"我们做了什么"成为事实——这正是失败恢复(§6.1)能补记的前提。

#### 单个调用的四段瀑布

`packages/core/tools/src/pipeline.ts` 定义共享调用管线,一次调用依次经过:

1. **prepare**:`tools/pre-execute` 瀑布(hooks、权限、沙箱)→ `ask` 决定交给 `ctx.approval` 一次性审批(审批通道缺失/不可答 → 一律 deny,fail-closed)→ 单调守卫 `checkGuards()`;
2. **dispatch**:`tools/execute` 瀑布(around dispatch:超时、重试、指标包裹真正工具体);调用方与包裹层信号熔接(`fuseToolSignals`),取消不抛弃已启动的 body——已启动的 promise **先排空**,结果才标记为 `ABORTED`;
3. **complete**:`projectContent` 安装执行准备好的内容 → `tools/post-execute` 瀑布(`accept` 保留/替换 content 或 value;`block` 把结果改写为带纠正反馈的 `isError`;两者都可附带 `additionalContexts`)→ `finalizeContent`(定义自有的最后内容变换)→ `tools/result` 同步通知(结果已冻结、无损 JSON);
4. **结算**:`tool/result` 持久落盘。

三段瀑布都可能改写调用;所有异常都被规范化为 `isError` 结果而不是外抛——工具失败是**给模型看的数据**,不是驱动器异常。

执行完毕,`additionalContexts`(工具或后置策略注入的上下文)按 FIFO 进入 `next-step` 队列,等待下一个 step 声明:

```ts
for (const context of result.additionalContexts) {
  this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context])
}
```

### 4.7 step/end、"工具还欠一次请求"与 turn 收尾

`step()` 的返回值只有两种:`StepEndReason`(`completed` / `max-tokens`)或 `null`。`null` 的含义是"**工具还欠一次模型请求**"——本 step 以工具调用收尾,结果已入日志,模型需要看到结果再继续:

```ts
const toolCalls = message.content.filter(block => block.type === 'tool-call')
if (toolCalls.length === 0) return { kind: 'completed' }
...
return result.concluded ? { kind: 'completed' } : null
```

`concluded` 来自工具结果上的 `concludesTurn` 标记(工具体调用 `concludeTurn()`):数据决定 turn 是否在本 step 终止。`turn()` 收到 `null` 后回到循环顶部,`target = 'next-step'`,claim 下一批输入(通常是空的,直接用工具结果续请求),开启下一个 step。

`step/end` 在 `finally` 中落盘——无论 step 正常完成、失败还是被取消,边界事件必须成对:

```ts
} finally {
  stopRecovery()
  this.session.append('step/end', { turn, step })
}
```

#### agent/turn-stopping:关闭前的最后检查点

当 `turnEnds` 已定且 next-step 队列为空时,turn 才真正走到关闭检查点:

```ts
if (turnEnds && this.inbox.nextStep.length === 0) {
  await this.dispatch.serial('agent/turn-stopping', { turn, signal })
  signal.throwIfAborted()
}
if (turnEnds && this.inbox.nextStep.length === 0) break
target = 'next-step'
```

`agent/turn-stopping` 是 **serial** 事件:逐个 await、没有 `next()`。它的契约是"turn 即将关闭,模型不欠响应"——监听器若想续命,**通过数据说话**:调 `agent.steer(...)` 放进一条 steering,循环重读收件箱,发现 next-step 有货就再跑一个 step;没有就关闭。监听器顺序不影响结果,决定权完全在数据。

#### turn/end 与 reason

`turn()` 的 `finally` 无条件落盘:

```ts
this.session.append('turn/end', { turn, reason: turnEnds! })
```

`turnEnds` 的取值(`packages/core/session/src/types.ts` 的 `TurnEndReasonMap`):

| reason | 触发 |
|---|---|
| `completed` | 自然完成(含 0-step turn) |
| `max-tokens` | 至少一个 step 撞到输出上限;**粘性**:后续正常完成的 step 不能降级这个结果 |
| `aborted` | 取消,附持久化 cause(`user`/`parent`/`hook`+reason/`disposed`) |
| `blocked` | 首条声明被 `agent/pre-step` 拒绝 |
| `error` | 结构化失败(`LlmError` 事实原样保留,其他错误压平为 `{ message, code }`) |
| `interrupted` / `forked` | 恢复专用标记:崩溃遗留 turn 的补记、fork 种子边界;活循环永不发出 |

关闭后若 `inbox.hasPending` 还有货(比如排队中的下一条 followup),驱动器换上**新的 AbortController**、`step` 归零,`turn()` 返回 `true`,`kick()` 的 `while (await this.turn())` 继续开下一个 turn。旧 controller 上遗留的唤醒闩锁因此失效——活驱动器自己 claim 队列。

---

## 5. 关键不变量:模型可见 ⟺ 已记录

architecture.md 的一句话:

> **Model-visible means logged.** Every model request must be reconstructable from the log. New model-visible inputs require session events.

### 为什么成立

这不是口号,而是 `buildRequest()` 的构造性结果:请求的每一个组成部分都来自日志——

- `messages` = `session.deriveMessages()`,纯日志投影,没有旁路;
- `tools` = 本 step 准入的工具 schema,且它们已经在 `request/header` 里落盘(信封也是日志事件);
- 系统提示词不是请求的独立字段,而是 `system/message` 历史节点,同样落盘;
- `toolHistory()` 也是日志派生。

反过来,所有进入请求的内容都在请求**之前**以持久事件形式提交:`user/message`(声明准入)、`system/message`(提示词调和)、`tool/result`(上一 step 的工具结果)、`request/header` + `request/context`(信封)。所以任意时刻,拿着会话日志就能逐字节重建当时发给模型的完整请求。

### 它约束了什么

1. **新增模型可见输入 = 必须新增会话事件**。想让模型看到任何东西(工具注入的上下文、插件补充的指令),没有捷径,只能走持久事件通道。这就是 `agent.inject()` 也要经 inbox splice 落盘的原因。
2. **改写既有消息内容 = 必须注册 pure message projections**——插件不能直接篡改历史节点,而是声明纯投影,读取方(包括离线读者)显式应用同一投影。
3. **`agent/request` 瀑布不能碰消息**(JSDoc 原文:"Model-visible content must use logged channels; this waterfall cannot mutate messages")。它只能改 controls。
4. **`agent/assistant-stream` chunk 是瞬态的,不违反不变量**——chunk 是输出方向的活增量,不是模型输入;真正持久的 `assistant/message` 内嵌完整流,回放读取持久结算。反例也成立:结算前崩溃,活流丢失、不留痕,因为它从未成为模型历史的一部分。
5. **失败尝试的流不进历史**(`assistant/attempt`),因为派生历史由持久 `assistant/message` 组成——不可信内容天然被挡在模型可见集合之外。

这条不变量是 fork、resume、转录、遥测、持久化全部"从日志派生"的地基:任何这些功能都不需要额外的辅助状态。

---

## 6. 失败与恢复

### 6.1 失败的 step:补记缺失的工具结果

模型可能中途失败(网络断、适配器错),而它已经声明的工具调用可能有的执行了、有的没执行。**带着未配对的 `tool/call` 关闭 step,会让后续请求的派生历史不合法**(provider 要求 call/result 成对)。驱动器用 `ToolCallRecovery` 在 step 关闭前补齐:

```ts
const toolRecovery = new ToolCallRecovery()
const stopRecovery = this.ctx.on('session/event', (session, event) => {
  if (session === this.session) toolRecovery.observe(event)
})
try {
  const stepEnd = await this.step(decision)
  ...
} catch (error: unknown) {
  try {
    for (const event of toolRecovery.results()) {
      this.session.append('tool/result', event.data, {
        surfaceOp: 'append',
        ...event.sourceEventSeqs === undefined ? {} : { sourceEventSeqs: event.sourceEventSeqs },
      })
    }
  } catch (recoveryError: unknown) {
    throw new AggregateError([error, recoveryError], 'Step failed and its pending tool results could not be recorded', { cause: error })
  }
  throw error
} finally {
  stopRecovery()
  this.session.append('step/end', { turn, step })
}
```

补记规则(packages/core/agent-loop/README.md "Unanswered calls after step failure"):

- **已记录 `tool/call` 但没有结果** → `TOOL_OUTCOME_UNKNOWN`:结果文本明说"Its outcome is unknown.",只允许对只读/幂等操作重试;可能有副作用的操作必须先检查外部状态或询问用户;
- **请求里有调用但连 `tool/call` 记录都没有** → `TOOL_NOT_STARTED`:"The tool call was interrupted before the Harness recorded it as started. Retry it if it is still needed."
- 已提交的结果原样保留;已启动的派发先排空再恢复;turn 的 `error` 结局保留**原始失败**,而不是被恢复过程掩盖。
- 若连补记都失败(日志本身写不进),原始错误与恢复错误一起包成 `AggregateError` 抛出——绝不让日志处于静默不一致状态。

### 6.2 请求失败与重试:不重复组装

`agent/request-error` 瀑布(`packages/core/agent/src/runtime-types.ts`)是模型请求失败的处理点。终态适配器选择、派发、迭代失败以"终态 finish"进入这里;监听器返回 `{ kind: 'retry' }` 时**不调用 `next()`**,表示"我接管了恢复";无人处理则原始失败终态化,抛出 `LlmError`。

重试发生在**同一个打开的 step 里**,而且是刻意廉价的:`step()` 的 `while (true)` 循环里 `firstAttempt` 在第一轮后被置 `false`,因此:

- **不重复组装**:提示词与工具集在 `preStep()` 只组装一次,`renderPrompt(assembly)` 在 step 开头渲染一次,重试直接复用;
- **不重复 `agent/pre-step`**:准入决定不重跑;
- **不重复用户准入**:`decision.messages` 只在 `firstAttempt` 时提交为 `user/message`;
- 每次尝试只重做:刷新注册的 runtime facts、调和系统提示词、按需补记 header/context、重新 `deriveMessages()` 冻结请求。

每次尝试的失败流照常结算为 `assistant/attempt`,所以 N 次尝试在日志里留下 N 条尝试痕迹 + 最多一条成功消息。已弃用的 `agent/request` 瀑布**不在重试中重跑**——重试沿用同一信封。内置消费者示例:`dsh-compaction-basic` 在上下文溢出时在此触发压缩再重试,`llm-retry` 实现通用退避。

### 6.3 取消语义

取消的入口是 `agent.cancel(cause, options)`:

```ts
cancel(cause: AgentCancelCause, options: CancelOptions = {}): void {
  if (!options.keepInbox) {
    this.inbox.clear()
    if (this.phase.kind !== 'idle') this.phase.wakeRequested = false
  }
  if (this.phase.kind !== 'idle') this.phase.abort.abort(cause)
}
```

- 默认**清空收件箱**(`next-step` 先于 `next-turn` 清);`keepInbox: true` 保留待处理输入;
- cause 取值:`user` / `parent` / `hook`(带 reason)/ `disposed`;
- 取消是**协作式**的:信号逐层传播——流式循环每个 chunk 后 `throwIfAborted()`,工具管线在 prepare/dispatch/complete 多个关口检查,已启动的工具 body 排空到静默才结算。

三个值得记住的精确行为:

1. **已交付的文本不丢**。取消的流若已有可见内容,结算为 `interrupted: true` 的 `assistant/message`,把用户看到的前缀固化进历史(§4.5);未派发的工具调用收到合成 `tool/call` + `ABORTED_BEFORE_DISPATCH` 结果对,保证历史可回放。
2. **日志里的 cause 是拷贝**。`abortedCancelCause()` 只复制 `turn/end` 需要的字段:`AbortSignal.reason` 是调用方的活对象,Node 的 fetch 会往上挂 `stack`——那不是可 JSON 化的数据,拷贝把它挡在日志之外,保证 `turn/end { kind: 'aborted', reason }` 永远可落盘。
3. **取消关闭 turn,但不关闭驱动器**。`turn()` 的 catch 把 cause 翻译成 `aborted` 结局后继续上抛,`kick()` 在驱动器边界收容(`Reported failures and cancellation are contained at the driver boundary`),`agent/error` 先行通知;`finally` 里 `turn/end` 照常落盘。已闩住的唤醒(`wakeRequested`)在下一次收敛时重放。

---

## 7. 流程步骤 → 源码 → 教程章节映射表

| 流程步骤 | 源码(相对主仓库根) | 本教程章节 |
|---|---|---|
| 收件箱命令与声明 | `packages/core/agent-loop/src/inbox.ts` | §1.2–1.4 |
| turn/step 状态机、取消、冻结请求 | `packages/core/agent-loop/src/agent.ts` | §3、§4.1–4.7 |
| 活流帧(start/chunk/end)结算 | `packages/core/agent-loop/src/assistant-stream.ts`;帧类型 `packages/core/agent/src/runtime-types.ts` | §4.5 |
| runtime context 投影 | `packages/core/agent-loop/src/runtime-context.ts` | §4.2 |
| `agent/*` 事件契约 | `packages/core/agent/src/runtime-types.ts` | §2.1、§4.2、§4.7、§6.2 |
| 提示词组装与 system/message 调和 | `packages/core/system-prompt/src/index.ts` | §4.3 |
| 工具调用管线(四段瀑布) | `packages/core/tools/src/pipeline.ts` | §4.6 |
| 工具批调度(barrier/滚动池) | `packages/core/tools/src/batch.ts` | §4.6 |
| `SessionEventMap`、`TurnEndReason` | `packages/core/session/src/types.ts` | §2.1、§4.7 |
| `deriveMessages()` / `toolHistory()` 投影 | `packages/core/session`(见该包 README) | §4.4、§5 |
| Turn flow 权威伪代码与可视化 | `docs/architecture.md`(Turn flow)、`docs/agent-lifecycle.md`、`docs/tool-execution-pipeline.md`、`docs/event-producer-consumer.md` | §2、§3、§4.6 |

后续章节深入链接:核心引擎篇将逐包拆解 `agent`、`agent-loop`、`tools`、`session`、`llm`、`system-prompt`(见 `docs/02-core-engine/` 各目录)。

---

## 小结

把一次对话的生命周期压缩成三句话:

1. **inbox 是唯一入口,claim 是唯一出口**:输入持久排队,驱动器在 turn 边界原子取走一批,交由 `agent/pre-step` 瀑布裁决接受、改写或拒绝——拒绝或空声明意味着一个零 step 的 turn。
2. **每个 step 是"从日志来、回日志去"的一个闭环**:快照模型选择 → 组装 → `agent/request` + `prepareCall` → 提交 system/user → 冻结派生历史 → 流式 → 工具四段瀑布 → `step/end`;工具还欠一次请求,就再来一个 step。
3. **模型可见 ⟺ 已记录**是全流程的纲:请求的每个字节都能从会话日志重建,所以 fork、resume、转录、遥测一律派生自日志,不需要任何影子状态。

下一章我们转向系统如何启动:profile、bundle 与 cordis.yml 的组合机制(《Boot 与 Profile》)。
