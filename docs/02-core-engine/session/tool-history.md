# `tool-history.ts` — 历史工具声明的有状态重建（动态工具更新的 fold）

> 源码：`packages/core/session/src/tool-history.ts`（75 行，模块注释："Stateful reconstruction of historical tool definitions for request projection."）

## 文件概览

`ToolHistoryProjection` 把日志中的 `request/header` 与 `developer/message` 两类事件 fold 成一个 `ToolHistory` 快照：**当前 series 的基线工具声明 + 历史上每次"动态加工具"的更新记录**。它解决的问题是：`developer/message` 的 `tool-addition` 块只记工具名与定义所在的 `headerSeq` 引用，而不内联定义——重建时必须顺着引用回到那份**不可变的历史 header**取定义，而不是用"当前最新 header"或"当前注册表"猜。

能力无关（capability-independent）：无论模型路由是否支持动态工具更新，这个 fold 都照常进行。活会话用增量 fold（`Session.toolHistory()` 持有一个实例），恢复/重放时从头消费继承历史。

## 关键代码走读

### 1. 状态字段：五份账

```typescript
export class ToolHistoryProjection {
  /** Historical declarations indexed by the header sequence referenced by additions. */
  private readonly headers = new Map<SessionSeq, readonly ToolSchema[]>()
  /** Definitions retained in the current declaration series, including removed tools. */
  private declared = new Map<string, ToolSchema>()
  /** Active definitions from the latest request header. */
  private active: readonly ToolSchema[] = []
  /** Active names reconstructed from the baseline and recorded updates. */
  private available = new Set<string>()
  /** Header that starts the current declaration series; absent before the first header. */
  private baselineSeq: SessionSeq | undefined
  /** Immutable request snapshot of the current baseline and resolved updates. */
  private history: ToolHistory = deepFreeze({ tools: [], updates: [] })
```

- `headers`：历史 header 的工具表，按 header 事件 seq 索引——`tool-addition` 的 `headerSeq` 引用就是查这张表。**跨重启、fork、表面替换都保留**（header 事件在日志里，fold 重放自然带回）。
- `declared`：当前 series 保留的完整定义集（**包括已移除的工具**——保留它们是为了检测"retained 换定义"）。
- `active`：最新 header 的活动定义。
- `available`：基线 ∪ 记录在案的 addition − removal 得到的可用名集合。
- `baselineSeq`：当前声明 series 的起点 header；series 重开（显式 series、替换后、retained 定义变更）时移位。

### 2. `apply()` — 处理 header：series 重开判定

```typescript
apply(event: SessionEvent): void {
  if (event.type === 'request/header') {
    const tools = event.data.header.tools ?? []
    this.headers.set(event.seq, tools)
    // A retained name cannot acquire a different definition without losing
    // prefix reuse; an unchanged restored name is re-offered by its recorded addition.
    const redeclared = tools.some((tool) => {
      const before = this.declared.get(tool.name)
      return before !== undefined && JSON.stringify(before) !== JSON.stringify(tool)
    })
    if (this.baselineSeq === undefined || event.data.reason === 'series' || event.data.startsSeries || redeclared) {
      this.baselineSeq = event.seq
      this.declared = new Map(tools.map(tool => [tool.name, tool]))
      this.history = deepFreeze({ tools, updates: [] })
      this.available = new Set(tools.map(tool => tool.name))
    }
    this.active = tools
  }
```

遇到 `request/header` 时：

1. 把该 header 的工具表存入 `headers`（供以后的 addition 引用）。
2. **series 重开判定**：首个 header、显式 `reason === 'series'`、`startsSeries` 标记、或某个 retained 名字的定义变了（`redeclared`），都会把当前 header 设为新基线并重置 `declared`/`history`/`available`。`redeclared` 的注释说明了两面：retained 名字换定义必然损失前缀复用（KV cache），而定义未变的恢复名字由其 recorded addition 重新提供——两条路径都正确。
3. 无论是否重开，`active` 都更新为最新 header 的定义。

### 3. `apply()` — 处理 developer 消息：顺 headerSeq 解析 addition

