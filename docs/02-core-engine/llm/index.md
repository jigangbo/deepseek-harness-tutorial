# LLM 模块 — 消息/流词汇表与适配器接缝

> `packages/llm/llm`(`@deepseek-ai/dsh-llm`)是整个 harness 调模型的**唯一入口**。它定义了一套 provider 中立的消息/内容块/流式词汇表,并提供 `ctx.llm` 服务:适配器注册表 + 可被 waterfall 拦截的流式调用 API。本模块自己**没有任何 provider wire 代码**——所有 HTTP/SSE 细节都在适配器包里。

## 文件地图

| 文件 | 一句话 |
|------|--------|
| `src/index.ts` | `LlmRuntime` 服务本体:注册表、目录、发现、prepareCall、流式边界(并入本页) |
| `src/types.ts` | `StreamChunk` 协议、内容块 map、finish 原因、`GenerateOptions` 等共享词汇 |
| `src/message.ts` | 不可变消息构造器与角色/来源类型 |
| `src/content.ts` | 内容块投影:文件/图片占位文本、图片预算、工具更新投影 |
| `src/assembler.ts` | `BlockAssembler`:增量把 chunk 折叠回内容块与 assistant 消息 |
| `src/assistant-stream.ts` | 紧凑带时间戳的流记录 + 免展开读取器 |
| `src/call-config.ts` | 调用配置类型、相等比较、loop 请求标记 |
| `src/api-key.ts` | API key 格式校验(所有适配器共享) |
| `src/adapter-failure.ts` | 适配器抛出值的失败归一化 |
| `src/error.ts` | `HarnessError`/`LlmError` 基类与 provider 中立错误码 |
| `src/retry-policy.ts` | provider 拥有的重试策略解析 |
| `src/attribution.ts` | 每个 provider 请求都带的 `User-Agent` 归属 |
| `src/brand.ts` | 本包拥有的 branded id |

## `ctx.llm` 是什么

`LlmRuntime` 是一个抽象的 Cordis 服务(挂在 `ctx.llm` 上),它把三件事收拢到一处:

1. **适配器注册表**——`registerAdapter(providers, adapter)` 把一个 `LlmAdapter` 实例绑定到若干 provider 路由名上;`GenerateOptions.provider` 用路由名选适配器。重复注册同一路由抛 `DUPLICATE_ADAPTER`,且是全有全无(all-or-nothing)。
2. **可配置 provider 目录**——`registerConfigurableProviders()` 声明"经配置可以激活"的路由(即使尚未注册),配置界面据此展示 dormant provider。
3. **流式调用**——`stream(options)` 走 `llm/stream` waterfall 到达适配器,产出原始 `StreamChunk` 流。

```yaml
# 最小组合(cordis.yml):服务本身无配置,必须搭配至少一个适配器
- name: '@deepseek-ai/dsh-llm'
- name: '@deepseek-ai/dsh-llm-deepseek-api-key'
  config:
    apiKeyEnv: DEEPSEEK_API_KEY
```

任何要调模型的组合——agent loop、会话标题生成器、compaction 摘要器——都经这里流式发请求。

## 适配器 seam:谁拥有什么

设计的核心分离是:**逻辑契约 provider 中立,适配器拥有 wire**。本包定义一次消息、内容块、流 chunk 的规范词汇;每个适配器只做"自家 wire 格式 ↔ 规范词汇"的翻译。

```text
           ┌──────────────────── 注册时捕获 ───────────────────┐
LlmAdapter │ providerInfo()  providerRetryPolicy()             │  每路由一份
(抽象基类) │ resolveModel()  listModels()                      │
           └───────────────────────────────────────────────────┘
           stream(options) ← 唯一必须实现的方法
```

适配器契约要点(消费者可依赖、适配器必须遵守):

