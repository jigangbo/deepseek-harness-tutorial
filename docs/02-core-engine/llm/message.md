# `message.ts` — 不可变消息构造器与来源词汇

> 源码:`packages/llm/llm/src/message.ts`(约 300 行)。定义持久会话消息的类型、来源(`MessageSource`)词汇,以及一组"构造即冻结"的工厂函数。

## 文件概览

四段内容:

1. **来源类型**:`AssistantProviderMetadata`、`MessageSourceMap`(merge-extensible)、`ContextForm`/`ContextFormed`/`ContextSnapshotSection`;
2. **角色类型**:`MessageRoleMap`(封闭)与五个角色消息接口;
3. **构造器**:`createMessage` 泛型工厂 + 各角色特化构造器 + `freezeMessage`;
4. **notice 摘要约束**:`CONTEXT_SUMMARY_MAX_CHARS` 与 `boundContextSummary`。

## 关键代码走读

### 消息基座与角色 map

```typescript
// packages/llm/llm/src/message.ts
interface MessageBase {
  /** Stable identity preserved across every representation boundary. */
  readonly id: MessageId
  /** Exact model-facing blocks. */
  readonly content: readonly ContentBlock[]
  /** Required source fields supplied by the producer. */
  readonly source: MessageSource
}

export interface MessageRoleMap {
  system: SystemMessage
  developer: DeveloperMessage
  user: UserMessage
  assistant: AssistantMessage
  tool: ToolResultMessage
}
/** Any persisted conversation message, discriminated by its `role`. */
export type Message = MessageRoleMap[keyof MessageRoleMap]
```

- 角色注释明说:`MessageRoleMap` 是**封闭**的——每个模型可见角色必须有持久 Session 事件和适配器投影,不像来源那样谁都能加。
- `source` 是必填的:每条消息都知道自己是谁产的。`@persistenceSource user developer` 标记这两类角色的 source 由 producer 自由声明。

### 来源:merge-extensible 的 producer 身份

```typescript
// packages/llm/llm/src/message.ts
export interface MessageSourceMap {
  user: { kind: 'user' }
  model: ModelMessageSource
  tool: ToolMessageSource
  'system-prompt': SystemPromptMessageSource
}
export type MessageSource = MessageSourceMap[keyof MessageSourceMap]
```

producer 在**自己的模块**里声明自己的 `kind`,没有共享的 `plugin` 兜底 kind——消费者遇到未知 kind 落穿处理。模型来源携带 provider/model 与可选回放状态:

```typescript
// packages/llm/llm/src/message.ts
export interface AssistantProviderMetadata {
  /** Provider route that produced the message. */
  provider: string
  /** Provider model id that produced the message. */
  model: string
  /**
   * Lossless-JSON adapter state needed to replay the provider response.
   * `LlmRuntime` exposes it to a target adapter only when that adapter instance
   * currently owns both this historical provider and the target provider.
   */
  replayState?: unknown
}
export interface ModelMessageSource extends AssistantProviderMetadata {
  kind: 'model'
}
```

`replayState` 就是 `ReplayEnvelope`(types.ts)落盘后的形态:不透明、由 `LlmRuntime` 按同适配器原则转发。

### ContextForm:producer 身份与信息形态是两个正交轴

```typescript
// packages/llm/llm/src/message.ts
export type ContextForm =
  | 'instructions'   // 工作区文件里模型要遵守的指令
  | 'catalog'        // 会话可用条目目录,变化时重新发布
  | 'snapshot'       // 当前状态,同 producer 后发覆盖先发
  | 'notice'         // 一次性事件记述,不覆盖任何东西
  | 'relay'          // 另一个 agent 发给本 agent 的消息
  | 'recall'         // 从别的会话日志抽出(可能已缩减)的材料
```

JSDoc 强调这条纪律:词汇是**语义的,不是视觉的**——颜色、图标、折叠默认值都是消费者的事,不得进入这个联合。`ContextFormed` 用判别联合保证声明形态就必须带上它需要的字段(`notice` 要 `summary`,`snapshot` 要 `sections`);省略 `form` 仍是合法的"未声明上下文"默认态。

### 工厂:构造即深冻结

