# 6.3 扩展实战：添加一个模型可见的工具


> 目标：从零写一个 `read_file` 风格的工具插件，理解工具契约、策略挂点、PTC 免费接入、UI 卡片与验证要求。规范细节以主仓库 `docs/cookbook/adding-a-tool.md` 为准；生产级三包示例是 `packages/shell/tool-bash`。

## 最小可用形状

一个工具插件 = 一个标准 Cordis 插件 + `ctx.tools.register(...)`：

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // 模型看到的就是这句
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // 不写 required 即可选
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args 已按 schema 校验并推导类型: { path: string; limit?: number }
      // exec.signal 是取消信号
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

三个立即生效的机制：

1. **注册即 effect**：插件 fiber 被释放（卸载/HMR）时工具自动注销。
2. **schema 自动进提示词**：注册后工具 schema 在下一次 step 组装时进入模型请求，无需手工拼接。
3. **args 已校验**：`defineTool` 在 `execute` 前对模型生成的 `arguments` 按统一 `ParameterSchemaSpec` 校验（类型、required、字面量约束、exact-one 联合、嵌套值），所以 `execute` 内的 args 类型就是 `InferArgs` 推导值。

## execute() 契约八条铁律

1. **参数校验帮你做了，但没全做**。DSL 表达不了的约束（非空字符串、正数、跨字段规则）仍要自己查。直接注册原生 JSON-Schema 的工具自己负责输入校验。
2. **注册借走你的只读定义**。同进程类型化贡献不是序列化边界——注册后不要改 schema 或换回调。要热替换工具：dispose 拥有的 effect，重新注册。
3. **执行身份受保护**。registry 把 `arguments` 物化成一次递归的 detached lossless JSON，在策略开始前冻结，并分配不透明 `exec.token`；`callId`、`name`、`arguments`、`agent`、`token`、caller 持有的 `signal`、可选的 `parent` 在派发全程不可变。`args` 当只读输入。
4. **返回一个规范 JSON 值**。`output.schema` 用 `ValueSchemaSpec`（对象/数组/标量/null 根都行）；`execute` 只返回推断值，registry 快照为 lossless JSON、校验、冻结，再交给 `output.render(args, value)`。**不要**在返回体里塞内容块，也不要让调用方去解析散文里的 id。
5. **抛错或非法返回 = `isError`**。基础设施失败用 throw；领域上的"不理想但成功"（如进程非零退出）放进规范值，由渲染层解释。
6. **尊重 `exec.signal`**：触发时取消在途工作。
7. **（可选）用 `presentationMeta` 投影持久卡片数据**：从同一规范值派生可回放 JSON，核心持久化在 `tool/result` 上——需要结果期事实的卡片（如 write/edit 的应用 hunk）因此可在回放中复现。
8. **（可选）用 `exec.agent` 做异步通知**：`agent.inject({ content, source: { kind: 'my-plugin' } })` 追加**下一次**模型请求可见的持久上下文——不是唤醒（空闲 agent 保持空闲）。`kind` 要通过 `MessageSourceMap` 声明（V4 会话格式拒绝旧 `{ kind: 'plugin', plugin }` 包装），并对已释放 agent 做防御。

## 长任务：接后台作业运行时

需要 `run_in_background` 时，不要自己管理线程：

```ts
const handle = await ctx.jobs.start({
  kind: 'bash', label: '…', owner: exec.agent,
  run: async (job) => { /* push 到 job 的输出环 */ },
})
// 成功分支返回类型化句柄: { kind: 'background', jobId }
```

- 一旦 `ctx.jobs.start()` 发布了 job id，改用 **task 自有的取消信号**而非 `exec.signal`——外层调用取消只是不再等待，不杀已发布的工作；`job_kill`、owner 释放、服务 teardown 才拥有那个生命周期。
- 前台工作保持耦合 `exec.signal`。
- PTC 模式绝不解析散文恢复 id，只读规范值里的 `jobId`。

## 策略与观测：别把部署策略烧进工具

六个挂点各有用途（详见核心引擎篇 tools 章）：

| 挂点 | 用途 |
|------|------|
| `tools/pre-execute` | 可扩展的 allow/deny/ask 策略（权限门示例见 extension-cookbook） |
| `ctx.tools.guard()` | 最终单调 deny，后续监听器无法撤销 |
| `tools/execute` | 包裹派发：deadline、重试、指标 |
| `tools/post-execute` | 替换呈现内容/返回值、拦截结果、附加模型可见上下文 |
| `tools/result` | 观察不可变的规范化结果 |
| 工具自身 executor | 沙箱等机制可在实现内做 |

## PTC 模式免费接入

每个可见注册的工具自动成为 PTC 程序里的 `await tools.<name>(args)`：

- 生成的 `ToolArgsMap` / `ToolOutputMap` 从同一 schema 推导精确类型；
- 调用重新进入正常执行管线（策略照走）；
- 成功 = 策略后的**最终规范 JSON 值**；失败 reject 真 `ToolCallError`（程序只能读 `name`/`toolName`/`message`）。

所以：把 `output.schema` 设计成好用的**程序化 API**——直接返回句柄与字段，诚实使用标量/数组/null 根，人类解释留给 `output.render`。

## UI 卡片：呈现与执行解耦

`output.render` 是给模型的；**UI 卡片**走纯函数投影 `presentCall` / `presentResult`，返回 `card` 标签的渲染意图：

- `generic`（默认）——可带 `kind` 图标与 `locations`（编辑器跟随跳转）；
- `terminal` ——调用即 shell 命令（tool-bash）；
- `diff` ——创建/修改文件（tool-fs write/edit）；
- `read` / `search` / `web` ——由持久化 `result.meta` 重建的完成态视图。

两条硬规则：

- **纯函数**：它们同时在直播流与会话日志回放上运行——禁止 I/O、禁止读会话状态、禁止时钟/随机。想拿"文件的旧内容"？那是持久化元数据或 UI 适配器的事。
- **UI 专用格式不进模型结果**：```console 代码块、diff、相对化路径都不该为了 UI 进规范值。`defineTool` 对显示路径做软校验：畸形历史参数返回 `undefined`（退回通用卡片）而不是抛错——**显示永不弄崩回放**。

Web Client 不消费 `presentCall/presentResult`：Client 插件在 `tool.call.toolview` 键控槽注册 wire 工具名，从原始 `tool/call`、`tool/result` 事件（含 `result.meta`）本地派生组件 props。

## 验证清单

- 单元：注册/注销 effect、args 校验、取消、错误路径。
- 真实组合：通过 Loader 启动含此工具的 `cordis.yml`，mock 模型断言请求里出现你的 schema、工具结果正确落日志。
- **快照**：模型可见的非平凡变更必须新增/更新 keyless 录制会话场景。
- UI 卡片改动加 Web 快照；检查 `presentationMeta` 的回放等价性。

## 常见坑

| 坑 | 后果 |
|----|------|
| 在 `execute` 里解析自己的 prose 找 id | PTC 模式下不可用 |
| 返回内容块而不是规范值 | 策略/PTC/卡片全链路失真 |
| `presentCall` 里读文件 | 回放崩溃或位置错乱 |
| 后台任务用 `exec.signal` 取消 | 外层取消误杀已发布作业 |
| 注册后改 schema | 类型承诺失效，静默坏行为 |
