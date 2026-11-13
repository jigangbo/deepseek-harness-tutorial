# `runtime-context.ts` — system prompt 与运行时上下文两个 surface 投影

> 源码：`packages/core/agent-loop/src/runtime-context.ts`（169 行）。导出 `SystemPromptCommit`、`SystemPromptDecisionInput`、`SystemPromptProjection`、`RuntimeContextProjection`；并通过声明合并向 `@deepseek-ai/dsh-llm` 的 `MessageSourceMap` 注册 `'runtime-context'` 来源。

## 文件概览

agent-loop 拥有两条由 system-prompt 插件"成形"的 surface 消息：

1. **system prompt**——`system/message` 节点（surface 节点 0 及任何 in-history 替换/追加）；
2. **运行时上下文快照**——`source.kind === 'runtime-context'` 的 `user/message`。

两个投影类都不"拥有"提交：它们只**决策**要提交什么（返回未提交的 message + intent），真正的 `session.append()` 由 `agent.ts` 执行，且都在取消点之后。这个拆分让投影可以被纯逻辑测试，也让提交时序全部留在驱动里。

## 关键代码走读

### 来源类型与清空哨兵

```typescript
declare module '@deepseek-ai/dsh-llm' {
  interface MessageSourceMap {
    'runtime-context': { kind: 'runtime-context' } & ContextFormed
  }
}

const SOURCE = 'runtime-context'
const CLEARED = 'Current runtime context: none. Earlier runtime-context snapshots no longer apply.'
```

`MessageSourceMap` 是 `…Map → derived-union` 模式：`MessageSource` 联合由它派生，本包经声明合并扩展。`ContextFormed` 携带快照由哪些具名贡献组成（`form: 'snapshot', sections`），用于 UI 归因每个部分属于哪个子系统。

### SystemPromptProjection：三种路由的决策器

先看状态读取——surviving 的 `system/message` 节点按 surface 顺序：

```typescript
/** The surviving `system/message` nodes in surface order. */
private systemNodes(): { seq: SessionSeq; text: string | undefined }[] {
  const nodes: { seq: SessionSeq; text: string | undefined }[] = []
  for (const seq of this.session.surface.nodes) {
    const event = this.session.eventAt(seq)
    if (event?.type !== 'system/message') continue
    const content = event.data.message.content
    const text = content.length === 0 ? '' : textOf(event.data.message)
    nodes.push({ seq, text })
  }
  return nodes
}
```

节点读的是 `session.surface.nodes`（当前存活 surface），空内容的节点记 `''`——它们是"休眠尾巴"。

核心决策 `project()`：

```typescript
/**
 * Reconcile effective text and retained nodes with the prepared route and series.
 * A series start re-baselines node 0 only when the effective text differs; a
 * capable continuing series keeps appended nodes in place to preserve the prefix.
 */
project(rendered: string, input: SystemPromptDecisionInput): SystemPromptCommit[] {
  const nodes = this.systemNodes()
  const head = nodes[0]
  if (head === undefined) {
    return [{ message: createSystemMessage(rendered), intent: { surfaceOp: 'append' } }]
  }
  const latest = nodes.findLast(node => node.text !== '') ?? head
  if (!input.inHistory || rendered.length === 0 || (input.startsSeries && latest.text !== rendered)) {
    const updates = nodes.slice(1).filter(node => node.text !== '')
      .map(node => this.replace(node.seq, ''))
    if (head.text !== rendered) updates.push(this.replace(head.seq, rendered))
    return updates
  }
  if (latest.text === rendered) return []
  return [{ message: createSystemMessage(rendered), intent: { surfaceOp: 'append' } }]
}

private replace(seq: SessionSeq, text: string): SystemPromptCommit {
  return {
    message: createSystemMessage(text),
    intent: { surfaceOp: { op: 'replace', startSeq: seq, endSeq: seq }, sourceEventSeqs: [seq] },
  }
}
```

输入是 `SystemPromptDecisionInput { inHistory, startsSeries }`——`inHistory` 来自 `preparedCall.systemPromptUpdate === 'in-history'`（route 能力），`startsSeries` 来自驱动汇聚的系列事实。四种结果：

