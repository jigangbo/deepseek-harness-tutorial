# `assistant-stream.ts` — 流式记录嵌入
> 紧凑流记录:无损压缩、严格校验、免展开读取


> 源码:`packages/llm/llm/src/assistant-stream.ts`(约 530 行)。一个模型流尝试的**无损紧凑表示**(`AssistantStreamRecord[]`),加上一组直接在记录上作答的读取器——telemetry、token 计量、Host 折叠都不必把流展开成逐成员 chunk。

## 文件概览

为什么需要这个文件:一个长响应可能产生数千个 token 级 chunk,每个都带事件信封。持久层若存原始 `TimedStreamChunk[]`,一个 assistant/attempt 事件就是一大坨;而消费者最常问的问题("首个 token 几点?"、"有没有可见内容?")其实一两个记录就能回答。

三条使用路径:

- **写入**:`AssistantStreamAccumulator.push()` 逐个吃 `{time, chunk}`,把连续同块同型的 delta 打包成一条 run 记录;`snapshot()` 产出可嵌入持久事件的脱离不可变记录列表。
- **校验展开**:`expandAssistantStream()` 严格校验后重建**精确的**带时间戳 chunk 序列——durable 边界读取记录时的验证路径。
- **免展开读取**:`assistantStreamFirstTokenTime`、`assistantStreamHasVisibleContent` 等在记录上一遍扫过、提前退出——大历史的成本是 O(记录数) 而不是 O(成员数)。类头 JSDoc 定调:读取器**信任静态记录类型**;durable 边界才走 `expandAssistantStream` 验证。

## 关键代码走读

### 记录形态:打包 delta run + 原始 chunk

```typescript
// packages/llm/llm/src/assistant-stream.ts
export interface TimedStreamChunk {
  readonly time: number
  readonly chunk: StreamChunk
}

export type AssistantStreamRecord =
  | {
    readonly type: 'text-chunks'
    readonly time0: number
    readonly index: number
    readonly dt: readonly number[]
    readonly texts: readonly string[]
  }
  | {
    readonly type: 'reasoning-chunks'
    readonly time0: number
    readonly index: number
    readonly dt: readonly number[]
    readonly texts: readonly string[]
  }
  | {
    readonly type: 'tool-call-chunks'
    readonly time0: number
    readonly index: number
    readonly dt: readonly number[]
    readonly id: ToolCallId
    readonly name?: string
    readonly args: readonly string[]
  }
  | { readonly type: 'chunk'; readonly time: number; readonly chunk: StreamChunk }

export type AssistantStreamRun = Exclude<AssistantStreamRecord, { type: 'chunk' }>
```

一条 run 记录的语义:**首成员时间 `time0` + 逐成员时间差 `dt`(长度 = 成员数 − 1)+ 成员数组**。时间戳关系精确保留(`time[i+1] = time[i] + dt[i]`),token 边界一个不丢(每个 delta 是数组的独立条目)。控制 chunk(block-start/end、usage、finish)从不打包,永远是一条 `chunk` 原始记录。

### Accumulator.push:打包判定

```typescript
// packages/llm/llm/src/assistant-stream.ts
push(value: TimedStreamChunk): TimedStreamChunk {
  const time = safeTime(value.time)
  const chunk = snapshotChunk(value.chunk)
  const timed = deepFreeze({ time, chunk })
  const previous = this.records.at(-1)
  switch (chunk.type) {
    case 'text-delta':
    case 'reasoning-delta': {
      safeIndex(chunk.index, chunk.type)
      if (typeof chunk.text !== 'string') throw new TypeError(`${chunk.type} text must be a string`)
      const type = chunk.type === 'text-delta' ? 'text-chunks' : 'reasoning-chunks'
      const gap = previous !== undefined && previous.type === type ? safeGap(previous.lastTime, time) : undefined
      if (previous !== undefined && previous.type === type && previous.index === chunk.index && gap !== undefined) {
        previous.dt.push(gap)
        previous.texts.push(chunk.text)
        previous.lastTime = time
      } else {
        this.records.push({ type, time0: time, index: chunk.index, dt: [], texts: [chunk.text], lastTime: time })
      }
      return timed
    }
    /* tool-call-delta 分支见下 */
    case 'block-start':
    case 'block-end':
    case 'usage':
    case 'finish':
      this.records.push({ type: 'chunk', time, chunk })
      return timed
    default:
      return assertNever(chunk, 'AssistantStreamAccumulator.push')
  }
}
```

