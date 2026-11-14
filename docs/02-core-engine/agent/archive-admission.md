# `archive-admission.ts` — 归档准入
> Workspace 归档准入的 `turn` 家族


## 文件概览

`packages/core/agent/src/archive-admission.ts`（模块标注 `@module @deepseek-ai/dsh-agent`）只有 39 行、一个导出函数 `installTurnArchiveAdmission(ctx, lookup)`，却是一个完整的**能力 seam 消费端**样例：Workspace 注册表在把 Session 归档/隐藏之前会问"这个 Session 还有什么在运行"（`workspace/session-activity`）并要求"让它停下"（`workspace/session-stop`）。本文件以 `turn` 活动家族（由 [types.md](types.md) 合并进 `SessionActivityKindMap`）回答这两个问题，让"Session 自己的 Agent 正在一个 turn 里"成为归档准入的一等活动。

安装时机在 `AgentRegistry` 构造函数里，所以它为注册表发布的**每一个** Agent 回答：

```typescript
// packages/core/agent/src/index.ts（构造函数摘录）
// Archive admission: the Workspace registry asks what still runs for a
// Session before hiding it; a running turn answers here, for every Agent.
installTurnArchiveAdmission(ctx, sessionId => this.get(sessionId))
```

## 关键代码走读

整个实现先完整看一遍：

```typescript
// packages/core/agent/src/archive-admission.ts
/**
 * Answer `workspace/session-activity` with the `turn` family while the
 * Session's Agent is running (a turn waiting for an approval or an answer
 * included), and `workspace/session-stop` by cancelling that turn the way
 * the user's own stop does — `agent.cancel({ kind: 'user' })`, but without
 * the stop button's `keepInbox`, so queued input is discarded with a logged
 * inbox splice instead of waking the archived Session later. Nothing is
 * awaited to settlement. Both listeners live as long as `ctx`'s fiber.
 */
export function installTurnArchiveAdmission(ctx: Context, lookup: (sessionId: SessionId) => Agent | undefined): void {
  ctx.on('workspace/session-activity', async ({ sessionId }, next) => {
    const running = lookup(sessionId)?.status === 'running'
    const rest = await next()
    if (!running) return rest
    const own: SessionActivity = { kind: 'turn' }
    return [own, ...rest]
  })
  ctx.on('workspace/session-stop', ({ sessionId }) => {
    const agent = lookup(sessionId)
    if (agent?.status === 'running') agent.cancel({ kind: 'user' })
  })
}
```

### 活动查询：先采样，后委托，再前置

```typescript
// packages/core/agent/src/archive-admission.ts
ctx.on('workspace/session-activity', async ({ sessionId }, next) => {
  const running = lookup(sessionId)?.status === 'running'
  const rest = await next()
  if (!running) return rest
  const own: SessionActivity = { kind: 'turn' }
  return [own, ...rest]
})
```

这是一个 waterfall 监听器（`next` 是链条的后续），注意三个细节：

1. **`lookup()` 在 `await next()` 之前采样**。`running` 是在链条推进前抓下的快照——`status === 'running'` 意味着 Agent 的驱动处于 drain 区间，**包括正在等审批或等回答的 turn**。先采样后委托，避免下游处理期间的并发状态翻转污染本次回答。
2. **委托必须完整**。`const rest = await next()` 调用了 `next()` 并透传其余家族——符合"瀑布监听者必须调用 `next()` 委托"的仓库规范；本监听者只**前置**自己的家族，不吞掉别人的回答。
3. **没有运行就不报告**。`running` 为 false 时原样返回 `rest`——不运行不等于 `idle` 之外的第三种状态；`turn` 家族只在真正有 turn 在跑时出现。

返回值把 `turn` 放在数组首位：`[own, ...rest]`——本 Session 最贴身的活动排最前。

### 停止请求：复刻用户的停止，但故意不带 keepInbox

