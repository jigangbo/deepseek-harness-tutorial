# 第 12 章 System Prompt：提示词组装

`packages/core/system-prompt`（npm 包名 `@deepseek-ai/dsh-system-prompt`）是 harness 的"模型可见输入装配台"。每个模型 step 之前，agent loop 调用一次 `ctx.systemPrompt.assemble(...)`，把散落在各个插件里的提示词素材合并成一份有序输出：

- **prompt sections**（`PromptSection`）：静态或按上下文求值的文本段，升序拼接成 system 消息；
- **prompt variables**（`variable()`）：段文本里 `{{name}}` 引用的运行时事实，如 `provider` / `model`；
- **dynamic contexts**（`PromptContext`）：与段分开的动态运行时事实（工作目录、沙箱策略、审批策略…），由 loop 落成 user 角色的快照消息；
- **tool schemas**（`tools()` provider）：本 step 模型可见的 `ToolSchema[]`。

设计上的一个关键决定：工具 schema 也属于组装——"模型被告知它能做什么"和"系统提示词说了什么"是一体的，即使线上传输时 schema 走独立字段。装配失败（多个 complete 段、未解析变量、未注册的 toolOrder 名字）会**响亮地抛错**，而不是发出一份畸形的提示词。

## 文件概览

| 成员 | 职责 |
|---|---|
| `SystemPrompt` 服务 | 注册表 + 组装管线，挂载为 `ctx.systemPrompt` |
| `PromptSection` / `PromptContext` / `AssembleContext` | 三类注册输入与组装上下文 |
| `SECTION_ORDERS` / `CONTEXT_ORDERS` | 仓库自有段/上下文的集中命名位置表 |
| `assemble()` / `refreshContext()` | 两条组装入口（完整组装 / 准入前上下文刷新） |
| `renderPrompt()` / `renderContextSections()` / `joinContextSections()` | 渲染函数（agent loop 消费） |
| `interpolate()` | 严格的 `{{variable}}` 插值 |
| `orderTools()` / `validateToolOrder()` | 配置驱动的工具顺序 |
| `system-prompt/assemble` waterfall / `system-prompt/change` emit | 两个事件 |

## 组装全景图

```text
assemble({ scope, signal, agent?, modelSelection?, toolSet? })
   │
   ├─ variables   全局 provider 先求值 → scope 链 provider 后求值（近层同名覆盖远层）
   ├─ sections    layers.merge(scope) 同名遮蔽 → comparePromptSections(order, name) 排序
   │              → text provider 用当前 AssembleContext 求值 → 顺带记录 complete 段
   ├─ contexts    merge + 按 order 排序 → 运行时上下文被 suppress 时只留 required
   └─ tools       全局 + scope 链的 schema provider 逐个求值
                  parameters structuredClone 脱关联 → orderTools(toolOrder, knownNames)
   ▼
system-prompt/assemble waterfall（scopeTarget(this, scope)：scope 内 listener 才收到）
   │  listener 可改写 sections / contexts / variables；tool 声明保持 provider 所有
   ▼
恢复 effective complete 段（若有）→ PromptAssembly { sections, contexts, variables, tools }
```

agent loop 拿到 `PromptAssembly` 后分两路消费：

```text
renderPrompt(assembly) ──► system 消息文本（append/replace 为 system/message 表面节点）
renderContextSections + joinContextSections ──► runtime-context 快照（append 为 user/message）
```

## ctx.systemPrompt 服务

服务声明在 `packages/core/system-prompt/src/index.ts`，通过 `declare module '@deepseek-ai/cordis'` 挂上 `Context.systemPrompt`。注册 API 一览：

| 方法 | 作用 | 同名/重复策略 |
|---|---|---|
| `section(section)` | 注册一个有序提示词段 | 同层重名抛错；scoped 段遮蔽全局同名段 |
| `context(context)` | 注册有序动态上下文 | 同上 |
| `variable(name, provider)` | 注册插值变量（名字必须匹配 `/^[a-z][a-z0-9_]*$/`） | 同上 |
| `tools(provider)` | 注册工具 schema provider | 匿名追加，可多条；返回保留名 `<unlisted-tools>` 会让组装失败 |
| `suppressRuntimeContext()` | 对调用方 scope 压制可选动态上下文 | 匿名追加，可叠加；全部解除后恢复 |
| `getSectionOrder(name)` / `getContextOrder(name)` | 解析集中命名的位置数字 | 未知名字类型报错 |
| `assemble(context?)` | 完整组装（async） | — |
| `refreshContext(assembly, context?)` | 准入前只刷新注册的上下文 | — |

