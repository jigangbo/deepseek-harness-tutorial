# `error.ts`

> 错误基类、规范错误码与 provider 文本分类器


> 源码:`packages/llm/llm/src/error.ts`(约 175 行)。三样东西:`HarnessError` 基类(stable code + cause 链)、一组 provider 中立的规范错误码常量、以及把 provider 错误文本分类成规范码的正则分类器与诊断渲染器。

## 文件概览

分类纪律:**消费者按 `code` 路由,永不解析 message 文本**。但 provider 报上来的只有文本——所以需要一个地方把"各家 provider 的说法"归一成规范码,这就是 `isContextWindowExceededError` / `isQuotaExceededError` 两个分类器的职责(适配器调用它们,而不是消费者)。

## 关键代码走读

### HarnessError 基类

```typescript
// packages/llm/llm/src/error.ts
export class HarnessError extends Error {
  /** Stable machine-routable failure class (e.g. `RATE_LIMIT`); route on this, never by parsing `message`. */
  readonly code: string

  constructor(message: string, code: string, options?: ErrorOptions) {
    super(message, options)
    this.code = code
    this.name = new.target.name
  }
}
```

- `code` 与人类可读 `message` 分离:code 是稳定的程序化路由键;`name` 用 `new.target.name`——子类实例的名字自动跟子类走,不用每个子类手动设。
- cause 链走标准 `ErrorOptions`,与 `errorChain` 渲染器配合。

### 规范错误码常量

```typescript
// packages/llm/llm/src/error.ts
export const CONTEXT_WINDOW_EXCEEDED_CODE = 'CONTEXT_WINDOW_EXCEEDED'
export const QUOTA_EXCEEDED_CODE = 'QUOTA'
export const ACCOUNT_QUOTA_EXCEEDED_CODE = 'ACCOUNT_QUOTA'

export const EMPTY_RESPONSE_CODE = 'EMPTY_RESPONSE'
export const INVALID_CREDENTIAL_CODE = 'INVALID_CREDENTIAL'
export const IMAGE_OFFLOAD_REQUIRED_CODE = 'IMAGE_OFFLOAD_REQUIRED'
```

每个常量都有值得记的语义边界:

- **`QUOTA` vs `ACCOUNT_QUOTA`**:`QUOTA` 是 provider 中立的耗尽;`ACCOUNT_QUOTA` 专指**当前产品可以充值的第一方账号余额**——UI 据此决定是否展示"去充值"入口。
- **`EMPTY_RESPONSE`**:正常结束但零内容块的退化完成。适配器把它归类为失败而不是产出空 assistant 消息,因为空消息会无声地结束 turn;且尝试没有产生持久物,重试是安全的(`dsh-llm-retry` 默认重试它)。
- **`INVALID_CREDENTIAL`** vs `MISSING_CREDENTIAL`:凭证**给了但格式不可用** vs **没给**——修复动作不同(改存储值 vs 提供值),所以必须两个码。它刻意不在默认可重试集合里:坏凭证每次尝试都一样地失败。
- **`IMAGE_OFFLOAD_REQUIRED`**:image-capable 路由的请求图字节数超预算;failure 的 `offloadImages` 命名还要 offload 多少个最旧出现,`dsh-compaction-image-offload` 记录选择后重试。

### 上下文溢出分类器

```typescript
// packages/llm/llm/src/error.ts
export function isContextWindowExceededError(detail: string): boolean {
  return STRUCTURED_CONTEXT_OVERFLOW.test(detail)
    || /\b(?:maximum|max)(?:\s+(?:allowed|supported))?\s+context\s+(?:length|window)\b/i.test(detail)
    || TOO_LARGE_FOR_CONTEXT.test(detail)
    || /\b(?:input|prompt|request)\s+(?:is\s+)?too\s+(?:long|large)\s+for\s+(?:this|the)\s+model\b/i.test(detail)
    || EXCEEDS_MODEL_CONTEXT.test(detail)
}
```

四个正则各管一种 provider 措辞,共同点是**只认明确指向上下文边界的说法**:

- `STRUCTURED_CONTEXT_OVERFLOW`:`context length/window exceeded/overflow/limit exceeded` 的结构化变体(连字符/下划线/空格都认);
- 字面量正则:`maximum context length` 这类 OpenAI 风格短语;
- `TOO_LARGE_FOR_CONTEXT`:把 "too large" 直接系到 model context 的请求尺寸措辞;
- `EXCEEDS_MODEL_CONTEXT`:"exceeds" 措辞只在宾语明确是 model context 时才算——避免把无关的 "exceeds" 误判成上下文溢出。

