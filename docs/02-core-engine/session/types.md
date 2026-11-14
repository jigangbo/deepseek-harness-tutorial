# `types.ts` — 会话事件日志的全部类型：事件词汇表、信封与序号品牌

> 源码：`packages/core/session/src/types.ts`（569 行）。这是整个模块的"宪法"：所有事件类型、信封形状、序号品牌、头信息都在这里声明。

## 文件概览

`types.ts` 定义五层内容：

1. ** branded 标识与序号**：`SessionId`、`SessionSeq`、`SessionLogOffset`、`SessionSeqCursor`、`OptionalSessionSeq`——跨边界 id 一律 branded，序号带运行时验证。
2. **存储元数据**：`SESSION_FORMAT_VERSION`、`SessionHeader`、`CreateSessionOptions` / `PrepareSessionOptions` / `RestoredSessionOptions`。
3. **回合结束语义**：`TurnEndReasonMap`（merge-extensible）与取消原因。
4. **事件词汇表**：`SessionEventMap`、`SurfaceEventType`、`SurfaceOp`、`SurfaceIntent`、`SessionEvent` 信封。
5. **实验性插件记录**：`PluginRecordType`、`PluginRecordMap`、`PluginRecord`。

它在模块中的角色：被 `surface.ts`（消费事件类型集合）、`index.ts`（实现 `Session`/`SessionStore`）、以及全仓库几乎所有 agent/persistence/客户端包共同导入。

## 关键代码走读

### 1. 序号品牌：两个角色，两种类型

```typescript
/** Sequence number of one existing event in a Session log. */
export type SessionSeq = BrandedNumber<'SessionSeq'>

/**
 * Admit a numeric value as an existing Session event position.
 * @param value - non-negative safe integer admitted by the owning log operation.
 * @returns the same number with the Session-sequence brand.
 */
export function SessionSeq(value: number): SessionSeq {
  if (!Number.isSafeInteger(value) || value < 0 || Object.is(value, -0)) {
    throw new TypeError(`SessionSeq must be a non-negative safe integer, got ${String(value)}`)
  }
  return brandNumber<SessionSeq>(value)
}

/** A Session log gap, prefix length, or read offset, which may equal the event count. */
export type SessionLogOffset = BrandedNumber<'SessionLogOffset'>
```

会话日志位置用两个**语义不同**的数字类型：

- `SessionSeq` 标识**一个已存在的事件**（或包含式水位线）；
- `SessionLogOffset` 标识**间隙、前缀长度或读取边界**，可以等于事件总数（指向"下一个空位"）；
- `SessionSeqCursor` 增加 `-1`（"还没有任何事件"）；`OptionalSessionSeq` 用 `null` 表示"缺席本身是数据"。

品牌在运行时消失（就是普通 number），所以持久化 JSON 与线上值无需转换；但构造器做运行时验证：只接受非负 safe integer、显式拒绝 `-0`。算术运算的结果退化为普通 `number`，调用方必须用构造器按其意图重新"准入"。这个设计把 `seq` 与"偏移"的混用错误在编译期拦下。

### 2. `SessionEvent` 信封：可判别 union + 条件性表面元数据

```typescript
export type SessionEvent<T extends SessionEventType = SessionEventType> = {
  [K in SessionEventType]: {
    type: K
    /** Monotonic sequence number within the session. */
    seq: SessionSeq
    /** Unix epoch milliseconds. */
    time: number
    data: SessionEventMap[K]
    /**
     * Marks an event a reader may safely skip when it does not recognize
     * `type`. Absent means required: a reader meeting an unrecognized type
     * without this marker MUST refuse to reconstruct the session instead of
     * silently dropping the event, because an unrecognized required event may
     * change how the rest of the log is interpreted. A writer sets `true` only
     * on purely informational records whose loss cannot affect reconstruction;
     * defaulting to required means a forgotten marker over-refuses (an
     * inconvenience) rather than silently resuming a gutted session.
     */
    ignorable?: true
  } & (K extends SurfaceEventType ? SurfaceIntent<K> : {
    surfaceOp?: never
    sourceEventSeqs?: never
  })
}[T]
```

这是全模块最重要的类型工程：

