# `assembler.ts` — BlockAssembler:增量把流折回消息

> 源码:`packages/llm/llm/src/assembler.ts`(约 210 行,但语义密度极高)。`BlockAssembler` 是**唯一的规范装配算法**:agent loop 一边把原始 chunk 记进日志(回放保真),一边喂给同一个 assembler,流结束后读出内容块、usage、finish 与回放元数据,组装成持久 assistant 消息。

## 文件概览

一次流式响应是交错的 delta(文本、思考、多个工具调用)加上少量控制 chunk。装配器要解决三个问题:

1. **交错归位**——`index` 关联的 delta 如何归到各自的块;
2. **容错**——畸形流(闭块后还有 delta、重复 block-end)不能污染结果或涨内存;
3. **内容与元数据一致**——装配丢弃的块(截断的工具调用)必须同步从回放信封里丢弃,两者永远不能打架。

状态只有五样:`partials`(index → 半成品块)、`order`(首次出现顺序)、`_usage`、`_finish`、`_replayState`。全部私有,读取都走方法。

## 公开 API 全貌

消费者视角的完整接口(`docs/subsystems/llm-streaming.md` 的 BlockAssembler 一节收录了同款签名):

```typescript
// packages/llm/llm/src/assembler.ts(签名摘要)
declare class BlockAssembler {
  push(chunk: StreamChunk): void                    // 喂一个 chunk,流序
  blocks(): ContentBlock[]                          // 全部块;max-tokens 丢工具调用;开放未知块抛错
  interruptedBlocks(): ContentBlock[]               // 中断可安全固化的前缀
  get usage(): TokenUsage | undefined               // usage chunk 的值
  get finish(): FinishReason                        // 终止原因;缺省 {kind:'stop'}
  get replayState(): ReplayEnvelope | undefined     // 与 blocks 同步修剪的回放信封
  message(source: Omit<ModelMessageSource, 'kind'>): AssistantMessage  // 冻结的 assistant 消息
}
```

注意 `blocks()` / `replayState` 是**读取时装配**(每次调用重算),不是 push 时增量维护——这让 keep/drop 决策永远基于最终的 `finish`,流中途调用它们看到的也只是当时的累积状态。`message(source)` 的 `source` 不带 `kind` 标签:工厂 `createAssistantMessage` 会补上 `kind:'model'`。

## 关键代码走读

### 半成品块与 push 主循环

```typescript
// packages/llm/llm/src/assembler.ts
interface PartialBlock {
  blockType: string
  text: string
  toolCallId?: ToolCallId
  toolCallName?: string
  toolCallArguments: string
  /** Set by `block-end` — authoritative, and freezes the partial. */
  block?: ContentBlock
}

export class BlockAssembler {
  private partials = new Map<number, PartialBlock>()
  private order: number[] = []
  private _usage: TokenUsage | undefined
  private _finish: FinishReason | undefined
  private _replayState: ReplayEnvelope | undefined

  push(chunk: StreamChunk): void {
    switch (chunk.type) {
      case 'block-start': {
        if (!this.partials.has(chunk.index)) {
          this.order.push(chunk.index)
          this.partials.set(chunk.index, {
            blockType: chunk.blockType,
            text: '',
            toolCallArguments: '',
          })
        }
        return
      }
      case 'text-delta':
      case 'reasoning-delta': {
        const partial = this.ensure(chunk.index, chunk.type === 'text-delta' ? 'text' : 'reasoning')
        if (partial.block) return // closed by block-end; ignore stragglers
        partial.text += chunk.text
        return
      }
      case 'tool-call-delta': {
        const partial = this.ensure(chunk.index, 'tool-call')
        if (partial.block) return // closed by block-end; ignore stragglers
        partial.toolCallId = chunk.id
        if (chunk.name) partial.toolCallName = chunk.name
        partial.toolCallArguments += chunk.argumentsDelta
        return
      }
      case 'block-end': {
        const partial = this.ensure(chunk.index, chunk.block.type)
        // First close wins; ignoring re-close stragglers keeps streamed output
        // and the final assembled block in agreement.
        if (partial.block) return
        partial.block = chunk.block
        return
      }
      case 'usage': {
        this._usage = chunk.usage
        return
      }
      case 'finish': {
        this._finish = chunk.reason
        this._replayState = chunk.replayState
        return
      }
      default: return assertNever(chunk, 'BlockAssembler.push')
    }
  }
```

