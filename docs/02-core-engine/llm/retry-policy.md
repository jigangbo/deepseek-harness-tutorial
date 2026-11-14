# `retry-policy.ts` — provider 拥有的重试策略解析

> 源码:`packages/llm/llm/src/retry-policy.ts`(约 195 行)。定义重试策略的配置 schema 与解析函数。**本包不执行重试**——`LlmRuntime` 在路由注册时把解析后的策略捕获进注册表,真正执行它的是可选的 `dsh-llm-retry` 插件(agent 失败步扩展点)。

## 文件概览

两种模式:

- **normal**:有界瞬态重试——只重试 `retryableCodes` 里的稳定码,上限 `maxRetries`;
- **always**:无界重试——每个模型请求失败都重试,直到成功、取消或注册释放。

解析发生在**路由注册之前**(适配器的 `providerRetryPolicy()` 返回已解析策略,未提供则用 normal 默认),产物是冻结的不可变 `ResolvedRetryPolicy`,随注册捕获。这保证在途失败看到的恢复策略不会因注册被 replace/dispose 而改变。

## 关键代码走读

### 默认值与 schema

```typescript
// packages/llm/llm/src/retry-policy.ts
const DEFAULT_MAX_RETRIES = 5
const DEFAULT_INITIAL_DELAY_MS = 500
const DEFAULT_MAX_DELAY_MS = 10_000
const DEFAULT_JITTER_RATIO = 0.1
const DEFAULT_RETRYABLE_CODES = Object.freeze([
  EMPTY_RESPONSE_CODE,
  'RATE_LIMIT',
  'SERVER',
  'TIMEOUT',
  'TRANSPORT',
])
```

默认可重试集合正好是"瞬态失败"家族:空响应(无持久物,安全重试)、限速、服务端、超时、传输。`AUTH`、`INVALID_CREDENTIAL`、`CONTEXT_WINDOW_EXCEEDED` 这类重试也一样失败的码都不在集合里。

Cordis 配置 schema(嵌入每个具体 provider 的配置)与运行时校验**并存**:

```typescript
// packages/llm/llm/src/retry-policy.ts
const backoffSchema: z<BackoffConfig> = z.object({
  initialDelayMs: z.number().max(MAX_TIMER_DELAY_MS).default(DEFAULT_INITIAL_DELAY_MS),
  maxDelayMs: z.number().max(MAX_TIMER_DELAY_MS).default(DEFAULT_MAX_DELAY_MS),
  jitterRatio: z.number().min(0).max(1).default(DEFAULT_JITTER_RATIO),
})

export const RetryPolicySchema: z<RetryPolicyConfig> = z.union([
  normalPolicySchema,
  alwaysPolicySchema,
])
```

schema 管 cordis.yml 的加载期校验与默认值;`resolveRetryPolicy` 是**独立入口**(适配器直接调用,不经 cordis 加载路径),所以自己也做全套验证——misconfiguration fails loud 在两个入口都成立。

### resolveRetryPolicy:三种情形

```typescript
// packages/llm/llm/src/retry-policy.ts
export function resolveRetryPolicy(
  config: RetryPolicyConfig | undefined,
  path: string,
): ResolvedRetryPolicy {
  if (config === undefined) {
    return Object.freeze({
      mode: 'normal',
      maxRetries: DEFAULT_MAX_RETRIES,
      retryableCodes: DEFAULT_RETRYABLE_CODES,
      ...resolveBackoff(undefined, `${path}.backoff`),
    })
  }
  switch (config.mode) {
    case 'normal': {
      validateKeys(config, NORMAL_POLICY_KEYS, path)
      const maxRetries = config.maxRetries ?? DEFAULT_MAX_RETRIES
      const retryableCodes = config.retryableCodes ?? [...DEFAULT_RETRYABLE_CODES]
      if (!Number.isSafeInteger(maxRetries) || maxRetries < 0) {
        throw new Error(`${path}.maxRetries must be a non-negative safe integer`)
      }
      if (retryableCodes.length === 0) {
        throw new Error(`${path}.retryableCodes must not be empty`)
      }
      if (retryableCodes.some(code => typeof code !== 'string' || code.length === 0)) {
        throw new Error(`${path}.retryableCodes must contain only non-empty strings`)
      }
      if (new Set(retryableCodes).size !== retryableCodes.length) {
        throw new Error(`${path}.retryableCodes must not contain duplicates`)
      }
      return Object.freeze({ mode: 'normal', maxRetries, retryableCodes: Object.freeze([...retryableCodes]), ...resolveBackoff(config.backoff, `${path}.backoff`) })
    }
    case 'always':
      validateKeys(config, ALWAYS_POLICY_KEYS, path)
      return Object.freeze({ mode: 'always', ...resolveBackoff(config.backoff, `${path}.backoff`) })
    default:
      throw new Error(`${path}.mode must be "normal" or "always"`)
  }
}
```