所有注册都返回**精确的 Cordis effect disposer**——插件卸载时自动反注册，并触发 `system-prompt/change`（这个通知故意不做 scope 过滤：一次全局变更影响所有 scope）。

配置字段（`SystemPrompt.Config`）负责"部署作者写的那部分"：

| 字段 | 默认 | 说明 |
|---|---|---|
| `includeHarnessIdentity` | `true` | 在 order −1000 注入固定开场白 `harness:identity` |
| `includeRuntimeContext` | `true` | 为 false 时在构造函数里直接调 `suppressRuntimeContext()` |
| `personaPrefix` / `personaSuffix` | `''` | 全局 persona 模板，落在 `deployment:persona-prefix`（order 0）与 `deployment:persona-suffix`（order 10200）两个具名槽位 |
| `toolOrder` | — | 模型可见工具顺序；缺少唯一 `<unlisted-tools>` 哨兵或含重复名在**加载时**抛错，列出未注册工具名则在**组装时**抛错 |

构造函数里这三段是服务自己注册的（`packages/core/system-prompt/src/index.ts`）：

```ts
// Keep harness-owned openers independent of the selected loop plugin.
if (config.includeHarnessIdentity ?? true) {
  this.section({
    name: 'harness:identity',
    order: this.getSectionOrder('HARNESS_IDENTITY'),
    text: 'You are an AI agent powered by DeepSeek Harness.',
  })
}
this.section({ name: PERSONA_PREFIX_SECTION, order: this.getSectionOrder('DEPLOYMENT_PERSONA_PREFIX'), text: config.personaPrefix ?? '' })
this.section({ name: PERSONA_SUFFIX_SECTION, order: this.getSectionOrder('DEPLOYMENT_PERSONA_SUFFIX'), text: config.personaSuffix ?? '' })
```

`PERSONA_PREFIX_SECTION`（`'deployment:persona-prefix'`）导出为常量的原因写在 JSDoc 里：agent preset 要用**同一个名字**注册 scoped 段来替换部署 persona——两边命名相同才能"替换"而不是"重复"。

## 段注册与排序：getSectionOrder

段的完整输入契约是 `PromptSection`（JSDoc 摘要）：

```ts
export interface PromptSection {
  readonly name: string        // Unique name — a duplicate registration throws.
  readonly order: number       // Ascending; equal orders use code-unit name order.
  readonly text: string | ((context: AssembleContext) => string)  // static or per-assembly provider
  readonly interpolate?: boolean // defaults true; false preserves literal text
  readonly complete?: boolean  // become the exact complete prompt after assembly
}
```

排序函数只有一行：`comparePromptSections` 先比 `order`，同序时用 `compareNames`（码元比较，locale 无关，任何机器上顺序一致）。

仓库自有贡献者不允许随手写魔法数字，而是通过 `getSectionOrder()` 解析集中分配的位置表：

```ts
const SECTION_ORDERS = {
  HARNESS_IDENTITY: -1000,
  DEPLOYMENT_PERSONA_PREFIX: 0,
  PLAN_POLICY: 500,
  TEAM_POLICY: 600,
  PTC_ONLY: 800,
  FILE_REFERENCE: 900,
  TOOL_BASH: 1000,
  ...
  TOOLS_SDK: 5000,
  DELIVERABLE_FILE_REFERENCES: 9000,
  STRUCTURED_OUTPUT: 9900,
  // Local paths and endpoints follow reusable instructions.
  HARNESS_SOURCE: 10000,
  WEB_SURFACE: 10100,
  DEPLOYMENT_PERSONA_SUFFIX: 10200,
} as const
```

