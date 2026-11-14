# 02 · 会话日志与存储后端

> SessionHandle 与 JSONL


> 内存会话（`docs/subsystems/session.md` 描述的 append-only 日志）如何变得**持久**。核心源码：`packages/session/session-persistence/src/index.ts`（缝定义）与 `packages/session/session-persistence-jsonl/`（交付后端）。

## 缝的形状

`ctx.sessionPersistence` 是一个标准能力缝：抽象服务 `SessionPersistence` 暴露 `create` / `open` / `stat` / `list`，`create`/`open` 返回每会话的 `SessionHandle`（`read`/`append`/`flush`/`close`），**所有日志访问都经过这个 handle**——它是跨进程写租约守卫的唯一门。

```text
            ctx.sessionPersistence (抽象缝)
             ├─ create(header) ───► SessionHandle(write)
             ├─ open(id, 'read'|'write') ─► SessionHandle
             ├─ stat(id) ──► SessionPersistenceSnapshot (轻量观察)
             └─ list() ───► SessionPersistenceSnapshot[]
                          │
                    SessionHandle
             read / append / flush / close
```

## SessionHandle：一个打开的通道

句柄设计的关键决策（摘自 `packages/session/session-persistence` 的类型文档）：

- **单写者状态**：handle 是单 owner 状态，不是共享服务。写 owner 活跃时第二个 `open(id, 'write')` 抛 `SessionAlreadyOwnedError`。
- **单调读**：`read` 永不回退——同一 handle 上重复读不会观察到比上一次更旧的状态；读只返回**合法的连续前缀切片**，撕裂的物理尾巴永不返回。
- **append 尽力而为，flush 才是屏障**：append 解析后批次已接受、有序、本实例可见，但只有 flush 解析才承诺崩溃后存活。后端可以在 append 后面缓冲/批量物理写。
- **新鲜度承诺**：一旦某 write handle 的 append/flush 解析，同后端实例上**之后开始**的任何读（任何 handle、stat/list）至少观察到该前缀。
- **close 是唯一 teardown**：幂等、不可取消（`Symbol.asyncDispose` 委托给它）；关闭的 handle 上一切操作抛 `SessionHandleClosedError`。

```ts
interface SessionHandle extends AsyncDisposable {
  readonly id: SessionId
  readonly header: SessionHeader            // 不可变存储元数据
  readonly inheritedEventCount: number      // fork 继承前缀长度（0 = 非种子）
  readonly access: SessionAccess            // 'read' | 'write'
  read(offset?, length?, options?): Promise<SessionHandleReadResult>
  append(events, options?): Promise<void>
  flush(options?): Promise<void>
  close(): Promise<void>
}
```

读结果带 `eventState`（`'detached' | 'shared-frozen'`）——**调用方拥有的外层数组 + 事件值的所有权状态由生产者声明**，切片不会从结果长度反推所有权。

## flush 检查点：写入如何在后台流动

`session/event` 是**同步**通知；后端按 session id 把事件路由进活跃写 handle 的**有界 write-behind 窗口**，不阻塞生产者：

```text
session/event ──► 路由进写 handle 的缓冲窗口
                    │ 第一个待定事件开窗（固定 deadline）
                    │ 后续事件加入，不重置 deadline
                    ▼
                 窗口到期 ──► 一次持久 append（经写 handle）
                    ▲        │ 写期间进入的事件另起下一批
  session/flush ────┘ 取消等待并排空至静默
```

要点：

- **`session/flush` 是 loop 的排序与错误观察检查点**——在认领下一个普通 turn 之前使用。
- **失败保留顺序**：被拒绝的后台写把事件按序保留、暂停自动路径、经 logger 报告；下一次显式 flush 重试并向调用者响亮拒绝。
- `session/disposed` 做同样的最终排空并关闭 handle；服务级 `flush()` 一次屏障排空所有活跃写 handle，单个失败以 `AggregateError` 命名，其余照常。
- 窗口只限制**有意的批量等待**，不限制事件循环调度或后端持久化延迟。

## 崩溃恢复：不修剪，由读者修复

日志在 turn 中途崩溃时以开放的 `turn/start` 结尾、没有 `turn/end`。持久化层**不截断不修复**——单个 turn 在长任务里可能巨大（多 step、大工具输出），且那些事件在崩溃前已持久追加。它返回物理有效的连续日志；只有**撕裂物理尾巴**中属于未解析 append 的不完整片段被丢弃（其中完整记录由 JSONL 后端部分解码，写入路径在第一次新 append 前持久重写）。