逐段看:

- **`block-start` 只在首次出现时登记**(顺序 + 半成品),重复的 start 无害;
- **delta 三态共用"闭块即忽略"守卫**:`partial.block` 已被 `block-end` 设置后,后续同 index 的 delta 直接丢弃——畸形适配器既不能改写已完成的块,也不能借 stragglers 涨内存;
- **`block-end` 首次关闭获胜**(`block` 字段权威),重复 close 同样忽略——流式输出与最终装配结果始终一致;
- **`ensure(index, blockType)` 兼容 delta-only 协议**:没有 block-start 的适配器也能用,delta 到达时按需建半成品并推入 `order`;
- `switch` 以 `assertNever` 收尾——`StreamChunk` 封闭联合,新增变体此处编译失败。

### 装配单个半成品

```typescript
// packages/llm/llm/src/assembler.ts
private assemble(partial: PartialBlock, index: number): ContentBlock {
  if (partial.block) return partial.block
  switch (partial.blockType) {
    case 'text': return { type: 'text', text: partial.text }
    case 'reasoning': return { type: 'reasoning', text: partial.text }
    case 'tool-call': return {
      type: 'tool-call',
      id: partial.toolCallId ?? brandString<ToolCallId>(`call-${index}`),
      name: partial.toolCallName ?? '',
      arguments: partial.toolCallArguments,
    }
    default: throw new Error(`cannot assemble incomplete block of type "${partial.blockType}"`)
  }
}
```

- 有权威 `block` 直接用;否则从累积 delta 合成。
- **tool-call 缺 id 时合成 `call-${index}`**(mock/降级协议兜底),name 缺省为空串——参数保持原始拼接的 JSON 字符串。
- **未知块类型且从未被 `block-end` 关闭 → 抛错**。这不是防御过度:插件加的自定义块类型,其流永远不会被本装配器闭合,类 README 的 Known Limitations 明说 `blocks()` 会 throw——fail loud 优于默默造出一个错误块。

### assembled():唯一的 keep/drop 决策点

```typescript
// packages/llm/llm/src/assembler.ts
private assembled(): { blocks: ContentBlock[]; replay: ReplayEnvelope | undefined } {
  const all = this.order.map(index => this.assemble(this.mustGet(index), index))
  const kept = this.finish.kind === 'max-tokens'
    ? all.map(block => block.type !== 'tool-call')
    : undefined
  const blocks = kept === undefined ? all : all.filter((_, position) => kept[position])
  const envelope = this._replayState
  if (envelope?.blocks === undefined) return { blocks, replay: envelope }
  if (envelope.blocks.length !== all.length) return { blocks, replay: undefined }
  return {
    blocks,
    replay: kept === undefined || blocks.length === all.length
      ? envelope
      : { response: envelope.response, blocks: envelope.blocks.filter((_, position) => kept[position]) },
  }
}
```

这段是整个文件的核心契约,**一个决策覆盖内容与元数据**:

- **`max-tokens` 结束丢弃所有工具调用**——被截断的调用不安全,不可执行;文本/思考块保留;
- 同一个 `kept` 布尔数组同时用于过滤 `blocks` 与 `envelope.blocks`(按位置对齐),所以 `blocks()` 与 `replayState` **结构上不可能** disagree;
- **信封长度对不上就整个丢弃**(replay 置 undefined)——适配器的逐块元数据与实际发出的块数不一致时,宁可放弃回放状态也不存"描述错位内容的元数据";
- 无 `blocks` 字段的信封(适配器元数据与块结构无关)原样穿过,不受过滤影响。

### interruptedBlocks():取消时能安全收尾的前缀

```typescript
// packages/llm/llm/src/assembler.ts
interruptedBlocks(): ContentBlock[] {
  return this.order
    .map((index) => {
      const partial = this.mustGet(index)
      const type = partial.block?.type ?? partial.blockType
      if (type !== 'text' && type !== 'reasoning') return undefined
      return this.assemble(partial, index)
    })
    .filter((block): block is ContentBlock =>
      (block?.type === 'text' || block?.type === 'reasoning') && block.text.trim() !== '')
}
```