- **proper discriminated union**：`switch (event.type)` 能无 cast 地窄化 `event.data`——不是独立的 `type`/`data` 两个 union。
- **required-on-read（`ignorable`）**：默认缺席即"必需"。写入方只对"丢失不影响重建"的纯信息记录标 `true`；默认必需意味着**忘标标记只会"过度拒绝"（不便）**，而不会静默重建一个被掏空的会话——安全侧偏置。
- **条件性字段**：只有 `SurfaceEventType` 变体（`system/message`、`developer/message`、`user/message`、`assistant/message`、`tool/result`）携带 `surfaceOp`/`sourceEventSeqs`；非表面事件（边界、attempt、错误）在类型层禁止这两个字段，`Session.append()` 调用点由编译器强制。
- **merge-extensible**：`SessionEventType = keyof SessionEventMap`，插件可扩展。因此对 `SessionEvent` 的 switch **不能用 `assertNever`**——插件新增变体是合法的未知值，须处理已知情形后落 `default`。

### 3. `SessionEventMap`（节选）与 `SurfaceOp`

```typescript
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/message': {
    turn: number
    step: number
    message: AssistantMessage
    /** Exact timed model stream, compacted without joining delta boundaries. */
    stream: AssistantStreamRecord[]
    usage?: TokenUsage
    interrupted?: true
  }
  'tool/call': { turn: number; step: number; callId: ToolCallId; name: string; arguments: string }
  'tool/result': {
    turn: number
    step: number
    message: ToolResultMessage
    error?: { name: string; code: string; reason?: string }
    meta?: JsonValue
  }
  'request/header': {
    header: EpochHeader
    reason: RequestHeaderReason
    startsSeries?: true
  }
  'session/end-seed': { inherited?: true }
}
```

（完整成员见源文件与 `docs/subsystems/session.md`；此处节选代表性成员。）值得注意的 payload 约定：

- `assistant/message` 把**精确的紧凑时间模型流** `stream` 嵌入事件——每次尝试的持久结算就是一个事件，token 用量随消息一起走（没有独立的 usage 记录）；`interrupted: true` 标记中途取消但已定稿的前缀。
- `tool/call` 保留模型产出的**原始未解析** `arguments` JSON 字符串。
- `tool/result.error` 只允许出现在 `isError: true` 的消息上（运行时在 `surface.ts` 校验）；`meta` 是工具私有的展示载荷，对 core 不透明但必须可序列化。
- `request/header` 是 log-only 的完整请求信封快照；最新一份快照重建请求头。

```typescript
export type SurfaceOp =
  | 'append'
  | { op: 'replace'; startSeq: SessionSeq; endSeq: SessionSeq }
```

`'append'` 是常规尾追加；`replace` 携带**精确三键**（`op`/`startSeq`/`endSeq`，无别名无多余键），包含式地遮蔽当前表面上 `[startSeq, endSeq]` 区间、把新事件插到它们的位置——compaction 用它"隐藏"已总结的历史而不删日志。

### 4. `TurnEndReasonMap` 与 `SessionHeader`

```typescript
export interface TurnEndReasonMap {
  completed: { kind: 'completed' }
  aborted: { kind: 'aborted'; reason: TurnEndCancelCause }
  blocked: { kind: 'blocked' }
  error: { kind: 'error'; error: LlmFailure }
  'max-tokens': { kind: 'max-tokens' }
  interrupted: { kind: 'interrupted' }
  forked: { kind: 'forked' }
}
export type TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]
```

两个"合成原因"只有特定路径会写：`interrupted` 由崩溃恢复在事后补写（循环从不在线写它）；`forked` 只出现在 fork seed 里（源会话中该 turn 在 fork 切点仍开着）。`max-tokens` 赢过后续继续：只要有一个 step 撞顶，整个 turn 以 `max-tokens` 收尾。

```typescript
export interface SessionHeader {
  readonly version: typeof SESSION_FORMAT_VERSION
  readonly id: SessionId
  readonly createdAt: number
  readonly cwd?: string
  readonly parentSession?: SessionId
  readonly isSeeded: boolean
  readonly origin?: 'subagent'
  readonly delegationDepth?: number
  readonly agentPreset?: string
}
```