这张表就是整个 first-party 提示词的目录：身份（−1000）→ 部署 persona 前缀（0）→ 策略段（plan 500 / team 600 / PTC 800）→ 各工具的使用说明（1000–3000）→ 可复用指令（TOOLS_SDK 5000）→ 结构化输出（9900）→ 环境性后缀（本仓库来源 10000、Web 面 10100、persona 后缀 10200）。动态上下文有独立的一张 `CONTEXT_ORDERS`（工作目录 100、沙箱策略 110、审批策略 115、子代理委派 120…），两条位置轴互不干扰。

外部贡献者可以用任意有限 order——`section()` 只拒绝非有限值，然后经 `this.layers.effect(this.ctx, ...)` 注册（`this.layers` 是 `ScopedLayers<PromptLayer>`）。**调用上下文的 scope 标签同时决定可见性与销毁所有权**——这正是 scope 模块的语义（见 [scope 章节](../scope/index.md)）；全局层与 scoped 层的重名报错文案不同，scoped 重名会提示"per-agent override 请走该 agent 的 `agent.ctx`"。

## 变量与严格插值

变量由 `variable(name, provider)` 注册，在段或上下文文本里写成 `{{name}}`。agent loop 注册了两个（`packages/core/agent-loop/src/index.ts`）：

```ts
ctx.systemPrompt.variable('provider', context => context.modelSelection?.provider ?? context.agent?.options.provider)
ctx.systemPrompt.variable('model', context => context.modelSelection?.model ?? context.agent?.options.model)
```

所以部署可以在 `personaPrefix` 模板里写 `You are {{model}}, ...` 引入模型名简介。注意 loop **不注册 `{{cwd}}`**——工作目录由 working-directory 服务以字面上下文贡献，自定义 persona 模板里引用 `{{cwd}}` 会让组装在第一次模型请求前就失败。

插值发生在渲染阶段（`renderPrompt`），不是组装阶段，且极其严格——`renderPrompt` 把每段按 `interpolate` 选项走 `interpolate()`，去空段后以空行拼接。`interpolate()` 的失败路径有四类，全部抛错：畸形引用（`{{` 后有 `}}` 但不构成完整 group）、名字不匹配正则、**未注册**的名字（用 `Object.hasOwn` 检查，避免原型链污染）、已注册但本次求值为 `undefined`。两个刻意的宽容点：孤立的 `{{` 后面再也没有 `}}` 时视为字面散文；替换后的值不会再被扫描。`interpolate: false` 让整段保持字面量——生成型工具文档或含 `{{…}}` 的目录名需要它。

## 工具 schema：step 准入（toolSet）与 provider 的交互

工具 schema 不是插件直接塞进提示词的，而是通过一条三层协作链：

1. **dsh-tools 给 `AssembleContext` 增加准入字段**。`AssembleContext` 是 merge-extensible 接口，`packages/core/tools/src/index.ts` 通过声明合并加入 step 准入集合：

```ts
declare module '@deepseek-ai/dsh-system-prompt' {
  interface AssembleContext {
    /** Admitted registrations for the prepared step; absent outside a model step. */
    toolSet?: ResolvedToolSet
  }
}
```

2. **agent loop 在 step 准备时求出准入集合并传入组装**（`packages/core/agent-loop/src/agent.ts` 的 `preStep`）：

```ts
const toolSet = await this.loopCtx.tools.resolve({ agent: this, modelSelection: selection, signal })
const assembly = await this.loopCtx.systemPrompt.assemble({ ...assembleContextFor(this, signal), modelSelection: selection, toolSet })
```

`assembleContextFor`（`packages/core/agent/src/dispatch.ts`）把 agent 与 scope 一起填上——`{ agent, scope: agent, ...signal === undefined ? {} : { signal } }`——避免 agent 级贡献被静默漏掉。

3. **ToolRuntime 注册的 schema provider 在组装时被回调**（`packages/core/tools/src/index.ts` 构造函数）：

```ts
ctx.systemPrompt.tools(({ scope, toolSet }) => {
  const executor = this.selectExecutor(scope)
  const description = executor.describe(this.definitions(scope, toolSet))
  return { ...description, knownNames: description.knownNames ?? [...this.view(scope).knownNames] }
})
```

