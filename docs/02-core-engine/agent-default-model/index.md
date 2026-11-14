# 2.8 Agent Default Model — 部署默认模型选择

> 源文件：`packages/core/agent-default-model/src/index.ts`（97 行）。回答一个问题：**新建的 agent 没有显式模型选择时，默认用哪个？** 核心源码：`packages/core/agent-default-model/src/index.ts`。

## 文件概览

`AgentDefaultModelConfig` 服务挂在 `ctx.agentDefaultModel`，拥有部署默认的模型选择（provider 路由 + 模型 id + 可选推理档位），独立于任何 Host 或传输层。消费者：`api-session-controller`（UI 模型目录的默认值）与 `headless`（一次性任务的模型）。

## 关键代码走读

### 服务声明与 volatile 配置

```ts
export interface Config {
  /** Registered provider route. */
  provider: Volatile<string>
  /** Provider-owned model id. */
  model: Volatile<string>
  /** Adapter-owned reasoning effort; omission follows the provider default. */
  reasoningEffort: Volatile<string | undefined>
}

export class AgentDefaultModelConfig extends Service {
  static Config = z.object({
    provider: z.string().required().volatile(),
    model: z.string().required().volatile(),
    reasoningEffort: z.string().volatile(),
  })
  ...
}
```

三个字段全部 `.volatile()`——用户可在设置 UI 里改（配置章的表单链路），改动经 `ctx.configEditor` 持久化并热生效。`reasoningEffort` 可省——**省略即跟随适配器默认档**，与 LLM 章的"适配器拥有 effort 映射"一致。

### 投影为 Agent 面的选择类型

```ts
function selection(settings: { provider: string; model: string; reasoningEffort?: string }): ModelSelection {
  return {
    provider: settings.provider,
    model: settings.model,
    ...settings.reasoningEffort === undefined
      ? {}
      : { reasoningEffort: ReasoningEffortId(settings.reasoningEffort) },
  }
}
```

存储设置 → `ModelSelection`（agent 包 `types.ts` 定义的选择类型）的纯投影：`ReasoningEffortId` 品牌化包装，undefined 时**不产出键**（显式优于隐式：适配器默认不可被 `undefined` 意外覆盖）。

### 设置集成与序列化保存

```ts
constructor(private readonly ownerContext: Context, private config: Config) {
  super(ownerContext, 'agentDefaultModel')
  ownerContext.inject(['settings'], (child) => {
    child.effect(() => child.settings.configure({ auto: false }, ownerContext.fiber))
  })
```

- 可选注入 `settings`：注册 `auto: false` 的配置策略（有自带设置页的插件不想要自动生成的表单——配置章）；
- **业务插件不因 Settings 缺席而失败**：策略放在 inject 子上下文的 effect 里，Settings 晚加载或被替换也能接上。
- `saves: Promise<void>` 把保存操作串行化——并发保存请求排队而非竞写。

## 核心类与函数

| 名称 | 职责 |
|------|------|
| `AgentDefaultModelConfig` | 服务：读默认选择、保存变更 |
| `Config` | volatile 三字段（provider/model/reasoningEffort） |
| `selection()` | 存储设置 → `ModelSelection` 纯投影 |
| `saves` 队列 | 串行化保存，避免并发竞写 |

## 与其他模块的关系

- **agent 包**：消费 `ModelSelection` 类型；loop 在组装时对会话模型选择做快照（agent/pre-step 的 `modelSelection`），本包只在"无会话级选择"时提供默认。
- **settings/config-editor**：volatile 字段经表单编辑、经编辑器持久化（配置章）。
- **llm 包**：`ReasoningEffortId` 的品牌化 id——**id 不等于 wire 表示**，映射归适配器（精通篇适配器章）。

## 小结

97 行解决"默认模型"：volatile 配置 + 纯投影 + 可选 settings 策略 + 串行保存。它示范了一个小服务该有的样子——**单一职责、显式投影、可选集成不破坏主路径**。