```typescript
// packages/core/agent/src/archive-admission.ts
ctx.on('workspace/session-stop', ({ sessionId }) => {
  const agent = lookup(sessionId)
  if (agent?.status === 'running') agent.cancel({ kind: 'user' })
})
```

语义上这是"用用户自己的方式停掉那个 turn"：cancel cause 是 `{ kind: 'user' }`，与停止按钮同源（该 cause 会作为 `turn/end` 的 `{ kind: 'aborted', reason }` 落入持久记录）。但有一个刻意的**不对称**：不传 `{ keepInbox: true }`。停止按钮保留排队输入（用户可能只是想打断当前回合），而归档场景必须丢弃排队输入——否则挂起的输入会在稍后唤醒一个已被归档的 Session。丢弃动作本身不是静默的：它带一条日志化的 inbox splice（`agent/inbox/spliced` 的 `outcome: 'canceled'` 事件），所以"归档丢弃了什么"仍可从日志重建（并参与 [consumed-work.md](consumed-work.md) 的 `droppedUnrun` 记账）。

这里能看到 [runtime-types.md](runtime-types.md) 中 `cancel` 契约的两个条款同时生效：

- cause `{ kind: 'user' }` 是第一个 cause，赢得该活动；
- 没有 `keepInbox` 时排队与 steering 输入被清空。

### 生命周期：fire-and-forget，随 fiber 存亡

函数 JSDoc 明确两点：**不对 settle 等待任何东西**（"Nothing is awaited to settlement"）——`cancel()` 是同步 fire-and-forget，归档流程不等 turn 真正停下；**两个监听器活到 `ctx` 的 fiber 结束**——`ctx.on()` 的注册返回 disposer 并挂在安装上下文上，注册表（连同它发布的所有 Agent）拆除时监听器随之注销。还有一个推论：**没有活体 Agent 的 Session 没有能运行的 turn**——`lookup()` 返回 `undefined` 时两个监听器都直接短路，所以归档一个"从未创建过 Agent"或"Agent 已拆除"的 Session 不需要本家族的任何参与。

## 核心导出速查

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `installTurnArchiveAdmission` | `(ctx: Context, lookup: (sessionId: SessionId) => Agent \| undefined) => void` | 安装 `workspace/session-activity`（报告 `turn` 家族）与 `workspace/session-stop`（取消运行中的 turn）两个监听器 |

## 与其他模块的关系

- **`types.ts`**：`SessionActivityKindMap.turn` 声明合并定义了本文件回答的家族；`SessionActivity` 类型来自 `dsh-workspace`。
- **`index.ts`**：唯一的安装点——注册表构造函数传入 `sessionId => this.get(sessionId)`，把准入回答接到活体注册表上；这保证了"每个发布的 Agent 都被覆盖"。
- **`runtime-types.ts`**：`agent.status` 与 `agent.cancel(cause, options)` 是本文件对 Agent 契约的全部依赖。
- **`dsh-workspace`**：`workspace/session-activity`（waterfall）与 `workspace/session-stop` 事件的拥有者——归档准入 seam 的 Service Definition 一侧。
- **`dsh-agent-loop`**：cancel 的实际执行者（中止活动 turn、写 canceled inbox splice、收敛到 idle）。
- **`consumed-work.ts`**：归档丢弃留下的 canceled splice 会进入其 `droppedUnrun` 记账。

## 小结

- 39 行完成一个 seam 消费端：waterfall 先采样 `status`、再 `await next()` 委托、有 turn 则前置 `{ kind: 'turn' }`；stop 侧以用户 cause 取消运行中的 turn。
- 归档取消与停止按钮的唯一差别是刻意省略 `keepInbox`：排队输入被丢弃且留日志化 splice，避免唤醒已归档的 Session。
- fire-and-forget、随 fiber 存亡、无 Agent 短路——三个生命周期事实让"归档时谁在跑"这个问题有了确定且无悬挂的答案。
