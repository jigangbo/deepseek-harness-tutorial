# 第 7 章 Session：会话事件日志

> 源码：`packages/core/session/`（`@deepseek-ai/dsh-session`）。本文同时覆盖该包的入口文件 `src/index.ts`。

## 文件概览

`dsh-session` 是整个 Harness 的**事实记录层**：一条 **append-only 的 `SessionEvent` 日志**是 agent 交互历史的唯一来源（source of truth）。仓库最高约定 **"Model-visible ⟺ logged"**——任何进入模型请求的内容都必须能从这条日志重建出来。模型看到的 `Message[]` 历史**不单独存储**，而是用 `deriveMessages()` 从日志**派生**；重放（resume）就是用同一批事件重新派生一遍。

关键设计决策：

- **`Session` 是普通类，不是 Cordis Service**。它自己不持久化、不调模型；持久化是插件的事（订阅 `session/event`、在 `session/flush` 上排空缓冲）。
- **`SessionStore`（`ctx.sessions`）是 Cordis 服务**，负责创建、进入、宣布、fork、flush 会话，并持有发布钩子。
- **派生视图全部增量缓存**：surface（有序表面）、request header、request context、tool history、derived messages——每个事件只 fold 一次，后续读取只消费新事件。
- **所有事件数据必须无损 JSON 可序列化**，在 append 现场一次性验证（一次迭代式 pass 同时完成读取、验证、快照），坏事件进不了日志。

## 架构图

```
                    ┌─────────────────────────────────────────────────┐
                    │              agent-loop（主要写入方）             │
                    │  turn/*  step/*  user/*  assistant/*  tool/*    │
                    │  request/header  request/context                │
                    └───────────────────────┬─────────────────────────┘
                                            │ session.append(type, data, opts?)
                                            ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ Session（普通类，append-only log）                                             │
│                                                                              │
│   log: SessionEvent[]   ◄── snapshotJsonValue 一次遍历验证 + 冻结快照           │
│   seq = log.length（从 0 连续，seq-ranges.ts 编码源引用）                        │
│                                                                              │
│   ├─ SurfaceManager (surface.ts) ── 增量验证 + 有序表面投影                      │
│   │      nodes / replaceGeneration / contentGeneration                        │
│   ├─ requestHeader() / requestContext() / toolHistory() ── 增量 fold           │
│   └─ deriveMessages() ── 派生 Message[]（深冻结、缓存、每次返回新数组）            │
└────────────┬─────────────────────────────────────────────────┬───────────────┘
             │ commit → session/event（提交后推送，fire-and-forget）│ session/flush
             ▼                                                 ▼
┌──────────────────────────────┐                  ┌─────────────────────────────┐
│ SessionStore（ctx.sessions）  │                  │ 持久化插件                     │
│ create / prepare / enter /    │                  │ （session-log-jsonl 等）      │
│ announce / fork / flush       │                  │ 异步缓冲 → session/flush 落盘  │
└──────────────────────────────┘                  └─────────────────────────────┘
             │
             ▼  下游消费者（读日志或派生视图）
   agent-loop（deriveMessages 组装模型输入） · Web 客户端（@deepseek-ai/dsh-session/surface 子路径，
   browser-safe）· session-query（冷读 foldSurface）· session-telemetry · session-title …
```

## 事件大类清单

事件词汇表是 `SessionEventMap`（定义在 `types.ts`），**merge-extensible**：插件通过 declaration merging 追加自己的 log-only 事件类型。核心成员按用途分组：

| 大类 | 事件类型 | 说明 |
|---|---|---|
| 回合/步骤边界 | `turn/start`、`turn/end`、`step/start`、`step/end` | 结构性标记，不产生模型消息；`turn/end` 携带 `TurnEndReason`（completed / aborted / blocked / error / max-tokens / interrupted / forked，可扩展） |
| **表面事件**（`SurfaceEventType`，产生模型消息） | `system/message`、`developer/message`、`user/message`、`assistant/message`、`tool/result` | 必须携带 `surfaceOp`（`'append'` 或 `{ op:'replace', startSeq, endSeq }`），声明自己如何进入有序表面 |
| 工具流 | `tool/call`、`tool/result` | `callId` 配对；`tool/result` 可带 `error`（仅当 `isError: true`）与工具私有 `meta` |
| 模型尝试 | `assistant/attempt` | 一次没有产生表面消息的模型尝试（失败/重试/取消/流错误），内嵌紧凑时间流，不进入派生历史 |
| 请求状态 | `request/header`、`request/context` | log-only：完整请求信封快照（call config、adapter 默认值标记、工具 schema）与路由元数据；最新快照重建请求 |
| 会话边界 | `session/end-seed` | log-only 标记：分隔继承/恢复历史与生命周期自有工作；fork seed 用 `{ inherited: true }` 标出继承切点 |
| 插件贡献（merge 扩展） | `compaction/*`、`hook/*`、`approval/*`、`todo/write`、`llm/retry`、`subagent/*`、`team/*` 等 | 各插件自有语义，core 不拒绝未知类型；完整清单见 `known-event-types.ts` |
| 实验性插件记录 | `plugin:<owner>/<record>` | 走 `appendPluginRecord()`，恒带 `ignorable: true`，永不进入模型表面 |

