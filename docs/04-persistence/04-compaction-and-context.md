# 05 · 上下文压缩与请求上下文

> 让长会话活下去


> 长会话的敌人是上下文窗口。dsh 用两条互补战线应对：**压缩**（compaction，把旧历史折叠成摘要）与**请求上下文**（context，在每次请求注入当前工作环境事实）。压缩不是 agent-loop 脊柱的一部分——它是一个可选能力缝。

## 压缩缝三角色

| 角色 | 包 | 说明 |
|------|-----|------|
| Service Definition | `dsh-compaction`（`ctx.compaction`，`CompactionEngine`） | 接口必然依赖 `dsh-session` 与 `dsh-llm`：动词作用于 agent 拥有的 Session，持久摘要事件用 `ContentBlock` 词汇 |
| Service Provider | `dsh-compaction-basic`（以及任何同接口兄弟包） | 拥有阈值、保留尾策略、溢出上限、路由摘要调用与失败处理 |
| Human Consumer | `dsh-command-compact` | 手动 `/compact` 命令 |

与 bash 缝的不同：bash 的接口零依赖；压缩的接口**必然**依赖 session 与 llm——它的动词天然作用于 Session 与模型。

## 三个 log-only 会话事件

压缩通过声明合并扩展 `SessionEventMap`，三个事件**全部只进日志、不进 surface**（`SurfaceEventType` 刻意不扩展——只有产生消息的事件才到达模型）：

| 事件 | 载荷要点 | 作用 |
|------|----------|------|
| `compaction/start` | `{ turn }`（数字=自动回合；`null`=独立手动尝试） | 获取日志记录的锁 |
| `compaction/summary` | 摘要、可选 rawOutput/usage、`shadowedRange`/`shadowedSeqs`、token 数、provider/model 信封 | 安全摘要投影 + 被遮蔽范围的完整记录 |
| `compaction/end` | `{ turn, error? }` | 释放锁（error 记录失败尝试） |

**摘要本体靠一条带 `surfaceOp: { op: 'replace', startSeq, endSeq }` 的 `user/message` 落地**——这是摘要压缩执行的唯一 surface 变更。

锁括住**整个**操作：`start` → 摘要化 → `summary` 记录 → `user/message` 替换 → 最后 `end`。锁最后释放，把"操作中途崩溃"变成**可检测的孤儿锁**（有 start 无 end），而不是一个谎称"压缩完成"的 end。活跃的未匹配 start 阻塞所有入口；在更新的 `session/end-seed` 之前的未匹配 start 是旧生命周期的陈旧证据，被忽略。

## CompactionEngine 三个动词

```ts
compactIfNeeded(agent, trigger, signal)  // 自动：'pressure' | 'context-overflow'
compactNow(agent, signal, sourceCommandId?)   // 手动：空闲会话的有用缩减（低于压力也行）
compactRegion(start, end, agent, signal?)     // 强制：按 surface 位置折叠一个区间
```

- `compactNow()` 在 turn 之间作为 agent 维护运行；没有有用范围时不写任何东西返回 `null`。
- 压力压缩跑在 `agent/pre-step`（针对已提交请求头）与 `agent/request`（针对变更后的目的地，头部记录前）。
- **模型切换摘要保留前一路由**，除非显式配置了摘要目标；失败的切换保留头部与已完成的缩减。
- 区间边界**保持工具调用/结果配对但不保持整 turn**——一个超大 turn 里较早关闭的 step 也能被压缩。
- 手动失败分类：`busy | cancelled | changed | summary | commit | persistence`；`changed`/`summary` 关闭并持久化失败尝试（无摘要替换）；取消单独处理，原样抛出 abort 原因。

## 工具结果修剪（pruning）

`ctx.toolResultPruner` 是可选的确定性裁剪服务，在范围选择**之前**运行：

1. 按头/中/尾裁剪超预算的当前工具结果文本（按 **Unicode 码点**切，不会劈开代理对）。
2. 每个替换保留完整事件数据、只改 `content`，引用被遮蔽节点让回放可恢复；
3. 替换前紧邻一条 `compaction/prune` 定价事件（经 token meter 计价阴影节点），纯消费者无需逐节点状态即可扣减。
4. `pruneSession` 在稳定 surface 快照上工作；失败的请求恢复走 `agent/request-error`，只有 surface 替换世代前进才返回重试动作。

摘要失败恢复走 **`compaction/summary-error` 瀑布**：监听器同步记录对选中输入的持久变更（如摘除图片），只有取得进展才返回 true；提供者随后重新推导、重新计价再重试。决定会跨后续摘要失败或取消存活。

## 图片卸载

`compaction-image-offload` 拥有 `image/offload` 事件及其纯消息投影：目标标识当前输入节点与精确的深度优先图片出现位置（`{ seq, imageIndexes }`），保留节点与消息身份、不带 `surfaceOp`。恢复策略与脱离回放归包 README。

## 请求上下文（packages/context）

模型可见的请求上下文由 `packages/context/` 组的插件在每次组装时贡献，作为 system-prompt 的段或注入消息：

| 能力 | ctx 键 | 作用 |
|------|--------|------|
| 工作目录 | `ctx.workingDirectory` | 一个 Session 投影拥有**有效执行目录**；文件系统校验变更，用户上下文报告变更，而原始项目身份与沙箱写根保持不变 |
| 文件引用 | `ctx.fileReferences` | 路径补全候选（只返回路径，provider 拥有命名空间访问与排序） |
| 会话引用 | `ctx.sessionReferenceResolver` | 把有界的当前 surface 会话快照投影为**持久的不可信消息上下文**；host 适配器拥有 @ 提及语法 |
| 工作区指令/时间上下文 | context 组插件 | 工作区说明、时间事实等固定段（见核心引擎篇 system-prompt） |

注入即记录：`agent.inject()` 落到下一次被准许的请求，源 `kind` 需在 `MessageSourceMap` 声明——"模型可见 ⟺ 已记录"在这里同样成立。

## 计量

缝本身**没有定价 API**：单例 `ctx.tokenMeter` 直接拥有估算与回放；`dsh-compaction-basic` 拥有保留、事件排序、路由摘要调用及其配置。计量与压缩的关系是"测量服务被压缩消费"，不是"压缩内嵌计量"。

## 设计启发

1. **log-only 事件 + 单一 surface 变更**：摘要的完整历史（锁、范围、用量、调用信封）都可从日志重建，唯一 surface 变化是一条 `user/message` replace——"模型可见 ⟺ 已记录"的最严格实践。
2. **锁是事件对而非互斥量**：crash 后的可检测孤儿锁比一个假"完成"事件诚实得多。
3. **位置 span 而非 seq 区间**：`shadowedRange.start` 可以大于 `end`（早先的替换把新摘要节点落到旧位置）——用 surface 位置思考，别用 seq 算术。
4. **压缩可插拔**：接口是缝，tokenizer/模板后端是兄弟包；`compactCheckpointSource` 构造器让消费者跨后端识别替换检查点。
