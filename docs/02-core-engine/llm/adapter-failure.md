# `adapter-failure.ts`

> 适配器抛出值的失败归一化


> 源码:`packages/llm/llm/src/adapter-failure.ts`(约 110 行)。`LlmRuntime.adapterStream` 捕获的任何抛出值(`unknown`)都要先经过 `normalizeLlmFailure`,变成可放进终态 finish chunk 的 `LlmFailure`。适配器边界之外的世界不可信——SDK 错误、跨包副本、甚至敌意 getter——归一化要全部防住。

## 文件概览

输入是任意的 `unknown`:可能是 `LlmError`(自带 failure 快照)、第三方 SDK 错误、非 Error 值(`throw 'boom'`)、甚至带 getter 陷阱的对象。输出是冻结的、字段全部验证过的 `LlmFailure`(见 types.md)。

三条规则贯穿全文件:

1. **只信任自己拥有的代码体系**:`harnessErrorCode` 对非 `HarnessError` 一律给 `UNKNOWN`——"第三方 SDK 的 code 不是我们的分类";
2. **只读 own data property,不触发 accessor**:`Object.getOwnPropertyDescriptor` 而不是 `error.failure`——SDK 定义的 getter 可能抛;
3. **携带的 failure 快照必须通过验证、且与自身 code 一致才被采信**——跨包结构化克隆保留 own data 但不保留类身份,所以不能只看 `instanceof`。

## 关键代码走读

### 主入口

```typescript
// packages/llm/llm/src/adapter-failure.ts
export function normalizeLlmFailure(value: unknown): LlmFailure {
  const error = value instanceof Error
    ? value
    : new HarnessError(thrownMessage(value), 'UNKNOWN', { cause: value })
  // Cross-package copies preserve own data but not class identity. Trust the
  // carried facts only when both own properties agree after validation.
  const carried = ownFailureSnapshot(error)
  if (carried !== undefined && carried.code === ownErrorCode(error)) return carried
  return Object.freeze({
    message: errorMessage(error),
    code: harnessErrorCode(error),
  })
}
```

- 非 Error 抛出值包成 `HarnessError(..., 'UNKNOWN', { cause: value })`,原值挂在 cause 上不丢信息;`thrownMessage` 用 `String(value)` 渲染且整体 try 住(`toString` 抛异常的敌意值回退到 `'LLM adapter failed'`)。
- **采纳条件是双重的**:error 上有一个**own data** 的 `failure` 属性(经 `failureSnapshot` 验证),**并且**它自己的 own data `code` 与快照里的 code 一致。两个独立信号都对上才信"这是一个结构化失败";否则退回 `message + harnessErrorCode` 的保守组合。

### own 属性读取:永不触发 getter

```typescript
// packages/llm/llm/src/adapter-failure.ts
function ownErrorCode(error: Error): unknown {
  try {
    const descriptor = Object.getOwnPropertyDescriptor(error, 'code')
    return descriptor !== undefined && 'value' in descriptor ? descriptor.value : undefined
  } catch (_sdkPropertyTrap) {
    return undefined
  }
}

function ownFailureSnapshot(error: Error): LlmFailure | undefined {
  try {
    const descriptor = Object.getOwnPropertyDescriptor(error, 'failure')
    return descriptor !== undefined && 'value' in descriptor
      ? failureSnapshot(descriptor.value)
      : undefined
  } catch (_sdkPropertyTrap) {
    return undefined
  }
}
```

两个函数同一个模式:descriptor 上必须有 `value`(own **data** property;own accessor 的 getter 也不触发),外层 try 兜住 descriptor 枚举本身抛异常的极端情况。空 catch 都有名字(`_sdkPropertyTrap`)说明放弃的原因。

### failureSnapshot:字段逐一验证