每个事件信封上还有两个关键字段：

- **`seq`**：单调递增的事件序号，恒等于 `log.length`（`seq = log.length` 连续性契约）。构造器 `SessionSeq(value)` 只接受非负 safe integer，拒绝 `-0`。
- **`ignorable?: true`**：**required-on-read 约定**的开关。缺席意味着"必需"——读取方遇到不认识的类型且没有此标记时，**必须拒绝重建会话**而不是静默丢弃（未知必需事件可能改变日志其余部分的解释）；带 `true` 的是纯信息性记录，丢失不影响重建。

## 关键代码走读（`src/index.ts`）

`index.ts` 包含四块内容：Cordis 事件声明、`Session` 类、`SessionStore` 服务、插件记录读写。

### 1. 四个 Cordis 事件

```typescript
declare module '@deepseek-ai/cordis' {
  interface Events {
    'session/created'(this: Scoped<Session>, session: Session): void
    'session/disposed'(this: Scoped<Session>, session: Session): void
    'session/event'(this: Scoped<Session>, session: Session, event: SessionEvent): void
    'session/flush'(this: Scoped<Session>, session: Session): Promise<void> | void
  }
}
```

- `session/created`：进入 store 后的**只读创建宣布**，同步派发；同步抛出会否决创建并回滚。
- `session/event`：**提交后**的追加推送。监听器快照先解析、回调在日志 push 之后运行；监听器失败只记日志并逐个隔离，绝不影响已提交的 append。
- `session/flush`：**awaited parallel** 检查点——每个持久化监听器都运行、调用方等全部完成。
- `session/disposed`：已宣布的会话离开 store 时成对发出。
- 三类派发都经 `@deepseek-ai/dsh-scope` 按 agent 作用域过滤。

### 2. `Session.append()` — 唯一的写入口

```typescript
append<T extends SessionEventType>(
  type: T,
  data: SessionEventMap[T],
  ...opts: T extends SurfaceEventType ? [opts: SurfaceIntent<T>] : []
): SessionEvent<T> {
  this.publicationEntry()
  const surfaceOpts: SurfaceIntent | undefined = opts[0]
  const surfaceMetadata = {
    ...surfaceOpts?.sourceEventSeqs === undefined ? {} : { sourceEventSeqs: surfaceOpts.sourceEventSeqs },
    ...surfaceOpts?.surfaceOp === undefined ? {} : { surfaceOp: surfaceOpts.surfaceOp },
  }
  const dataSnapshot = snapshotJsonValue(data)
  if (dataSnapshot === undefined) {
    throw new Error(`session event "${type}" carries non-JSON-serializable data`)
  }
  const surfaceMetadataSnapshot = snapshotJsonValue(surfaceMetadata)
  if (surfaceMetadataSnapshot === undefined) {
    throw new Error(`session event "${type}" carries non-JSON-serializable surface metadata`)
  }
  // Payload getters can detach the Session while the snapshots are read.
  const entry = this.publicationEntry()
  const event = deepFreeze({
    type,
    seq: SessionSeq(this.log.length),
    time: Date.now(),
    data: dataSnapshot,
    ...(surfaceMetadataSnapshot as { surfaceOp?: unknown; sourceEventSeqs?: unknown }),
  } as unknown as SessionEvent<T>)
  validateSessionEventData(event, `session event "${type}" at seq ${event.seq}`)
  this.commit(event as SessionEvent, entry)
  return event
}
```

要点：

- **类型层强制表面元数据**：`...opts` 的条件元组让非表面事件（如 `turn/start`）在编译期就无法传 `surfaceOp`；表面事件则必须传。
- **快照语义**：`snapshotJsonValue` 一次遍历完成读取、验证、拷贝——有状态 getter 不可能给验证和存储提供两个不同的值。返回的 `event.data` 是进入日志的快照，不是调用方仍可变的输入。
- **现场失败**：日志是持久事实，坏事件在 append 现场抛出，而不是等到后端 flush 时才炸。
- **写入门槛**：`publicationEntry()` 要求 store 内的会话已完成 `session/created` 宣布才可写，并拒绝在发布边界内重入 append。

### 3. `Session.commit()` — 先验证，后提交