打包条件(文本/思考):

1. 上一条记录**就是**当前 run 尾部(`this.records.at(-1)`,中间不能隔着别的记录);
2. 类型相同(`text-chunks` 与 `reasoning-chunks` 不互并)且 `index` 相同;
3. `safeGap` 能无损表示时间差:`previous + gap === next` 且 gap 是安全整数——时间回退或溢出就断开 run 另起一条。

`safeTime`/`safeIndex` 是入口校验(时间必须是安全整数;index 必须非负安全整数且非 -0);`snapshotChunk` 经 `snapshotJsonValue` 做**无损 JSON 快照**——不可序列化的 chunk 在写入时就报 TypeError,而不是等持久化时炸。

tool-call 打包更挑剔:

```typescript
// packages/llm/llm/src/assistant-stream.ts
case 'tool-call-delta': {
  safeIndex(chunk.index, chunk.type)
  if (typeof chunk.id !== 'string') throw new TypeError('tool-call-delta id must be a string')
  if (Object.hasOwn(chunk, 'name') && typeof chunk.name !== 'string') {
    throw new TypeError('tool-call-delta name must be a string')
  }
  if (typeof chunk.argumentsDelta !== 'string') {
    throw new TypeError('tool-call-delta argumentsDelta must be a string')
  }
  if (chunk.id.length === 0 || chunk.name === '') {
    this.records.push({ type: 'chunk', time, chunk })   // 畸形 id/name 不打包,原样成原始记录
    return timed
  }
  const gap = previous?.type === 'tool-call-chunks' ? safeGap(previous.lastTime, time) : undefined
  const sameName = previous?.type === 'tool-call-chunks'
    && Object.hasOwn(previous, 'name') === Object.hasOwn(chunk, 'name')
    && previous.name === chunk.name
  if (previous?.type === 'tool-call-chunks'
    && previous.index === chunk.index
    && previous.id === chunk.id
    && sameName
    && gap !== undefined) {
    previous.dt.push(gap)
    previous.args.push(chunk.argumentsDelta)
    previous.lastTime = time
  } else { /* 另起一条 tool-call-chunks */ }
  return timed
}
```

额外条件:**id 相同**且 **name 的存在性与值完全一致**(`Object.hasOwn` 区分"没给 name"和"给了空串";空 id / 空 name 的畸形 chunk 直接落到原始 `chunk` 记录,不参与打包)。

### snapshot:脱离不可变快照

```typescript
// packages/llm/llm/src/assistant-stream.ts
snapshot(): readonly AssistantStreamRecord[] {
  const records = this.records.map((record): AssistantStreamRecord => {
    if (record.type === 'chunk') return { ...record }
    const { lastTime: _lastTime, ...durable } = record
    if (durable.type === 'tool-call-chunks') {
      return { ...durable, dt: [...durable.dt], args: [...durable.args] }
    }
    return { ...durable, dt: [...durable.dt], texts: [...durable.texts] }
  })
  return deepFreeze(records)
}
```

内部 `MutableRecord` 带 `lastTime` 工作字段(打包含并的游标);快照把它剥掉、拷贝数组、整体深冻结——**累积器可以继续 push 而快照不受影响**,这正是"边流边发布、结束后落盘"的用法。

### expandAssistantStream:精确重建

```typescript
// packages/llm/llm/src/assistant-stream.ts
export function expandAssistantStream(stream: readonly AssistantStreamRecord[]): readonly TimedStreamChunk[] {
  const chunks: TimedStreamChunk[] = []
  for (const candidate of stream) {
    const record = validateRecord(candidate)
    if (record.type === 'chunk') {
      chunks.push({ time: record.time, chunk: record.chunk })
      continue
    }
    const members = record.type === 'tool-call-chunks' ? record.args : record.texts
    let time = record.time0
    for (let index = 0; index < members.length; index += 1) {
      if (index > 0) time += record.dt[index - 1] as number
      let chunk: StreamChunk
      if (record.type === 'text-chunks') {
        chunk = { type: 'text-delta', index: record.index, text: members[index] as string }
      } else if (record.type === 'reasoning-chunks') {
        chunk = { type: 'reasoning-delta', index: record.index, text: members[index] as string }
      } else {
        chunk = {
          type: 'tool-call-delta',
          index: record.index,
          id: record.id,
          ...Object.hasOwn(record, 'name') ? { name: record.name } : {},
          argumentsDelta: members[index] as string,
        }
      }
      chunks.push({ time, chunk })
    }
  }
  return chunks
}
```

