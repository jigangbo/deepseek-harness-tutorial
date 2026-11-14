# 3.8 子代理与团队协作

> 一棵 agent 树的诞生


> 一个 agent 干不完的活，分给子代理。`ctx.subagents` 是子代理的 provider 缝——从"同进程新 agent"到"委托给外部产品"六种后端共存于一个接口之后。团队模式再往上加共享任务板。核心源码：`packages/subagent/`、`packages/team/`。

## 子代理缝

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-subagent`（`ctx.subagents`，含**激活管理器**） |
| Provider | `subagent-spawn-in-process`（同进程新 agent）、`subagent-fork-in-process`（fork 已有会话）、`subagent-dsh-sdk`（另一个 dsh 运行时）、`subagent-acp`、`subagent-codex`、`subagent-claude-code`（外部产品委托） |
| Consumer | `tool-subagent`（委派）、`tool-subagent-control`（跟进控制）、`experimental/tool-ralph` |

六种后端一句话概括差异：

```text
spawn-in-process:  同进程新建子 agent（最常见）
fork-in-process:   从现有会话分叉（继承事件前缀，isSeeded）
dsh-sdk:           委派给另一个 dsh 进程（SDK 协议）
acp / codex / claude-code:  委派给外部 agent 产品（都经 ctx.subprocess 孵子进程）
```

- **providers 准备本地子代理或执行外部任务；一个激活管理器统一拥有两种生命周期**——无论子代理活在哪个进程，激活记录、发现、中断、收件箱等待都是同一套词汇。
- 普通子代理控制拥有：身份、消息传递、发现、中断、收件箱等待。
- `tool-subagent` 还拥有 `ctx.subagentModelSelection`——默认关闭的设置命名空间，委派工具组合新顶层会话时采样。

## 会话谱系与持久化

子代理是**真的会话**：header 携带 `parentSession`（直接创建者）、`origin: 'subagent'`（粗分类，供导航隐藏重复子行）、`delegationDepth`（持久化的递归预算——重启不重置）。fork 创建时 `ctx.agents.create({ sessionId, seed, inheritedEventCount, meta: { isSeeded: true } })`，只有 agent-loop 发布的会话才会持久化。

## Agent Teams（team 组）

`ctx.agentTeams`（`dsh-agent-team`）在子代理树之上加**共享任务 DAG**：

- 一个根会话启用团队后，`ctx.agentTeams` 拥有这棵树的共享任务板；
- `tool-agent-team` 贡献任务控制工具（创建/认领/完成/依赖）；
- 普通子代理控制（消息/中断/等待）继续照常工作——团队是**增量的任务策略层**，不是替代品。

```text
根会话（启用 Teams）
  ├─ 共享任务 DAG ──── tool-agent-team 读写
  ├─ 子代理 A ── 认领任务 3 ── 完成后回报
  ├─ 子代理 B ── 认领任务 1（依赖任务 3）── 等待
  └─ 激活管理器：统一的身份/中断/收件箱词汇
```

## 委派的完整链路

```text
模型调用 tool-subagent({ backend, task })
  → ctx.subagents.select(backend) 校验
  → provider 准备子代理（进程内新建 / fork / 外部进程）
  → 激活管理器登记 → 子会话走正常 agent-loop
  → 结果/完成事件回到父会话（激活结果可被工作流等待）
  → tool-subagent-control 发后续跟进消息
```

工作流引擎的 `agent()` 调用同样扇出到 `ctx.subagents`——工作流章的编排原语与这里的委派共享同一缝。

## 设计启发

1. **生命周期统一，执行位置自由**：激活管理器把"进程内 agent"与"外部产品进程"纳入同一词汇——上层（工具、工作流、UI）不知道也不关心子代理跑在哪。
2. **血统进 header，不进运行时**：深度、父会话、origin 全部持久化——重启后的谱系查询（session-query 的 traceSession）依然准确。
3. **团队是策略层**：任务 DAG 是可选叠加，基础委派原语保持简单。
