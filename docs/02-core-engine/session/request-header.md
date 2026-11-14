# `request-header.ts` — 请求信封快照的规范化、判等与折叠重建

> 源码：`packages/core/session/src/request-header.ts`（69 行，模块注释："Request-header reconstruction utilities over full `request/header` session events."）

## 文件概览

每次模型请求的**信封**——`EpochHeader`（call config + adapter 默认值标记 + 组装好的工具 schema）——是 log 状态：循环在每次请求派发前写一份完整的 `request/header` 快照。于是**"每个对话请求都是日志的纯函数"**：任何持有日志的人，取最新规范快照就能重建当时请求所用的信封（reconstructable-requests 约定）。

本文件提供三个纯函数：

- `canonicalHeader`：把 header 归范为唯一表示（空可选字段缺席）。
- `headerEquals`：字段级判等——循环用它避免记录未变化的 header。
- `foldRequestHeader`：折叠日志（或任意前缀）为"最后一份快照之后的生效 header"。

活会话的 `Session.requestHeader()` 维护的是同一 fold 的增量版本；本文件是离线/无状态版本。

## 关键代码走读

### 1. `canonicalHeader()` — 空可选字段必须缺席

```typescript
export function canonicalHeader(header: EpochHeader): EpochHeader {
  const adapterDefaults = header.adapterDefaults
  return {
    config: header.config,
    ...adapterDefaults?.reasoningEffort === true || adapterDefaults?.maxTokens === true
      ? { adapterDefaults }
      : {},
    ...header.tools !== undefined && header.tools.length > 0 ? { tools: header.tools } : {},
  }
}
```

规范化规则：

- **空工具列表 → 字段缺席**：`tools: []` 与"无工具"必须是同一个表示，否则判等与折叠都会被表示差异欺骗。
- **`adapterDefaults` 只在真有内容时保留**：它是"由 adapter 实际确定（而非调用方提出）的 config 字段"的标记集（`reasoningEffort: true` / `maxTokens: true`），全空时省略。
- 三个字段之外不透传任何其他键——输出永远是规范形状。

配套的运行时校验（在 `surface.ts` 的 `validateSessionEventData`）**拒绝**非规范输入（`header.system`、`tools: []`、`adapterDefaults: {}`）而不是归范化——写侧必须自己干净。

### 2. `headerEquals()` — 字段级判等，schema 按序比较

```typescript
export function headerEquals(a: EpochHeader, b: EpochHeader): boolean {
  if (
    !callConfigEquals(a.config, b.config)
    || a.adapterDefaults?.reasoningEffort !== b.adapterDefaults?.reasoningEffort
    || a.adapterDefaults?.maxTokens !== b.adapterDefaults?.maxTokens
  ) return false
  const at = a.tools ?? []
  const bt = b.tools ?? []
  return at.length === bt.length && at.every((tool, i) => sameSchema(tool, bt[i] as ToolSchema))
}
```

判等的三个维度：

- **config**：用 `dsh-llm` 的 `callConfigEquals`（语义级比较采样参数等，而不是引用或 JSON 串比较）。
- **adapterDefaults**：两个字段逐一比较（标记值只有 `true`）。
- **tools**：长度相等且**逐位置** `JSON.stringify` 相等（`sameSchema`）。工具 schema 经由同一条组装路径产出，JSON 序列化比较既简单又足够；顺序参与比较——顺序不同的同一组工具视为不同 header。

这个函数被 `agent-loop` 用来决定"本次请求的信封与上次相同吗"：相同则不写新快照（同 series 继承最新快照），不同则写 `reason: 'change'`。

### 3. `foldRequestHeader()` — 取最新快照

```typescript
export function foldRequestHeader(events: readonly SessionEvent[], from?: EpochHeader): EpochHeader | undefined {
  let state = from
  for (const event of events) {
    if (event.type === 'request/header') state = canonicalHeader(event.data.header)
  }
  return state
}
```

fold 逻辑极简：跳过一切非 header 事件，最后一个 header 赢。两个细节让它可组合：

- **`from` 参数**：接受先前折叠的状态继续折叠——`Session.requestHeader()` 的增量缓存正是用它对 `log.slice(headerFoldSeq)`（自上次消费以来新增的事件）续 fold。
- **返回 `undefined`** 表示"日志里还没有任何 header"——第一个请求前没有可重建的信封。
- 折叠时再次 `canonicalHeader`：即使历史格式迁移漏了规范化，读取侧也统一表示。

为什么"取最新快照"就够了？因为每个 header 事件都是**完整快照**而非增量（delta 机制 `request/header-delta` 已废弃；含它的遗留 v0 日志被拒收而不是不完整重放）。完整快照的代价是每个消息 series 重复一次信封，换来的是部分窗口渲染与精确重建的自足性。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `canonicalHeader` | `(header: EpochHeader) => EpochHeader` | 归范为唯一表示：空 `tools`/空 `adapterDefaults` 缺席 |
| `headerEquals` | `(a, b) => boolean` | 字段级判等：config 语义比较 + adapterDefaults 逐字段 + tools 按序 JSON 比较 |
| `sameSchema` | 模块私有 `(a, b) => boolean` | 工具 schema 的规范化 JSON 相等 |
| `foldRequestHeader` | `(events, from?) => EpochHeader \| undefined` | 折叠日志/前缀为最新生效 header；`from` 支持增量续 fold |

## 与其他模块的关系

- **`agent-loop`**（`packages/core/agent-loop/src/agent.ts` 直接导入 `canonicalHeader`、`headerEquals`）：每个循环实例边界写 `reason: 'initial' | 'resume'` 快照；信封变化写 `'change'`；未变化但显式开新消息 series 或表面发生替换时写 `'series'`；同 series 的后续 turn/step/重试**继承**最新快照不重写。`'initial'/'resume'/'change'` 携带 `startsSeries: true` 表示该请求同时开启新 series。
- **`index.ts` 的 `Session.requestHeader()`**：同一 fold 的增量形态——headerFold 缓存按 `headerFoldSeq` 游标对新增事件续 fold，结果深冻结后按引用暴露（消费者若原地改写会在下次比较时造成失同步，因此冻结）。
- **`tool-history.ts`**：消费折叠输入的同一批 header 事件，把工具表另存一份供 `developer/message` 的 addition 引用解析。
- **`request/context`（`types.ts`）**：路由元数据（provider/model/contextWindow/`systemPromptUpdate`）**有意放在 EpochHeader 之外**——`headerEquals` 是请求重建契约的字段级比较，把容量或更新模式折进去会让"路由变了"误判为"请求信封变了"。
- **系统提示**：`EpochHeader.system` 已废弃（`@persistenceReserved`，类型为 `never`）；渲染的系统提示是 `system/message` 表面节点（node 0），不是 header 状态——提示变化走表面替换，header 不动。

## 小结

本文件把"请求可重建性"压缩成三个小而确定的函数：规范化保证唯一表示，判等支撑去重与 series 语义，折叠保证"最新快照即生效信封"。记住两个边界：header 只包含**请求输入**（config/adapterDefaults/tools），路由元数据与系统提示都不在其中；快照永远是**完整**的——没有 delta，重建不需要任何日志之外的上下文。