```typescript
private commit(event: SessionEvent, entry: SessionEntry | undefined): void {
  this.surfaceManager.validateNext(event)

  if (entry !== undefined) entry.appending = true
  try {
    const callbackArgs: unknown[] = [this, event]
    const callbacks = entry === undefined
      ? undefined
      : collectSessionCallbacks(entry.emitCtx, [entry.carrier, 'session/event', ...callbackArgs])
    this.log.push(event)
    this.eventsSnapshot = undefined
    if (entry !== undefined && callbacks !== undefined) {
      invokeContainedSessionObservers(entry.emitCtx, 'session/event', entry.id, callbackArgs, callbacks)
    }
  } finally {
    if (entry !== undefined) {
      entry.appending = false
      if (entry.detachRequested && entry.announcement !== 'dispatching') entry.detach()
    }
  }
}
```

表面候选**先经 `SurfaceManager.validateNext` 验证**（不改动已提交状态），监听器快照在日志 push 前解析、回调在 push 后运行。事件一旦入栈即视为已提交。

### 4. `Session.deriveMessages()` — 从日志派生模型历史

```typescript
deriveMessages(): Message[] {
  const surface = this.surface
  const nodes = surface.nodes
  const generation = surface.contentGeneration
  if (generation !== this.derivedGeneration) {
    this.derived = []
    this.derivedNodes = 0
    this.derivedGeneration = generation
  }
  for (const seq of nodes.slice(this.derivedNodes)) {
    // Surface sequences are built from this.log — seq is always a valid
    // index by construction. The non-null assertion expresses that invariant.
    // oxlint-disable-next-line typescript/no-non-null-assertion
    const msg = this.deriveEventMessage(this.log[seq]!)
    // A surface node is one of the five message-producing types, but an
    // empty-content assistant/message (a max-tokens step that hosts only
    // usage) derives to null and must not enter the transcript.
    if (msg) this.derived.push(msg)
  }
  this.derivedNodes = nodes.length
  return [...this.derived]
}
```

派生完全由**表面**驱动：表面节点是消息产生型事件在模型可见顺序下的 seq 列表；`replace` 操作（compaction）把被遮蔽节点从派生中剔除，但原始日志记录从不删除。缓存策略：纯尾部增长只派生新节点（O(新增)）；`contentGeneration` 变化（位置替换或插件消息投影）触发整表重建。返回值每次是新数组，其中 `Message` 对象**共享且深冻结**，调用方无法通过任何一条路径改写日志。

### 5. `SessionStore` — `prepare` / `enter` / `announce` 三段式

```typescript
enter(session: Session): () => void {
  const id = session.id
  const carrier = scopeTarget(session, scopeOf(this.ctx))
  if (this.store.has(id)) throw new Error(`session "${id}" already exists`)
  if (attachments.get(session) != null) throw new Error(`session "${id}" is already attached to a store`)
  const entry: SessionEntry = { /* id, session, carrier, emitCtx, announcement: 'pending', … */ }
  this.store.set(id, entry)
  attachments.set(session, entry)
  // …返回 detach：若正在创建派发或 append 发布中，则登记 detachRequested 延迟执行
}
```

- `prepare()`：只构造（验证 id/cwd、构建 `SessionHeader`），不进 store；对象置 `null` 附件（store 持有但只读）。
- `enter()`：安装模块私有的 append 发布钩子并加入 store；返回 **detach disposer**。重复 id 在这里是权威冲突边界——陈旧的 prepared 会话绝不能覆盖同 id 的存活条目。
- `announce()`：与 `enter` 分离，让调用方先在 effect 中 yield detach（**回滚安全**：同步 `session/created` 监听器抛出时，生成器 effect 会处置已 yield 的 disposer，而不是泄漏 store 条目）。宣布成功前会话只读；失败的宣布不可重试。
- `create()` 是便捷路径：`prepare` + 单个 `ctx.effect`（yield enter → announce）。
- `fork()`：见 [fork.md](fork.md)；`flush()` 是唯一的持久化检查点入口——store 拥有 carrier，调用方必须经由它而不是自己派发原始 `session/flush`。