- **一次适配器调用 = 一次 provider 尝试**。适配器关闭库内重试;重试由 `dsh-llm-retry` 在 agent 失败步扩展点执行。
- **`usage` 在 `finish` 之前,`finish` 之后什么都不发**。
- **工具参数全程保持原始 JSON 字符串**(增量走 `argumentsDelta`,不解析)。
- **错误有且只有一条归一化路径**:适配器可以 throw(传输/协议错误),也可以流内以 `finish {kind:'error'|'aborted', failure}` 收尾;`LlmRuntime.stream()` 把前者转成后者,消费者看到的永远是一个终态 `finish` chunk。
- **每个 provider HTTP 请求都带 `attributionHeaders()`**(`User-Agent` 基线)。

## 消息类型全集

会话是 `Message[]`;一条消息是带角色的不可变值,内容为类型化**内容块**数组。角色 map 是**封闭的**(每个角色必须有持久 Session 事件与适配器投影):

| role | 类型 | 来源(source.kind) |
|------|------|--------------------|
| `system` | `SystemMessage` | `system-prompt` |
| `developer` | `DeveloperMessage`(工具增删等增量记录) | 任意 producer |
| `user` | `UserMessage` | 任意 producer(`MessageSourceMap` 是 merge-extensible 的) |
| `assistant` | `AssistantMessage` | `model`(带 provider/model/replayState) |
| `tool` | `ToolResultMessage` | `tool`(带 callId) |

内容块(`ContentBlockMap`,merge-extensible):`text` / `reasoning` / `image` / `file` / `tool-call` / `tool-addition` / `tool-removal`。注意:**工具结果是一等 `ToolResultMessage`,不是内容块**;`file` 块永远不进 provider——请求装配会把每个文件出现投影为确定性 handle 文本。

## 流式事件模型

`StreamChunk` 是**封闭**可判别联合——`switch` 以 `assertNever` 收尾,新增变体会在所有消费者的编译期报错:

```text
block-start(index, blockType)
  text-delta(index, text)        ┐
  reasoning-delta(index, text)   │ index 关联交错的块
  tool-call-delta(index,id,Δ)    ┘
  block-end(index, block)        ← 携带装配完成的块,消费者不必自己拼
  usage(usage)                   ← 必须在 finish 前
  finish(reason, replayState?)   ← 终态,恰好一个:stop / tool-calls / max-tokens / aborted / error
```

## 与 agent-loop 的分工

**loop 组请求,llm 发请求**。agent-loop 从 session log 派生请求(深冻结、带 `markAgentLoopRequest` 进程内标记),经 `prepareCall()` 把配置与适配器代际绑定,再一次性派发;本包负责路由解析、能力校验、默认值材料化、投影与终态归一化,不碰请求内容。

```text
 agent-loop                    LlmRuntime (ctx.llm)               适配器 (llm-deepseek …)
──────────────                ──────────────────────             ────────────────────────
 组装 GenerateOptions
 (深冻结 + loop 标记)
   │
   ├─ prepareCall(config) ───▶ registration(provider)  ──查无──▶ 抛 NO_ADAPTER
   │                            │
   │                            ├─ adapter.prepareCall() ────▶ resolveModel()(精确模型元数据)
   │                            ├─ normalizeModelInfo(校验+脱离)
   │                            ├─ resolveCallWithInfo(默认值材料化/effort 校验)
   │                            ▼
   │                       PreparedLlmCall
   │                       (config 冻结 + retryPolicy + 一次性 stream())
   │
   ├─ prepared.stream(options) ─▶ ctx.waterfall('llm/stream') ─▶ adapter.stream(options)
   │                                                        │        │ SSE / HTTP wire
   │                          (适配器 throw → 归一化)        │        ▼
   │ ◀═════════ for await (chunk of stream) ════════════════┘   StreamChunk 流
   │                                终态 finish chunk(error/aborted 或正常)
   ▼
 BlockAssembler.push(chunk)…      AssistantStreamAccumulator.push({time, chunk})
   ▼                                   ▼
 blocks()/message()/usage/finish   AssistantStreamRecord[](紧凑记录)
   │                                   │
   └──▶ assistant/message 事件 ◀───────┘  (attempt 流嵌入 durable settlement)
```

关键不变量(节选自包 README):