| 路由 | 触发条件 | 产出 |
|---|---|---|
| **首次追加** | 无头部节点（日志从未有过 system 节点） | append 渲染文本。**即使为空也追加**——首个 prompt 预留 surface 节点 0，只是不产生 wire 消息 |
| **清空** | `rendered.length === 0`（无论 route 或系列状态） | 每个非空的后续节点一条逐节点空替换（logged per-node empty replacement），头部仍非空则也替换为空。空头部 + 无活跃后续节点 = 无 prompt；重复清空与恢复后保持空 |
| **合并（归一化）** | route 不支持 `inHistory`，或新系列且有效文本不同（`!input.inHistory \|\| ... \|\| (input.startsSeries && latest.text !== rendered)`） | 先把所有非空后续节点替换为 `''`，再视需要重写头部。旧指令不再模型可见；逐节点空替换保留中间历史而不需要 surface 删除操作 |
| **in-history 追加** | capable route 的持续系列且文本有变 | append 一个新 `system/message` 节点挂在缓存历史之后——前缀到历史为止仍可复用（KV Cache 友好） |

外加一个**无操作**结果：`latest.text === rendered`（capable 持续系列文本未变，或新系列但最新存活节点已持有该文本）返回 `[]`——"把同样文本挪到节点 0 只会改变请求前缀而不改变 prompt"，所以系列开始时若无差异则不产生任何 system 事件，请求保留原有前缀。

两个细节值得注意：

- **休眠尾巴**（`text === ''` 的后续节点）不提供有效文本、不需要重复替换——`filter(node => node.text !== '')` 把它们排除出清空/合并路径。
- **新系列且文本相同不重基线**：`input.startsSeries && latest.text !== rendered` 才走合并；文本相同时即使 `startsSeries` 为真也走无操作，保住前缀缓存。

驱动的调用点（`agent.ts` 的 `step()`）把三种 `startsSeries` 来源喂进来：

```typescript
const commits = this.systemPrompt.project(renderedPrompt, {
  inHistory: preparedCall?.systemPromptUpdate === 'in-history',
  startsSeries: startsRequestSeries
    || this.requestSurfaceGeneration !== this.session.surface.contentGeneration
    || (preparedCall?.toolUpdate === undefined && this.toolsChanged(assembly.tools)),
})
for (const { message, intent } of commits) {
  this.session.append('system/message', { turn, step, message }, intent)
}
```

即：pre-step 声明的 `startsRequestSeries`、surface generation 变化、或不支持 `toolUpdate` 的 route 上工具 schema 有变。支持工具与 system-prompt 双更新的 route 可以追加变更的 prompt 并激活新增工具而不重写更早的消息。

### RuntimeContextProjection：快照的恢复与跟随

```typescript
/** Tracks the last retained runtime-context snapshot without owning its commit. */
export class RuntimeContextProjection {
  /** `undefined` means no snapshot ever existed; `null` means none is retained. */
  private retained: { seq: SessionSeq; text: string | undefined } | null | undefined
```

`retained` 的三态是文件里最微妙的类型：`undefined`（从未有过快照）、`null`（有过但已被替换）、`{ seq, text }`（当前保留的快照）。

```typescript
constructor(ctx: Context, session: Session) {
  const surface = new Set(session.surface.nodes)
  for (const event of eventsNewestFirst(session)) {
    if (event.type !== 'user/message' || !isOwned(event.data)) continue
    this.retained ??= null
    if (surface.has(event.seq)) {
      this.retained = { seq: event.seq, text: textOf(event.data) }
      break
    }
  }

  ctx.on('session/event', (subject, event) => {
    if (subject !== session) return
    if (event.type === 'user/message' && isOwned(event.data)) {
      this.retained = { seq: event.seq, text: textOf(event.data) }
    } else if (this.retained
      && isReplacementSurfaceEvent(event)
      && event.sourceEventSeqs?.includes(this.retained.seq) === true) {
      this.retained = null
    }
  })
}
```

构造做两件事：

1. **一次性恢复**：从最新向旧扫描 `user/message`，只认本包拥有的消息（`isOwned` = `source.kind === 'runtime-context'`）。扫到任何 owned 消息先把 `retained` 定为 `null`（"有过快照"），再检查它是否还在存活 surface 上——在则恢复其 seq 与文本。
2. **持续跟随**：`ctx.on('session/event')` 只看本 session。新 owned `user/message` 直接更新 `retained`；替换型 surface 事件（`isReplacementSurfaceEvent`，如 compaction）引用了保留快照的 seq 时置 `null`——快照被 compaction 阴影掉了。