两个参数分工明确：`scope` 决定**可见集合**（scope 链上的注册 + restriction 过滤），`toolSet` 决定**本 step 的准入集合**（`when` 条件激活的结果）。`definitions(scope, toolSet)` 在有 toolSet 时读 `toolSet.view(scope)`，没有时读注册表视图。native 执行器直接把每个工具投影成 schema，PTC 执行器折叠成一个 `run_code`——schema 形态是执行器的事，registry 只供料。

provider 返回的 `ToolProviderResult` 除 `schemas` 外还有一个可选 `knownNames`：**restriction 之前的名字全集**。`orderTools()` 用它区分"配置里写错名字"（抛错）与"已知工具在此 scope 被有意隐藏"（允许缺席）：

```ts
function orderTools(tools: ToolSchema[], toolOrder: string[] | undefined, knownNames: ReadonlySet<string>): ToolSchema[] {
  const reserved = tools.find(tool => tool.name === TOOL_ORDER_REST)
  if (reserved !== undefined) throw new Error(`tool provider returned reserved tool name "${TOOL_ORDER_REST}" ...`)
  if (toolOrder === undefined) return tools.sort(compareToolNames)
  const unknown = toolOrder.filter(name => name !== TOOL_ORDER_REST && !knownNames.has(name))
  if (unknown.length > 0) throw new Error(`toolOrder lists unregistered tool...`)
  const listed = new Set(toolOrder)
  const rest = tools.filter(tool => !listed.has(tool.name)).sort(compareToolNames)
  return toolOrder.flatMap(name =>
    name === TOOL_ORDER_REST ? rest : tools.filter(tool => tool.name === name))
}
```

每个 provider 的 `parameters` 都会 `structuredClone`——收集到的 schema 与注册表里的定义脱关联，后续 waterfall 和调用方都改不到原对象。没有配置 `toolOrder` 时按名字典序排序（注册顺序是插件加载的偶然产物，不能当模型可见顺序）。

## assemble() 走读：遮蔽、complete 段与 waterfall

组装主体（省略已讲过的工具收集）：

```ts
async assemble(context: AssembleContext = {}): Promise<PromptAssembly> {
  const scope = context.scope
  const scopeLayers = this.layers.chainLayers(scope)
  // Scoped variables shadow globals; scope-chain providers run farthest first,
  // so the nearest scope wins a name.
  const variables: Record<string, string | undefined> = {}
  for (const [name, provider] of this.layers.global.variables.entries()) variables[name] = provider(context)
  for (const layer of scopeLayers) {
    for (const [name, provider] of layer.variables.entries()) variables[name] = provider(context)
  }
  // Scoped sections shadow globals before the deterministic order sort.
  const sectionByName = this.layers.merge(scope, layer => layer.sections)
  const contextByName = this.layers.merge(scope, layer => layer.contexts)
  ...
```

遮蔽的实现是"先远后近地覆盖同一个 Map"，而不是过滤——所以近层定义可以替换远层同名段而不留痕迹。persona preset（`packages/preset/persona/src/index.ts`）就是这样替单个 agent 换身份的：

```ts
ctx.effect(() => ctx.systemPrompt.section({
  name: PERSONA_PREFIX_SECTION,
  order: ctx.systemPrompt.getSectionOrder('DEPLOYMENT_PERSONA_PREFIX'),
  text: config.prefix,
  ...(config.complete ? { complete: true } : {}),
}), 'persona.section()')
```

complete 段的处理分两步：组装前检查**最多一个** effective complete 段（多个直接抛错）；waterfall 之后恢复它为唯一的段列表。这样 cooperative waterfall 仍然跑完——工具、上下文、变量照常解析——但 listener 无法往一个声明了 complete 的 scope 提示词里添内容：

```ts
const transformed = await this.ctx.waterfall(
  scopeTarget(this, scope), 'system-prompt/assemble', assembly, context,
  () => Promise.resolve(assembly),
)
if (completeSection === undefined && !runtimeContextSuppressed) return { ...transformed, tools }
return {
  ...transformed,
  tools,
  sections: completeSection === undefined ? transformed.sections : [completeSection],
  contexts: runtimeContextSuppressed ? transformed.contexts.filter(entry => requiredContexts.has(entry.name)) : transformed.contexts,
}
```

