# `events.ts` — 管线事件
> 工具执行策略的五个共享事件


`packages/core/tools/src/events.ts` 只有 58 行，却是整个工具管线的"插头面板"：它通过 Cordis 的声明合并把 5 个事件挂进全局 `Events` 接口 —— 三段 waterfall（`tools/pre-execute`、`tools/execute`、`tools/post-execute`）和两个 emit（`tools/result`、`tools/change`）。文件里没有任何实现：管线（`pipeline.ts`）负责触发，插件负责监听。类型上的一个贯穿设计是 `this: Scoped<ToolsService>` —— 监听器通过 `dsh-scope` 的作用域过滤分发，agent-scoped 的 listener 只收到那个 agent 的调用。

## 文件概览

| 事件 | 模式 | 签名要点 | 触发位置 |
|---|---|---|---|
| `tools/pre-execute` | waterfall | `(exec, next) => PreToolDecision` | 管线 `prepare()`，参数冻结后、guard 前 |
| `tools/execute` | waterfall | `(exec: ToolDispatchExecution, next) => ToolExecutionResult` | 管线 `dispatch()`，包住工具正文 |
| `tools/post-execute` | waterfall | `(exec, result, next) => PostToolDecision` | 管线 `complete()`，正文结果规范化后 |
| `tools/result` | emit | `(exec, result) => undefined` | 管线 `finish()`，结果冻结后 |
| `tools/change` | emit | `() => void` | 注册表条目增删（`ScopedLayers` 回调） |

## 关键代码走读

### 1. 声明合并的骨架

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    /**
     * Allow, deny, cancel, or ask before dispatch. `next()` delegates to allow;
     * ...
     * @param exec - the pending call (name, parsed arguments, caller agent).
     * @mode waterfall
     */
    'tools/pre-execute'(this: Scoped<ToolsService>, exec: ToolExecution, next: () => Promise<PreToolDecision>): Promise<PreToolDecision>
```

全部事件都长在 `@deepseek-ai/cordis` 的 `Events` 接口上（typed events use declaration merging）。每个 JSDoc 都带 `@mode`（waterfall / emit）和 `@param` —— 这是仓库对事件文档的硬性要求。`this: Scoped<ToolsService>` 把作用域过滤语义写进类型：监听器注册在哪个 scope，就只看到哪个 agent 的调用（`tools/change` 是唯一无参数、无过滤的例外）。

### 2. `tools/pre-execute` — 可重排的准入门

```ts
'tools/pre-execute'(this: Scoped<ToolsService>, exec: ToolExecution, next: () => Promise<PreToolDecision>): Promise<PreToolDecision>
```

JSDoc 契约的要点：

- `next()` 委托为 allow —— 这是 waterfall 语义：不拦截就调用 `next()` 把决定权交给下一个 listener，全部放行则默认 allow；
- 返回 `{ kind: 'deny' }` 短路整条链，`reason` 会变成模型可见的 `Error: <reason>`，可选 `info` 附带结构化错误；
- 返回 `{ kind: 'cancel' }` 选择规范的前分发取消结果（`ABORTED_BEFORE_DISPATCH`），不呈现策略拒绝；
- 返回 `{ kind: 'ask' }` 交给审批 seam（`ctx.approval`）：只有 `allowed-once` 放行，审批服务缺失、无 agent 可路由、用户拒绝/取消/通道不可用都退化为 deny；
- **异步 listener 必须观察 `exec.signal`**；管线在它们落定后会重查取消，但从不抛弃它们的 promise；
- `exec` 是冻结参数后的只读 `ToolExecution` —— 参数改写被刻意排除（日志、审计、UI 与实际执行必须一致）。

**审批挂在哪一段**：就在这一段。`pipeline.ts` 的 `prepare()` 跑完这个 waterfall 后，若决定是 `ask`，会调 `serviceAsk()` 走 `ctx.approval.request()`，映射回 allow/deny，然后才查 guard（见下）。

### 3. `tools/execute` — around 包装（唯一能换 signal 的一段）

```ts
'tools/execute'(this: Scoped<ToolsService>, exec: ToolDispatchExecution, next: () => Promise<ToolExecutionResult>): Promise<ToolExecutionResult>
```

这是三段中唯一拿到 `ToolDispatchExecution` 的一段：包装者可以替换 `exec.signal`（比如超时插件给下游一个限时信号），但身份字段（callId/token/args 等）不可变。`next()` 直接返回规范化的 `ToolExecutionResult`，包装者可以透传、替换或据此重试。两个安全阀保证"替换无法甩掉调用方取消"：

1. 管线的 `dispatchToolBody()` 把**原始调用方 signal 与包装者替换的 signal 融合**后再交给工具正文；
2. 包装者自己仍须恢复 signal 并等自己的工作 quiescence。

**超时挂哪一段**：`@deepseek-ai/dsh-tool-call-timeout-policy` 就是一个 `tools/execute` wrapper —— 这也解释了 `ToolDefinition.timeoutMs` 为何只是声明：registry 不强制，强制逻辑属于这个可选包装。

### 4. `tools/post-execute` — 结果的检查站

```ts
'tools/post-execute'(this: Scoped<ToolsService>, exec: ToolExecution, result: Readonly<ToolExecutionResult>, next: () => Promise<PostToolDecision>): Promise<PostToolDecision>
```

拿到的是**规范化后的 dispatch 结果** —— 注意"工具抛错也算跑完了"：thrown tools still reach this waterfall as errors，所以策略插件看到的结果全集与模型将看到的一致。listener 可以：

- `next()` 原样接受；
- 返回 `accept`（可换 `content` 或 `value`，二选一；可附 `additionalContexts` 注入下一条用户消息）；
- 返回 `block`：把纠错 `feedback` 变成 isError 结果，feedback 文本成为 error.message。

异步 listener 同样必须观察 `exec.signal`；落定后若调用方已取消，管线只把**成功的 accepted 结果**替换成取消结果（错误结果保留更具体的失败身份），替换成哪个取消码取决于正文是否已启动。

### 5. `tools/result` — 只读终点

```ts
/**
 * Observe the frozen, lossless-JSON final outcome. Listener failures are contained.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): keyed by `exec.agent`.
 * @param exec - the execution object that traversed the pipeline.
 * @param result - a deep-frozen snapshot of the final returned result.
 * @mode emit
 */
