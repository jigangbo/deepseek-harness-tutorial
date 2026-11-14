# 第 30 章 会话投影

> 从日志折叠领域状态


> 会话日志是事实，但 UI 和领域服务不该各自翻日志。**会话投影**把"从日志折叠状态"标准化：框架驱动折叠，领域只负责计算。核心源码：`packages/session/session-projection/src/index.ts`、`packages/session/session-projection-cache/`。

## 三角色

| 角色 | 包 | 职责 |
|------|-----|------|
| Service Definition + 注册表 | `dsh-session-projection`（`ctx.sessionProjections`） | 订阅一次 `session/event`，把每个已提交事件流经每个单元 |
| 领域贡献者 | todo、goals、agent-loop、plan… | 各注册一个**纯单元**（ProjectionDefinition），不持有任何订阅 |
| 载体 | `dsh-session-controller` | 历史尾部页与 `session/projection` 推送帧，把成品值送到客户端 |

核心分工口号：**"The framework drives, the domain computes"**——领域不做订阅，客户端不做折叠，只收成品。

```text
session/event (每次提交，同步)
     │ 注册表订阅一次
     ▼
┌──────────────────────────────────────────┐
│ 对每个注册单元执行 apply(state, event)   │
│   todo 单元 → todo 状态                  │
│   goals 单元 → 目标状态                  │
│   agent-loop 单元 → inbox/modelSelection │
└──────────────────────────────────────────┘
     │ view(state) (Object.is 比较)
     ▼
snapshot() 同步一致读切 / 变更推送帧 ──► 客户端
```

## 单元（ProjectionDefinition）契约

每个单元是**纯同步折叠**，五个字段缺一不可：

```ts
interface ProjectionDefinition<K extends string, S> {
  key: K                                  // 本单元拥有的投影键
  stateSchema: ZodType<S>                 // 校验持久化状态再播种折叠
  init(header, inheritedEventCount): S    // 空日志的初始状态
  apply(state, event): S                  // 纯转换；不关心的事件必须返回同一引用
  wire?: { viewSchema, view(state), summary? }   // 客户端视图；host-only 单元省略
  stateVersion: number                    // 持久缓存失效版本
}
```

三条硬纪律：

1. **全同步**：异步单元会撕裂载体的一致性读切——所以 apply 与 view 都必须是同步函数。
2. **状态必须是纯 JSON**：这是持久化缓存的前置条件。
3. **引用纪律即性能**：不关心某事件就返回**同一 state 引用**（`Object.is` 相同 → 下游零工作）；对象值视图必须复用引用以抑制"仅内部状态变化"的发布。

`apply` 是确定性回放：源事件携带整值或领域自有操作，单元的确定性 `apply` 拥有回放语义——**检查点 + 前向尾部回放重建出同一状态**。

## 快照与水位线

```ts
interface ProjectionSnapshot {
  asOfSeq: SessionSeqCursor   // 共享水位线：所有值都反映到的最后一个事件 seq（空日志 -1）
  values: Partial<SessionProjectionMap>   // 每个键的完整当前客户端值
}
```

- `snapshot(session)` **完全同步**：载体在与页面切片同一个 tick 里读它，`asOfSeq` 用同一个序号覆盖两处读取——一致性不用锁，用"同一 tick"。
- 每个值返回前都过本单元的 `viewSchema` 校验。
- `handle.stateOf(session)` 只读一个 host 状态、不算无关视图；`readerFor(session)` 返回绑定单一会话的同步回调，可在类型化对象里组合多个读者（领域服务私有持有）。
- 卸载不失效：`ProjectionHandle` 保留同步读者，注销后仍可读——直到键重新注册前，读还会推进 checkpoint 服务的 released 单元。

## 变更馈送

两级监听器，都只在 `Object.is` 变化时触发：

| 监听器 | 粒度 |
|--------|------|
| `ProjectionChangeListener` | 单单元：某单元的 raw view 结果变化 |
| `ProjectionEventChangeListener` | 单事件：一个提交事件改变了 ≥1 个客户端可见单元的视图（按注册序逐键携带） |

没有改变任何客户端视图的事件不触发事件级调用。

## 摘要读与排除标记

会话摘要（列表行）携带**摘要投影**而非完整投影。排除用纯类型标记：

```ts
type SessionProjectionSummaryExcluded<T> = T & { readonly [sessionProjectionSummaryExcluded]: true }
```

- 标记**只存在于类型层**：它决定客户端视图类型；`wire.summary: false` 决定运行时键集——两者必须人工保持一致（声明 `false` 于带标记的键）。
- `summaryKeys()` 从活跃注册派生键集，注册或释放变化前返回同一实例；摘要消费者把它传给 `snapshot(session, keys)` / `viewCheckpoint(rows, keys)`——**没有摘要专用读面，也没有第二份键集定义**。
- 列表行承诺摘要键集且仅此；带键子集的 `snapshot` 只计算返回的单元。

## 持久投影缓存

`packages/session/session-projection-cache`（`ctx.sessionProjectionCache`）：

- 按会话**节流地持久化检查点**（强制点：`turn/end` 与 detach），加速 prepared-Session 的投影水合。
- 消费者：session-controller（基线）、session-query、session-reference。
- `stateVersion` 是失效开关：状态字段或折叠语义变化时 bump——**可选字段添加**若"缺席保持历史含义且旧检查点后缀回放=全量回放"可以不 bump，但必须**测试那个等价性**，仅 schema 通过不算数。

## 谁在用投影

| 键 | 单元来源 | 内容 |
|----|----------|------|
| `todo` | `dsh-tool-todo` | 待办清单状态 |
| `goals` | `dsh-goal` | 会话目标（修订版状态） |
| `plan` / `planMode` | `dsh-plan-mode` | 计划协作状态 |
| inbox / modelSelection | agent-loop | 待处理输入、模型选择快照 |
| turn/step 状态 | agent 注册表 | `turnBoundaryOf(session)` 暴露 |

注册生命周期：领域服务在激活时**等待注册完成**、私有持有 handle、只暴露业务方法（`dsh-session-projection` 的 README 拥有驱动/缓存/馈送契约细节；`dsh-session-projection` 技能拥有创建/评审投影的流程）。

## 设计启发

1. **状态 = 日志的确定性函数**：崩溃恢复、fork、回放、测试全部退化为"重新折叠"——没有第二份事实来源。
2. **同步折叠 + JSON 状态 + 引用纪律**：三个看似工程细节的约束，合起来让"框架驱动、领域计算"真正可行。
3. **水位线代替锁**：`asOfSeq` + 同 tick 读切，用序号表达一致性，天然可缓存。
4. **类型标记做策略**：摘要排除是纯类型层的标记，零运行时表示——策略进类型，运行时零成本。