```typescript
// packages/llm/llm/src/adapter-failure.ts
function failureSnapshot(value: unknown): LlmFailure | undefined {
  if (typeof value !== 'object' || value === null) return undefined
  try {
    const candidate = value as Partial<LlmFailure>
    const message = candidate.message
    const code = candidate.code
    /* …status / providerRetryAfterMs / requestId / offloadImages 读取… */
    if (typeof message !== 'string' || message.length === 0
      || typeof code !== 'string' || code.length === 0
      || (status !== undefined && (!Number.isInteger(status) || status < 100 || status > 599))
      || (providerRetryAfterMs !== undefined
        && (!Number.isFinite(providerRetryAfterMs) || providerRetryAfterMs <= 0))
      || (requestId !== undefined && (typeof requestId !== 'string' || requestId.length === 0))
      || (offloadImages !== undefined && (!Number.isSafeInteger(offloadImages) || offloadImages <= 0))) return undefined
    return Object.freeze({
      message,
      code,
      ...status === undefined ? {} : { status },
      /* …可选字段条件展开… */
    })
  } catch (_sdkFailureGetter) {
    return undefined
  }
}
```

验证规则与 `LlmError` 构造器完全同构(status 是 100–599 整数、retryAfter 正有限、requestId 非空串、offloadImages 正安全整数)。**任一字段不过就整体拒绝**(返回 undefined 走保守路径)——半个坏快照比没有快照更危险,因为它会被下游按 code 路由。产物冻结;可选字段用条件展开保持键集精确。外层 try 兜住访问候选对象属性时抛异常的 getter。

### message 与 code 的保守读取

```typescript
// packages/llm/llm/src/adapter-failure.ts
function errorMessage(error: Error): string {
  try {
    const message: unknown = error.message
    if (typeof message === 'string' && message.length > 0) return message
  } catch (_sdkMessageGetter) {
    // The fallback below preserves a serializable failure beside the original Error.
  }
  return 'LLM adapter failed'
}

function harnessErrorCode(error: Error): string {
  return error instanceof HarnessError ? error.code : 'UNKNOWN'
}
```

`error.message` 允许读(这是主失败信息),但读失败/非串/空串都回退 `'LLM adapter failed'`——message getter 抛异常不能顶替主失败。code 则严格要求真实 `HarnessError` 实例:跨包副本丢了类身份,`instanceof` 不过就 `UNKNOWN`,这是故意的——信任面越小越稳。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `normalizeLlmFailure` | `(value: unknown) => LlmFailure`(internal) | 归一化主入口:非 Error 包装、快照双重采信、保守回退 |
| `thrownMessage` | `(value) => string` | 非 Error 抛出值的安全渲染 |
| `ownErrorCode` / `ownFailureSnapshot` | `(error) => unknown/LlmFailure?` | 只读 own data property,不触发 getter |
| `failureSnapshot` | `(value) => LlmFailure?` | 逐字段验证 + 冻结;不过即整体拒绝 |
| `errorMessage` / `harnessErrorCode` | `(error) => string` | 保守的 message 读取 / 只认 HarnessError 的 code |

## 与其他模块的关系

- `index.ts` 的 `adapterFailureChunk` 调用本函数,把结果包成终态 `finish` chunk(`error`/`aborted` 按 signal 与 code 判定)。
- `error.ts` 提供 `HarnessError` 基类;`types.ts` 提供 `LlmFailure` 目标类型。
- 字段验证规则与 `LlmError` 构造器(index.ts)成对存在:一边是"构造时保证",一边是"读取时验证"——同一个契约的两个入口。
- 下游 `dsh-llm-retry` 按归一化后的 `code`(对照 retryableCodes)决定是否重试。

## 小结

- 归一化的信任模型:**类身份靠 instanceof,结构化事实靠 own-data 双重验证**,两者缺一不可;其余一律降级到 `message + UNKNOWN`。
- 所有属性读取都绕开 getter,所有读取都 try 住——适配器边界外的任何值都不能把主失败本身变成异常。
- 验证规则与构造端同构,保证"合法构造的 LlmError 必然通过快照采信",而伪造/损坏的快照整体被拒。