重建顺序:逐记录校验(`validateRecord`),run 记录按 `time0 + Σdt` 逐成员重放时间,恢复出的 chunk 形态与打包前**逐字段一致**(包括 name 是否存在)。校验器 `validateRecord` 值得细看:

- **exactKeys**:每个 record 变体的键集合精确匹配(不多不少)——未知键是未来格式的嫌疑,直接拒绝;
- `texts`/`args` 必须非空字符串数组;`dt` 长度必须等于成员数 − 1(`validateRun`),且逐成员累加后仍是安全整数(重放出的时间戳不能溢出);
- `tool-call-chunks` 的 id/name 非空;`chunk` 记录的 chunk 必须能过 `snapshotChunk` 无损 JSON;
- 校验产物深冻结。**校验失败抛 TypeError**——durable 边界上的记录要么完全可信要么拒收,没有降级读取。

### 免展开读取器

先看单 chunk 语义的三个判定,它们也是读取器的语义锚点:

```typescript
// packages/llm/llm/src/assistant-stream.ts
export function isTokenDelta(chunk: StreamChunk): boolean {
  switch (chunk.type) {
    case 'text-delta':
    case 'reasoning-delta':
      return chunk.text !== ''
    case 'tool-call-delta':
      return chunk.argumentsDelta !== '' || chunk.name !== undefined
    default:
      return false
  }
}

export function isVisibleChunk(chunk: StreamChunk): boolean {
  switch (chunk.type) {
    case 'text-delta':
    case 'reasoning-delta':
      return hasNonWhitespace(chunk.text)
    case 'block-start':
      return chunk.blockType !== 'text' && chunk.blockType !== 'reasoning' && chunk.blockType !== 'tool-call'
    case 'block-end':
      return blockIsVisible(chunk.block)
    default:
      return false
  }
}

export function chunkHasVisibleText(chunk: StreamChunk): boolean {
  if (chunk.type === 'text-delta') return hasNonWhitespace(chunk.text)
  return chunk.type === 'block-end' && chunk.block.type === 'text' && hasNonWhitespace(chunk.block.text)
}
```

- **isTokenDelta**(首 token 延迟测量):非空文本/思考增量、非空参数增量或带 name 的工具增量都算"模型吐出了第一个 token";
- **isVisibleChunk**(转录读者可见性):文本/思考要有非空白;块级 chunk 中 text/reasoning/tool-call 的 block-start 不算(它们的可见性由成员 delta 体现),tool-call 整体是协议而非内容;usage/finish 永不;
- **chunkHasVisibleText**(可见文本):只认 text-delta 与闭合 text 块,思考/工具不算。

run 级与流级读取器全部提前退出:

```typescript
// packages/llm/llm/src/assistant-stream.ts
export function runFirstTokenTime(run: AssistantStreamRun): number | undefined {
  if (run.type === 'tool-call-chunks' && run.name !== undefined) return run.time0
  return firstRunMemberTime(run, fragment => fragment !== '')
}

export function assistantStreamFirstTokenTime(stream: readonly AssistantStreamRecord[]): number | undefined {
  for (const record of stream) {
    const time = record.type === 'chunk'
      ? (isTokenDelta(record.chunk) ? record.time : undefined)
      : runFirstTokenTime(record)
    if (time !== undefined) return time
  }
  return undefined
}

export function assistantStreamHasVisibleContent(stream: readonly AssistantStreamRecord[]): boolean {
  return stream.some(record => record.type === 'chunk'
    ? isVisibleChunk(record.chunk)
    : runFirstVisibleTime(record) !== undefined)
}

export function assistantStreamHasVisibleText(stream: readonly AssistantStreamRecord[]): boolean {
  return stream.some(record => record.type === 'text-chunks'
    ? record.texts.some(hasNonWhitespace)
    : record.type === 'chunk' && chunkHasVisibleText(record.chunk))
}
```

`firstRunMemberTime` 在 run 内部也**扫到第一个合格成员就停**(时间按 dt 累加重放)。所以"整条 5000 token 的流找首 token 时间"最多扫到第一个非空成员。

### 免展开的其余读取器

