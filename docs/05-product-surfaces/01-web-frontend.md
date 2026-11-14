# 第 35 章 Web 前端架构

> 浏览器里的第二个 Cordis


> Web Client 是一个**浏览器侧的 Cordis 应用**：由独立加载的插件组装而成。它与 Host 共享插件化世界观，但没有任何特权核心——UI 是从日志与投影**派生**出来的视图。

## 六层所有权

| 层 | 主要包 | 职责 |
|----|--------|------|
| Host 应用 | 业务服务 + `packages/api/*-controller` | 权威状态、持久化、变更排序、访问策略、流生产 |
| 传输与 API 组装 | `client/connection`、`api/gateway`、`api/remotes` | 建立连接代、暴露生成的 `ctx.remote` 方法与流、转发选定事件、携带取消与结果 |
| Client 模型 | `api/session-controller/client`、`api/workspace-controller/client` | Host 状态的**无 React 镜像**、流/一元竞态、对象身份与订阅 |
| UI 适配器 | `client/ui-session`、`client/ui-workspace` | 模型 observable → Slot 源；视图级导航与状态策略 |
| 会话数据 | `client/ui-conversation`、`ui-chat`、`ui-trajectory` | 标准事件 + 历史 Assistant 运行 → 独立目标快照 |
| 组合与渲染 | `client/ui-slots`、`ui-renderer`、`ui-layout`、各 feature 包 | 声明扩展位、派生 props、绑 observable 到 React hooks |

依赖方向严格单向：**Host 状态 → Remote 传输 → Client 模型 → UI 适配器 → 会话/呈现 → Slots → React**。用户动作经回调原路返回。**呈现组件永不接触 Cordis `ctx`、传输对象或另一个 feature 插件的实现。**

## 浏览器启动序列

```text
1. Host 把组合好的 WebBootGraph 写到 window.__DSH_BOOT__
2. 解析器预加载脚本执行前安装浏览器模块加载器门面
3. 模块系统 = 惰性 CommonJS 表：bundle 注册 factory；
   物化条目 = 用同步 require 跑 factory（平台模块 + 声明的动态依赖）
4. boot 内核：创建模块系统 → 预取 immediately 条目
   → 挂载 vendored Cordis Loader → 创建每个图条目
5. 全名册稳定 + boot 审计通过（浏览器必需清单上的非活跃条目拒绝启动；
   其他非活跃条目一条 console 警告）
6. ui-renderer 水合无框架 boot DOM，调用唯一的上下文级 renderSlot('root')
```

**Cordis 服务注入决定激活；模块图顺序只决定同步 import 能否物化**——两个顺序系统各管一摊。

## 远程通信

- Host 业务服务用 Typert Remote 装饰器标注可调用方法；Host 生成严格描述符、运行时编解码器、声明合并与 source map。
- Client 侧 `api-remotes` 装配选择这些生成贡献，挂到 `ctx.remote.<namespace>` 与会话作用域 `agentCtx.remote.<namespace>`。**feature 包依赖生成的服务面，不依赖 Gateway 实现或 Host 包的运行时入口。**
- Connection 拥有请求 URL 解析、关联、`/api` 载体、信任检查、精确 Fetch 路由与**连接代（generation）**。
- API Gateway 拥有 Remote 派发、取消、逻辑流与选定 Host 事件转发。
- `$events` 逻辑流是连接代之源：首个 `ready` 帧携带 Host home 并在 Host 监听器挂好后、任何控制器基线读之前建立代。
- `ctx.remote.$on()` 把白名单普通事件投到根 Client Context，作用域瀑布事件投到解析后的 Session Context——瀑布监听器返回结果、调用 `next()` 或拒绝（与主进程同样的纪律）。

## Client 模型：会话与工作区

**Sessions**（`ClientSessions → SessionManager → Session` 三级）：

- `ClientSessions`：拥有引用、来源计数、Session 作用域、稳定 `SessionBinding`；投影目录状态而**不选全局当前会话**。
- `SessionManager`：列表基线、活更新、惰性 Session 实例、投影 store、拉取与后续更新的冲突排序。
- 每个 `Session`：一个连续逻辑事件窗口（`SessionEventLikeEntry`）、翻页、follow、prompt/控制状态、适配器消费的 observable 快照。

