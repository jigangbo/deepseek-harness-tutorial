# 06 · 会话检索

> 精确读、过滤、追踪与全文搜索


> 「应用代码需要程序化地访问呈现给模型的那份历史」——这就是 `ctx.sessionQuery` 的存在理由。核心源码：`packages/session-query/session-query`（接口 + 逻辑语料）与 `packages/session-query/session-query-sqlite`（交付后端）。

## 缝的分工

```text
ctx.sessionQuery (接口包定义)
  ├─ 精确读 / 过滤 / 追踪 ── 任何存储设置都可用
  └─ 排名全文搜索 ────────── 需要具体后端（交付组合挂 dsh-session-query-sqlite）
        后端补充：全文调和、排名、片段、游标世代
```

接口包永远不单独挂载——服务由具体后端插件提供。**模型消费者拥有工作区权威与无游标渲染**；接缝不管 UI 呈现。

## 能做什么

| 操作 | 返回 |
|------|------|
| `listSessions()` | 每个逻辑会话（新→旧），带 `live`/`persisted` 可用性标志与可选 `formatStatus` |
| `readSession(id)` | 完整的回放校验原始事件日志，**不**让会话变活跃 |
| `filterSessions(filters)` | 匹配 AND 元数据与可用性谓词的会话 |
| `filterEvents(id, filters)` | 匹配元数据与字面文本谓词的语义事件文档 |
| `readTitleSnapshots(ids)` | 每会话最新折叠标题（绑定源 header） |
| `listEvents(id)` / `readSurface(id)` | 轻量逐事件记录，或完整当前模型 surface |
| `readEvent(request)` | 一个完整事件 + 围绕它的有界原始日志窗口 |
| `traceSession(id)` | 已知祖先链 + 递归后代树 |
| `traceEvent(request)` | 一个事件的位置替换与被引用源事件关系 |
| `searchSessions(request)` / `searchEvents(request)` | 全文搜索分页（由挂载后端实现） |

两条隐私/血统纪律：

- **无 body 记录只暴露 `SessionHeader.isSeeded`**——不读 body 就不知道 cwd 等元数据。
- 返回事件 body 的读（`readSession`/`readSurface`/`readEvent`）携带精确 `inheritedEventCount`，调用者无需从日志推断切点就能区分继承与自有事件。

## 过滤器语义

`SessionResultFilter` 按会话：id、可空 cwd、创建时间范围、可空 parent、来源可用性。`SessionEventResultFilter` 按事件：seq/时间范围、事件类型、surface、字面文本。

- 过滤数组 **AND**；同一子句内的列表值 **OR**。
- 空列表值匹配不到任何东西；范围闭区间。
- 畸形范围或未知封闭联合值 → `SESSION_QUERY_INVALID_FILTER` 失败（响亮失败原则）。

## 冷读缓存：revision 键控

架构上最有意思的部分——**读持久会话不用每次全量解码**：

```text
stat(id).revision (不透明变更令牌)
        │
        ▼
每 id 缓存一个"平衡过的冷 Session"
   （内存中用 interruptedTurnClosers 平衡中断日志，不写回）
        │ revision 未变 → 直接复用
        │ revision 变了 → 重新读
        ▼
会话列表读 header + 身份校验的投影缓存提示，绝不为此开冷日志
```

- 平衡是**内存中**的：查询层用与 agent-loop resume 相同的关闭事件补齐中断 turn，但**不写回存储**（持久化层不修复、读者自修相同逻辑的只读面）。
- 列表从不读 body：缓存或格式升级不会把启动变成 body 扫描。
- 一个构建读不了的高世代被 `SessionFormatUnsupportedError` 拒绝，而不是部分解读。

## 与投影缓存的关系

`session-projection-cache`（持久投影检查点）与 session-query 协作：列表的身份校验提示来自投影缓存，但两者职责分明——投影缓存服务**领域状态的加速水合**，session-query 服务**日志语料的检索**。

## 模型可见的工具面

`experimental/tool-session-query` 把检索能力暴露给模型（会话搜索工具）；`context/session-reference` 用同一缝做跨会话引用解析。consumers 清单：session-query → session-reference、tool-session-query。

## 设计启发

1. **接口/后端分离到了检索域**：精确读人人可用，全文搜索按后端能力降级——接口按"无后端也能诚实工作"的最小面设计。
2. **revision 令牌代替失效回调**：派生读模型自己决定何时重新计算，缓存与存储零耦合。
3. **一致观察的脱离克隆**：读不碰活会话、返回脱离克隆——检索层永远不会干扰运行中的 agent。
