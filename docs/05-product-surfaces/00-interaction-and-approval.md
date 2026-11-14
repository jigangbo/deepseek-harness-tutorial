# 第 34 章 人机交互、审批与提问

> 模型何时要问人


> Agent 自主性与安全的平衡点：哪些动作必须问人、怎么问、答案如何审计。本章覆盖 interaction 组的四个部件：审批（user-approval）、提问（user-questions）、权限预设（permission-presets）、人类命令（commands）。

## 审批缝：may this action proceed?

`dsh-user-approval` 回答一个具体问题：**这个特定动作是否可以继续？** 它拥有共享的请求/结果词汇、`ctx.approval` 派发服务、`approval/request` 应答瀑布、仅审计的事件对、以及每会话 `ask`/`never` 策略。

```text
工具执行前 (tools/pre-execute 等)
     │ ctx.approval.request({ agent, toolName, callId?, reason?, signal? })
     ▼
  策略检查（在服务内部、瀑布之前！）
     ├─ 'never'  → 确定性 'rejected'，任何应答器都看不到请求
     └─ 'ask'    → approval/request 瀑布
                     UI 应答器 ──► 人点按钮
                     ACP 桥   ──► 自动化一次性决策
                     无人应答 ──► 'unavailable'（失败关闭）
     ▼
  落审计事件对：approval/asked + approval/decided（log-only，不进模型转写）
     ▼
  返回封闭结果：'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'
```

### 失败关闭（fail closed）是底线

`ApprovalOutcome` 是封闭联合，调用方在 `rejected`、`cancelled`、**`unavailable`** 上都拒绝——`allowed-once` 只授予被问的那个动作。缺席的、非本主、抛错的、返回不合词汇值的应答器一律归一化为 `unavailable`，**而不是打开闸门**。

### 请求身份与审计

- 每次请求一个全新 `ApprovalRequestId`（品牌化类型），把 `approval/asked` 与 `approval/decided` 配对——审批 id 与工具调用/会话 id 不可混用。
- **审计事件对只进日志、不进模型转写**；模型可见的行为是调用方派生的工具结果 + 当前运行时上下文快照。
- `request()` 要求会话**处于打开的 turn 内**（审计对必须被持久日志的提交/回放边界包住）；空闲 ask 在追加任何东西之前就拒绝。两个审计事件有一个没提交就整体拒绝——返回一个未记录的决定会破坏事件对的完整性。

### 请求体刻意不带工具参数

`ApprovalRequest` 足以路由与审计（agent、toolName、可选 callId、reason），但**刻意省略工具参数**：应答器通过 `callId` 把提示附到**已经流式展示过的**工具调用上，而不是渲染第二份可能漂移的副本。`signal` 中止即撤回问题（立即 `cancelled`，迟到的答案被丢弃）。

### 每会话策略

```ts
type ApprovalPolicy = 'ask' | 'never'
```

- 有效值 = 会话日志中最后一条 `approval/policy` 事件，回退到服务配置；`setApprovalPolicy()` 是唯一写路径，回放可重建覆盖。
- `never` 的典型场景：CI/无人值守的严格 headless 立场。
- 策略切换对模型可见：两种策略都把完整当前含义贡献给缓存安全的运行时上下文快照；`setPolicy(agent, policy)` 切换活 agent 并为下一步排队转换通知。
- `never` 在**服务内部、瀑布派发之前**强制——即使事后有人 `prepend` 注册应答器也无法绕过。

### 监听器即应答者

`approval/request` 是作用域过滤的瀑布（`dsh-scope`）：agent 作用域监听器只收到自己 agent 的请求。监听器要么返回结果认领请求，要么 `next()` 委托；**第一个答案占据唯一的决定槽**。

## 提问缝：模型主动问人

`ctx.userQuestions`（`dsh-user-questions`）与审批方向相反——**模型/工具主动向人提问**：

- UI 前端提供活跃的人类应答 provider；
- `dsh-tool-ask-user` 让工具调用在 provider 中立的 `ask()` promise 上**暂停**，等人回答后继续；
- 缝本身不知道任何 UI——Web、ACP 各自注册自己的 provider。

审批 vs 提问的对照：

| | 审批 approval | 提问 user-questions |
|---|---|---|
| 谁发起 | 调用方（工具策略/守卫） | 模型经由 ask-user 工具 |
| 回答谁 | UI 应答器 / ACP 桥（监听器） | UI 注册的 provider |
| 结果词汇 | 封闭四值，失败关闭 | 开放答案 |
| 审计 | log-only 事件对（必审计） | 由实现决定 |

## 权限预设：一键换安全姿态

`ctx.permissionPresets`（`dsh-permission-presets`）是用户可见的预设表，把两个旋钮打包：

- `workspace-write` —— 沙箱内工作区写 + 默认审批策略；
- `danger-full-access` —— 全访问（危险，显式确认）。

切换预设**写一条 `permission/preset` 事件**，穿过并同时更新两个底层旋钮事件（沙箱模式 + 审批策略）——一次用户操作，一条持久事实，两个旋钮保持同源。

## 人类命令（ctx.commands）

`dsh-commands` 注册**直接的人类命令**——不经过模型：`/compact`、`/plan`、`/feedback` 这类斜杠命令由 `ctx.commands` 派发，不发模型请求。与模型可见工具的分界线：命令改变宿主状态或触发流程；工具是模型为了完成任务而调用的能力。

## 与工具管线的位置关系

```text
tool/call
   → tools/pre-execute   ← 审批策略在这里拦截（allow/deny/ask）
   → tools/execute
   → tools/post-execute
   → tool/result
```

审批是 pre-execute 阶段最典型的消费者；`ctx.tools.guard()` 提供不可撤销的最终单调 deny。全链路见核心引擎篇 tools 章。

## 设计启发

1. **失败关闭的封闭联合**：安全语义只有一个授权值（`allowed-once`），其余全拒绝——类型系统直接表达安全立场。
2. **审计对 + 开 turn 约束**：决策与理由成为持久事实，且必须落在日志的提交边界内——审计不是旁路文件，是日志的一部分。
3. **参数不复制**：审批请求带 `callId` 而不带参数，UI 永远只有一份工具调用展示——漂移无从发生。
4. **策略在派发前强制**：可扩展性（瀑布）与不可绕过性（服务内检查）分层放置。
