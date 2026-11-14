# `brand.ts`

> 本包拥有的 branded id


> 源码:`packages/llm/llm/src/brand.ts`(约 77 行)。五个跨边界 id 的名义类型(brand)与无状态构造器。`Branded<B>` 原语本身住在 `@deepseek-ai/dsh-brand`,每个 id 的**拥有者**各自在自己的模块里声明 brand——dsh-llm 不向外出租它的 brand。

## 文件概览

仓库约定:跨边界的不透明 id 必须是 branded(`Branded<B>`),不能是裸 `string`。本文件是 dsh-llm 拥有的五个 id 的登记处,每个都是同一个三件套:类型别名 + 构造函数 + JSDoc。

## 关键代码走读

```typescript
// packages/llm/llm/src/brand.ts
/** Stable identity carried by one message across inbox, log, and model-request boundaries. */
export type MessageId = Branded<'MessageId'>

export function MessageId(id: string): MessageId {
  return brandString<MessageId>(id)
}

/** Correlates a model-issued tool call with its result. Provider-issued for real adapters; synthesized by mocks/assembler fallbacks. */
export type ToolCallId = Branded<'ToolCallId'>

export function ToolCallId(id: string): ToolCallId {
  return brandString<ToolCallId>(id)
}

/** Provider-issued request identifier retained for diagnostics across package boundaries. */
export type ProviderRequestId = Branded<'ProviderRequestId'>

/** Identity of one model streaming attempt, unique within one Agent lifecycle. */
export type LlmAttemptId = Branded<'LlmAttemptId'>

/** Adapter-owned identifier for one model's selectable reasoning effort. */
export type ReasoningEffortId = Branded<'ReasoningEffortId'>
```

(ProviderRequestId / LlmAttemptId / ReasoningEffortId 的构造函数与上同型,略。)

五个 id 的语义分工:

| id | 谁签发 | 关联什么 |
|----|--------|----------|
| `MessageId` | 创建消息的一方 | 一条消息跨 inbox、日志、模型请求的身份 |
| `ToolCallId` | provider(真实适配器);mock/装配器兜底合成(`call-${index}`) | 模型发出的调用 ↔ 其结果 |
| `ProviderRequestId` | provider | 跨包诊断用的请求 id(`LlmFailure.requestId`) |
| `LlmAttemptId` | agent loop | 一次模型流尝试,Agent 生命周期内唯一 |
| `ReasoningEffortId` | 适配器 | 一个模型的可选推理 effort(`GenerateOptions.reasoningEffort`) |

模式要点:

- **同名函数即构造器**:`MessageId('…')` 与类型 `MessageId` 共用名字(TS 里类型与值命名空间分离,合法且惯用);构造**零验证**——brand 的意义是让类型系统区分"这个字符串属于哪个域",不是内容校验,校验归属各域自己的解析层;
- **每域一个 brand,不共享**:`Branded<'ToolCallId'>` 的字面量参数是唯一的域标签;两个不同 id 即使都是 string 也不能互相赋值——这正是"显式 > 隐式"在类型层的表达;
- JSDoc 明确 **ToolCallId 的合成来源**:assembler 在适配器没发 id 时用 `call-${index}` 合成并打 brand,保证不变量(工具调用必有 id)始终成立。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `MessageId` / `MessageId(id)` | type + `(id: string) => MessageId` | 消息身份 |
| `ToolCallId` / `ToolCallId(id)` | type + 同上 | 调用↔结果关联 |
| `ProviderRequestId` / `ProviderRequestId(id)` | type + 同上 | provider 请求诊断 id |
| `LlmAttemptId` / `LlmAttemptId(id)` | type + 同上 | Agent 生命周期内的尝试 id |
| `ReasoningEffortId` / `ReasoningEffortId(id)` | type + 同上 | 适配器拥有的 effort id |

## 与其他模块的关系

- `types.ts` import `MessageId`/`ToolCallId`/`ProviderRequestId`/`ReasoningEffortId`(`GenerateOptions.reasoningEffort`、`ToolCallBlock.id`、`LlmFailure.requestId` 的类型)。
- `message.ts` 的 `createMessage` 用 `brandString<MessageId>(randomUUID())` 现场签发;`assembler.ts` 兜底合成 `ToolCallId`。
- `index.ts` 的 `LlmErrorOptions.requestId` 用 `ProviderRequestId`。
- `@deepseek-ai/dsh-brand` 提供原语(`Branded<B>`、`brandString`);session、tools 等包各自拥有别的 brand——**brand 不跨包借用**。

## 小结

- 五个 id 覆盖本包全部跨边界标识;同名构造器零验证是设计而非偷懒——brand 是类型层的域标签。
- `ToolCallId` 的"provider 签发 + 装配器合成兜底"双来源写在 JSDoc,是"不变量永远成立"的记录方式。