'tools/result'(this: Scoped<ToolsService>, exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): undefined
```

emit 模式、没有 `next`、观察者**无法改变结果** —— 管线在触发前把执行对象和结果都 `Object.freeze`，观察者抛错或返回被拒绝的 promise 只会被 `ctx.logger.warn` 记录，不会污染结果。这是遥测/审计/计费类插件的挂点。作用域按 `exec.agent` 过滤。

### 6. `tools/change` — 注册表变更通知

```ts
/**
 * A tool was registered or unregistered, or a scoped restriction changed.
 * Unfiltered registry notification: scoped listeners receive changes from every
 * scope because a global contribution can affect every agent's next assembly.
 * @mode emit
 */
'tools/change'(): void
```

无参数、**不过滤作用域**：scoped listener 也会收到所有 scope 的变化 —— 因为一个全局注册可能影响任何 agent 的下一次提示词组装。`ToolsService` 构造 `ScopedLayers` 时把 `() => this.ctx.emit('tools/change')` 作为层变化回调传入，所以工具注册/反注册、restriction 增删都会触发。典型消费者是提示词前缀缓存失效逻辑。注意：**执行器的注册与卸载不触发它**（提示词组装与批次在运行时各自解析当前执行器）。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `tools/pre-execute` | waterfall `(exec, next) => PreToolDecision` | 准入门：allow/deny/cancel/ask；审批策略挂点 |
| `tools/execute` | waterfall `(exec: ToolDispatchExecution, next) => ToolExecutionResult` | around 包装：超时/重试/指标；唯一可换 signal |
| `tools/post-execute` | waterfall `(exec, result, next) => PostToolDecision` | 结果检查站：accept/replace/block + 附加上下文 |
| `tools/result` | emit `(exec, result) => undefined` | 冻结结果的只读观察，观察者失败被包容 |
| `tools/change` | emit `() => void` | 注册表/限制变化通知，无作用域过滤 |

## 与其他模块的关系

- `contracts.ts` 提供全部载荷类型（`ToolExecution`、`ToolDispatchExecution`、`ToolExecutionResult`、`PreToolDecision`、`PostToolDecision`）。
- `pipeline.ts` 是唯一的触发方：`prepare()` 触发 pre-execute，`dispatch()` 触发 execute，`complete()` → `postExecute()` 触发 post-execute，`finish()` → `notifyResult()` 触发 result。
- `index.ts` 是 `tools/change` 的触发方（经 `ScopedLayers` 回调）与 `Scoped<ToolsService>` 的 `this` 提供方。
- guard **不在这里**：`ctx.tools.guard()` 是普通函数注册（`GuardLayer`），管线在 pre-execute 决议之后调 `checkGuards()`，这是"审批可重排、owner 策略不可重排"的分层设计。
- `dsh-scope` 提供作用域过滤分发（`Scoped<T>`），`tool-executor-ptc` 另有 `tools/ptc-dispatch-log` waterfall（声明在 PTC 包，改写的只是持久化日志副本）。

## 小结

`events.ts` 把工具执行的全部扩展点收敛成 5 个有严格签名契约的事件：**前置**（allow/deny/cancel/ask，审批在这）、**around**（换 signal 的唯一位置，超时在这）、**后置**（改结果/阻断/注入上下文）、**终点**（只读冻结结果）、**注册表**（变化广播）。三段 waterfall 共享同一个 Cordis 惯例 —— 调 `next()` 表示放行默认，返回 decision 表示短路 —— 而 guard 刻意不走事件系统，用单调函数补上"绝不可翻案的拒绝"。理解这 5 个签名，就理解了所有工具策略插件的能力边界。
