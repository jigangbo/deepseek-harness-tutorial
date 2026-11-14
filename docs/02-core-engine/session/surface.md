# `surface.ts` — 有序表面投影：日志如何变成模型可见的有序消息

> 源码：`packages/core/session/src/surface.ts`（712 行）。模块注释开宗明义："Surface layer on top of the session event log: an ordered view of events that produce LLM messages. The append-only log remains the source of truth." 另一个硬约束：**browser-safe**——Web 客户端直接消费这个子路径导出，禁止任何 `node:` 导入。

## 文件概览

`surface.ts` 实现四件事：

1. **事件 → 消息的纯投影**：`deriveEventMessage()` 是唯一的投影规则函数。
2. **有序表面**：五类表面事件按 `surfaceOp`（`'append'` / `{ op:'replace', … }`）维护一个"模型可见顺序"的 seq 列表（`nodes`）；`replace` 遮蔽区间但不删日志。
3. **校验器**：事件局部 payload 校验（`validateSessionEventData`）、表面元数据校验（`validateSurfaceMetadata`）、替换合法性（sourceEventSeqs 覆盖、tool/result 改写限制、node 0 系统提示保护）。
4. **两种 fold**：`foldSurface()`（完整重放，供冷读）与 `SurfaceManager`（增量视图 + append 前验证器，供活会话），共用同一套转移函数——保证活会话与离线重放不可能分歧。

它是"Model-visible ⟺ logged"的机制化：日志是唯一事实，表面是它的**派生投影**。

## 关键代码走读

### 1. `deriveEventMessage()` — 单事件的投影规则

```typescript
export function deriveEventMessage(
  event: SessionEvent,
  projectedMessages?: ReadonlyMap<SessionSeq, Message>,
): Message | null {
  const projected = projectedMessages?.get(event.seq)
  if (projected !== undefined) return projected
  switch (event.type) {
    case 'user/message': {
      return event.data
    }
    case 'system/message':
    case 'developer/message':
    case 'assistant/message': {
      if (event.data.message.content.length === 0) return null
      return event.data.message
    }
    case 'tool/result': {
      return event.data.message
    }
    default:
      // A non-surface event (boundary, attempt, log-only record) projects to
      // no message. Merge-extensible union: no assertNever here.
      return null
  }
}
```

规则极小而精确：

- `user/message` 的 `data` 就是 `UserMessage`，原样投影（普通 prompt 与注入上下文都保留 producer 自己的 framing，不加包装）。
- **空内容节点不投影**：空 system/developer 节点是"休眠"的（保留表面位置但不产生 wire 消息）；空 assistant 事件承载 max-tokens step 的 usage，绝不能给 provider transcript 注入一个无内容的 assistant 回合。
- `tool/result` 投影为一等 tool 消息。
- 其余（`turn/*`、`step/*`、`assistant/attempt`、插件 log-only 事件）返回 `null`——**trace/replay 数据，不是历史**。
- `default` 不用 `assertNever`：union 是 merge-extensible 的，插件事件是合法未知值。
- 第二参数 `projectedMessages` 来自 `foldSurface`：外部重建者传入同一前缀的投影结果，保证与活会话缓存**使用完全相同的规则**、不可能 disagree。

### 2. `validateSessionEventData()` — 事件局部一致性校验

```typescript
if (event.type === 'request/header') {
  if (!isRecord(data)) throw new Error(`${subject} data must be an object`)
  const header = data['header']
  if (!isRecord(header)) throw new Error(`${subject} header must be an object`)
  if (Object.hasOwn(header, 'system')) throw new Error(`${subject} must omit header.system; use system/message`)
  if (Array.isArray(header['tools']) && header['tools'].length === 0) {
    throw new Error(`${subject} must omit empty tools`)
  }
  const defaults = header['adapterDefaults']
  if (isRecord(defaults) && Object.keys(defaults).length === 0) {
    throw new Error(`${subject} must omit empty adapterDefaults`)
  }
} else if (event.type === 'tool/result') {
  if (data['error'] === undefined) return
  const message = data['message']
  if (!isRecord(message) || message['isError'] !== true) {
    throw new Error(`${subject} error requires message.isError === true`)
  }
}
```