- **Model-visible ⟺ logged**:每个 provider 请求的模型可见输入必须能从 session log 重建;loop 请求深冻结,监听器与适配器只读不写。
- **Prepared calls 一次性**:prepared call 只能派发一次,且请求的 call-config 字段必须与 prepared config 一致(`INVALID_PREPARED_CALL`)。
- **Replay state 只在同适配器内旅行**:`forAdapter()` 在派发前剥掉历史路由不属于当前适配器实例的 `replayState`。
- **文件投影无条件,图片投影跟随捕获的路由**:文件块→handle 文本(所有路由);图片块→仅 image-capable 路由保留,文本模型收到确定性占位。

## `index.ts` 走读(服务本体)

### LlmError 与 assertUsableApiKey

```typescript
// packages/llm/llm/src/index.ts
export class LlmError extends HarnessError {
  /** Serializable facts retained beside this live Error. */
  readonly failure: LlmFailure

  constructor(message: string, code: string, options?: LlmErrorOptions) {
    if (typeof message !== 'string' || message.length === 0) throw new Error('LlmError message must be a non-empty string')
    if (typeof code !== 'string' || code.length === 0) throw new Error('LlmError code must be a non-empty string')
    // …status/providerRetryAfterMs/requestId 逐一验证…
    super(message, code, options)
    this.name = 'LlmError'
    this.failure = Object.freeze({
      message,
      code,
      ...options?.status === undefined ? {} : { status: options.status },
      /* …可选字段条件展开… */
    })
  }
}
```

构造器在**入口处**校验所有字段(loud 失败),并把可序列化的 `failure` 快照冻结在错误对象上——这就是"抛出的 Error 与流内 `LlmFailure` 携带同样事实"这条契约的实现载体。`assertUsableApiKey(raw, pkg, ref)` 把 `normalizeApiKey` 的判定转成带 `INVALID_CREDENTIAL` 码的 `LlmError`,诊断文本指明**去哪里修**(引用名,Models 页),绝不回显密钥值。

### 原子注册:prepareRoutes + commitRoutes

```typescript
// packages/llm/llm/src/index.ts
private prepareRoutes(providers: string[], adapter: LlmAdapter, owned: ReadonlySet<string>): AdapterRegistration[] {
  const unique = new Set<string>()
  const registrations: AdapterRegistration[] = []
  for (const provider of providers) {
    if (provider.length === 0) throw new LlmError('adapter provider names must be non-empty', 'INVALID_ADAPTER')
    if (unique.has(provider) || (this.adapters.has(provider) && !owned.has(provider))) {
      throw new LlmError(`an adapter for provider "${provider}" is already registered`, 'DUPLICATE_ADAPTER')
    }
    const info = adapter.providerInfo(provider)
    if (/* id 必须等于 provider、name 非空 */) throw new LlmError(/* … */, 'INVALID_ADAPTER')
    unique.add(provider)
    const retryPolicy = adapter.providerRetryPolicy(provider)
      ?? resolveRetryPolicy(undefined, `llm: provider "${provider}" retryPolicy`)
    registrations.push({ adapter, provider: { id: info.id, name: info.name }, retryPolicy })
  }
  return registrations
}
```

先**完整校验候选集**(不写任何状态),再由 `commitRoutes()` 在一个同步段内换入。`registerAdapter` 返回的 handle 除 disposer 外还有 `replace(providers)`:原子替换同一适配器实例的路由集(配置驱动的路由变化用),disposed 后调用抛 `REGISTRATION_DISPOSED`。每次 commit 都发 `llm/adapters-updated`(无载荷,消费者重读注册表),且监听器失败被逐个 contain,不会否决注册本身(`emitAdaptersUpdated` 特意绕过 Cordis emit 的 map 语义,防止单个同步 throw 饿死后续监听器)。

### adapterStream:终态归一化的边界

