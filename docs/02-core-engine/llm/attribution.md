# `attribution.ts`

> 每个 provider 请求都带的静态应用归属


> 源码:`packages/llm/llm/src/attribution.ts`(约 70 行)。定义发给 provider 的静态公开应用身份(`AppIdentity`)并渲染成 `User-Agent` 头。适配器契约要求:每个 provider HTTP 请求都必须带 `attributionHeaders()`,并用 wire 级测试证明。

## 文件概览

文件头 JSDoc 点明目的:把非机密的产品身份**集中一处**,防止适配器各自漂移。配套约束写在 `docs/subsystems/llm-streaming.md` 的 AppIdentity 一节:每个字段都是公开产品事实——无机密、无本地路径、无 session id、无提示词文本、无 per-user 标识,且**任何 per-request 信息都不得影响这些值**。

## 关键代码走读

### 版本的单一来源:包清单

```typescript
// packages/llm/llm/src/attribution.ts
import { createRequire } from 'node:module'

// The package's own manifest is the single source of the version so the
// User-Agent cannot drift from what is published (`./package.json` is an
// export of this package; the relative path resolves from both `src/` and
// the bundled `lib/`).
const { version } = createRequire(import.meta.url)('../package.json') as { version: string }
```

- 版本从包自己的 `package.json` 读,而不是手抄常量——发布的 UA 永远与发布物一致,不存在"改了版本忘了改归属"这类漂移;
- `createRequire(import.meta.url)` + 相对路径 `../package.json`:注释明说这个相对路径同时解析自 `src/` 与打包后的 `lib/`(`package.json` 是本包的导出之一,见 package.json 的 `exports`)。

### APP_IDENTITY 与 userAgent

```typescript
// packages/llm/llm/src/attribution.ts
export interface AppIdentity {
  /** `User-Agent` product token (lowercase, hyphenated). */
  product: string
  /** Product version; sourced from package metadata, never hand-copied. */
  version: string
  /** Repository home URL of the app, used as the `User-Agent` comment. */
  url: string
}

export const APP_IDENTITY: AppIdentity = {
  product: 'deepseek-harness',
  version,
  url: 'https://github.com/deepseek-ai/deepseek-harness',
}

export function userAgent(identity: AppIdentity = APP_IDENTITY): string {
  return `${identity.product}/${identity.version} (+${identity.url})`
}

export function attributionHeaders(identity: AppIdentity = APP_IDENTITY): Record<string, string> {
  return { 'user-agent': userAgent(identity) }
}
```

- UA 形态是规范乘积 token + 注释:`deepseek-harness/0.2.2-rc.1 (+https://github.com/…)`——RFC 9110 §10.1.5 的 product + comment 惯例;
- **默认参数指向 `APP_IDENTITY`,且没有"不发归属"的路径**:省略 identity 回落到默认,而非置空——白标部署可以传入自己的 `AppIdentity`,但归属本身不可取消;
- 头名用小写(`user-agent`):wire 上 HTTP 字段名不区分大小写,小写是代码侧的统一约定;
- 契约刻意**不支持** OpenRouter 风格的额外 app 归属头(`X-Title` 之类)——`attributionHeaders` 只产出标准 `User-Agent`,provider 特定的归属是适配器自己的事。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `AppIdentity` | interface | product/version/url 三元组(全部公开事实) |
| `APP_IDENTITY` | const | harness 默认身份;版本读自包清单 |
| `userAgent` | `(identity?) => string` | 渲染 `product/version (+url)` |
| `attributionHeaders` | `(identity?) => Record<string,string>` | 适配器合并进请求的归属头(仅 `user-agent`) |

## 与其他模块的关系

- `index.ts` 通过 `export * from './attribution.ts'` 公开;`LlmAdapter` 的抽象契约 JSDoc 明说每个 provider 请求必须带这些头。
- 两个 shipped 适配器(直连 fetch 的 `llm-deepseek`、库背书的 `llm-pi-ai`)用不同内部机制合并同一份头;`llm-pi-ai/request-headers` waterfall 里 harness 归属**最后落笔、优先级最高**。
- 包 README 与 `docs/subsystems/llm-streaming.md` 的 AppIdentity 一节是本文件的规范出处。

## 小结

- 归属是**静态、公开、不可抑制**的:值只来自包清单与常量,per-request 事实碰不到它,省略参数也只会回落默认。
- 适配器契约用 wire 级测试证明归属头真实上线——"文档说有"不算数。