waterfall 的 `this` 是 `Scoped<SystemPrompt>` 载体（`scopeTarget(this, scope)`）：只有 tag 等于该 scope（或其祖先）的 listener 参与，未打 tag 的全局 listener 也参与。事件声明强制这个载体作为 `this` 类型，直接用裸 service 派发是编译错误。

### refreshContext：准入前的轻量刷新

`agent/request` 重试或路由准备之后，事实可能已经变化（比如工作目录被切换）。`refreshContext(assembly, context)` 只重跑**注册的**上下文 provider：waterfall 追加的上下文保留原值，当前 suppress 状态移除可选上下文，缺失的 required 注册按注册顺序补回位置。段、工具、变量保留已接受的组装值——provider 与 waterfall 不重跑。loop 在 `step()` 里每次尝试请求前调用它（见 `packages/core/agent-loop/src/agent.ts`）。

## system/message：提示词历史如何落账

渲染出的提示词不作为请求字段传输，而是作为**派生历史里的 system 消息节点**。`packages/core/agent-loop/src/agent.ts` 的 `step()`：

```ts
const renderedPrompt = renderPrompt(assembly)
...
const commits = this.systemPrompt.project(renderedPrompt, {
  inHistory: preparedCall?.systemPromptUpdate === 'in-history',
  startsSeries: ...,
})
for (const { message, intent } of commits) {
  this.session.append('system/message', { turn, step, message }, intent)
}
```

`SystemPromptProjection`（`packages/core/agent-loop/src/runtime-context.ts`）把"本次渲染文本"对账到既有的 `system/message` 表面节点：没有节点则 append（第一个 step 的表面节点 0）；文本未变则零提交，前缀缓存不受影响；文本变了默认原地替换（空文本时逐节点清空，历史里不留旧提示词），而声明 `systemPromptUpdate: 'in-history'` 的调用在延续请求系列里改为在缓存历史**之后追加**新提示词，保住既有前缀的复用。

动态上下文是另一路：`renderContextSections` + `joinContextSections` 拼出的快照（带头部 `Current runtime context. This snapshot supersedes earlier runtime-context snapshots.`）作为 user 消息 append，只在内容变化或被压缩移除后重新落账。段是 system 消息、上下文是 user 快照——这就是 `PromptSection` 与 `PromptContext` 分成两个注册 API 的原因。

## 扩展点：新增一个 section 的做法

**仓库自有贡献者**（占了集中位置表的名字）：在插件里用 `getSectionOrder()` 解析位置，不要手写数字。跨调用生效的使用说明段是典型——`packages/shell/tool-bash/src/index.ts`：

```ts
// Cross-call guidance belongs in the prompt rather than one-call schema prose.
ctx.systemPrompt.section({
  name: 'tool:bash',
  order: ctx.systemPrompt.getSectionOrder('TOOL_BASH'),
  text: 'Check the [exit code: N] marker on every bash result; investigate failures before moving on.',
})
```

需要新位置时先在 `SECTION_ORDERS` 里分配（保持稀疏、留出插入空间），再让贡献插件引用它。

**工具类段要感知 step 准入**：restriction 不移除段注册，所以工具说明段应该在工具不可见时输出空文本。执行器自有段由 `setExecutor()` 自动包装成 toolSet 感知回调（`packages/core/tools/src/index.ts`）：

```ts
text: ({ scope, toolSet }) => {
  if (this.selectExecutor(scope) !== executor) return ''
  return typeof section.text === 'function' ? section.text(this.definitions(scope, toolSet)) : section.text
},
```

外部贡献者可用任意有限 order；文本可以是 `string`，也可以是接收 `AssembleContext` 的函数（可读 `context.toolSet` 检查与执行同一步的准入集合）。

**动态事实放 context 而不是 section**：事实会变化的（工作目录、审批策略）注册成 `PromptContext`，由 working-directory 服务示范——`required: true` 保证被压制时仍然可见，`interpolate: false` 保证目录名里的 `{{…}}` 不会炸插值（`packages/session/working-directory/src/index.ts`）：

