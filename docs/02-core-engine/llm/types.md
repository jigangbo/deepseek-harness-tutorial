# `types.ts`

> provider 中立的消息/流/请求词汇表


> 源码:`packages/llm/llm/src/types.ts`(约 600 行,纯类型 + 两个事件声明,无运行时代码)。这是整个包的"宪法":loop、session log、每个适配器、UI、compaction 共用的全部词汇都在这里定义一次。

## 文件概览

文件按主题分四段:

1. **Cordis 事件声明**(模块合并进 `@deepseek-ai/cordis` 的 `Events`):`llm/adapters-updated` 与 `llm/models-updated` 两个 emit 事件;
2. **失败与内容块**:`LlmFailure`、七个内容块接口与 `ContentBlockMap`;
3. **流协议**:`FinishReasonMap`、`TokenUsage`、`ReplayEnvelope`、`StreamChunk`;
4. **请求与目录**:`GenerateOptions`、`ToolSchema`、`RequestUserInput`、provider/model 目录与能力元数据、`LlmRequestPurposeMap`。

贯穿全文件的类型组织手法是 **merge-extensible map**(键可扩展的映射):

```typescript
// packages/llm/llm/src/types.ts
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'file': FileBlock
  'tool-call': ToolCallBlock
  'tool-addition': ToolAdditionBlock
  'tool-removal': ToolRemovalBlock
}
export type ContentBlockType = keyof ContentBlockMap
/** Any known content block; switch on `type` and fall through unknowns (merge-extensible). */
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

- **map 是开放接口**:插件可以在自己的模块里 `declare module` 增加键,联合类型随之变宽;消费端 `switch` 到未知 `type` 时**落穿到 default**而不是编译报错。
- 与之对照,`StreamChunk` 是**封闭**联合(直接写死的 `type` 联合),消费端以 `assertNever` 收尾——新增流变体会破坏所有必须处理它的消费者的编译。
- 选择标准:内容块/来源/finish 原因这类"谁都能加一种"的词汇用 map;协议 chunk 这种"加一种就要全库跟进"的词汇封闭。

## 关键代码走读

### LlmFailure — 失败事实的序列化形态

```typescript
// packages/llm/llm/src/types.ts
export interface LlmFailure {
  /** Human-readable provider or transport failure. */
  readonly message: string
  /** Stable provider-neutral machine-routing code. */
  readonly code: string
  /** HTTP status returned by the provider, when available. */
  readonly status?: number
  /** Provider-requested delay in milliseconds, when valid and available. */
  readonly providerRetryAfterMs?: number
  /** Opaque provider-issued request identifier for diagnostics. */
  readonly requestId?: ProviderRequestId
  /** With code `IMAGE_OFFLOAD_REQUIRED`: … how many more of the oldest retained image occurrences … */
  readonly offloadImages?: number
}
```

所有终态失败(throw 或流内)都归一化成这个可序列化结构。要点:

- `code` 是机器路由键(`RATE_LIMIT`、`AUTH`、`CONTEXT_WINDOW_EXCEEDED`…),消费者**只按 code 路由**,永不解析 `message` 文本;
- `providerRetryAfterMs` 是 provider **请求**的延迟,不是重试决策——是否采纳由重试策略决定;
- `offloadImages` 只在 `IMAGE_OFFLOAD_REQUIRED` 下有意义,告诉 compaction 还要 offload 多少个最旧的图片出现才能放下同一请求。

### StreamChunk — 原始流协议

```typescript
// packages/llm/llm/src/types.ts
export type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: ToolCallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | {
    type: 'finish'
    reason: FinishReason
    /** Replay metadata for a successful response; see {@link ReplayEnvelope}. */
    replayState?: ReplayEnvelope
  }