（前半段校验 developer 角色与工具增删块、`headerSeq` 的"恰在有 addition 时出现"等规则，此处节选后半。）这是**拒绝而非规范化**的范例：非规范的 `request/header`（带 `system` 字段、空 `tools: []`、空 `adapterDefaults: {}`）直接抛错而不是悄悄修掉——"Misconfiguration fails loud"。同理 `error` 字段只允许在 `isError: true` 的消息上出现，失败身份与消息内容保持一致。

### 3. 替换校验三件套：source 覆盖、tool/result 限改、node 0 保护

```typescript
const range = replacementRange(state, surfaceOp)
assertSourceEventReferences(event, range.shadowedSeqs)
assertToolResultRewrite(event, range.shadowedSeqs, events, baseSeq)
assertSystemHeadRewrite(event, state, range.startIdx, range.shadowedSeqs, events, baseSeq)
```

`planSurfaceEvent` 在真正提交替换前执行三道闸：

- **`assertSourceEventReferences`**：`replace` 事件的 `sourceEventSeqs` 必须**完整包含**每一个被遮蔽的表面节点（`missing` 列表直接进错误消息）；引用必须更早、唯一、非空；`assistant/message` 类型禁止携带 `sourceEventSeqs`（它内嵌自己的 stream 作为证据）。
- **`assertToolResultRewrite`**：`tool/result` 的替换只能改写**一个**当前节点，且除 `content` 外的其余字段逐字段深度相等（比较时把双方 `content` 置 null）——工具结果改写是"重试场景下补内容"，不是任意改写。
- **`assertSystemHeadRewrite`**：node 0 若是 `system/message`（渲染的系统提示），只能被"恰好覆盖该节点的另一个 `system/message`"改写；node 0 之后的系统节点无保护，可被 compaction 遮蔽。

`planSurfaceEvent` 还强制**序列连续性**（`event.seq !== expectedSeq` 即抛）与消息投影解释器存在性（`MESSAGE_PROJECTION_EVENT_TYPES` 中的类型无定义直接抛）。所有校验在**变更状态前**完成（"validated replacement transition that has not mutated fold state yet"），失败不可能留下半提交的表面。

### 4. `SurfaceManager` — 增量视图与 append 前验证器

```typescript
export class SurfaceManager implements SessionSurface {
  private _state = createFoldState()
  private _lastProcessedSeq: SessionSeqCursor
  private _pendingPlan: { event: SessionEvent; expectedSeq: SessionSeq; plan: SurfacePlan | undefined } | undefined

  validateNext(event: SessionEvent): void {
    this._assertProjections()
    if (this._lastProcessedSeq < this.baseSeq + this.log.length - 1) this._processDelta()
    const expectedSeq = SessionSeq(this.baseSeq + this.log.length)
    this._pendingPlan = {
      event,
      expectedSeq,
      plan: planSurfaceEvent(this._state, event, expectedSeq, this.log, this.baseSeq, this.projections),
    }
  }
```

活会话的 `Session` 持有一个 `SurfaceManager`（以自己的 `log` 数组为 `log` 参数）：

- **`validateNext`**：append 提交前调用，验证候选但不改动已提交状态，把通过的计划暂存 `_pendingPlan`。
- **惰性 `_processDelta`**：每个访问器（`nodes`/`replaceGeneration`/`contentGeneration`/`deriveEventMessage`）先消费 log 尾部的新事件；若暂存计划对应的正是这个事件，直接 `applySurfacePlan` 提交它——避免重复规划。
- **`_assertProjections`**：缓存的消息不能活过解释它的定义——已使用的投影定义被移除后，任何读取都会抛错（`session message projection "…" was removed or replaced`）。
- **窗口化构造**：`SurfaceManager(log, baseSeq, projections)` 支持折叠"从绝对序号 `baseSeq` 开始的连续窗口"；跨窗口头的替换因区间不存在而失败。

对外暴露的只读视图 `SessionSurface`：

```typescript
export interface SessionSurface {
  readonly nodes: readonly SessionSeq[]
  readonly replaceGeneration: number
  readonly contentGeneration: number
}
```

