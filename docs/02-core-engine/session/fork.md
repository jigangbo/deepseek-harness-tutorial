# `fork.ts` — fork seed 构造：复制前缀、标记继承切点、闭合开口尾部

> 源码：`packages/core/session/src/fork.ts`（30 行，模块注释："Fork seed construction over an exact source-event prefix."）整个文件只有一个函数 `buildForkSeed`——但它是 fork/seed 语义的核心。

## 文件概览

fork（分支会话）的产物是一个**种子**：源日志某个包含式前缀的拷贝，加上子会话自有的合成事件。`buildForkSeed(events, boundary)` 生成这个种子数组，之后交给 `SessionStore.create()`（以 `isSeeded: true` + 精确的 `inheritedEventCount` 元数据）构造出活的子会话。

它的三条职责：

1. **复制包含式前缀** `[0, boundary]`；
2. **在切点放置 `{ inherited: true }` 的 `session/end-seed` 标记**——分隔"继承来的历史"与"子会话自有工作"；
3. **闭合源会话在切点处仍开着的尾部**：补缺失的错误 tool result，再用 `forked` 原因收尾 step/turn。**已闭合的 step/turn 原样保留**，包括历史上早已缺失的结果（不回溯修复）。

## 关键代码走读

### 1. `buildForkSeed` 全文

```typescript
export function buildForkSeed(events: readonly SessionEvent[], boundary: SessionSeqType): SessionEvent[] {
  const prefix = events.slice(0, boundary + 1)
  prefix.push({
    type: 'session/end-seed', seq: SessionSeq(boundary + 1),
    // oxlint-disable-next-line typescript/no-non-null-assertion -- the caller validated this exact source event.
    time: events[boundary]!.time,
    data: { inherited: true },
  })
  return prefix.concat(openTurnClosers(prefix, { kind: 'forked' }))
}
```

逐行解读：

- `events.slice(0, boundary + 1)`：**包含式**边界——boundary 指向的最后一个事件也被复制。种子保留**源事件对象本身**（引用共享）；随后 `Session` 构造器会对借来的事件做一次 lossless-JSON 快照（`snapshotJsonValue`）再发布，所以发布后子会话与源日志互不影响。
- 合成标记放在 `boundary + 1` 这个 seq（正好接在前缀之后），**时间戳复用边界事件的时间**——合成的 `seq`/`time` 规则贯穿整个 fork/repair 体系：序号接着日志数，时间不发明"未来"。
- `data: { inherited: true }`：**带标记的切点**。读取方通过"最后一个 `inherited: true` 标记"定位当前 fork 的继承边界，并由此推导 `inheritedEventCount`。即使前缀本身以某个祖先的标记结尾，子会话仍拥有自己的这一个标记（嵌套 fork 各有切点）。
- `prefix.concat(openTurnClosers(prefix, { kind: 'forked' }))`：把前缀（连同刚放入的标记）交给 repair 模块的 `openTurnClosers` 扫描，用 **`forked`** 原因生成合成收尾。合成事件排在标记**之后**——它们属于子会话，**不计入** `inheritedEventCount`。

### 2. caller 的契约与 `inheritedEventCount` 的语义

JSDoc 声明了两个前置条件："The caller validates that the boundary is an existing contiguous event seq; Session construction snapshots the borrowed events before publication." 也就是说：

- **边界验证在调用方**：`SessionStore.fork()` 的私有 `_forkBoundary` 检查 boundary 是非负 safe integer、小于日志长度、且与连续 seq 精确匹配（`events[boundary].seq === boundary`），不满足抛 `SessionForkError('INVALID_BOUNDARY')`。`buildForkSeed` 因此才敢用 `events[boundary]!`。
- **计数语义**：`SessionStore.fork` 随后以 `inheritedEventCount: SessionLogOffset(resolved + 1)` 创建子会话——**只数复制的源事件，不含标记与合成收尾**。`Session.inheritedEventCount` 由此保持"精确的继承前缀长度"，`ownEvents()`、`isOwnSeq()` 都以它为分界。
- **空源 fork 空子**：boundary 省略且源为空时，`_forkBoundary` 返回 undefined，`fork()` 以空 seed 创建子会话（`inheritedEventCount: 0`）。

### 3. 一个 fork seed 的实际形状

假设源日志在 `turn/start`(seq 5) → `assistant/message`(seq 6，请求调用 A) → `tool/call`(seq 7) 处被 fork（boundary=7）：

```
seq 0..7   复制的源前缀（已闭合的历史原样保留）
seq 8      session/end-seed { inherited: true }      ← 继承切点（含在标记里）
seq 9      tool/result (TOOL_NOT_STARTED 文案)        ← 子会话自有（forked 措辞）
seq 10     step/end                                   ← 闭合开着的 step
seq 11     turn/end { reason: { kind: 'forked' } }    ← 闭合开着的 turn
```

