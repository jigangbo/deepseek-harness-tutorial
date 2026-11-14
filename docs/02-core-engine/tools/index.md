# Tools — 工具注册表与执行管线

`packages/core/tools`（npm 包名 `@deepseek-ai/dsh-tools`）是 harness 的工具中枢：它提供 `ctx.tools` 这个服务，所有工具插件把 `ToolDefinition` 注册进来，所有 agent 循环通过它把模型发出的 tool-call 批次交给执行器执行。它是"注册表 + 执行管线"两层：

- **注册表层**：维护按 scope 分层的工具定义（`ScopedLayers`），自动把可见工具的 name/description/parameters 投影进 system prompt 组装；
- **执行管线层**：内置执行器（native / PTC）复用共享的 `ToolExecutionPipeline` —— 三段 waterfall（`tools/pre-execute` → `tools/execute` → `tools/post-execute`）加单调 guard 和只读 `tools/result`，把"权限、超时、改写结果、观察"这些横切策略从工具本体里剥离出去。

一个关键设计：**注册即可见**。工具插件只要 `ctx.tools.register(defineTool({...}))`，它的参数 schema 就会自动进入下一次提示词组装，不需要任何"把工具塞进提示词"的额外步骤。

## 文件概览

| 文件 | 角色 |
|---|---|
| `packages/core/tools/src/index.ts` | 插件入口：`ToolsService`（注册表 + scoped 执行器分发 + guard 注册），本文讲 |
| `packages/core/tools/src/contracts.ts` | 全部核心类型契约：`ToolDefinition`、执行对象、结果联合、决策类型、guard、restriction |
| `packages/core/tools/src/events.ts` | 5 个共享事件的声明合并（三段 waterfall + 两个 emit） |
| `packages/core/tools/src/executor.ts` | `ToolExecutor` 契约与原始批次输入/输出类型、批次级错误 |
| `packages/core/tools/src/pipeline.ts` | 共享调用管线：prepare / dispatch / complete 三阶段、取消语义、结果规范化 |
| `packages/core/tools/src/batch.ts` | 批次调度：exclusive 屏障 + parallel 滚动池、有序落账、Session 日志 |
| `packages/core/tools/src/schema.ts` | 统一 schema DSL、类型推导、`defineTool`、模型可见 schema 投影 |
| `packages/core/tools/src/json-schema.ts` | 受支持的 JSON Schema 子集：断言与值校验 |
| `packages/core/tools/src/presentation.ts` | UI 渲染意图词汇（generic/terminal/diff/search/read/web 卡片） |

## 执行管线总览

一次模型工具调用从发出到落账的完整路径（ASCII 版，对应 `docs/tool-execution-pipeline.md` 的 mermaid 图）：

```text
模型 assistant 消息携带 tool-call block
        │
        ▼
ctx.tools.execute(batch) ── 选择最近 scope 的 ToolExecutor（native / PTC）
        │
        ▼
batch.ts executeToolBatch ── exclusive 屏障 / parallel 滚动池排程
        │                     start(): 追加 tool/call 事件
        ▼
pipeline.ts prepare() ── 参数 lossless JSON 快照 + deepFreeze + 分配 token
        │
        ▼
tools/pre-execute waterfall ── 可重排的 allow / deny / cancel / ask
        │                        ask → ctx.approval 一次性审批
        ▼
ctx.tools.checkGuards() ── 注册式单调 guard：返回 reason 即 deny，
        │                    后来的 listener 无法翻案为放行
        ▼ allow
pipeline.ts dispatch() ── tools/execute waterfall（超时/重试等 around 包装，
        │                   唯一可以换 signal 的一段）
        ▼
工具 execute(args, exec) 正文 ── 正文必须观察 exec.signal 协作取消
        │
        ▼
pipeline.ts complete() ── projectContent（正文预备内容）
        │                   tools/post-execute waterfall（accept/replace/block）
        │                   finalizeContent（定义自有的最后内容变换）
        │                   tools/result（emit：冻结后的最终结果，观察者失败被包容）
        ▼
batch.ts commit() ── 追加 tool/result 事件（content/error/meta）
        │             additionalContexts 按模型顺序收集
        ▼
ToolBatchResult ── 回到 agent loop，进入下一步请求
```

三段 waterfall 各司其职：

