# 03 · 命令执行与沙箱

> shell、subprocess、sandbox 三缝联动


> 让模型安全地跑命令是 Agent Harness 最危险也最核心的能力。dsh 把它拆成三层缝：**执行器语义**（shell）、**进程原语**（subprocess）、**围栏**（sandbox）——层层叠加，各自可替换。

## 三层结构

```text
模型工具 tool-bash / tool-pwsh / tool-terminal
     │ ctx.shell（执行器缝：bash/pwsh 语义、退出码、输出）
     ▼
┌──────────────────────────────┐
│ bash-local  / bash-sandbox / pwsh-local   ← 执行器实现
└──────────────┬───────────────┘
               │ ctx.subprocess（进程原语缝：生成/进程树/stdio/终止升级）
               ▼
┌──────────────────────────────┐
│ subprocess-local / subprocess-ssh          ← 进程后端
└──────────────┬───────────────┘
               │ ctx.sandbox（围栏缝：包装待执行 argv）
               ▼
┌──────────────────────────────┐
│ sandbox-local（bwrap / Landlock / Seatbelt）/ sandbox-ssh
└──────────────────────────────┘
```

- **subprocess** 拥有进程坐标、树/会话生命周期、stdio 处置、终端机制与 kill 升级。
- **sandbox** 的消费者交出"即将 spawn 的精确 argv"，同世界后端按每调用策略包装它并报告执行情况——沙箱不是装饰，是 argv 的包装函数。
- **shell** 的消费者是模型工具与 hooks 桥；沙箱化、远程、PowerShell 执行器替换 `bash-local` 时它们一行不改。

## 环境与共享执行世界

`ctx.ssh` 展示了"换一个 subprocess 后端 = 整个执行世界搬家"：

```text
ctx.ssh (一条 OpenSSH 连接的所有者)
  ├─ fs-ssh          远程文件
  ├─ subprocess-ssh  远程进程
  └─ sandbox-ssh     远程围栏
→ Bash、PTY、LSP 全部随行
```

`ctx.shellEnv` 是受管 bash 环境注册表：插件声明 effect 作用域的 `DSH_*` 事实；每个 shell 工具在每次执行时收集一份受信快照，执行器重建命名空间。

## 沙箱策略的唯一家

`ctx.sandboxPolicy`（`sandbox-policy` 包）：

- 部署默认模式 + 工作区根的**唯一**来源；
- 只有沙箱化执行器（`bash-sandbox`、`fs-sandbox`、`terminal-bash`）读服务本体；
- 工具层用纯 `sandbox/mode` 折叠（服务同时导出）；
- **bash 与 fs 不会围到不同的根**——因为它们读同一个家。

## 终端（terminal 组）

`ctx.terminals`（`dsh-terminal`）+ `terminal-bash` 后端 + `experimental/tool-terminal` 工具：

- 注册表拥有**精确 agent 的会话身份与清理**；后端拥有终端机制（PTY）。
- terminal-bash 经 `ctx.subprocess` 孵 PTY，经 `ctx.sandbox` 围栏（读同一策略家）。
- `tool-terminal` 只暴露 owner 作用域的模型工具——一个 agent 碰不到另一个 agent 的终端。
- 后台终端输出成为 `ctx.jobs` 的生产者（见后台任务章）。

## 模型工具全景（执行侧）

| 工具 | 缝消费 | 备注 |
|------|--------|------|
| `bash` | ctx.shell + ctx.jobs + ctx.approval + ctx.sessionPersistence | 三个包的生产级示例（tool-bash） |
| `pwsh` | ctx.shell（pwsh-local） | Windows PowerShell 变体 |
| `terminal_*` | ctx.terminals | 持久 PTY 会话 |
| `job_*` | ctx.jobs | 后台作业控制 |

审批在 `tools/pre-execute` 拦截；`tools/execute` 的 deadline 强制器是 guard 组的一部分（见核心引擎篇 tools 章）。

## hooks 桥（experimental）

`hooks-claude-code`、`hooks-codex` 消费 `ctx.shell` 与 `ctx.sessionPersistence`，把外部工具生态的 hook 协议接到 dsh 扩展点上——第三方生态接进来的第一个实例。

## 设计启发

1. **三层各答一个问题**：shell 答"bash 语义"，subprocess 答"进程怎么生灭"，sandbox 答"argv 如何被围"——任何一层都可以单独替换（远程执行、Windows、新围栏机制）。
2. **策略单一家**：沙箱模式与根只有一份，多个执行家族读同一处——配置漂移在结构上不可能。
3. **argv 包装式沙箱**：消费者交出精确 argv、后端包装并报告——沙箱的适用范围与局限（"围 argv，不围意图"）在接口上就是透明的。
