# `constants.ts` — 默认并发常量
> 共享调度默认值


> 源码：`packages/core/agent-loop/src/constants.ts`（6 行）。导出 `DEFAULT_MAX_PARALLEL_TOOL_CALLS`。

## 文件概览

整个文件只有一个常量：

```typescript
/** Shared agent-loop scheduler defaults.
 * @module dsh-agent-loop/constants
 */

/** Default maximum in-flight parallel-safe calls per agent step. */
export const DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10
```

## 为什么单列一个文件

这个常量服务于两个互不依赖的消费者：

- **`AgentLoop.Config`**（`packages/core/agent-loop/src/index.ts`）：`maxParallelToolCalls` 字段的默认值。它是 **volatile** 的正整数 Config 字段——部署可以在 `cordis.yml` 里改，运行期经 `ctx.agentLoop.config.maxParallelToolCalls.get()` 读取，live 编辑对下一批工具调用生效。
- **可能的外部消费**：常量经 `index.ts` 再导出（`export { DEFAULT_MAX_PARALLEL_TOOL_CALLS }`），独立成文件让消费者引用默认值时不必 import 整个服务模块，避免为了一个数字拉起服务依赖图。

放进独立模块还有一层秩序意义：常量属于"协议/调度默认值"，不属于某个类或服务——`agent.ts`（状态机）与 `index.ts`（服务/配置）都不该是它的家，否则默认值的可见范围会与实现耦合。

## 它如何被使用（不是硬编码）

循环每一步都**实时读取** volatile 配置而不是缓存这个常量：

```typescript
// packages/core/agent-loop/src/agent.ts — step() 内
const maxParallelToolCalls = this.loopCtx.agentLoop.config.maxParallelToolCalls.get()
// ...
const result = await this.loopCtx.tools.execute({
  calls: toolCalls, signal, agent: this, log: { turn, step },
  maxParallelToolCalls, toolSet: decision.toolSet,
})
```

`ctx.tools.execute()` 收到的是当前生效值，优先于执行器自身配置；`1` 表示串行。常量只在配置 schema 的 `.default(...)` 处出现一次：

```typescript
// packages/core/agent-loop/src/index.ts — AgentLoop.Config
maxParallelToolCalls: z.number().step(1).min(1).default(DEFAULT_MAX_PARALLEL_TOOL_CALLS).volatile(),
```

这符合仓库的配置纪律：**部署可变的取值必须是可校验的 Config 字段，一个 `DEFAULT_*` 常量本身不是可配置性**——常量只是字段的出厂值，真正的可配置性由 `.volatile()` 与 `.get()` 提供。

## 核心类与函数（表格）

| 名称 | 类别 | 值 | 说明 |
|---|---|---|---|
| `DEFAULT_MAX_PARALLEL_TOOL_CALLS` | 常量 | `10` | 每个 agent step 并行安全调用（parallel-safe）的最大在途数；由 `AgentLoop.Config.maxParallelToolCalls` 作为默认值，经 `index.ts` 再导出 |

## 与其他模块的关系

| 对象 | 关系 |
|---|---|
| `index.ts` | 唯一引用处：Config schema 的 `.default()`；并再导出常量 |
| `agent.ts` | 不直接引用常量；每步经 `config.maxParallelToolCalls.get()` 取实时值传给 `ctx.tools.execute()` |
| `packages/core/tools` | 执行器按传入的 `maxParallelToolCalls` 决定并行池上限（exclusive 调用始终是屏障，`1` 即串行） |
| `README.md` 配置目录 | `docs/config-catalog.md` 记录该字段的默认值与 volatile 语义 |

## 小结

6 行文件，一个数字：`DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10`。它的存在方式比数值本身更值得记住——默认值只出现在配置 schema 的 `default()` 处，运行路径永远读 volatile Config；独立成文件把"调度默认值"从服务与状态机实现中解耦出来。