| 函数 | 语义 |
|------|------|
| `lastAssistantStreamChunk(stream, type)` | 从后向前扫,返回某**从不打包的** chunk 类型的最后一条(`RawStreamChunkType` = 排除三个 delta 类型,泛型 `Extract` 保证返回类型精确) |
| `assistantStreamChunks(stream, type)` | 该类型全部原始 chunk,流序 |
| `joinAssistantStreamText(stream)` | 全部 text-delta 片段按流序拼接(思考/工具参数不算)——"整条回复说了什么" |
| `assembleAssistantStream(stream, assembler?)` | 喂给 BlockAssembler:**每个 run 一个合并 delta**(`texts.join('')` / `args.join('')`) |

`assembleAssistantStream` 的正确性论证写在 JSDoc:装配只做**字符串拼接**,所以"一条合并 delta"与"N 条原始 delta"产出完全相同的块;原始 chunk 按记录原样推入。注意它的入参被**信任而非校验**——durable 边界的流先过 `expandAssistantStream` 验证(顺带拿到全部成员),进程内已知良性的流可以直接喂它省一次展开。

## 嵌入与读取全景

```text
流式期间                          流结束后(持久)
────────────                     ────────────────────────────────────────
Accumulator.push({time,chunk})   snapshot() → AssistantStreamRecord[]
  连续同块 delta → run 记录          └→ 嵌入 assistant/message 或
  控制chunk   → chunk 记录              assistant/attempt 事件(durable)
  返回深冻结 TimedStreamChunk           │
  (供装配与实时发布)                    ├→ 进程内读者:record 级读取器
                                       │   firstTokenTime / hasVisibleContent /
                                       │   lastChunk / joinText / assemble(免展开)
                                       └→ durable 边界读者:expandAssistantStream
                                           (严格校验 → 精确 TimedStreamChunk[])
```

包 README 给出成本理由:紧凑流的保留成本约为展开形态的十分之一,且 `expandAssistantStream` **不做 memoize**——一个被保留的展开结果会在事件存活期间持续占着十倍内存。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `AssistantStreamAccumulator.push` | `({time,chunk}) => TimedStreamChunk` | 入口校验 + 无损快照 + 打包判定 |
| `AssistantStreamAccumulator.snapshot` | `() => AssistantStreamRecord[]` | 脱离、剥工作字段、深冻结的快照 |
| `expandAssistantStream` | `(records) => TimedStreamChunk[]` | 严格校验 + 精确重建(durable 路径) |
| `isTokenDelta` / `isVisibleChunk` / `chunkHasVisibleText` | `(chunk) => boolean` | 单 chunk 的 token/可见性/可见文本语义 |
| `runFirstTokenTime` / `runFirstVisibleTime` | `(run) => number \| undefined` | run 内首个合格成员的时间(提前退出) |
| `assistantStreamFirstTokenTime` | `(records) => number \| undefined` | 流级首 token 时间 |
| `assistantStreamHasVisibleContent` / `…HasVisibleText` | `(records) => boolean` | 流级可见性判定 |
| `lastAssistantStreamChunk` / `assistantStreamChunks` | `(records, type) => chunk(s)` | 从不打包类型的原始 chunk 读取 |
| `joinAssistantStreamText` | `(records) => string` | text-delta 片段流序拼接 |
| `assembleAssistantStream` | `(records, assembler?) => BlockAssembler` | run 合并喂装配器(信任记录) |

## 与其他模块的关系

- `assembler.ts`:`assembleAssistantStream` 的落点;两者合起来构成"记原始紧凑流 + 得装配结果"的完整回放保真方案。
- `types.ts`:`StreamChunk` 是全部记录的载荷;`ToolCallId` 来自 `brand.ts`。
- `@deepseek-ai/dsh-util-values`:`snapshotJsonValue`(无损 JSON 快照)、`deepFreeze`、`assertNever`。
- 消费侧:Session 在 `assistant/message` / `assistant/attempt` 事件嵌入紧凑记录;`agent/assistant-stream` 进程内帧携带实时呈现;token meter 与 Host 折叠走 record 级读取器;SDK 两侧(session)投影同款词汇。

## 小结

- 紧凑表示丢掉的只有**重复的事件信封**:时间差数组 + 逐成员片段保住了每个 token 的边界与时刻,终态/usage/块/失败/回放事实一条不少。
- 打包条件苛刻(同型同 index、gap 可无损表示、tool-call 还要同 id 同 name),校验同样苛刻(exactKeys、dt 长度、安全整数)——**要么无损要么拒收**,没有模糊地带。
- 读取器的共同设计:在记录(而非成员)上扫描、语义锚点复用单 chunk 判定、扫到即停。大历史的每个问题都是 O(记录数)。