```ts
ctx.systemPrompt.context({
  name: 'working-directory:current',
  order: ctx.systemPrompt.getContextOrder('WORKING_DIRECTORY'),
  required: true,
  interpolate: false,
  text: context => context.agent === undefined ? '' : this.contextText(context.agent.session),
})
```

它还监听 `system-prompt/assemble`，在 waterfall 里把最新目录文本写回 assembly——这是上下文贡献的另一种合法形态（waterfall 追加）。

**per-agent 覆盖**：走 `agent.ctx.systemPrompt...` 注册，同名遮蔽全局，随 agent 销毁自动反注册。

## 核心函数表

| 函数/方法 | 位置 | 一句话 |
|---|---|---|
| `SystemPrompt.section/context/variable/tools/suppressRuntimeContext` | `packages/core/system-prompt/src/index.ts` | 五类注册，scope 感知，返回精确 disposer |
| `getSectionOrder` / `getContextOrder` | 同上 | 集中命名位置 → 数字 |
| `assemble(context)` | 同上 | 遮蔽合并 → 排序 → 工具收集排序 → scope 过滤 waterfall → complete 恢复 |
| `refreshContext(assembly, context)` | 同上 | 准入前只刷新注册上下文（required 恢复、suppress 生效） |
| `renderPrompt(assembly)` | 同上 | 插值 + 去空段 + 空行拼接 |
| `renderContextSections` / `joinContextSections` / `renderContextSnapshot` | 同上 | 上下文快照的具名渲染与拼接 |
| `interpolate` | 同上 | 严格 `{{variable}}` 插值（畸形/未知/未定义抛错） |
| `orderTools` / `validateToolOrder` | 同上 | 配置顺序应用；形状错误加载时抛，未知名字组装时抛 |
| `assembleContextFor(agent, signal?)` | `packages/core/agent/src/dispatch.ts` | agent + scope + signal 一起填的组装上下文 |
| `SystemPromptProjection.project` | `packages/core/agent-loop/src/runtime-context.ts` | 渲染文本对账到 `system/message` 表面节点 |

## 与其他模块的关系

- **scope**（`packages/core/scope`）：`PromptLayer` 基于 `ScopedLayers`/`NamedEntries`/`AnonymousEntries`；waterfall 用 `scopeTarget` 做 scope 过滤派发。scoping 语义见 [scope 章节](../scope/index.md)。
- **tools**：声明合并给 `AssembleContext` 加 `toolSet`；ToolRuntime 构造时注册 schema provider；`setExecutor` 把执行器自有段包成 toolSet 感知回调。
- **agent / agent-loop**：`assembleContextFor` 统一组装上下文；loop 是 `assemble()`、`refreshContext()`、`renderPrompt()` 的唯一调用方，并把渲染结果落成 `system/message` / user 快照。`agent/pre-step` waterfall 在组装之后运行，listener 可拒绝该 step。
- **配置与 persona**：`Config` 持有全局 persona；`packages/preset/persona` 用同名 scoped 段替换单个 agent 的 persona。
- **事实拥有者插件**：working-directory、sandbox-policy、user-approval、plan-mode、各工具插件等各自注册自己拥有的段/上下文——提示词事实的归属见 Agent Note（`.agents/notes/implemented/architecture/2026-07-05-prompt-variables-and-tool-guidance-ownership.md`）。

## 小结

`dsh-system-prompt` 是"注册表 + 协作组装管线"：插件通过 `ctx.systemPrompt` 注册段、变量、上下文与工具 schema provider，loop 每 step 调一次 `assemble()`。三个不变式贯穿全模块——**顺序确定**（order 升序 + 码元名序，位置由集中表分配）、**遮蔽显式**（scope 同名替换全局，重复注册响亮失败）、**失败响亮**（畸形变量、多个 complete 段、未知 toolOrder 名都不会产出畸形提示词）。渲染与落账的另一半在 agent loop：段变成 `system/message` 表面节点，上下文变成 user 快照，工具 schema 随请求头传输——模型可见的一切都能从 session 日志重建。
