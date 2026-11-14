# 第 21 章 网络访问

> web 缝与搜索/抓取 provider


> 模型上网的两件事：搜索（找 URL）与抓取（读内容）。dsh 用一个 `ctx.web` 缝承载两者，工具名稳定、provider 可替换。核心源码：`packages/web/` 组。

## 缝的构成

| 角色 | 包 |
|------|-----|
| Service Definition + 注册表 | `dsh-web`（`ctx.web`） |
| Search Provider | `web-search-deepseek`（DeepSeek 搜索） |
| Fetch Provider | `web-fetch-http`（HTTP 抓取） |
| 底座 API | `web-deepseek-api`（DeepSeek Web API 客户端） |
| Consumer | `tool-web`（稳定模型工具名） |

```text
tool-web (web_search / web_fetch 工具名)
     │
   ctx.web ──── 注册表：search providers + fetch providers
     ├── web-search-deepseek
     ├── web-fetch-http
     └── （第三方 provider 可并列注册）
```

**tool-web 拥有稳定的模型可见名**；provider 拥有实现。换搜索后端，模型看到的工具与 schema 不变。

## 与模型可见性的衔接

- `web_search` 的完成态卡片（`web` 意图，`kind: 'search' | 'fetch'`）由持久 `result.meta` 重建：结构化搜索来源或抓取摘要——**卡片不带正文拷贝**，没有 web 卡片能力的 UI 回退到原始结果文本（见核心引擎篇 tools 与精通篇工具章）。
- system prompt 组装会为 web 工具贡献使用说明段（`svc_systemPrompt` 消费者表中的 `tool-web`）。
- 搜索/抓取结果作为工具结果进入日志——"模型可见 ⟺ 已记录"同样适用。

## 双执行器下的免费接入

PTC 程序里 `await tools.web_search({...})` / `await tools.web_fetch({...})` 直接可用——返回规范化 JSON（`output.schema`），批量"搜 N 个→挑最优→抓全文"这类编排写起来像普通代码，而不是多轮工具往返。

## 安全与策略

- 网络访问属于沙箱世界的一部分：PTC 程序的沙箱化 Node **没有网络 API**（边界即能力），只能经 bindings 调工具；直接发起抓取的是 provider 进程，不是被沙箱的程序。
- 抓取的内容是不可信数据：进入模型上下文前经过溢出上限与工具结果管线（spill、修剪）。

## 设计启发

1. **搜索与抓取分开注册**：两者能力集不同（provider 可能只实现其一），接口按能力分面而不是一个大接口。
2. **工具名稳定性归 consumer**：provider 随便换，模型的"世界"不晃动。
3. **结果元数据进日志**：`presentationMeta` 让 UI 卡片可回放重建，而模型上下文只保留必要文本。