中断(用户取消)后装配器收到的不是 `finish` 而是流的提前终止。可安全固化的前缀:

- **只要 text/reasoning**(已关闭或开放中的都行)——工具调用**全部省略**,因为中断先于派发,留下一个调用就得伪造一个结果;
- **必须含非空白内容**(`trim() !== ''`)——纯空白增量不构成可见输出;
- 开放的未知类型块同样省略。

`mustGet` 是不变量访问器:`order` 里的每个 index 必须有半成品,违反即抛(内部不变量被破坏是 bug,不是可恢复状态)。

### 读取器与 message()

```typescript
// packages/llm/llm/src/assembler.ts
get usage(): TokenUsage | undefined {
  return this._usage
}

get finish(): FinishReason {
  return this._finish ?? { kind: 'stop' }
}

get replayState(): ReplayEnvelope | undefined {
  return this.assembled().replay
}

message(source: Omit<ModelMessageSource, 'kind'>): AssistantMessage {
  return createAssistantMessage({ content: this.blocks(), source })
}
```

- `finish` 缺省为 `{kind:'stop'}`:流没发 finish 就结束(如适配器生成器自然耗尽)按正常完成算——但注意配合 `adapterStream` 的归一化,失败路径总会送来显式 finish。
- `message(source)` 只要求 provider/model(不带 `kind` 标签),内部补 `kind:'model'` 并经 `createAssistantMessage` 冻结——装配产物与所有持久消息同一条不可变构造路径。

## 一次完整装配的时间线

```text
adapter 流                BlockAssembler 状态                 流结束后的读取
──────────────────────   ─────────────────────────────       ─────────────────────
block-start(0,text)  →   order=[0] partials[0]=text ''
text-delta(0,"Hel")  →   partials[0].text="Hel"
tool-call-delta(1,id)→   order=[0,1] partials[1]=tool-call
tool-call-delta(1,Δ) →   partials[1].args+="…"
block-end(0,{text})  →   partials[0].block=权威 text 块
   (此后 index 0 的 delta 一律忽略)
usage(usage)         →   _usage=usage
finish({tool-calls}, →   _finish / _replayState
        replayState)
                         blocks()      → [text("Hello"), tool-call]
                         usage         → TokenUsage
                         finish        → {kind:'tool-calls'}
                         replayState   → 信封(blocks 与块按位置对齐)
                         message(src)  → 冻结 AssistantMessage
```

`max-tokens` 结束时:`blocks()` 里 index 1 的 tool-call 被丢掉,`replayState.blocks` 同步丢掉第 2 个条目——两个读取器看到的是同一次修剪。

## 边界情形清单

装配器对畸形与边界输入的行为全部有定义,没有"未定义行为"区:

| 情形 | 行为 | 依据 |
|------|------|------|
| `block-end` 之后同 index 又来 delta | 静默忽略(不改已完成块) | push 里的 `if (partial.block) return` |
| 同 index 第二次 `block-end` | 静默忽略(首关获胜) | 同上 |
| 没有 `block-start` 的 delta | 正常装配(`ensure` 按需建半成品) | delta-only 协议容忍 |
| `finish` 缺席、生成器自然耗尽 | `finish` 读作 `{kind:'stop'}` | getter 缺省 |
| `usage` 缺席 | `usage` 读作 `undefined` | getter 缺省 |
| 开放的未知块类型(如插件自定义块) | `blocks()` 抛 Error | assemble 的 default 分支 |
| max-tokens 结束 + 工具调用已部分流式 | 工具调用块整体丢弃 | assembled 的 keep/drop |
| 回放信封 `blocks` 长度 ≠ 发出块数 | 整个信封丢弃(replayState 为 undefined) | assembled 对齐检查 |
| 中断(无 finish) | 只有非空白 text/reasoning 可固化 | interruptedBlocks |

