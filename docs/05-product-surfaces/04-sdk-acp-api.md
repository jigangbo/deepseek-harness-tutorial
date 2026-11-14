# 第 38 章 SDK、ACP 与远程 API

> 从外部进程驱动 Harness


> dsh 支持三种"外部世界接入"形态：TypeScript/Python SDK（JSON-RPC over stdio）、ACP（自动化 Agent Client Protocol）、远程 BFF API（Typert RPC 网关）。三者共享同一个核心运行时，只在协议层分叉。

## 三形态对照

```text
                         ┌─────────────────────────────┐
   TypeScript SDK ──────►│                             │
   （spawn dsh 子进程）  │   完整 Harness 运行时        │
   Python SDK ──────────►│   （profile: sdk /          │
   （捆绑运行时 wheel）  │     sdk-minimal）           │
                         │                             │
   ACP 自动化客户端 ────►│   profile: acp（仅自动化）  │
                         │                             │
   浏览器 ──────────────►│   Host Web 应用 + BFF 网关  │
                         └─────────────────────────────┘
```

| 形态 | 传输 | profile | 典型场景 |
|------|------|---------|----------|
| TypeScript SDK | 换行分隔 JSON-RPC over stdio | `sdk` | 把 agent 嵌进 TS 应用/CLI |
| Python SDK | 同一协议；运行时 wheel 打包 dsh CLI 为平台产物 | `sdk`（最小示例用 `sdk-minimal`） | Python 应用集成 |
| ACP | Agent Client Protocol 服务器 | `acp` | 编辑器/自动化框架的标准化接入 |
| 远程 API | Typert RPC 网关（Host 内） | `web` 组合的一部分 | 浏览器 UI 的后端 |

## SDK 家族（packages/sdk）

| 包 | 角色 |
|----|------|
| `protocol/` | 线协议：换行分隔 JSON-RPC 传输 + 命名的请求/结果/通知类型 |
| `client/` | TypeScript 客户端：spawn 运行时子进程，经高层与协议层 API 驱动 agent turn |
| `server/` | `jsonrpc` 插件：经 stdio 服务带外 SDK 客户端 |

要点：

- **一个协议、两个官方客户端**（TS 与 Python 说同一协议）。
- 客户端能打开会话、发送 prompt、观察会话事件、agent 状态变化与子代理完成。
- SDK 包**不创建开发者项目、不定义另一个应用**——它们驱动一个正常的 Harness 运行时。
- Python 的应用架构与 TS 同构：运行时 wheel 把常规 `dsh` CLI 打包为 `deepseek-harness-sdk-runtime-<platform>-<arch>`，客户端默认以显式 Harness home 启动 `dsh --profile sdk`；Python 暴露 profile 选择与有序 patch 文件，**而不是完整的 Cordis 树**；持久外部插件通过 `dsh plugin` 安装。
- `sdk-minimal` 是仓库拥有的独立 bundle：一个 bundle 拥有完整显式 SDK 树、**不叠 dsh-base**——需要最小面时用它。

## ACP（packages/acp）

仅自动化的 Agent Client Protocol 服务器：

- 为自己的 agent 提供**一次性机器审批决策**（审批缝的监听器——见交互与审批章）；
- 消费 `ctx.agents` 与 `ctx.approval`，把编辑器侧的标准化协议翻译进 dsh 事件；
- ACP 行为的录制快照（`snapshots/acp/`）是测试政策的一部分。

## 远程 API（packages/api）

`api/` 组是 Host 进程内的 BFF 层：

- **Typert RPC 网关**（`api-gateway`）：把生成的 Remote 描述符关联到活的 Cordis 服务，解析注册身份，经共享 Connection RPC 载体暴露一元调用。
- 各控制器包（`api-session-controller`、`api-settings-controller`、`api-workspace-controller`、`api-terminal-controller`、`api-job-controller`、`api-account-controller`）把领域缝**投影**到生成的 Remote 命名空间——注意职责位置：拒绝分类、脱敏读、批处理扇出都住在控制器，不在缝的 Definition 上。
- `api-remote-access` 决定非 owner Peer 可以调用哪些 Remote 方法（业务注册的角色策略），过滤列表结果，流式发送每个 Peer 的访问快照。
- `ctx.connection`（client-connection 包）拥有浏览器认证与共享 HTTP 请求派发。

## 客户端半边（packages/client）

浏览器侧由 `packages/client/` 组承担（60+ 个 ui-* 插件）：

- `connection`（认证传输）、`store`（客户端状态）、`locale`（类型化字典 + `t`——**UI 文案必须走 locale，硬编码文案被 `verify-client-ui-i18n` 拒绝**）、`modules`（`__DSH_BOOT__` 入口图组装）、`resources`。
- 每个产品能力一个 `ui-*` 插件（ui-chat、ui-session、ui-settings、ui-tool…），从 `session/event` 与投影快照**派生渲染**——客户端不折叠领域事件。
- 工具卡片：Client 在 `tool.call.toolview` 键控槽注册 wire 工具名，从原始 `tool/call`、`tool/result` 事件（含持久 `result.meta`）本地验证并派生组件 props；畸形输入回退通用行。

## 会话控制流（Host 侧）

`api-session-controller` 拥有会话命令、冷读、持久事件**跟随**（`session/page` 与 `session/follow`——原始事件流 + 投影推送帧）、活控制状态、模型目录、工作区打开与 Agent 激活策略。UI 收到的是两类东西：**日志事实**（session/page 翻页）与**活增量**（follow 推送）——这正是"持久事实 vs 活扩展点"事件二分在产品面的投影。

## 设计启发

1. **协议与载体分离**：JSON-RPC/ACP/HTTP 三种协议、三种载体，一个运行时——每加一种接入形态，成本只有协议适配层。
2. **控制器投影缝，缝保持纯粹**：批处理、脱敏、拒绝分类这些"产品语义"住在 api 控制器；缝的 Definition 保持领域纯净——同一缝可以被 CLI、SDK、UI 以不同产品语义消费。
3. **SDK 是运行时的遥控器，不是第二个运行时**：所有启动纪律（profile + patch）原样适用于 SDK 路径。