`nodes` 是模型可见顺序的表面 seq 列表；`replaceGeneration` 是已提交位置替换的单调计数（区分"纯尾增长"与"重写"）；`contentGeneration` 计入替换**和**插件消息变化（`Session.deriveMessages()` 用它决定是否重建缓存）。`foldSurface()` 返回的 `SurfaceFoldResult` 额外带 `replacements: SurfaceFoldReplacement[]`（每个替换实际遮蔽了哪些节点）与 `projectedMessages`（投影消息表），供冷读者精确重放。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `deriveEventMessage` | `(event, projectedMessages?) => Message \| null` | 单事件投影规则（纯函数、公开供外部重放对齐） |
| `validateSessionEventData` | `(event, subject) => void` | 事件局部 payload 校验：developer 角色/工具块/headerSeq 配对、request/header 规范性、tool error 一致性 |
| `validateSurfaceMetadata` | `(event) => SurfaceOp \| undefined` | 表面元数据校验（marker 形状、 eligibility、replace 端点必须更早）；未知 ignorable 记录返回 undefined |
| `foldSurface` | `(events, projections?) => SurfaceFoldResult` | 完整重放：nodes + 替换历史 + 投影消息（冷读路径） |
| `SurfaceManager` | `class implements SessionSurface` | 增量表面视图 + `validateNext` append 前验证器（活会话路径） |
| `isSurfaceEvent` / `isAppendSurfaceEvent` / `isReplacementSurfaceEvent` / `isSurfaceEligibleType` | 类型守卫 | 区分表面事件 / 尾追加事件（人类 transcript 的素材）/ 替换事件 / 可上表面类型 |
| `SessionMessageProjection` / `SessionMessageProjectionContext` | interface | 插件拥有的消息投影定义（纯解释器）与其只读上下文 |
| `SurfaceFoldReplacement` / `SurfaceFoldResult` | interface | 单次替换记录（含实际 `shadowedSeqs`）/ 完整 fold 结果 |
| 内部：`surfaceOpOf` / `assertSourceEventReferences` / `assertDeveloperHeader` / `assertToolResultRewrite` / `assertSystemHeadRewrite` / `planSurfaceEvent` / `applySurfacePlan` | — | 校验与原子转移的实现细节 |

## 与其他模块的关系

- **`index.ts`**：`Session` 构造器把 `SurfaceManager` 建在自己的 log 上；`append()` → `commit()` → `surfaceManager.validateNext(event)`；`deriveMessages()`/`deriveEventMessage()` 全部委托给它。seed 校验也走同一条 `validateNext` 路径（"候选在进入 log 前先规划，失败不可能部分变更表面"）。
- **`agent-loop`**：不直接消费 surface，但它的每次 `append('user/message'|'assistant/message'|'tool/result', data, { surfaceOp: 'append' })` 都受此处校验；系统提示的 node 0 语义（何时 replace、何时 in-history 追加）由 `requestContext()` 读到的 `systemPromptUpdate` 模式决定，此处保护 node 0 不被其他生产者改写。
- **`compaction-*`**：`replace` 操作的合法生产者——把已总结区间替换为一个 summary 消息节点；node 0 系统提示保护与 tool/result 限改规则约束它能遮蔽什么。
- **Web 客户端**：`@deepseek-ai/dsh-session/surface` 子路径被 `packages/client/ui-chat` 等多个包直接导入（browser-safe 约束的来源）；人类 transcript 用 `isAppendSurfaceEvent` 过滤 append-origin 事件——替换副本只对模型生效，用户已经看到的对话不能"消失"。
- **`session-query` / 快照套件 / 格式目录**：冷读用 `foldSurface` + `projectedMessages` 精确重放派生规则。

## 小结

`surface.ts` 是"日志 → 模型历史"的翻译器，核心思想是**一条规则、两处使用**：`foldSurface`（离线全量）与 `SurfaceManager`（在线增量）共享同一套 plan/apply 转移函数，所以重放与活会话永不分歧。记住三组不变量：`seq` 连续性、`replace` 必须完整引用被遮蔽节点、node 0 的系统提示只许系统提示改写；再记住两个"空值语义"：空内容节点不投影、空 `sourceEventSeqs` 不合法（要么不引用，要么完整非空）。