```

设计要点:

- **`index` 关联交错的块**:一个流式响应会同时吐文本、思考、多个工具调用;每个 delta 用 `index` 标明属于哪个块。
- **`block-end` 携带装配完成的 `ContentBlock`**:懒消费者可以直接取终值,不必自己拼 delta。
- **工具参数保持原始 JSON 字符串**(`arguments` / `argumentsDelta`)端到端不解析——provider 返回已解析对象时,适配器在 `block-end` 重新序列化。
- **协议顺序**:`usage` 先于 `finish`;`finish` 之后什么都不发。适配器实现可以 throw,但 `LlmRuntime.stream()` 会把它归一化成终态 `error`/`aborted` finish 再暴露给消费者。

### ReplayEnvelope — 适配器私有回放状态的共享切分

```typescript
// packages/llm/llm/src/types.ts
export interface ReplayEnvelope {
  /** Response-level adapter-private metadata (ids, native stop reason). */
  response: unknown
  /**
   * Per-block adapter-private metadata, one entry per emitted block in
   * first-seen stream order. When assembly drops a block it drops the entry at
   * the same position; entries whose length does not match the emitted block
   * count discard the whole envelope.
   */
  blocks?: readonly unknown[]
}
```

两半都对 harness 不透明,只有**切分方式**是共享词汇:`blocks[i]` 描述第 i 个发出的块。装配丢弃一个块时按位置同步丢弃对应条目,长度对不上就整个丢弃信封——这样"存下来的元数据永远描述存下来的内容",装配层无需读懂任何一半。

### TokenUsage — 不相交的计数

```typescript
// packages/llm/llm/src/types.ts
export interface TokenUsage {
  inputTokens: number
  outputTokens: number
  totalTokens?: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  reasoningTokens?: number
}
```

计数**不相交**:`inputTokens` 只算未缓存输入,缓存读写单独报告(计费输入 = 三者之和)。把缓存命中折进 prompt 总数的 provider(如 DeepSeek 的 `prompt_tokens`)由适配器减回去。`reasoningTokens` 已包含在 `outputTokens` 里,求和时不得再加。

### GenerateOptions — 完整装配的单次模型请求

```typescript
// packages/llm/llm/src/types.ts
export interface GenerateOptions<Purpose extends string = LlmRequestPurpose> {
  provider: string
  model: string
  reasoningEffort?: ReasoningEffortId
  /** Ordered conversation messages, exactly as the provider sees them. */
  messages: RequestMessage[]
  /** System prompt text for one-shot callers; adapters map it to the provider's system slot. */
  system?: string
  tools?: ToolSchema[]
  toolHistory?: ToolHistory
  temperature?: number
  maxTokens?: number
  stop?: string[]
  signal?: AbortSignal
  sessionId?: Branded<'SessionId'>
  purpose?: Purpose
}
```

- `messages` 接受持久 `Message` 或请求专用的 `RequestUserInput`(无 `id`/`source`);loop 构造的请求传派生历史,leading `system` 消息即渲染后的系统提示,所以 loop 请求**不设** `system` 字段——那是给标题生成器这类 one-shot 调用者的。
- `sessionId` 是本地声明的 brand:直接 import dsh-session 的 `SessionId` 会造成依赖环,所以这里 `Branded<'SessionId'>`。
- `purpose` 由生产者包通过模块合并 `LlmRequestPurposeMap`(空接口)声明键;普通会话请求不设。

### ToolSchema — 发给模型的工具线格式

```typescript
// packages/llm/llm/src/types.ts
export interface ToolSchema {
  /** Requests deferred loading of the tool definition into model context. */
  deferLoading?: true
  name: string
  description: string
  /** JSON Schema object for the arguments. */
  parameters: Record<string, unknown>
}
```

声明在本包(而非 dsh-tools)的理由写在 JSDoc 里:它是 `GenerateOptions` 的一部分,而 dsh-tools 的 `ToolDefinition` 和 dsh-system-prompt 都要 import 它。`deferLoading` 借用 Anthropic 的 defer_loading 术语:定义先不进上下文,等 developer 消息里的 `tool-addition` 激活。

### 能力元数据:目录(咨询性)vs 精确解析(权威)

两套元数据刻意分开:

```typescript
// packages/llm/llm/src/types.ts
export interface LlmModelInfo {
  provider: string
  id: string
  name: string
  description?: string
  /** Accepted request modalities; absent means unknown, while an explicit omission is negative capability. */
  inputModalities?: readonly ModelModality[]
  badge?: LlmModelBadge        // 纯展示数据,请求与 resolveModelInfo 都不携带
}