```typescript
} else if (event.type === 'developer/message') {
  const { message, headerSeq } = event.data
  const definitions = headerSeq === undefined ? [] : this.headers.get(headerSeq)
  const additions = message.content.flatMap((block) => {
    if (block.type !== 'tool-addition') return []
    const tool = definitions?.find(tool => tool.name === block.toolName)
    if (tool === undefined) throw new Error(`tool history: missing definition for ${block.toolName}`)
    return [tool]
  })
  for (const tool of additions) this.declared.set(tool.name, tool)
  for (const block of message.content) {
    if (block.type === 'tool-addition') this.available.add(block.toolName)
    else if (block.type === 'tool-removal') this.available.delete(block.toolName)
  }
  this.history = deepFreeze({
    tools: this.history.tools,
    updates: [...this.history.updates, { messageId: message.id, additions }],
  })
}
```

关键点：

- addition 的定义**必须**能在 `headerSeq` 指向的历史 header 里找到且**恰好一个**（append 时由 `surface.ts` 的 `assertDeveloperHeader` 保证；fold 里找不到直接抛错——"missing definition for X"）。引用的是历史快照，所以"当前注册表"或"最新 header"都不能替代。
- `removal` 只按名字从 `available` 中删除，不需要 header 引用。
- `history` 以**不可变追加**方式演进：每次产生新的深冻结快照（`tools` 保持基线不变，`updates` 追加 `{ messageId, additions }`）——以前返回的快照不会被后续事件改写。
- 其余类型的事件不动 fold 状态（`toolHistory()` 只对 `log.slice(toolHistorySeq)` 增量 apply）。

### 4. `snapshot()` — 旧日志的兼容回退

```typescript
snapshot(): ToolHistory {
  // Sessions written before tool-update emission have headers without matching updates.
  if (this.active.length !== this.available.size || this.active.some(tool => !this.available.has(tool.name))) {
    return deepFreeze({ tools: this.active, updates: [] })
  }
  return this.history
}
```

在"工具更新机制"出现之前写入的会话，header 与 `developer/message` 更新记录之间没有配对关系，此时 `available`（由基线+更新重建的名字）与 `active`（最新 header 的名字）对不上。fold 检测到这种不一致就退化为**只报告最新 header 的完整活动声明**（`updates: []`）——注释原话："Headers lacking matching update records use complete active declarations"。一致性成立时返回累积的基线 + 更新历史。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ToolHistoryProjection` | `class` | 有状态 fold：历史 header 索引 + 当前 series 声明 + 可用名集合 |
| `apply` | `(event: SessionEvent): void` | 消费下一个已提交事件（log 顺序）；只响应 `request/header` 与 `developer/message` |
| `snapshot` | `(): ToolHistory` | 返回不可变快照：series 基线声明 + 已解析 additions；旧日志回退为完整活动声明 |

## 与其他模块的关系

- **`index.ts` 的 `Session.toolHistory()`**：`Session` 持有一个实例与游标 `toolHistorySeq`，每次读取只 `apply` 新提交的事件（增量 fold），然后取 `snapshot()`。agent-loop 与 compaction 都经由该方法取工具历史（`packages/core/agent-loop/src/agent.ts` 中 `toolHistory: session.toolHistory()`）。
- **`surface.ts`**：append/restore 时由 `assertDeveloperHeader` 校验 `headerSeq` 引用与工具定义的"恰一个、完整、deferLoading 必须为 true"约束——fold 抛的 "missing definition" 是同一契约在读取侧的镜像。
- **`request-header.ts`**：本 fold 消费的 header 工具表正是 `canonicalHeader` 规范化后的内容；`reason`/`startsSeries` 字段驱动 series 重开。
- **模型请求组装**：`ToolHistory`（定义在 `dsh-llm`）描述请求应携带的基线工具与更新序列；"同名 schema 变更被允许并使前缀复用失效（即便 in-history 路由）"的策略记录在 `.agents/notes/implemented/architecture/2026-09-20-dynamic-tool-updates.md`。

## 小结

`tool-history.ts` 只做一件事：把"工具声明随会话演化"还原成模型请求可用的快照。三个不变量值得记住——addition 的定义**永远解析自它引用的那份历史 header**；series 由显式标记或 retained 换定义重开；对没有更新记录的旧日志，fold 退化为完整活动声明而不是猜测历史。