`inheritedEventCount = 8`（seq 0..7）；子会话的"自有"历史从标记开始。模型在子会话里看到的是一份自洽历史：没有悬空的工具调用，开着的回合有明确的 `forked` 结束原因。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `buildForkSeed` | `(events, boundary) => SessionEvent[]` | 复制包含式前缀 → 追加 `{ inherited: true }` 标记 → 以 `forked` 原因合成开口尾部的 tool result 与 step/turn 收尾；返回新数组 |

### 4. `Session` 构造器如何消费这个 seed

`buildForkSeed` 的产物最终进入 `Session.create()`（快照模式）。构造器对 fork seed 的验收规则（`packages/core/session/src/index.ts`，构造器尾部）：

```typescript
const seedMarker = this.log[inheritedEventCount]
const markedSeed = seedMarker?.type === 'session/end-seed' && seedMarker.data.inherited === true
if (mode === 'snapshot' && this.header.isSeeded && inheritedEventCount !== this.log.length && !markedSeed) {
  throw new Error('seeded session constructor seed must equal its inherited prefix or mark its inherited cut')
}
if (markedSeed && this.log.slice(inheritedEventCount + 1).some(event => event.type === 'session/end-seed' && event.data.inherited === true)) {
  throw new Error('session inherited event count must identify the final inherited marker')
}
```

两种合法的 seed 形态：

- **只有继承前缀**（seed 长度 == `inheritedEventCount`，无标记）：构造器自己补一个 `{ inherited: true }` 标记（"the `Session` constructor supplies it when given only the inherited prefix"）。
- **已带标记 + 子会话收尾**（`buildForkSeed` 的完整产物）：构造器接受 `inheritedEventCount` 指向的标记，并验证它之后**再无**第二个带标记的 marker——`inheritedEventCount` 必须指向**最后一个**继承标记。

不满足即抛错，绝不猜测。这保证了 `Session.inheritedEventCount`、`ownEvents()`、`isOwnSeq()` 的分界永远与日志内的标记一致。

### 5. fork 错误码

`SessionStore.fork` 的拒绝路径使用类型化错误 `SessionForkError`（`index.ts`），四个代码对应四类失败：

| 代码 | 触发条件 |
|---|---|
| `SESSION_NOT_FOUND` | 源 id 不在活 store 中；或传入对象查不到同 id 活会话 |
| `SESSION_NOT_LIVE` | 传入了 Session 对象，但它不是 store 里的存活实例（陈旧引用） |
| `SESSION_ALREADY_EXISTS` | 请求的 `childSessionId` 已被占用 |
| `INVALID_BOUNDARY` | 边界不是非负 safe integer、超出日志长度、或与连续 seq 不匹配 |

## 与其他模块的关系

- **`repair.ts`**：`buildForkSeed` 直接复用 `openTurnClosers`，只是把 cause 换成 `{ kind: 'forked' }`——错误结果文案（"父会话可能在 fork 点之后执行了它"）与 `turn/end` 原因都由 cause 决定；错误码（`TOOL_NOT_STARTED`/`TOOL_OUTCOME_UNKNOWN`）两系共用，因为它们陈述的是关于调用记录周期的同一事实。
- **`index.ts` 的 `SessionStore.fork()`**：解析 source（活对象或活 id）→ `_forkBoundary` 验证边界 → `buildForkSeed(events, boundary)` → `create(childSessionId, { seed, inheritedEventCount, meta: { cwd, isSeeded: true } })`。子会话继承 `cwd`，**不设置** `parentSession`（普通 fork 不记录来源关系；`parentSession` 只给 subagent 用，分类看 `origin`）。
- **`types.ts`**：`session/end-seed` 事件的 JSDoc 明确"只有 `Session` 构造器与 `buildForkSeed` 可以创建该标记"；`Session.append` 并不拒绝其他写入者——插件乱写一个会悄悄把之前的活工作全部错分类为 seed 历史，这是文档化的禁区。
- **消费方**：`dsh-session-fork` 相关路径（`session-format-v3-to-v4` 的 fork 测试、`session-controller` 的远程 fork 命令）都以本函数的产物为种子契约。

## 小结

`fork.ts` 用 30 行实现了 fork 的全部语义：**前缀复制 + 切点标记 + forked 收尾**。两个不可妥协的细节——标记必须在任何合成收尾**之前**（`inheritedEventCount` 数的是标记之前的前缀），合成收尾**只处理开口尾部**（闭合的 step/turn 里的历史缺失结果不修复，fork 生成结果的措辞也只描述继承前缀里的事实："父会话可能……"）。