export interface LlmResolvedModelInfo extends LlmModelInfo {
  context?: LlmModelContext          // contextWindow
  defaultMaxTokens?: number          // 适配器配置的输出上限
  reasoning?: LlmModelReasoningInfo  // 有序 effort + 可选 defaultEffort
  systemPromptUpdate?: SystemPromptUpdate   // 唯一值 'in-history'
  toolUpdate?: ToolUpdate            // 'in-history' | 'addition-only'
}
```

- `listModels()` 的目录是**咨询性的**:路由仍以注册的 provider 为准,未列出的 model id 也能发请求;GUI 这类目录驱动入口可以要求在册。
- `resolveModelInfo()` 才是正确性敏感的解析:上下文窗口、默认输出上限、reasoning efforts、模态、system prompt/工具更新模式。`inputModalities` **缺席 = 未知,显式空 = 负能力**(image 门控靠这个区分)。
- `SystemPromptUpdate = 'in-history'`:模型把 messages 中任意位置的最新 `system` 消息读作生效系统提示,于是 loop 可以把变更后的提示**追加在缓存历史之后**而不重写第 0 条消息;缺席意味着只读最前面的 system 消息。
- `ReasoningEffortId` 是 adapter 拥有的不透明 brand id,核心不枚举值;每个适配器按 effort 递增排列并给出展示名。

### 目录与发现

`LlmConfigurableProvider`(settingsNs/settingsPath/declared/defaultBaseURL/baseURLEnv/nativeAuth/error)描述"配置可激活"的路由;`LlmModelDiscoveryRequest` + `LlmDiscoveredModel` 描述一次对端点的草稿询问——请求直接携带 endpoint 与一次性 key(还没存进配置的路由无可引用),harness 永不存储这次询问的凭证。

## 核心类与函数

本文件没有运行时导出,核心类型一览:

| 名称 | 形态 | 职责 |
|------|------|------|
| `StreamChunk` | 封闭联合 | 适配器产出的原始流协议;`assertNever` 收尾 |
| `ContentBlockMap` / `ContentBlock` | merge-extensible map | 七种内容块;未知 kind 落穿 |
| `FinishReasonMap` / `FinishReason` | merge-extensible map | stop/tool-calls/max-tokens/aborted/error |
| `TokenUsage` | interface | 不相交的 token 计数 |
| `LlmFailure` | interface | 序列化的 provider 中立失败事实 |
| `ReplayEnvelope` | interface | 适配器私有回放状态的共享切分 |
| `GenerateOptions` | interface | 完整装配的单次模型请求 |
| `ToolSchema` | interface | 模型可见的工具声明(线格式) |
| `RequestUserInput` / `RequestMessage` | interface/union | 无持久身份的请求专用输入 |
| `LlmModelInfo` / `LlmResolvedModelInfo` | interface | 咨询目录 / 精确能力解析 |
| `LlmConfigurableProvider` / `LlmNativeAuth` | interface | 配置可激活路由的目录条目 |
| `LlmModelDiscoveryRequest` / `LlmDiscoveredModel` | interface | 端点草稿询问 |
| `ToolHistory` | interface | 会话折叠出的工具声明历史(投影输入) |
| `LlmRequestPurposeMap` / `LlmRequestPurpose` | 空接口 / keyof | 辅助请求 purpose 的声明合并点 |
| `'llm/adapters-updated'` / `'llm/models-updated'` | 事件声明 | 拓扑/目录变化通知(无载荷/带 provider) |

## 与其他模块的关系

- `message.ts` 从这里 import `ContentBlock`;`ContentBlockType` 又引用 `message.ts` 的角色 map——两个文件用 `export type` 互引避免运行时环。
- `assembler.ts`、`assistant-stream.ts` 全部围绕 `StreamChunk` 工作;`content.ts` 投影 `ContentBlock`。
- `index.ts` 的 `LlmRuntime` API 签名几乎全部取自本文件;`call-config.ts` 的 `LlmCallConfig` 与 `GenerateOptions` 字段一一对应。
- 适配器包(`llm-deepseek`、`llm-pi-ai`)的 `stream()` 签名就是 `(options: GenerateOptions) => AsyncIterable<StreamChunk>`。

## 小结

- types.ts 是 harness 调模型词汇的唯一定义点:**merge-extensible 的开放词汇**(块/来源/finish)+ **封闭的协议词汇**(chunk)分工明确。
- 失败是数据(`LlmFailure`),按 `code` 路由;回放状态两半不透明、切分共享;token 计数不相交。
- 目录咨询、解析权威;`inputModalities` 缺席与显式空语义不同——这是 image 门控的依据。