## 核心类与函数（index.ts）

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `Session` | `class`，普通类 | 事件溯源会话：append-only 日志 + 表面投影 + 各类增量 fold + 派生消息缓存 |
| `Session.create` | `static create(id, seed?, header?, inheritedEventCount?, projections?)` | 分离式创建：验证并快照借来的 seed/header |
| `Session.fromRestore` | `static fromRestore(id, seed, header, inheritedEventCount, eventState, projections?)` | 恢复式创建：采纳独立持有或深冻结的 seed，不复制不冻结 |
| `Session.append` | `append<T>(type, data, ...opts)` | 唯一写入口：快照 → 验证 → 冻结 → 提交 → 通知 |
| `Session.deriveMessages` | `(): Message[]` | 表面驱动的模型历史派生（缓存、深冻结） |
| `Session.requestHeader / requestContext / toolHistory` | 增量 fold 读取 | 分别重建请求信封、路由元数据、历史工具声明 |
| `Session.seq / firstLiveSeq / firstLifecycleSeq / inheritedEventCount` | 只读偏移 | 日志长度、构造 seed 长度、生命周期起点、fork 继承前缀长度 |
| `SessionStore` | `extends Service` | `ctx.sessions`：create/prepare/enter/announce/fork/flush/get/list + registerMessageProjection |
| `appendPluginRecord` | `(session, type, data)` | 写实验性 `plugin:` 记录（恒 `ignorable`，仅 `packages/experimental/` 允许调用） |
| `pluginRecordOf` | `(event): PluginRecord \| undefined` | 把 ignorable 的 `plugin:` 事件读回为记录（`data` 保持 `unknown`，由 owner 校验） |
| `adoptSessionEvent / snapshotSessionEvent` | 事件采纳/快照 | 跨持久化、查询边界采纳事件：验证事件局部形状并深冻结 identified message |

## 与其他模块的关系

- **`agent-loop`（packages/core/agent-loop）**：最大的写入方与读取方。每个循环实例边界与每次变更写 `request/header`（用 `canonicalHeader`/`headerEquals` 判等去重），每次 turn/step/tool 尝试写对应事件；组装模型输入时调 `session.deriveMessages()`、`session.toolHistory()`、`session.requestHeader()`（见 `packages/core/agent-loop/src/agent.ts`）。崩溃恢复用 `interruptedTurnClosers`/`ToolCallRecovery`，fork 用 `buildForkSeed`。此外有 `dsh-session-checkpoint-policy` 决定每请求的持久化检查点（调 `ctx.sessions.flush`）。
- **`session-persistence` / `session-persistence-jsonl`（packages/session/）**：订阅 `session/event` 异步缓冲、在 `session/flush` 落盘；格式迁移链（`session-format-v0-to-v1` … `v3-to-v4`）用 `encodeSeqRanges`/`decodeSeqRanges` 无损搬移 `sourceEventSeqs`。
- **`session-query` / Web 客户端**：冷读路径用 `foldSurface(events, projections)` + `deriveEventMessage` 精确重放派生规则；客户端 UI 直接消费 browser-safe 的 `@deepseek-ai/dsh-session/surface` 子路径（`isAppendSurfaceEvent` 等——人类 transcript 读 append-origin 事件而非表面，因为表面会遮蔽被替换区间）。
- **`compaction-*`**：`replace` 表面操作的第一个生产者；`compaction-image-offload` 注册 `SessionMessageProjection` 解释 `image/offload`。
- **`typert` 协议**：`Session` 注册为 `session` 资源，`SessionId` 作为线上字段。

## 推荐阅读顺序

1. [types.md](types.md) — 事件词汇表与信封（一切的地基）
2. [known-event-types.md](known-event-types.md) — 构建级已知类型清单与 required-on-read 的落地
3. [surface.md](surface.md) — 有序表面、replace 校验、`deriveEventMessage`（理解"日志如何变成模型历史"）
4. 本页（index.ts）— `Session` / `SessionStore` 生命周期与 append 提交路径
5. [request-header.md](request-header.md)、[tool-history.md](tool-history.md) — 请求可重建性与动态工具历史
6. [fork.md](fork.md)、[repair.md](repair.md) — fork/seed 与中断尾部修复
7. [preparation.md](preparation.md)、[seq-ranges.md](seq-ranges.md) — 两个小工具件

## 配置与扩展点

- **扩展事件词汇**：declaration merging `SessionEventMap`（log-only 事件）；消息内容变更类事件须在 JSDoc 标 `@messageProjection` 并经 `ctx.sessions.registerMessageProjection()` 注册纯解释器——缺定义时 append、restore、fold 全部拒绝；卸载已使用的定义会让缓存读取失败。
- **扩展回合结束原因**：merge `TurnEndReasonMap`（闭合可扩展 union，`switch` 落 `default` 而非 `assertNever`）。
- **实验性插件记录**：在 owner 包 augment `PluginRecordMap`，用 `appendPluginRecord` 写、`pluginRecordOf` 读。
- **无配置字段**：本包没有可调 Config；持久化策略、检查点时机都由 sibling 插件（persistence、checkpoint-policy）决定。

## 小结

`dsh-session` 用一个"只追加、可重建、一次验证"的日志统一了会话的全部事实：模型历史是 `surfaceOp` 标记驱动下的派生视图，请求信封是最新 header 快照的 fold，fork 是"前缀 + 标记 + 合成收尾"，崩溃是"未闭合尾部的合成收尾"。理解了 required-on-read 的 `ignorable` 信封与 `seq = log.length` 连续性契约，就掌握了这个模块与世界对话的方式。
