# 扩展实战：添加一个 LLM 适配器

> 目标：把一个新的模型 provider 接进 dsh。参考实现两看：`packages/llm/llm-deepseek`（直连 HTTP、SSE 用 `eventsource-parser` 分帧）与 `packages/llm/llm-pi-ai`（包装现成 LLM 库）。动笔前先读 `packages/llm/llm/src/types.ts` 里的 `StreamChunk` 文档——两个适配器都是对着它验证的。

## 形状

```ts
class MyAdapter extends LlmAdapter {
  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> { … }
}

export const name = 'llm-myprovider'
export const inject = ['llm']
export const Config: z<Config> = z.object({ apiKey: z.string(), /* … */ })

export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(/* … */))
}
```

- 注册即 effect（HMR 安全）；一条 provider 路由一个适配器——重复注册抛错，多路由注册全有或全无。
- `options.provider` 选择适配器，`options.model` 是 provider 侧模型 id——因此**动态目录型适配器**无需生命周期重配就能服务新模型。
- 秘密走 Cordis 原生方式：schemastery Config + 环境变量回退，cordis.yml 里用 `!!js process.env.MY_KEY` 注入；**不要在代码里读临时 key 文件**。

## 协议义务（两个实现验证过的契约）

| # | 义务 | 说明 |
|---|------|------|
| 1 | **`usage` 在 `finish` 前，`finish` 后什么都不发** | 稳妥做法：缓冲 finish/usage 直到 provider 的流结束标记再冲刷（兼容尾部只发 usage 的 provider） |
| 2 | **工具调用 `arguments` 端到端是原始 JSON 字符串** | 流片段用 `argumentsDelta`；provider 给的是解析对象就在 `block-end` 时重新 stringify |
| 3 | **block `index` 按首次出现顺序分配** | 同一块的每个 delta 复用同一 index |
| 4 | **错误只有两条合法路径** | 从 `stream()` throw（传输/协议失败，用稳定 code 的 `LlmError`）；或以 `finish {kind:'error'|'aborted'}` 结束流（provider 带内失败）。按失败类别选择并写进文档 |
| 5 | **尊重 `options.signal`** | 传给 fetch / SDK |
| 6 | **不支持的选项要响亮失败** | provider 没有 stop 序列？throw `LlmError(..., 'UNSUPPORTED_OPTION')`，不许静默丢弃 |
| 7 | **原生回放状态走 `finish.replayState`** | provider 需要在后续调用带 response id/签名等元数据时，发最小 lossless-JSON 投影；重建历史时校验。`LlmRuntime` 只在历史路由与目标路由当前由**同一适配器实例**拥有时才传回它；你的适配器自行裁决同模型/跨模型/跨 provider 恢复是否合法——状态缺席时绝不凭 provider/model 名推断原生回放 |

Provider 特有的思考模式开关留在适配器自己的 Config 里。模型元数据走唯一的 provider 中立能力缝：

```ts
resolveModel(identity) // provider/model 身份 + 可选 context/reasoning 字段
// 有默认档才声明 defaultEffort；尊重解析器的 AbortSignal
```

Reasoning effort 是**有序的不透明 id**，由适配器映射到 provider 请求；保留适配器的权威可选项列表（包括适配器自定义的 `off`），不暴露 wire 拼写、不夹逼不支持的值——**id 不必等于它的 wire 表示**。

## 实现结构

保持五个职责分离（`llm-deepseek` 是参考布局）：

```text
wire 类型 ──► 请求序列化 ──► 传输解析 ──► chunk 翻译 ──► 适配器类
( provider API 形状)  (组装请求)  (SSE 分帧)   (StreamChunk 映射)  (LlmAdapter 子类)
```

## 与 loop 的协作关系（你在哪里被调用）

```text
agent-loop prepareCall()
   │ 冻结模型历史、绑定适配器
   ▼
LlmRuntime ──► yourAdapter.stream(GenerateOptions)
   │              ├─ content delta / block 事件
   │              ├─ tool-call argumentsDelta
   │              ├─ usage (finish 之前!)
   │              └─ finish { kind: 'stop' | 'error' | 'aborted', replayState? }
   ▼
loop 把压缩流结算为 assistant/message（成功）或 assistant/attempt（失败留痕）
```

## 验证清单

- 单元：chunk 翻译表驱动测试（含 usage-before-finish、arguments 原样拼接、index 分配）、两条错误路径、signal 取消、UNSUPPORTED_OPTION。
- 真实 provider 冒烟（`test:e2e`，带 key）：一条 prompt 走通流式 + 一次工具调用 + 中途取消。
- **快照**：涉及模型转写的变更更新录制会话场景。
- 内建冒烟：构建产物入口（`lib/`）可被纯 node 启动。

## 常见坑

| 坑 | 后果 |
|----|------|
| finish 之后还发 chunk | 下游断言崩，协议违规 |
| 把解析后的对象直接当 arguments 发 | 模型历史里的工具调用不可回放 |
| 静默忽略不支持的选项 | 用户以为 stop 生效了 |
| 凭 provider/model 名推断原生回放 | 跨实例状态错乱 |
| 在适配器里读自己的 key 文件 | 绕过凭证缝，轮换/来源追踪全部失效 |