"静默忽略 stragglers"与"开放未知块抛错"看似矛盾,其实是一致的容错观:**流协议内的畸形**(重复、迟到、缺失)不影响已确定的输出;**词汇表外的块**意味着装配器无法给出正确结果,宁可 fail loud 也不产出错误块。

## 实例:max-tokens 修剪与回放信封对齐

设一个流发出了三个块:index 0 是文本、index 1 是工具调用、index 2 又是文本,最后 `finish {kind:'max-tokens'}` 且信封带两个逐块条目(适配器声明其元数据与块结构无关,`blocks` 长度恰为 3):

```text
order = [0, 1, 2]
partials:
  0: text  "Hello, I will check"           (closed by block-end)
  1: tool-call id=call_7 name=read_file    (closed by block-end)
  2: text  " the file."                    (open, delta-only)
finish: {kind:'max-tokens'}, replayState: {response:…, blocks:[b0,b1,b2]}
```

`assembled()` 的执行:

1. `all = [text0, tool-call1, text2]`;
2. `finish.kind === 'max-tokens'` → `kept = [true, false, true]`(工具调用不安全不可执行);
3. `blocks = [text0, text2]`;
4. 信封 `blocks.length(3) === all.length(3)`,对齐通过;`kept !== undefined` 且 `blocks.length !== all.length` → 修剪信封:`replay = {response:…, blocks:[b0,b2]}`。

于是 `message(source)` 产出的 assistant 消息只含两段文本;存储的逐块回放元数据也只剩两个条目——**存下的元数据永远描述存下的内容**。若适配器的信封声明了 2 个条目(与实际 3 块不符),整条信封被丢弃(`replayState` 为 undefined),消息照常装配:回放状态可以没有,不能错。

对比正常结束(`{kind:'tool-calls'}`):`kept === undefined`,`blocks` 全保留、信封原样——只有截断才触发修剪。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `BlockAssembler.push` | `(chunk: StreamChunk) => void` | 喂入一个 chunk;闭块后 delta/重复 close 忽略 |
| `BlockAssembler.blocks` | `() => ContentBlock[]` | 按流序装配全部块;max-tokens 丢工具调用;开放未知块抛错 |
| `BlockAssembler.interruptedBlocks` | `() => ContentBlock[]` | 中断可安全固化的前缀(仅非空 text/reasoning) |
| `BlockAssembler.usage` | getter | `usage` chunk 的值;未到则 undefined |
| `BlockAssembler.finish` | getter | finish 原因;缺省 `{kind:'stop'}` |
| `BlockAssembler.replayState` | getter | 与 `blocks()` 同步修剪的回放信封 |
| `BlockAssembler.message` | `(source) => AssistantMessage` | 冻结的 assistant 消息(走 createAssistantMessage) |
| (私有)`ensure` / `assemble` / `assembled` / `mustGet` | — | 按需建半成品 / 单块合成 / keep-drop 决策 / 不变量访问 |

## 与其他模块的关系

- `assistant-stream.ts` 的 `assembleAssistantStream()` 把紧凑记录喂给本类(每 run 一个合并 delta,装配结果与逐成员展开一致)——两个文件共享"装配"这一规范语义。
- `message.ts` 的 `createAssistantMessage` 是 `message()` 的落点;`ModelMessageSource` 提供 provider/model 归属。
- `types.ts` 的 `StreamChunk`/`ReplayEnvelope` 是输入词汇;`@deepseek-ai/dsh-util-values` 的 `assertNever` 与 `@deepseek-ai/dsh-brand` 的 `brandString` 是工具。
- dsh-agent-loop:边记原始 chunk 边 `push`,结束后把 `message(source)` 与紧凑流一起写进 `assistant/message` / `assistant/attempt` 事件(模型可见 ⟺ 已记录)。

## 小结

- BlockAssembler 是全库唯一的 chunk→块装配算法:容错(闭块即忽略、首关获胜)、兼容 delta-only 协议、未知开放块 fail loud。
- **一个 keep/drop 决策同时管内容和元数据**:max-tokens 丢工具调用,回放信封按同一位置数组修剪;长度不齐整个丢弃。装配层的输出从根上不可能自相矛盾。
- `interruptedBlocks()` 的取舍体现"工具调用不可伪造":中断后只有非空白文本/思考可固化。