| 事件 | 模式 | 挂什么 | 能做什么 |
|---|---|---|---|
| `tools/pre-execute` | waterfall | 审批策略、hooks、沙箱预检 | allow / deny / cancel / ask（ask 走 `ctx.approval`）；**不能改写参数** |
| `tools/execute` | waterfall | 超时（`dsh-tool-call-timeout-policy`）、重试、指标 | around 包装 dispatch；只能替换 `exec.signal`，身份字段不可变 |
| `tools/post-execute` | waterfall | 结果改写、纠错反馈、上下文注入 | accept（换 content 或 value）/ block（变成 isError） |
| `tools/result` | emit | 遥测、审计 | 只读观察冻结结果；观察者抛错被记录并包容 |
| `tools/change` | emit | 提示词缓存失效等 | 注册表变化通知（无参数） |

guard 不经过事件系统：`ctx.tools.guard(fn)` 把函数存进 `GuardLayer`，管线在 pre-execute 决议之后、dispatch 之前调 `ctx.tools.checkGuards(exec)`。它的返回类型刻意**没有 allow** —— 返回 reason 只能把权限调低，所以 listener 顺序永远无法把一次拒绝翻案成放行（单调 deny）。

## `index.ts` 走读 — `ToolsService`

### 服务声明与配置

```ts
/** Concrete tool registry and scoped executor dispatcher. */
export class ToolsService extends Service {
  static inject = ['systemPrompt']
  static Config = z.object({
    mode: z.never().description('Removed. Mount a native or PTC tool executor instead.'),
    maxParallelSubCalls: z.never().description('Moved to the PTC tool executor configuration.'),
  })
```

两个配置字段用 `z.never()` 显式声明为"已移除"：旧部署里还在写 `mode` / `maxParallelSubCalls` 的 cordis.yml 会在加载时炸出明确错误（misconfiguration fails loud），而不是被静默忽略。并发上限已经搬去 PTC 执行器自己的配置。

### 注册表的三层结构

```ts
/** One registered definition and its step-time activation condition. */
interface ToolRegistration {
  readonly definition: ToolDefinition
  readonly when: ToolRegistrationOptions['when']
}

/** One scope's complete tool-registry contribution. */
class ToolLayer implements ScopeLayer {
  readonly tools: NamedEntries<ToolRegistration>
  readonly restrictions = new AnonymousEntries<CompiledToolRestriction>()
```

每个 scope 一层 `ToolLayer`：`tools` 是具名条目（同名注册报错，全局层与 scoped 层的报错文案不同），`restrictions` 是匿名条目（一个 scope 可以叠加多条限制）。整个服务再包一层 `ScopedLayers`，层的任何增删都会触发 `tools/change`：

```ts
private readonly layers = new ScopedLayers(scope => new ToolLayer(scope), () => { this.ctx.emit('tools/change') })
```

### 构造函数：schema 自动进提示词

```ts
constructor(ctx: Context) {
  super(ctx, 'tools')
  ctx.systemPrompt.tools(({ scope, toolSet }) => {
    const executor = this.selectExecutor(scope)
    const description = executor.describe(this.definitions(scope, toolSet))
    return { ...description, knownNames: description.knownNames ?? [...this.view(scope).knownNames] }
  })
}
```

这就是"注册即可见"的机制：`system-prompt` 组装请求时会回调这里。registry 把**当前 scope 可见的定义列表**交给**当前 scope 选中执行器的 `describe()`**，由执行器决定线上的 `ToolSchema[]` 形态 —— native 直接暴露每个工具，PTC 则折叠成一个 `run_code`。`knownNames` 用于提示词里工具顺序的校验兜底。

### register()：注册即生效

```ts
register(definition: ToolDefinition, options: ToolRegistrationOptions = {}): () => void {
  const name = definition.name
  const output = (definition as Partial<ToolDefinition>).output
  if (output === undefined || typeof output !== 'object'
    || typeof output.render !== 'function'
    || (output.presentationMeta !== undefined && typeof output.presentationMeta !== 'function')) {
    throw new TypeError(`tool "${name}" must declare output { schema, render, presentationMeta? }`)
  }
  assertSupportedJsonSchema(output.schema)
  const timeoutMs = definition.timeoutMs
  if (timeoutMs !== undefined
    && (!Number.isFinite(timeoutMs) || timeoutMs <= 0)) {
    throw new TypeError(`tool "${name}" timeoutMs must be a positive finite number`)
  }
  // Reserved unconditionally: any agent may select a code mode for itself,
  // so a name free to take under the deployment default would become a
  // collision the moment a preset mounted.
  if (name === RUN_CODE_NAME) {
    throw new Error(`tool name "${RUN_CODE_NAME}" is reserved for the PTC mode presentation transport and cannot be registered or shadowed`)
  }
  return this.layers.effect(
    this.ctx,
    layer => layer.tools.insert(name, { definition, when: options.when }),
    { label: 'tools.register()' },
  )
}
```

