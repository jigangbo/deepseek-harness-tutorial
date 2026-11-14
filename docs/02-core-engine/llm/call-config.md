# `call-config.ts` — 会话级调用配置与 loop 请求标记

> 源码:`packages/llm/llm/src/call-config.ts`(约 100 行)。两件事:provider/model/effort/采样这组"请求头状态"的类型与相等比较;loop 构造请求的进程内身份标记(WeakSet)。

## 文件概览

文件头 JSDoc 点明设计动机:这组配置是**会话级**的、影响 KV cache 复用的请求头状态;请求 waterfall 可以替换它们,但 loop 记录变更快照(header)而不允许每调用静默漂移。所以这里只有类型、一个纯比较函数和两个 WeakSet 标记函数——没有默认值、没有校验(那在 `index.ts` 的 prepareCall)。

## 关键代码走读

### LlmCallConfig:与 GenerateOptions 一一对应

```typescript
// packages/llm/llm/src/call-config.ts
export interface LlmCallControls {
  reasoningEffort?: ReasoningEffortId
  temperature?: number
  maxTokens?: number
  stop?: string[]
}

export interface LlmCallConfig extends LlmCallControls {
  provider: string
  model: string
}
```

每个字段 1:1 映射到同名的 `GenerateOptions` 字段。区别在于:`GenerateOptions` 是**一次请求**的完整装配(消息、工具、signal 全在里面);`LlmCallConfig` 是**会话的请求头**——loop 从持久 header 重建请求,而不是逐调用接受这些值。TODO 注释坦白:`model` 与模型拥有的 effort 是明确的 epoch 级字段,采样标量放进这里只是出于谨慎(`FIXME(call-config-shape)` 在包 README 也有对应记录)。

### ConfigureCall:默认值材料化之前的纯选择

```typescript
// packages/llm/llm/src/call-config.ts
export type ConfigureCall = (
  controls: Readonly<LlmCallControls>,
  model: Readonly<LlmResolvedModelInfo>,
) => LlmCallControls
```

- 两个入参都是**深冻结的脱离副本**:proposed controls(不含默认值)与捕获代际的模型元数据;
- 同步、纯、可组合:后写覆盖先写,返回值里**改不了** provider/model(类型上就没这个字段);
- 抛错 = prepareCall 在派发前拒绝;只有解析后的配置可记录。
- `LlmCallConfigAdapterDefaults`(`reasoningEffort?: true; maxTokens?: true`)标记哪些字段是适配器默认材料化的而非调用者提议的——loop 把它记进 header,回放时才知道"这个值是当时路由给的"。

### callConfigEquals:字段级相等(含 stop 逐元素)

```typescript
// packages/llm/llm/src/call-config.ts
export function callConfigEquals(a: LlmCallConfig, b: LlmCallConfig): boolean {
  if (
    a.provider !== b.provider
    || a.model !== b.model
    || a.reasoningEffort !== b.reasoningEffort
    || a.temperature !== b.temperature
    || a.maxTokens !== b.maxTokens
  ) return false
  if (a.stop === undefined || b.stop === undefined) return a.stop === b.stop
  return a.stop.length === b.stop.length && a.stop.every((s, i) => s === b.stop?.[i])
}
```

用途写在 JSDoc:调用者用它判断"提议的配置是真变更(值得记一条 header 快照)还是把持有的配置重述了一遍"。`stop` 数组按元素比较而非引用比较;一边 undefined 一边有值直接不等。

### loop 请求标记:对象身份而非内容

```typescript
// packages/llm/llm/src/call-config.ts
const AGENT_LOOP_REQUESTS = new WeakSet<GenerateOptions>()

export function markAgentLoopRequest<T extends GenerateOptions>(request: T): T {
  AGENT_LOOP_REQUESTS.add(request)
  return request
}

export function isAgentLoopRequest(request: GenerateOptions): boolean {
  return AGENT_LOOP_REQUESTS.has(request)
}
```

- **按对象身份**标记,不复制不包装:loop 把组装好的请求对象 mark 后深冻结送进 waterfall;`llm/stream` 监听器用 `isAgentLoopRequest` 区分"loop 构造的会话请求"与"手搓的一次性请求"(后者由调用者自己保证输入不变直到流 settle)。
- WeakSet 意味着零内存泄漏、零全局可变状态泄漏:请求对象被回收,标记随之消失。
- 这是**进程内**标记,不进日志、不进 wire——深冻结(变异即抛)+ 派生自 session log 才是持久层的保证,标记只是让观察者不把辅助调用误认成会话请求。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `LlmCallControls` | interface | effort/temperature/maxTokens/stop 四个可选控制项 |
| `LlmCallConfig` | interface | controls + provider + model(会话请求头状态) |
| `ConfigureCall` | type | 派发前的同步纯配置回调(不能改路由) |
| `LlmCallConfigAdapterDefaults` | interface | 标记由适配器材料化的字段 |
| `callConfigEquals` | `(a, b) => boolean` | 字段级相等;stop 逐元素 |
| `markAgentLoopRequest` | `<T>(request) => T` | WeakSet 标记 loop 请求对象 |
| `isAgentLoopRequest` | `(request) => boolean` | 按对象身份查询标记 |

## 与其他模块的关系

- `index.ts` 的 `prepareCall` 消费 `ConfigureCall`/`LlmCallConfig`/`LlmCallConfigAdapterDefaults`,并调 `callConfigEquals` 实现 prepared call 的"config 未被改"检查;`adapterStream` 对未 prepared 的请求也用它比较解析前后配置。
- `types.ts` 的 `GenerateOptions` 是标记函数的目标类型;`retry-policy.ts` 无关但同属"会话级状态"讨论。
- dsh-agent-loop 是 `markAgentLoopRequest` 的生产者;`llm/stream` waterfall 监听器(如 retry、replay、routing)是 `isAgentLoopRequest` 的消费者。
- `EpochHeader`(session 侧)持久 `LlmCallConfig` 快照与 adapterDefaults 标记——本文件提供的 `callConfigEquals` 也用于"配置是否真变了"的判定。

## 小结

- call-config 把"每调用参数"提升为"会话请求头状态",变更必须经记录(header 快照),不许静默漂移。
- `ConfigureCall` 是**显式 > 隐式**的样板:选择发生在拥有方(prepareCall)的显式 `resolve` 步骤,而不是 `run()` 里藏 `?? default`。
- WeakSet 标记是最小机制的典范:对象身份、进程内、零清理成本,持久保证另由深冻结 + session log 提供。
