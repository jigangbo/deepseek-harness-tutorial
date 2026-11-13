# 终端与 PTC 运行时 — 持久 PTY 与沙箱化程序执行

## 一、终端（terminal 组）

`ctx.terminals` 管理**持久 PTY 会话**——与一次性 bash 执行相对的"活着的 shell"。

| 包 | 角色 |
|----|------|
| `terminal` | Service Definition：PTY 会话注册表（owner 作用域） |
| `terminal-bash` | Provider：本地 shell 后端（经 `ctx.subprocess` 孵 PTY、经 `ctx.sandbox` 围栏） |
| `experimental/tool-terminal` | Consumer：owner 作用域的模型工具 |

设计分工：

- **注册表拥有精确 agent 的会话身份与清理**——终端绑定到特定 agent，agent 消失则终端清理；
- **后端拥有终端机制**——PTY 生命周期、屏幕恢复（有界的 screen recovery 由 `api-terminal-controller` 负责）；
- **工具只暴露 owner 作用域的操作**——一个 agent 无法触碰另一个 agent 的终端。
- 用户终端进程由 `api-terminal-controller` 经子进程 provider 与类型化 Remote 传输管理（Host 侧）；模型终端与用户终端是两个面。

后台终端输出进入 `ctx.jobs`（后台任务章）——PTY 的"运行中"语义与作业的"可观察/可终止"语义共享一套记录。

## 二、PTC 运行时（ptc-runtime 组）

**PTC（Programmatic Tool Calling）**：模型不逐个调工具，而是写一段**程序**，程序里 `await tools.read_file({...})`、`await tools.bash({...})` 批量编排。这是 dsh 最有特色的能力之一。

### 缝的三角色

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-ptc-runtime`（`ctx.ptcRuntime`） |
| Service Provider | `dsh-ptc-runtime-node`（沙箱化 Node，交付实现）、`experimental-ptc-runtime-python`（实验 Python） |
| Consumer | `tool-executor-ptc`（PTC 模式的工具执行器）、`workflow-ptc`（工作流编排引擎） |

### 一次运行：请求进，结果出

```ts
interface PtcRunRequest {
  program: string          // 运行时语言；async function body 支持 top-level await/return
  stateOwner?: object      // 可选内存状态的同一性 owner；保留状态永不持久
  bindings: PtcBindingNamespace[]   // 暴露给程序的宿主函数，每命名空间一个全局对象
  cwd?: string             // 挂载文件系统与子进程执行世界中的工作目录
  timeoutMs?: number | null // 缺省=provider 默认；null=无时限；数字=封顶预算
  sandboxPolicy?: SandboxExecutionPolicy  // 无围栏的 provider 拒绝显式策略
  signal?: AbortSignal     // 硬停（甚至循环中途）；在途 binding 调用由调用方结算
}
```

`resolve` 阶段校验支持的选项并施加部署默认；`run` 收到带**显式目录与截止选择**的 `PtcRunSpec`——run 永不补默认值（显式优于隐式原则的又一实例）。不支持的选项在**执行前**拒绝。

### 类型化返回契约

- 程序里 `tools.<name>(args)` 对每个**可见注册工具**自动可用（无需额外集成）；
- `ToolArgsMap`/`ToolOutputMap` 从同一 schema 推导精确类型；
- 调用重新进入正常执行管线（策略照走）；
- 成功 = 策略后的**最终规范 JSON 值**；失败 reject 真 `ToolCallError`——程序只能读 `name`/`toolName`/`message`，看不到内部错误码或失败联合。

### 沙箱化 Node 交付 provider

交付的执行 provider 是**沙箱化 Node**（架构决策记录 `2026-09-11-sandboxed-node-ptc-runtime`）：程序跑在受限 Node 运行时里，只能看到 `bindings` 显式暴露的宿主函数——**边界即能力**：没给的网络/文件 API 就不存在，而不是"存在但被拦"。

### PTC 与 native 双执行器

工具调用有两个执行器（见核心引擎篇 executors 章）：

```text
native:   模型逐个发 tool/call → 每次一个往返
PTC:      模型写一段程序     → 一次往返内多次工具调用 + 控制流
```

PTC 模式下**每个可见工具免费可用**——这是工具契约设计（规范 JSON 返回、不可解析 prose）的回报。

### 与 workflow 的关系

`workflow-ptc` 是 `ctx.workflowEngine` 的交付引擎：工作流复用 PTC 的程序执行（agent() 调用经 `ctx.subagents` 扇出），但拥有更长的编排生命周期（见后台任务与工作流章）。

## 设计启发

1. **owner 作用域的资源身份**：终端、内存状态（stateOwner）都绑定到显式 owner——泄漏与清理有明确责任人。
2. **边界即能力**：沙箱化 Node 通过"未暴露 = 不存在"实现隔离，比"存在但拦截"更强。
3. **一次运行、显式默认**：resolve/run 两阶段让部署默认显式化且在执行前校验——运行中永不出现"隐式补默认"。