值得注意的细节:

- **`path` 参数**让每个错误消息都点名"哪个 provider 的哪段配置"(`llm: provider "deepseek-official" retryPolicy.backoff.jitterRatio …`)——本地化、可操作的报告;
- **未知键即抛**(`validateKeys`),分层配置残渣也不静默吞;
- **always 模式的键集合与 normal 相同**(都含 `maxRetries`/`retryableCodes`)——注释解释:分层设置可能在切到 always 后残留 normal-only 字段;解析器**忽略这些失活字段**但仍拒绝键集合之外的未知键。切模式不用用户先手清理旧字段;
- normal 的 `retryableCodes` 必须非空、无重复、全非空串——空集合没有意义(等于永不重试,声明出来是误导)。

### resolveBackoff:数值不变量

```typescript
// packages/llm/llm/src/retry-policy.ts
function resolveBackoff(config: BackoffConfig | undefined, path: string): ResolvedRetryBackoff {
  if (config !== undefined) validateKeys(config, BACKOFF_KEYS, path)
  const initialDelayMs = config?.initialDelayMs ?? DEFAULT_INITIAL_DELAY_MS
  const maxDelayMs = config?.maxDelayMs ?? DEFAULT_MAX_DELAY_MS
  const jitterRatio = config?.jitterRatio ?? DEFAULT_JITTER_RATIO

  if (!Number.isFinite(initialDelayMs) || initialDelayMs <= 0 || initialDelayMs > MAX_TIMER_DELAY_MS) { throw … }
  if (!Number.isFinite(maxDelayMs) || maxDelayMs <= 0 || maxDelayMs > MAX_TIMER_DELAY_MS) { throw … }
  if (initialDelayMs > maxDelayMs) {
    throw new Error(`${path}.initialDelayMs must be less than or equal to maxDelayMs`)
  }
  if (!Number.isFinite(jitterRatio) || jitterRatio < 0 || jitterRatio > 1) { throw … }

  return Object.freeze({ initialDelayMs, maxDelayMs, jitterRatio })
}
```

- 上限 `MAX_TIMER_DELAY_MS`(来自 dsh-timeout)保证调度器真的能定时;`initial ≤ max` 保证指数退避曲线有意义;
- jitter 是**对称**的围绕每个本地延迟的随机乘数(±`jitterRatio`),比值 0–1;
- 产物冻结。三种 resolved 类型(`ResolvedNormalRetryPolicy` / `ResolvedAlwaysRetryPolicy`)都是只读接口,注册表据此可以安全地把它交给任意消费者。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `RetryPolicySchema` | `z<RetryPolicyConfig>` | cordis 配置加载期校验(normal/always 联合) |
| `resolveRetryPolicy` | `(config?, path) => ResolvedRetryPolicy` | 独立入口的完整验证 + 默认值 + 冻结 |
| `resolveBackoff`(私有) | `(config?, path) => ResolvedRetryBackoff` | 退避数值不变量与默认值 |
| `validateKeys`(私有) | `(value, allowed, path) => void` | 未知键即抛(带路径) |
| `ResolvedRetryPolicy` | type | normal(有界 + retryableCodes)或 always(无界) |
| `BackoffConfig` / `ResolvedRetryBackoff` | interface | initial/max/jitter 三元组 |

## 与其他模块的关系

- `index.ts`:`prepareRoutes` 调 `adapter.providerRetryPolicy(provider) ?? resolveRetryPolicy(undefined, …)`,把产物存进 `AdapterRegistration`;`providerRetryPolicy(provider)` 与 `PreparedLlmCall.retryPolicy` 是读取面。
- `error.ts`:`EMPTY_RESPONSE_CODE` 是默认 retryableCodes 的成员。
- `dsh-llm-retry`(另一个包)在 agent 失败步扩展点**执行**本策略;失败事实来自 `LlmFailure`。
- 设置 UI/配置目录:生成的配置目录文档列出可选输入字段。

## 小结

- **策略与执行分离**:本文件只解析与冻结;"一次流 = 一次 provider 尝试"由服务保证,重试在 durable 步边界由 retry 包执行——同一个失败不会被两层机制重复恢复。
- 双入口双校验(cordis schema + resolve 函数)让 misconfiguration 无论从哪条路进来都 fail loud,错误消息永远点名配置位置。
- always 模式"忽略失活字段但拒绝未知键"是对分层配置现实的精确处理:宽容于合法残留,严格于结构错误。