误判的代价是不对称的:把普通失败判成溢出会触发错误的恢复路径,所以措辞必须"点名"上下文。配额分类器 `isQuotaExceededError` 同理,只认 terminal 的 quota/balance/credit/budget 措辞,不把瞬时限速(`RATE_LIMIT`)混进来。

### errorChain:带 cause 链与 AggregateError 的诊断渲染

```typescript
// packages/llm/llm/src/error.ts
export function errorChain(value: unknown): string {
  // Tracks the active recursion path (entries removed on exit), so only true
  // cycles are flagged and a diamond-shared cause still renders in full.
  const path = new Set<unknown>()
  const render = (current: unknown): string => {
    if (path.has(current)) return '<circular cause>'
    path.add(current)
    try {
      /* …渲染当前层… */
      const causeText = current.cause === undefined || current.cause === null
        ? ''
        : render(current.cause)
      const cause = causeText === '' || causeText === message ? '' : `: ${causeText}`
      return `${message}${members}${cause}`
    } catch {
      return '<unrenderable value>'
    } finally {
      path.delete(current)
    }
  }
  return render(value)
}
```

设计细节:

- **环检测用进出栈**(进入 add、finally 里 delete):真正的循环 cause 链被标记 `<circular cause>`;菱形共享的 cause(两个分支指向同一对象但不成环)完整渲染——用 `Set.has` 常驻判断会误伤后者;
- **AggregateError 成员**用 `[a; b]` 括起来渲染(undici 的 fetch 失败常裹成员);
- **逐帧独立兜底**:内层 throw 只让那个敌意节点变成 `<unrenderable value>`,整条链不塌;
- **逐字重复的 cause 不再渲染**——`new HarnessError(String(value), code, { cause: value })` 这类包装只会添噪;
- 非 Error 值读 own `message` data property,拿不到就 `String()`;
- JSDoc 限定用途:**诊断表面**(消息、通知、日志)专用,永不解析其结果做路由。

### isHarnessError

```typescript
// packages/llm/llm/src/error.ts
export function isHarnessError(value: unknown): value is HarnessError {
  return value instanceof HarnessError
}
```

运行时边界的 instanceof 收窄:鸭子类型与跨 realm 错误**不**收窄(与 adapter-failure 的保守策略一致)。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `HarnessError` | `class extends Error` | code + cause 链基类;name 自动取子类名 |
| `CONTEXT_WINDOW_EXCEEDED_CODE` 等 6 个常量 | `const string` | provider 中立规范码 |
| `isContextWindowExceededError` | `(detail) => boolean` | 识别 OpenAI 兼容 provider 的上下文溢出措辞 |
| `isQuotaExceededError` | `(detail) => boolean` | 识别终端配额措辞(区别于瞬时限速) |
| `errorChain` | `(value) => string` | cause 链 + AggregateError 成员的安全诊断渲染 |
| `isHarnessError` | 类型守卫 | 运行时 instanceof 收窄 |

## 与其他模块的关系

- `index.ts` 的 `LlmError extends HarnessError`,并 import `INVALID_CREDENTIAL_CODE`;适配器包 import 全部常量与两个分类器(两类 DeepSeek 适配器都经 `isContextWindowExceededError` 保证上下文溢出只有一个规范码,无论失败是 thrown 还是流内)。
- `adapter-failure.ts` 用 `HarnessError` 做非 Error 值的包装与 code 信任判定。
- `retry-policy.ts` 的默认 retryableCodes 引用 `EMPTY_RESPONSE_CODE`。
- 消费端(`dsh-llm-retry`、UI、session)按 code 路由;`errorChain` 只进日志与用户可见诊断。

## 小结

- 基类只做一件事:让 `code` 成为第一等公民,`message` 永远只是给人看的。
- 常量的价值在**语义边界**:QUOTA/ACCOUNT_QUOTA(能否充值)、INVALID/MISSING(修复动作)、EMPTY_RESPONSE(可安全重试)——每组差异都对应一条不同的下游路径。
- 分类器把"解析 provider 文本"收敛在唯一一处,且措辞必须明确点名上下文/配额才归类;渲染器则保证任何敌意值都不能打断诊断输出。