```typescript
// packages/llm/llm/src/index.ts
private async * adapterStream(options: GenerateOptions, prepared?: PreparedDispatch): AsyncGenerator<StreamChunk> {
  let iterator: AsyncIterator<StreamChunk>
  try {
    const registration = prepared?.registration ?? this.registration(options.provider)
    // …prepareCall / normalize / 投影(files→text、text-model 图片占位、工具更新)…
    const stream = dispatch(this.forAdapter(projectedOptions, adapter))
    iterator = stream[Symbol.asyncIterator]()
  } catch (error: unknown) {
    yield adapterFailureChunk(error, options.signal)   // 选择/派发失败 → 终态 chunk
    return
  }
  let completed = false
  try {
    while (true) {
      // iterator.next() 单独 try:迭代失败也归一化为终态 chunk
      // yield 在适配器拥有的 try 之外:消费者失败保持抛出
    }
  } finally {
    if (!completed) await iterator.return?.()          // 提前退出时关闭适配器流
  }
}

function adapterFailureChunk(error: unknown, signal?: AbortSignal): StreamChunk {
  const failure = normalizeLlmFailure(error)
  return {
    type: 'finish',
    reason: signal?.aborted || failure.code === 'ABORTED'
      ? { kind: 'aborted', failure }
      : { kind: 'error', failure },
  }
}
```

注意 try/yield 的分层:**适配器选择、派发、迭代失败**变成终态 `finish` chunk;**middleware、嵌套调用、清理、消费者失败**保持抛出。这是"最终适配器边界"的精确含义——出这块边界的失败才被流协议吸收。

### prepareCall:注册-代际绑定

`prepareCall(config, signal?, configure?)` 的流水线:

1. 查路由 → `adapter.prepareCall()`(动态适配器借此把模型元数据与派发绑定到同一代际,HMR 不能把 A 代的 capability 和 B 代的 endpoint 拼在一起);
2. `normalizeModelInfo` 校验/脱离;
3. 可选的纯函数 `configure(controls, model)` 在默认值材料化**之前**选控制项(后写覆盖先写);
4. `resolveCallWithInfo` 材料化 `defaultMaxTokens` / `defaultEffort`,显式 effort 不受支持就抛 `UNSUPPORTED_REASONING_EFFORT`(不 clamp);
5. 计算并冻结 `adapterDefaults`(哪些字段是适配器给的而非调用者提的——loop 据此记录 header);
6. 返回冻结的 `PreparedLlmCall`,`stream()` 内部检查一次性 + config 相等(`callConfigEquals`)。

直接 `ctx.llm.stream(options)`(不经 prepareCall)也能用:`adapterStream` 内部会对未 prepared 的请求现场做同样的 resolve/校验/投影,只是不提供跨 header-logging 与派发的注册绑定。

## 推荐阅读顺序

1. **本页** → 建立全景;
2. `types.md` — `StreamChunk`/`ContentBlock`/`GenerateOptions` 词汇表(一切的地基);
3. `message.md` — 消息构造与 source 词汇;
4. `assembler.md` — chunk 如何折回内容块与消息(**重点**);
5. `assistant-stream.md` — 紧凑流记录与免展开读取器(**重点**);
6. `content.md` — 请求投影(文件/图片/工具更新);
7. `call-config.md` / `retry-policy.md` / `adapter-failure.md` / `error.md` — 配置、重试、失败词汇;
8. `api-key.md` / `attribution.md` / `brand.md` — 小而准的共享设施;
9. 然后读同级 `providers.md`(适配器实现族)与 `../agent-loop/`(请求如何被组装);
10. 想动手写适配器:直接跳 **精通篇 `06-mastery` 的 `03-recipe-new-llm-adapter.md`**(新增一个适配器的完整路径:实现 `LlmAdapter`、遵守流契约、注册路由、wire 级测试证明归属头)。

## 小结

- `packages/llm/llm` = 词汇表(类型文件)+ 服务(`index.ts`)。词汇表是唯一规范,适配器只做翻译。
- `ctx.llm.stream()` 的输出保证**恰好一个终态 `finish` chunk**;失败归一化发生在最终适配器边界,消费者按 `code` 路由,永不解析 message 文本。
- 注册表变更是原子的、effect-scoped 的、带无载荷通知事件的;prepared call 把"解析出的配置"与"实际派发的适配器代际"锁在一起。