四个检查：`output` 声明必须有 schema + render；原始 output schema 必须落在受支持的 JSON Schema 子集里；`timeoutMs` 必须是正有限数（注意它只是声明，registry 从不强制超时 —— 强制需要 `dsh-tool-call-timeout-policy` 这个 `tools/execute` 包装插件）；`run_code` 是 PTC 传输保留名，任何 scope 都不能注册或遮蔽。返回值是**精确的 disposer**（effect 注册，随插件卸载自动反注册）。

### view()：scope 链上的可见性推导

```ts
private view(scope?: ScopeKey): ToolView {
  const layers = this.layers.chainLayers(scope)
  const own = this.layers.peek(scope)
  const inherited = new Map(this.layers.global.tools.entries())
  for (const layer of layers) {
    if (layer === own) continue
    for (const [name, registration] of layer.tools.entries()) inherited.set(name, registration)
  }
  const visible = new Map<string, ToolDefinition>()
  ...
  for (const [name, registration] of inherited) {
    knownNames.add(name)
    restrictableNames.add(name)
    if (layers.every(layer => layer.admits(name))) {
      visible.set(name, registration.definition)
      registrations.set(name, registration)
    }
  }
  if (own !== undefined) {
    for (const [name, registration] of own.tools.entries()) {
      knownNames.add(name)
      visible.set(name, registration.definition)   // 自有注册豁免 restriction
      registrations.set(name, registration)
    }
  }
  return { visible, registrations, knownNames, restrictableNames }
}
```

规则：沿 scope 链从远到近收集继承的工具，**近层遮蔽远层**；每一层的 restriction 都要放行，继承工具才可见；而**查看 scope 自己注册的工具豁免所有 restriction** —— 这样被委派的子 agent 不会把自己要用的工具过滤掉。一次遍历同时产出四样东西：可见定义、注册条目（给 `executable` 判断 `when` 用）、限制前的全部名字（`knownNames`）、可被 restriction 点名的全局名。

### resolve()：条件激活（step 级准入）

```ts
async resolve(context: ToolActivationContext): Promise<ResolvedToolSet> {
  const { agent, signal } = context
  signal.throwIfAborted()
  const view = this.view(agent)
  const admitted = new Map<string, ToolDefinition>()
  for (const [name, definition] of view.visible) {
    const registration = view.registrations.get(name)
    let enabled: boolean
    try {
      enabled = registration?.when === undefined || await registration.when(context)
    } catch (cause: unknown) {
      signal.throwIfAborted()
      throw new ToolActivationError(name, cause)
    }
    ...
  }
  let closed = false
  return Object.freeze({
    dispose: () => { closed = true },
    view: (scope) => { /* closed 或 scope 不匹配 → 空；否则按当前注册表复核 */ },
  })
}
```

`register(def, { when })` 的条件函数在每个模型 step 前被 `resolve()` 逐一求值：条件抛错不是"该工具这步不可用"，而是 `ToolActivationError`（`TOOL_ACTIVATION_FAILED`）让 step 准备失败 —— 因为"不知道工具在不在"和"确定不在"是两回事。返回的 `ResolvedToolSet` 被 agent loop 传给提示词组装和本 step 的整个批次（含 PTC 子调用），step 结束后 dispose。`view()` 每次读取都按**当前注册表**复核"当初准入的定义是否还是原来那个注册条目"，所以 dispose、同名校换、新 restriction 都会让未启动的调用失效，但已被准入的注册无法靠"迟到的条件结果"复活。

### 执行器选择与批次分发

```ts
async execute(input: ToolBatchInput): Promise<ToolBatchResult> {
  return this.selectExecutor(input.agent).execute(input)
}

private selectExecutor(scope?: ScopeKey): ToolExecutor {
  let record: Registration | undefined
  for (const key of scopeChainOf(scope)) {
    record = this.scoped.get(key)
    if (record !== undefined) break
  }
  record ??= this.root
  const executor = record?.executor
  if (executor === undefined) throw new ToolExecutorUnavailableError()
  return executor
}
```

`setExecutor()` 每个 scope 只允许一个执行器（重复注册报错）；卸载时只把 `record.executor = undefined` 而**保留记录** —— 这样一个已接受的批次不会被"掉落"到另一个执行器上，新批次则拒绝直到换上替代品。registry 与执行器的分工是：registry 管"谁能被调用"（定义、限制、准入），执行器管"怎么调用"（原生 function-calling 还是 PTC 的 `run_code` 程序）。两者的详细对照见 [executors 章节](../executors/index.md)。