```typescript
// packages/llm/llm/src/message.ts
export function createMessage<T extends NewMessage>(
  input: T & { readonly id?: never },
): T & Pick<Message, 'id'> {
  return deepFreeze(structuredClone({
    ...input,
    id: brandString<MessageId>(randomUUID()),
  }))
}

export function createAssistantMessage(
  input: NewAssistantMessage & { readonly id?: never; readonly role?: never },
): AssistantMessage {
  return createMessage({
    role: 'assistant',
    content: input.content,
    source: { kind: 'model', ...input.source },
  })
}

export function createSystemMessage(text: string): SystemMessage {
  return createMessage({
    role: 'system',
    content: text.length === 0 ? [] : [{ type: 'text', text }],
    source: { kind: 'system-prompt' },
  })
}

export function createToolResultMessage(input: ToolResultMessageInput): ToolResultMessage {
  return createMessage({
    role: 'tool',
    source: { kind: 'tool', callId: input.callId },
    toolCallId: input.callId,
    content: input.content,
    isError: input.isError,
  })
}
```

- 输入类型用 `id?: never` / `role?: never` 在编译期禁止调用者传身份或角色——id 由工厂生成(随机 UUID 打 brand),role 由特化工厂固定。
- `structuredClone` + `deepFreeze`:发布前脱离调用者的可变别名并冻结,消息从此不可变(这也是 `LlmRuntime.forAdapter` 能安全重建消息的前提)。
- `createSystemMessage('')` 记录"无系统提示"(空 content)——空渲染也是一条要落日志的事实。
- `createAssistantMessage` 的输入 `source` 不带 `kind`,工厂补 `kind: 'model'`;`createToolResultMessage` 把 `callId` 同时放进 `source.kind` 字段与顶层 `toolCallId`(后者才是适配器投影用的)。

### freezeMessage 与 notice 摘要约束

```typescript
// packages/llm/llm/src/message.ts
export function freezeMessage<T extends Message>(message: T): T {
  return deepFreeze(structuredClone(message))
}

export const CONTEXT_SUMMARY_MAX_CHARS = 120

export function boundContextSummary(summary: string): string {
  return summary.length <= CONTEXT_SUMMARY_MAX_CHARS
    ? summary
    : `${summary.slice(0, CONTEXT_SUMMARY_MAX_CHARS - 1)}…`
}
```

`freezeMessage` 保留已有身份(不换 id),用于派发前对已有消息快照。`boundContextSummary` 把 notice 摘要截到 120 字符(留 1 字符给省略号)——输入是 caller 文本,长度不受控,所以 commit 到持久日志前先约束。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `MessageRoleMap` / `Message` | 封闭 map / union | 五个持久角色,按 `role` 判别 |
| `MessageSourceMap` / `MessageSource` | merge-extensible map / union | producer 身份;未知 kind 落穿 |
| `ModelMessageSource` | interface | `kind:'model'` + provider/model/replayState |
| `ToolMessageSource` | interface | `kind:'tool'` + callId |
| `SystemPromptMessageSource` | interface | `kind:'system-prompt'` |
| `ContextForm` / `ContextFormed` | union / 判别联合 | 上下文信息形态(语义词汇)及其必带字段 |
| `createMessage` | `<T extends NewMessage>(input) => T & {id}` | 泛型工厂:生成 id、深冻结 |
| `createDeveloperMessage` / `createUserMessage` | 特化工厂 | 固定 role 后委托 `createMessage` |
| `createAssistantMessage` | `(input) => AssistantMessage` | 补 `kind:'model'` 后委托 |
| `createSystemMessage` | `(text) => SystemMessage` | 空文本 → 空 content |
| `createToolResultMessage` | `(input) => ToolResultMessage` | callId 双写(source + toolCallId) |
| `freezeMessage` | `<T extends Message>(message) => T` | 保留身份的脱离+冻结 |
| `boundContextSummary` | `(summary) => string` | notice 摘要截断到 120 字符 |

## 与其他模块的关系

- `types.ts` re-export 本文件的全部类型(`Message`/`UserMessage`/`ModelMessageSource`…),`GenerateOptions.messages` 的元素就是这里的值。
- `assembler.ts` 的 `message(source)` 用 `createAssistantMessage` 收尾——装配产出的 assistant 消息与其他消息走同一条不可变构造路径。
- `index.ts` 的 `forAdapter()` 用 `freezeMessage` 重造剥掉 replayState 的 assistant 消息。
- Session/attachment 包持久这些值;developer 角色的 `tool-addition`/`tool-removal` 块由 `content.ts` 的 `projectToolUpdates` 按路由能力投影。

## 小结

- 消息 = 身份 + 模型可见块 + producer 来源;角色封闭、来源开放,`switch` 纪律各不相同。
- 所有工厂"构造即冻结",且在类型层禁止调用者提供 id/role——不可变性由构造路径保证,而不是事后约定。
- `ContextForm` 语义词汇与视觉表现彻底分离;notice 摘要在 commit 前被约束到固定长度。