**修复是读者（agent-loop resume）的工作**：通过写 handle 读日志，计算 `interruptedTurnClosers`——缺失的工具错误、任何开放的 `step/end`、一个合成的 `turn/end { reason: { kind: 'interrupted' } }`——并在发布 Session 前作为普通批次经同一 handle 追加。`interrupted` 是唯一一个 loop 从不发出的 `TurnEndReason`。

修复只在写所有权下写：活跃会话的写 handle 由其生命周期 owner 持有，并发的 `open(id, 'write')` 会拒绝而不是让修复与活跃 turn 竞争。只读观察者（session-query）在**内存中**用同样的关闭事件平衡中断的冷日志，不写回。

## SessionHeader：与日志分开旅行的元数据

每个会话的元数据**与事件日志分开**存储：`version`、`cwd`、`isSeeded` 血统位、`parentSession`、`delegationDepth`、`agentPreset` 等。它们不属于 `SessionEventMap`、不进 `deriveMessages()`。

两个值得注意的持久化理由（为什么这些字段必须在 header 里而不是运行时推导）：

- `delegationDepth` 持久化，让递归预算在重启/恢复后存活——运行时深度会把恢复的子会话重置为顶层。
- `agentPreset` 持久化，因为 preset 决定会话的工具与提示词——恢复成不同组合的会话会回放模型再也无法执行的历史。

## 格式拒绝与轻量观察

- **`SessionFormatUnsupportedError`** 区别于损坏错误：**什么都没坏**，只是这个构建不能忠实解读。`stat`/`list` 对最高世代做 header-only 分类并翻译受支持的历史 header，不读不改 body。未来更高版本即使存在更旧的可读世代也照样拒绝。
- **`SessionPersistenceSnapshot`**（stat/list 返回）：header + `formatStatus` + 不透明 `revision` 令牌 + 可选 `eventCount`/`sizeBytes`/`location`。相等 revision 可视为日志未变；不等不承诺任何事；写所有权更迭不改变 revision。session-query 用它做冷读缓存的键——**会话列表绝不靠它去开冷日志**，缓存升级不会把启动变成全量 body 扫描。
- **`SessionLocation`** 是诊断/导出用的观察性元数据（如 JSONL 的绝对转写路径），不授权读，不是持久会话内容。

## 操作与保证速查

| 操作 | 语义保证 |
|------|----------|
| `append` | 尽力而为：解析后本实例可见、有序；不承诺崩溃后存活 |
| `flush` | 持久性屏障：解析后崩溃可恢复；空会话在此物化可列出 |
| `read` | 单调：合法连续前缀，永不回退，永不返回撕裂尾巴 |
| `close` | 唯一 teardown：幂等、不可取消；写 handle 完成待定持久化后释放所有权 |
| `stat`/`list` | 不读 body：header + revision 令牌；写入所有权更迭不改 revision |

## JSONL 后端（交付实现）

`packages/session/session-persistence-jsonl` 实现同一契约并通过共享持久化契约套件：

- 每会话一个**只追加的逻辑 JSONL 日志**，默认存储为**带校验和的 Zstandard 拼接帧**（可配置为原始行）。
- **崩溃安全的原子物化**、每批次 `fsync` 追加、第一次新 append 前截断撕裂尾巴。
- `stat`/`list` 携带 `sizeBytes` 与基于 `fs.stat` 的尽力 revision。

物化语义：`create` 解析后本进程立即可观察；后端可把物理物化推迟到首次 append/flush（纯优化）；**其他进程只看得到已物化的会话——崩溃前从未物化的会话等于从未存在**。`SessionHandle.flush` 强制物化。

## 消费者如何创建/恢复

```ts
// fork：ctx.agents.create({ sessionId, seed, inheritedEventCount, meta: { isSeeded: true } })
// 恢复已持久会话：ctx.agents.resume({ resumeSessionId })
```

只有 agent-loop 发布的会话才会持久化；loop 在发布前先经新会话的写 handle 存下 seed。`SessionStore.prepare()` 接受普通创建选项或 `RestoredSessionOptions`（seed + meta + `inheritedEventCount` + `eventState`），验证并**采用**存储值而不做第二次拷贝/冻结；`SessionPreparation` 拥有这个未发布 Session 直到发布或回滚（同步、幂等释放）。

## 设计启发

1. **句柄而非 id 寻址方法**：把"谁能写"从方法参数变成对象所有权，跨进程租约守卫就有了唯一收口。
2. **尽力 append + 显式 flush**：吞吐与持久性解耦，调用者按崩溃语义自选等待点。
3. **持久化层永不修复语义**：存储层只诚实报告物理事实，语义修复归协议 owner（loop）——职责不越界。
4. **header-only 元数据走旁路**：列表/统计永不读 body，启动成本与日志大小无关。