持久事件路径：`follow()` 首帧含当前 header、尾页、游标与**完整投影基线**；历史记录有显式 `event`/`chunks` 判别符；每个物理代**原子替换**保留窗口，之后活事件按 seq 追加；`page()` 专供更旧历史与缺口修复。

**Workspaces**：Host 拥有变更策略与权威 follow 馈送；`ClientWorkspaceModel` 拥有浏览器行、排序、归档/置顶数组、命令回声与竞态消解。每个流代以完整基线开始，然后 `upsert`/`remove`/`order`/`archived`/`pinned` 增量；重连从新基线整体替换。

> 这不是第二份业务真相：Host 控制器决定持久状态与变更结果；Client 模型维护"最新可用的本地投影"，并编码延迟响应与替换基线如何合并。

## 会话与呈现

- `ui-session` 安装 Session 作用域适配器（`useSessions`/`useSession`/`sessionId`/`useProjection`…）；`SessionProvider` 继承外层绑定或绑定显式引用——**并发子树可以指向不同会话**。
- `ui-conversation` 对每个 `SessionBinding.eventSource` 绑定一次：事件注册表把持久会话事件与 Client-only `assistant/live-chunk` 关联成稳定业务 Context；视图注册表物化目标快照。
- Chat Assistant、Trajectory Assistant、Turn Tail 同时解释**活 chunk 与持久结算里嵌入的压缩流**——重连与翻页历史复现相同 Assistant 状态，无需持久 token 行。
- `ui-chat` 与 `ui-trajectory` 注册独立 Definition：可以解释同一事件族，但**不导入不共享彼此的最终显示模型**。
- `ui-slots` 提供类型化注册表与生命周期台账；**`ui-renderer` 是唯一**经 `useSyncExternalStore` 绑定裸 observable、拥有 React contexts、渲染根树的包。

## 六条数据路径

| 路径 | 序列 |
|------|------|
| 持久会话显示 | Host 日志 → follow/page 历史 → Client 事件窗口 → Conversation Context → 目标快照 → Slot → React |
| 瞬时会话控制 | 控制基线 → Remote 快照流 → 投影 store → 快照 → hooks → 组件 |
| 后台任务 | job 注册表 → job.list/follow → 客户端花名册 → 列表与面板 |
| 工作区状态 | 基线 + 增量 → ClientWorkspaceModel → useWorkspaces → 侧栏 |
| 作用域交互 | Host 瀑布 → `$events` → `ctx.remote.$on()` → UI 包 → 结果或 next() |
| 用户命令 | 组件回调 → 注入面或 Slot owner → Client 服务/Remote → Host 控制器 → 权威更新 → 流回投 |

## 重连：物理与逻辑分离

- Gateway mux 恢复**物理** WebSocket；每个 `RemoteStream` 在连接发布可用代时重开自己的**逻辑**源。
- 载体失败可重试；业务错误、畸形开头项或协议违规对该逻辑流**终局**。
- 恢复跟随数据语义：持久日志验证 seq 范围并从每代开头快照替换窗口；控制/工作区流断连时保留最后值再原子替换；普通转发通知**不重放**（有状态域需要基线/游标/显式查询）。
- **没有**大一统的 Client `Runtime`、`events.mux` 或万能 `resync()` API。

## 包边界纪律

- feature 包之间：`import type` 共享声明；**禁止**运行时 import 或 re-export 别的 feature 包的值；跨包行为走注入服务，跨包 UI 走 Slots。
- 共享运行时值需要**无 feature 生命周期的窄静态 owner**（`client/store`、`ui-primitives` 等）。
- 传输与生成的 API 装配可以 import 运行时贡献——组装协议就是它们的显式职责。

## 设计启发

1. **浏览器里重放同样的架构**：插件、服务注入、生成 API——Web Client 不是"一组 React 页面"，而是 Cordis 世界观在浏览器的完整投影。
2. **基线 + 增量的流协议**：每个流代以完整基线开始——重连的正确性不靠记忆，靠代边界。
3. **派生式 UI**：客户端从不折叠领域事件、从不拥有业务真相；它把日志+投影渲染出来，重连/回放天然正确。
4. **槽（Slots）做跨包 UI**：类型化注册表 + 生命周期台账代替组件 import——feature 包互相不可见，但能在同一棵树共存。