### guard 注册与检查

```ts
guard(guard: ToolGuard): () => void {
  return this.guards.effect(this.ctx, layer => layer.guards.append(guard), { label: 'tools.guard()' })
}

checkGuards(exec: ToolExecution): string | undefined {
  for (const layer of [this.guards.global, ...this.guards.chainLayers(exec.agent)]) {
    for (const guard of layer.guards.values()) {
      const reason = guard(exec)
      if (reason !== undefined) return reason
    }
  }
  return undefined
}
```

guard 也是 scoped 分层的：全局层先查，再沿 agent 的 scope 链；第一个返回 reason 的 guard 赢。管线（`pipeline.ts` 的 `prepare`）在 pre-execute waterfall 决议为 allow 之后调用这里，所以 guard 是**最后的、不可协商的**预分发策略。

### 模块声明合并

`index.ts` 顶部有三处 `declare module`：给 `Context` 挂 `tools` 属性、给 `AssembleContext` 挂 `toolSet`（step 准入）、给 `MessageSourceMap` 挂 `'tool-registry'`（注册表变化作为消息来源标记）。文件尾部 `export *` 重导出四个子模块的全部公开 API，并提供别名 `export { ToolsService as ToolRuntime }`。

## 推荐阅读顺序

1. **本文** + [contracts.md](contracts.md) —— 先建立类型词汇表：工具定义、执行对象、结果联合、决策类型。
2. [events.md](events.md) —— 5 个事件各自的签名与挂点。
3. [executor.md](executor.md) —— 批次的输入/输出契约，以及执行器插件要实现什么。
4. [pipeline.md](pipeline.md) —— 单次调用的 prepare / dispatch / complete 三阶段，本模块最核心的文件。
5. [batch.md](batch.md) —— 多调用的并发调度、有序落账、取消与日志。
6. [schema.md](schema.md) —— `defineTool` 与 schema DSL，工具作者每天都打交道的一层。
7. [json-schema.md](json-schema.md) —— 上面一切的校验底座。
8. [presentation.md](presentation.md) —— UI 卡片词汇，与执行完全解耦。

配套外部文档：`packages/core/tools/README.md`（包契约）、`docs/subsystems/tools.md`（完整类型与生成目录）、`docs/tool-execution-pipeline.md`（管线图）。

## 扩展点速查

| 想做什么 | 用什么 | 位置 |
|---|---|---|
| 提供一个新工具 | `ctx.tools.register(defineTool({...}))` | 工具插件 |
| 按 step/model 条件启用工具 | `register(def, { when })` + agent loop 的 `resolve()` | `contracts.ts` / `index.ts` |
| 收窄某 agent 能看到的全局工具 | `ctx.tools.restrict({ allow? , deny? })` | `index.ts` |
| 不可翻案的策略拒绝 | `ctx.tools.guard(fn)`（单调 deny） | `index.ts` / 管线 prepare |
| 可重排的 allow/deny/ask 审批门 | 监听 `tools/pre-execute` waterfall | `events.ts` |
| 给所有调用加超时/重试/指标 | 监听 `tools/execute` waterfall（around） | `events.ts` |
| 检查/替换/阻断结果、注入上下文 | 监听 `tools/post-execute` waterfall | `events.ts` |
| 只读观察最终结果 | 监听 `tools/result`（emit，观察者失败被包容） | `events.ts` |
| 更换模型交互形态（原生 vs 代码模式） | `ctx.tools.setExecutor(executor)` —— native 或 PTC | 见 executors 章节 |
| 自定义卡片渲染意图 | `presentCall` / `presentResult`（纯函数） | `presentation.ts` |

## 小结

`packages/core/tools` 把"工具"这件事拆成四个互相正交的关注面：**声明**（`defineTool` + schema DSL，类型安全）、**准入**（scope 分层注册、restriction、条件 `when`、单调 guard）、**执行**（executor 选择 + 三段 waterfall 管线 + 协作取消）、**呈现**（`presentation.ts` 渲染意图 + `presentationMeta` 持久化元数据）。每个面都可以独立扩展，而所有这些机制共同维持两条铁律：模型可见的一切都来自注册表的白名单投影（`output`/`execute`/回调永不上 wire），任何失败（未知工具、非法参数、策略拒绝、取消）都变成结构化的 `isError` 结果而不是中断 turn。
