# `preparation.ts`

> 未发布会话的所有权封装（Disposable 生命周期件）


> 源码：`packages/core/session/src/preparation.ts`（49 行，模块注释："Ownership of one unpublished Session before registry publication."）

## 文件概览

`SessionPreparation` 是一个小而精确的生命周期件：把一个**尚未进入 registry（store）的 `Session`** 和"提供方为了让它保持可用而持有的状态"封装成同一个可处置对象。典型场景是恢复（restore）路径：从持久化后端读出 seed 后、`SessionStore.prepare/enter/announce` 发布前，中间可能隔着任意异步工作；这段窗口里缓存、连接等 provider 状态需要有人负责释放。

设计要点（JSDoc 原话）：

- "Disposal is synchronous and idempotent."——同步、幂等。
- "Providers decide whether release returns the Session to a cache or discards it; publication may consume that state before disposal, making the callback a no-op."——释放行为由提供方定义：回缓存或丢弃；若发布过程已消费了该状态，dispose 时回调自然退化为 no-op。

## 关键代码走读

### 1. 选项：可选的释放回调

```typescript
/** Options for a preparation whose provider retains unpublished state. */
export interface SessionPreparationOptions {
  /** Release provider-owned state when the Session was not published. */
  readonly release?: () => void
}
```

`release` 是同步回调，语义只有一个触发点：**这个 Session 没有被发布**（preparation 离开调用方的掌控）。成功发布路径会先处置（此时状态多半已被发布方消费，回调按提供方逻辑成为 no-op）——"成功"与"回滚"共用同一个 dispose 入口，避免两条释放路径的分叉。

### 2. 类本体：一次性语义由自身保证

```typescript
export class SessionPreparation implements Disposable {
  private released = false

  /** The exact Session to use for setup and publication. */
  readonly session: Session

  private constructor(
    session: Session,
    private readonly options: SessionPreparationOptions,
  ) {
    this.session = session
  }

  /**
   * Wrap an unpublished Session in one preparation lifetime.
   * @param session - exact unpublished Session.
   * @param options - optional provider release behavior.
   * @returns a preparation disposed after publication or rollback.
   */
  static create(session: Session, options?: SessionPreparationOptions): SessionPreparation {
    return new SessionPreparation(session, options ?? {})
  }

  /** Release provider state once when this preparation leaves its caller. */
  [Symbol.dispose](): void {
    if (this.released) return
    this.released = true
    this.options.release?.()
  }
}
```

四个实现细节：

- **`implements Disposable` + `[Symbol.dispose]()`**：使用 TypeScript 原生的 explicit resource management 协议——调用方可以 `using const prep = SessionPreparation.create(session)`，作用域结束自动处置，异常路径也不例外。
- **`released` 标志保证幂等**：多次 dispose（正常路径 + finally 路径）只触发一次 `release`。
- **构造器私有**：唯一的创建入口是 `create`，保证所有 preparation 都带着一致的文档语义。
- **`session` 是 readonly 公开字段**：preparation 的全部价值就是"把准确的 Session 交给发布流程"——它不做额外包装或代理，发布方直接用 `prep.session` 走 `prepare/enter/announce`。

### 3. 在恢复流程中的位置

restore 流程的时序（结合 `packages/core/session/src/index.ts` 与包 README）：

```
持久化后端读出 seed + header
        │
        ▼
SessionPreparation.create(session, { release })   ← provider 状态（如恢复缓存）此刻被持有
        │   （期间可做任意异步工作：迁移预检、校验、等待调度）
        ▼
SessionStore.prepare(id, { eventState, seed, meta, inheritedEventCount })
        │   session 处于 store 持有但未进入状态（attachments 置 null，写入禁用）
        ▼
ctx.effect(function* { yield enter; announce })   ← 发布事务
        ▼
prep[Symbol.dispose]()   ← 发布已消费 provider 状态；release 退化为 no-op
```

两条失败路径分别由两层清理负责：

- **发布失败**（`session/created` 监听器同步抛出）：generator effect 回滚已 yield 的 detach；preparation 的 dispose 负责把 provider 状态放回缓存或丢弃。
- **发布前放弃**（调用方取消、迁移预检失败）：只有 preparation 的 dispose 生效——`release` 真正执行（例如把会话从"待恢复"缓存中逐出）。

这解释了为什么 `release` 的语义是"Session **未被发布**时释放"而不是"prep 离开作用域时释放"：发布成功与否决定 release 是实质操作还是 no-op，而这一切都收敛在同一个幂等的 dispose 里。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `SessionPreparationOptions` | `interface { release?: () => void }` | 提供方注册"未发布时释放状态"的同步回调 |
| `SessionPreparation` | `class implements Disposable` | 封装一个未发布的 `Session` + provider 状态的所有权；同步幂等 dispose |
| `SessionPreparation.create` | `static create(session, options?)` | 唯一创建入口 |
| `[Symbol.dispose]` | `(): void` | 一次性释放 provider 状态；`released` 标志保证幂等 |

## 与其他模块的关系

- **`index.ts`**：`SessionStore.prepare()` 构造未进入 store 的会话（附件置 `null`，只读）；`SessionPreparation` 在"prepare 之后、enter 之前"这段窗口为调用方（如 `dsh-session-persistence` 的恢复流程）提供所有权封装——会话与"读出 seed 的那个后端会话缓存"同生共死。
- **`Session`（index.ts）**：preparation 只持有引用，不改变 `Session` 的任何行为；未发布/已发布的区分由 `attachments` WeakMap（`null` vs `SessionEntry`）管理，与本文件无关。
- **Cordis effect 模型**：发布路径是 `ctx.effect(function* { yield enter; announce })`——preparation 的 dispose 契约与"generator effect 在抛出时处置已 yield 的 disposer"互补：发布失败时 detach 由 effect 回滚，provider 状态由 preparation 回滚，两层清理互不重叠。

## 小结

49 行解决一个窄问题：**"拿在手里还没发布的会话"也需要所有权语义**。`using` + `[Symbol.dispose]` 让"发布成功或回滚"共用一次释放；幂等标志让调用方不必关心 dispose 被调用几次。它是 restore 路径上连接持久化后端状态与 `SessionStore` 发布流程的胶水件。
