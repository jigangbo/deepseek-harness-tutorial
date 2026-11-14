# LLM Provider 适配器族 — 从接缝到实现

> `ctx.llm` 是全库最重要的缝之一。本章过一遍注册进这个缝的各个包：主力 DeepSeek 适配器、两种认证形态、wire 扩展注册表、第二个 provider（pi-ai）、重试执行器与 token 计量。

## 家族地图

| 包 | 角色 | 一句话 |
|----|------|--------|
| `llm-deepseek` | Provider | 主力适配器：DeepSeek Messages、推理、图片输入 |
| `llm-deepseek-account` | 认证 Provider 变体 | DeepSeek 账号认证、模型列表、模型门控 |
| `llm-deepseek-api-key` | 认证 Provider 变体 | API key 认证、模型发现、官方模型清单 |
| `deepseek-llm-api-extensions` | Service Definition（子缝） | 官方 DeepSeek 请求扩展注册表 |
| `llm-pi-ai` | Provider | pi-ai 背书的多 provider 适配器 |
| `llm-retry` | Consumer（执行器） | 持久 agent-step 边界的重试恢复 |
| `token-meter` | 服务 | 回放感知的 token/上下文压力计量 |

## DeepSeek 适配器解剖（llm-deepseek）

```text
agent-loop prepareCall()
   │ 组装好的请求（消息、工具 schema、模型选择）
   ▼
DeepSeekAdapter.stream(GenerateOptions)
   ├─ adapter.ts      请求构造 + SSE 解析 → StreamChunk 映射
   ├─ config.ts       Config（模型、温度等）
   ├─ credential.ts   经 ctx.credentials 解析密钥引用（永不存明文）
   ├─ files-api.ts / file-store.ts / file-id.ts   图片输入：附件 → provider 文件
   ├─ defaults.ts     默认请求参数
   └─ host.ts         Host 侧装配
```

- **SSE 分帧**用 `eventsource-parser`；思考/工具调用增量映射为 dsh 流事件（usage-before-finish、arguments 原样拼接等协议义务见精通篇适配器章）。
- **图片输入**走附件缝：`ctx.attachments` 的持久引用解析为 provider 原生内容。
- **认证**经 `ctx.credentials`/`ctx.deepseekAccount`——适配器自己不碰密钥文件。

## 认证形态分叉

同一个 DeepSeek API，两种账号体系各自成包（都注册为 provider，路由名相同或不同由配置决定）：

- **`llm-deepseek-api-key`**：经典 API key。密钥值在凭证库，配置里只有引用；模型发现 + 官方模型清单。
- **`llm-deepseek-account`**：账号认证（浏览器授权流程，经 `ctx.deepseekAccount` 缝——Host 拥有浏览器授权与本地凭证，UI 只见状态不见 token）+ 模型列表与**模型门控**。

这是"一个 API、多个 provider 包"而非"一个包多模式"——认证形态影响密钥生命周期与模型目录来源，独立演化才拆包（能力缝的角色拆分纪律）。

## wire 扩展注册表（deepseek-llm-api-extensions）

`ctx.deepseekLlmApiExtensions` 是一个**子缝**：插件可以贡献**生命周期拥有的顶层 API 字段**（如会话日志上传声明），官方适配器合并它们并在 HTTP 接受后提交投递状态。

- 实现者：`session-log-deepseek`（会话日志上传策略）、`plugin-package-inventory-deepseek`；
- 消费者：`llm-deepseek`（合并进请求）；
- 设计点：**各插件准备独立的顶层字段**——适配器是唯一的合并点， wire 语义不会被多个插件各自打补丁。

## 第二个 provider：llm-pi-ai

pi-ai 背书的**多 provider 适配器**：经 pi-ai 目录 + 手工声明的网关路由模型。它的存在证明接缝的可替换性——一个全新的 provider 家族（多种后端模型）以一个包接入，agent-loop 与工具体系零改动。它消费 `ctx.attachments`、`ctx.credentials`、`ctx.authorization`（授权流注册表：谁能发起什么登录流）。

## 重试执行器（llm-retry）

`ctx.llm` 的调用可以在持久 agent-step 边界包上重试策略：

- 失败分类（可重试的传输/限流 vs 终局的协议错误）决定重试动作；
- 重试**不重复组装**：loop 的首次尝试置位保证 agent/pre-step 与组装只发生一次（总览篇生命周期章）；
- 配置化：provider 路由 → 重试策略的映射。

## token 计量（token-meter）

`ctx.tokenMeter` 是单例测量服务：

- **回放感知**：对同一路由的估算与回放折叠（breakdown/measurement/surface-fold 投影）；
- 消费者：`compaction-basic`（压力检测触发压缩，见持久化篇压缩章）与占用率显示；
- **路由定价**（route-pricing）：不同 provider/模型的 token 计价差异归它拥有——压缩缝自己没有定价 API。

## 实现者速记

| 包 | 注册进 | 关键配置 |
|----|--------|----------|
| `llm-deepseek` | `ctx.llm` | provider 路由、模型、凭证引用 |
| `llm-deepseek-api-key` | `ctx.llm` | key 引用、模型清单来源 |
| `llm-deepseek-account` | `ctx.llm` | 账号缝、门控 |
| `llm-pi-ai` | `ctx.llm` | pi-ai 目录、网关声明、授权流 |
| `llm-retry` | 包裹 `ctx.llm` 调用 | 路由 → 重试策略 |
| `token-meter` | `ctx.tokenMeter` | 定价表 |
| `deepseek-llm-api-extensions` | `ctx.deepseekLlmApiExtensions` | （注册表本体） |

## 设计启发

1. **认证形态 = 独立 provider 包**：密钥生命周期不同，就拆包——"一个 API 一个包"不是教条。
2. **wire 扩展走注册表**：第三方对请求的扩展经唯一合并点，字段所有权清晰（生命周期拥有 + 投递状态提交）。
3. **计量独立成缝**：压缩不知道 token 怎么算；token-meter 拥有估算、回放与定价——测量与决策分离。