头信息是**存储关注点**，不进事件日志、不可重放。注意 `parentSession` 只记录子 agent 的直接创建者；普通 fork 的历史 fork 可能残留此字段，所以**分类要用 `origin`**。`isSeeded` 报告是否存在 fork 继承前缀（用布尔而非暴露位置整数；精确切点长度是 `Session.inheritedEventCount`）。`delegationDepth`、`agentPreset` 持久化是为了递归预算与工具组合能**跨重启存活**。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `SessionId` | `Branded<'SessionId'>` + 构造函数 | 会话身份品牌（`brandString`，无运行时校验） |
| `SessionSeq` / `SessionLogOffset` | `BrandedNumber` + 验证构造函数 | 事件位置 / 日志偏移；非负 safe integer、拒绝 `-0` |
| `SessionSeqCursor` / `OptionalSessionSeq` | union 别名 | 含 `-1` 的水位线 / `null` 表示显式缺席 |
| `SESSION_FORMAT_VERSION` | `const = 4` | 当前逻辑格式版本；新写头盖此戳，读侧只接受此值（历史格式经相邻迁移链转换） |
| `SessionHeader` | interface | 深冻结的存储元数据（版本、id、cwd、谱系、isSeeded、origin、delegationDepth、agentPreset） |
| `CreateSessionOptions` / `PrepareSessionOptions` / `RestoredSessionOptions` / `SessionSeedEventState` | interface/union | store 创建参数：seed、inheritedEventCount、meta；恢复路径的别名状态（`'detached' \| 'shared-frozen'`） |
| `SessionEventMap` | interface（merge-extensible） | 事件词汇表：每个事件的 data 载荷 |
| `SessionEvent` | 映射类型 union | 单条日志条目：type/seq/time/data/ignorable + 条件表面元数据 |
| `SurfaceEventType` / `SurfaceEvent` | union / 别名 | 五个消息产生型事件类型及其事件形状 |
| `SurfaceOp` / `SurfaceIntent` | union / 条件类型 | 表面放置标记；`append()` 的 opts 参数（assistant 禁止 `sourceEventSeqs`） |
| `EpochHeader` / `RequestContext` / `RequestHeaderReason` | interface / union | 请求信封（config + adapterDefaults + tools，`system` 字段废弃为 `never`）、路由元数据、header 快照原因 |
| `TurnEndReasonMap` / `TurnEndReason` / `AgentCancelCause` / `TurnEndCancelCause` | interface / union | 回合结束原因（可扩展）、取消原因（含导入用的 `legacy`） |
| `PluginRecordType` / `PluginRecordMap` / `PluginRecord` | 模板串类型 / interface / interface | `plugin:` 命名空间、实验包 augment 的载荷映射、读回的记录（`data: unknown`） |
| typert module merge | `TypertResourceMap` / `RemoteErrorDetailsMap` | `session` 资源与 `session/not-found` 错误详情 |

## 与其他模块的关系

- **`dsh-brand`**：`Branded`/`BrandedNumber`/`brandString`/`brandNumber` 全部来自 `@deepseek-ai/dsh-brand`——"跨边界不透明 id 必须 branded"约定的落点。
- **`dsh-llm`**：`UserMessage`、`AssistantMessage`、`ToolResultMessage`、`AssistantStreamRecord`、`LlmCallConfig`、`ToolSchema` 等消息与请求类型都在 llm 包定义，此处只做事件化包装。
- **`surface.ts` / `index.ts`**：运行时验证与 `Session` 实现都按本文件的形状执行；`SESSION_FORMAT_VERSION` 被 `index.ts` 的头验证与 `SessionStore.prepare` 盖戳。
- **持久化与格式迁移链**：`version: 4` 与 `ignorable` 信封是相邻迁移包（`session-format-v*`）的分诊依据；`@persistenceReserved` 标记的 `system?: never` 字段禁止新写入。
- **SDK / 客户端**：`SessionEvent` 的判别 union 让 `switch (event.type)` 在 SDK 与 Web 客户端都能无 cast 窄化。

## 小结

`types.ts` 用三个机制守住日志的可重建性：**品牌序号**（两种位置语义分开）、**required-on-read 信封**（未知事件默认拒绝，`ignorable: true` 才可跳过）、**条件性表面元数据**（编译期区分消息产生事件与 log-only 事件）。`SessionEventMap` 与 `TurnEndReasonMap` 都是 merge-extensible——消费它们时用"已知分支 + default"而不是 `assertNever`。