决策函数：

```typescript
/**
 * Create an uncommitted snapshot only when the retained value differs.
 * @param current - fully rendered dynamic context.
 * @param sections - named contributions that formed the current snapshot.
 * @returns a candidate user message, or `undefined` when no update is needed.
 */
project(current: string, sections: readonly ContextSnapshotSection[]): UserMessage | undefined {
  if (this.retained === undefined && current.length === 0) return
  const snapshot = current.length === 0 ? CLEARED : current
  if (this.retained?.text === snapshot) return
  return createUserMessage({
    content: [{ type: 'text', text: snapshot }],
    // The cleared marker has no contributions left to attribute.
    source: sections.length === 0
      ? { kind: SOURCE }
      : { kind: SOURCE, form: 'snapshot', sections },
  })
}
```

三个分支：

- **从未有过快照且当前为空** → `undefined`，日志里不出现哨兵；
- **保留值与当前渲染相同** → `undefined`，不重复提交（KV Cache 前缀保持）；
- **否则** → 新候选消息。快照为空时以 `CLEARED` 哨兵文本显式声明"先前快照不再适用"，让模型知道旧上下文已失效，而不是默默消失。

`agent.ts` 的消费侧有两个位置：`preStep()` 的默认瀑布终点用它生成"跟随 claim 批次追加的上下文候选"；`step()` 里渲染后再次 `project()`，把 claim 到的 `runtime-context` 消息**原位替换**为新鲜候选（`source.kind === 'runtime-context'` 的消息不直接提交），重试间快照变化则走循环尾部的兜底追加。

## 核心类与函数（表格）

| 名称 | 类别 | 职责 |
|---|---|---|
| `'runtime-context'` 消息来源 | 声明合并 | 向 `MessageSourceMap` 注册，携带 `ContextFormed`（具名贡献归因） |
| `CLEARED` | 常量 | 空快照的哨兵文本，声明先前快照不再适用 |
| `SystemPromptCommit` | interface | 一条未提交的 system surface 操作：`message` + `intent`（append 或逐节点 replace） |
| `SystemPromptDecisionInput` | interface | 决策输入：route 能力 `inHistory` 与系列事实 `startsSeries` |
| `SystemPromptProjection` | class | system prompt 的三种路由决策（首次追加 / 清空 / 合并 / in-history 追加 / 无操作）；`systemNodes()` 读存活 surface |
| `RuntimeContextProjection` | class | 最后一保留快照的恢复（构造）与跟随（`session/event`）；`project()` 仅在差异时产出候选 |
| `eventsNewestFirst(session)` | function（模块内） | 新到旧的事件快照，恢复扫描用 |
| `isOwned / textOf` | function（模块内） | 来源判定；单文本块内容提取 |

## 与其他模块的关系

| 对象 | 关系 |
|---|---|
| `packages/core/session` | `session.surface.nodes` / `eventAt()` / `snapshotEvents()`（恢复）；`SurfaceIntent` / `isReplacementSurfaceEvent` 描述提交意图 |
| `packages/core/system-prompt` | `renderContextSections()` / `joinContextSections()` 产出快照文本与具名 sections；`refreshContext()` 在不重装 prompt/tools 的前提下刷新注册的运行时事实 |
| `packages/llm` | `createSystemMessage` / `createUserMessage` 工厂；`MessageSourceMap` 声明合并扩展 |
| `agent.ts` | 唯一消费者：`preStep()` 默认决策与 `step()` 准入都调用两个投影；`session.append()` 在取消点之后执行投影的决策 |
| `cordis` | `RuntimeContextProjection` 用 `ctx.on('session/event')` 做事件跟随（agent-scoped context，随 scope 卸载） |

## 小结

这个文件把两条最容易出错的消息流做成纯决策器：**system prompt** 按 route 能力与系列事实在"追加 / 清空 / 合并"三种路由中选择，绝不留下旧指令模型可见，同时尽力保住 KV Cache 前缀；**运行时上下文**以三态 `retained` 跟踪最后快照，只在差异时产出候选，清空用显式哨兵声明。两者都不碰 `session.append()`——提交时序（取消点之后、user 准入旁边、重试恢复）全部留在 `agent.ts`，决策逻辑保持可独立测试。
