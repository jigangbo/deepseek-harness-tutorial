# `api-key.ts` — 密钥传输不变量
> 所有适配器共享的 API key 格式判定


> 源码:`packages/llm/llm/src/api-key.ts`(42 行)。判断一个**已提供**的 provider API key 能否放进 HTTP 头——这个判定全库只有这一份,每个要把 key 放进请求头的适配器都复用它。

## 文件概览

文件极短,但把两个容易做错的决策写对了:

1. **空白修剪是静默的**(padded key 只有一种合理读法),其余缺陷都要报告;
2. 合法字符集是**传输不变量**而非某家 provider 的策略——`fetch` 拒绝构建含非法字符的头,与其得到一个报 UTF-16 码点的 opaque 错误,不如本地给出解释清楚的拒绝。

## 关键代码走读

```typescript
// packages/llm/llm/src/api-key.ts
const LEGAL_API_KEY = /^[\x21-\x7E]+$/

export type ApiKeyRejection = 'empty' | 'illegalCharacters'

export type ApiKeyCheck =
  | { readonly ok: true; readonly value: string }
  | { readonly ok: false; readonly reason: ApiKeyRejection }

export function normalizeApiKey(raw: string): ApiKeyCheck {
  const value = raw.trim()
  if (value.length === 0) return { ok: false, reason: 'empty' }
  if (!LEGAL_API_KEY.test(value)) return { ok: false, reason: 'illegalCharacters' }
  return { ok: true, value }
}
```

逐段解释:

- **`[\x21-\x7E]`** = 可打印 ASCII、不含空格(`\x20` 被排除)。JSDoc 解释了为什么刻意排除 Latin-1:HTTP 头理论上能携带它,但没有 provider 签发这种 key;放宽字符集只会把"本地一次说清楚"换成"线上一个 opaque 401"。
- **trim 先行且静默**:存储的 key 来自凭证缝、`.env` 行或 shell 导出,都会夹带周围空白;padded key 有唯一无歧义读法,不值得为它报错。空串(纯空白 trim 后)则明确报 `empty`。
- **返回判别联合而不是抛错/返回 `string | null`**:调用者被迫处理 `ok: false` 分支,reason 是封闭的两个字面量。
- **"缺席"不归这个函数管**:profile 不指定凭证引用时走 provider 自身的 ambient 发现或 OAuth——那是配置状态,不是格式缺陷。调用者先决定"有没有值",再问"这个值能不能用"。

`assertUsableApiKey`(在 `index.ts`,不在本文件)把这里的判定升级成带 `INVALID_CREDENTIAL` 码的 `LlmError`,诊断文案指明去哪里修(引用名、Models 页),且**绝不回显密钥值**。之所以分两个文件:api-key.ts 保持零依赖纯判定,index.ts 才引入 LlmError 与面向用户的诊断。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `LEGAL_API_KEY` | RegExp | 可打印 ASCII(排除空格)传输不变量 |
| `ApiKeyRejection` / `ApiKeyCheck` | type | 判定结果的封闭联合 |
| `normalizeApiKey` | `(raw) => ApiKeyCheck` | trim + 判定;空/非法字符分别报告 |

## 与其他模块的关系

- `index.ts` 的 `assertUsableApiKey` 消费本函数,把拒绝转成 `INVALID_CREDENTIAL` 的 `LlmError`(`error.ts` 提供常量)。
- 两个 shipped 适配器(`llm-deepseek`、`llm-pi-ai`)都在凭证解析后调用同一判定,不再各自保留近似拷贝。
- 凭证值本身来自 credentials 缝;本文件只管"这个值能不能进 HTTP 头"。

## 小结

- 一份判定、两处复用:格式缺陷在本地一次说清(哪个引用、哪类问题),而不是漏到线上变成 opaque 401。
- 修剪静默、其余报错;判别联合强制调用者处理失败分支;"key 没配"是配置问题,不归格式判定管。
