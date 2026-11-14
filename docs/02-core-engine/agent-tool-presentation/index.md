# 第 15 章 Tool Presentation：形态选择（已弃用）

> 源文件：`packages/core/agent-tool-presentation/src/index.ts`（54 行）。**这是一个已弃用（@deprecated）的兼容插件**：历史上它用 `mode: 'native' | 'ptc'` 选择工具形态；现在应直接挂载对应执行器包。留它是因为已有组合里还写着 `mode:` 配置——迁移需要一个过渡。

## 文件概览

`tool-presentation` 插件读取一个必填的 `mode` 配置，`apply` 时把对应的执行器包挂载进来：

```ts
export function apply(ctx: Context, config: Config): void {
  ctx.plugin(config.mode === 'native' ? NativeToolExecutor : PtcToolExecutor)
}
```

## 关键代码走读

### 为什么 mode 必填而不是默认

```ts
export interface Config {
  /**
   * The form this agent's model sees. `native` sends every visible schema,
   * `ptc` sends only `run_code` plus a generated SDK.
   * Required rather than defaulted: the deployment default is what a preset
   * without this row already gets, so an omitted value would mean the row was
   * composed for nothing.
   */
  mode: 'native' | 'ptc'
}
```

注释解释了**必填的架构理由**：一个组合里这行存在就是为了做选择——没有默认值可"继承"（部署默认是"没写这行的 preset"已经得到的），省略值意味着这行是白写的。这是「显式优于隐式」与「配置错误响亮失败」的组合应用：与其猜，不如让组合器写清楚。

### 动态插件挂载

```ts
export const inject = ['tools']
export function apply(ctx: Context, config: Config): void {
  ctx.plugin(config.mode === 'native' ? NativeToolExecutor : PtcToolExecutor)
}
```

- `ctx.plugin(...)` 动态挂载执行器包——兼容层本身零逻辑，纯转发；
- `inject = ['tools']` 保证注册表存在；注释说明 PTC 分支的细节：**PTC 子插件会等它的 runtime 就绪才在该作用域注册执行器**（native 不需要程序运行时）。

### 弃用标注

包、Config、apply 全部 `@deprecated`，指向替代品：`@deepseek-ai/dsh-tool-executor-native` / `@deepseek-ai/dsh-tool-executor-ptc` 直接挂载。lint 对 deprecated 配置的自校验做了豁免标注（兼容 schema 校验自己的弃用配置）。

## 核心类与函数

| 名称 | 职责 |
|------|------|
| `Config.mode` | 必填：`'native' | 'ptc'` |
| `apply` | 按模式转挂对应执行器包 |

## 与其他模块的关系

- **tool-executor-native / tool-executor-ptc**：真正的执行器（见前两篇）；本包只是迁移期的形态选择器。
- **preset 作用域**：兼容层的典型使用位置是 preset 的 `config.plugins` 行——preset 组合章（配置体系）。

## 小结

54 行的兼容层示范了**优雅弃用**：旧配置继续工作、新组合有明确替代、注释保留设计理由（为何必填）。等存量组合迁移完成，这个包的退役就是删文件。
