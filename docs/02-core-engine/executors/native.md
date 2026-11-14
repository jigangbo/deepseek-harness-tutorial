# `index.ts` — native 执行器：直接工具执行

> 源文件：`packages/core/tool-executor-native/src/index.ts`（36 行）。整个 native 执行器小到一屏读完——这正是"机制层薄、策略层厚"设计的体现。

## 文件概览

`NativeToolExecutor` 实现 `ToolExecutor` 接口（见 tools 章 `contracts.ts`）：`describe` 决定模型看到什么，`execute` 处理一个批次。它是"每个可见工具直接可调"的最小实现，拥有完整批次的结算。

## 关键代码走读

```ts
export class NativeToolExecutor implements ToolExecutor {
  private readonly pipeline: ToolExecutionPipeline

  constructor(ctx: Context, private readonly maxParallel: () => number = () => 10) {
    this.pipeline = new ToolExecutionPipeline(ctx)
  }

  describe(tools: readonly ToolDefinition[]): ToolModelInput {
    return { schemas: tools.map(definition => projectToolSchema(definition, false)) }
  }

  execute(input: ToolBatchInput): Promise<ToolBatchResult> {
    return executeToolBatch(input, this.pipeline, this.maxParallel)
  }
}
```

逐段解读：

1. **构造时只做一件事**——建共享管线。审批、guard、超时、结果规范化全部在管线里，执行器本身没有策略代码。
2. **`describe` 逐个投影 schema**：`projectToolSchema(definition, false)` 把定义投影为模型可见 schema（第二参数控制投影面）。模型看到全部可见工具——每个都可直调。
3. **`execute` 委托给 `executeToolBatch`**：批次调度（exclusive 屏障 + parallel 滚动池）、abort 排空与合成结果都在 tools 包的 batch 实现里；`maxParallel` 是函数而非数值——**并发上限可以动态读 volatile Config**（默认 10，即 `DEFAULT_MAX_PARALLEL_TOOL_CALLS`）。

```ts
export const name = 'native-tool-executor'
export const inject = ['tools']

export function apply(ctx: Context): void {
  ctx.tools.setExecutor(new NativeToolExecutor(ctx))
}
```

4. **注册即选择**：插件挂载 → `setExecutor` → 该作用域内模型请求由 native 描述。插件卸载，选择随之回收（effect 语义）。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `NativeToolExecutor` | `constructor(ctx, maxParallel?)` | native 执行器：describe + execute |
| `describe` | `(tools) => { schemas }` | 投影全部可见工具 schema |
| `execute` | `(input) => Promise<ToolBatchResult>` | 经共享管线跑一个批次 |
| `apply` | `(ctx) => void` | 挂载即选择 native |

## 与其他模块的关系

- **tools 包**：管线、批次调度、schema 投影全部复用；本包零策略。
- **system-prompt**：describe 的输出在 step 组装时进入请求（step-admitted tool schemas）。
- **agent-loop**：prepareCall 阶段绑定执行器能力；取消承诺、冻结历史与执行器无关——那是 loop 的职责。
- **tool-executor-ptc**：同一接口的另一种实现；两者可按作用域共存。

## 小结

Native 执行器是"执行器有多薄"的基准：**36 行，零策略，全部委托**。读它之后再读 PTC 版本，差异（`run_code` 单工具 + SDK 段 + 嵌套调用守卫）就一目了然。
