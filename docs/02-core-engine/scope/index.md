# 第 11 章 Scope：作用域注册

`packages/core/scope`（npm 包名 `@deepseek-ai/dsh-scope`）是一个零依赖库，不是 Cordis 服务。它回答 harness 里一个反复出现的问题：**同一个插件，如何在不同 agent 手里表现为不同的实例与配置？**

没有 scope 时，一个 Cordis 服务的注册表是全局唯一的：工具、提示词段、限制策略对所有 agent 一视同仁。有了 scope，"注册"多了一个维度——**通过哪个上下文注册**。经由 `agent.ctx` 注册的贡献只对该 agent 可见、随该 agent 销毁而撤销；全局注册保持原样，同名时 agent 级定义遮蔽全局定义。core 组的全部注册型服务（`ctx.tools`、`ctx.systemPrompt`、审批策略等）都建立在这个原语上，而它自己不依赖 agent loop 或 presets——任何自定义注册表或 scope 过滤事件都能复用。

## 三个角色

| 角色 | 类型 | 一句话 |
|---|---|---|
| 作用域身份 | `ScopeKey = object` | 不透明、按引用比较的对象；shipped loop 直接用活的 `Agent` 对象当自己的 key |
| 作用域句柄 | `Scope` | `{ ctx, rawDispose, dispose }`：注册上下文 + 两条销毁路径 |
| 事件路由载体 | `Scoped<T>` | `scopeTarget()` 产出的不透明接收器，编译期品牌 |

源码只有两个文件：`src/index.ts`（身份、载体、父链、mint）与 `src/store.ts`（注册表存储原语，见 [store.md](store.md)）。

## createScope：铸造一个作用域

`createScope(ctx, key)` 在 `ctx` 的 fiber 下造一个子插件，再给它的上下文 `extend` 一个 scope 标签（`packages/core/scope/src/index.ts`）：

```ts
export function createScope(ctx: Context, key: ScopeKey, options?: CreateScopeOptions): Scope {
  if (options?.parent !== undefined) bindScopeParent(key, options.parent)
  const fiber = ctx.plugin(scope)
  const scoped: Context = fiber.ctx.extend({ [kScope]: key })
  let disposing: Promise<void> | undefined
  return {
    ctx: scoped,
    rawDispose: fiber.dispose,
    dispose: () => (disposing ??= quiesceFiber(fiber)),
  }
}
```

三个要点：

1. **可见性与所有权同源**。`scopeOf(ctx)` 读取 `kScope` 标签，注册型服务用它决定贡献落在哪一层；同时 `ctx.plugin(scope)` 的 fiber 让"通过 scope.ctx 做的一切注册"都成为该 fiber 的 effect——`dispose()` 时统一反注册。一个贡献不可能"在某 scope 可见却被另一个 scope 销毁"。
2. **两条销毁路径**。`rawDispose` 是精确的 Cordis disposer（身份不变），供有序复合 effect 嵌套时使用；`dispose()` 是公共静止边界，`disposing ??=` 让并发调用共享同一次完成。`quiesceFiber` 会跟随 fiber 的 `inertia`（异步收尾）直到真正静止。
3. **继承铸造者的依赖 API**。scope.ctx 继承铸造插件的依赖解析链——所以 scope 从"其依赖正好是 scoped 注册所需"的那个插件铸造。发出去的 scope 交出了铸造插件的服务可见性，这不是权限边界，而是**同进程可信插件间的路由原语**。

agent loop 是第一个用户（`packages/core/agent-loop/src/agent.ts` 构造函数）：

```ts
this.dispatch = agentEvents(loopCtx, this)
this.scope = createScope(loopCtx, this)
this.ctx = this.scope.ctx
```

`Agent` 对象既是事件 payload 里的 subject，也是 `ScopeKey`。公开给插件的是 `agent.ctx`——插件写 `agent.ctx.tools.register(...)` 就得到了"只属于这个 agent 的工具"。驱动退出后由生命周期所有者统一 `scope.dispose()`（`packages/core/agent-loop/src/index.ts`）。

## 父链：一个关系，两个方向

scope 可以嵌套（preset 组合出一棵 agent 树）。父子关系存进 `scopeParents` 这个 `WeakMap<ScopeKey, ScopeKey>`，写入只有一条受控通道 `bindScopeParent(key, parent)`：

- 已有父的 key 再 bind 直接抛错——**绑定一次**，只有最初 binder 拿到的 `ScopeParentBinding` 可以 `rebind()`（blank-session 重组契约，由持有者保证旧父下没有留存产物）；
- 每次链接沿父链走一遍查环，环会让所有链消费者走不到根。

这条单一关系同时驱动两个方向，方向相反：

```text
注册视图继承 DOWN：子 scope 的读取会看到祖先层（ScopedLayers.chainLayers）
事件准入延伸 UP：  打了祖先 tag 的 listener 能收到派发到后代 key 的事件（scopeTarget）
                  反方向都不成立：父看不见子的注册，子收不到派发到父的事件
```

`scopeChainOf(key)` 返回最近优先的链 `[key, parent, grandparent, …]`，是两种遍历的公共底座。

## scopeTarget 与 Scoped<T>：事件的作用域过滤

事件系统按 `this`（接收器）过滤 listener。`scopeTarget(base, key)` 造一个只带路由状态的对象，保留 base 原有的 Cordis filter，再叠加 scope 谓词：

```ts
export function scopeTarget<T extends object>(base: T, key: ScopeKey | undefined): Scoped<T> {
  const baseFilter = (base as { [CordisContext.filter]?: (ctx: Context) => boolean })[CordisContext.filter]
  const carrier = {
    [CordisContext.filter](ctx: Context): boolean {
      if (baseFilter !== undefined && !baseFilter.call(base, ctx)) return false
      const tag = scopeOf(ctx)
      if (tag === undefined) return true
      for (let cursor = key; cursor !== undefined; cursor = scopeParents.get(cursor)) {
        if (cursor === tag) return true
      }
      return false
    },
  }
  carrierKeys.set(carrier, key)
  return carrier as unknown as Scoped<T>
}
```

判定规则：

| listener 的注册上下文 | 派发 key | 是否收到 |
|---|---|---|
| 未打 tag（全局 ctx 注册） | 任意 | 收到 |
| 打了 tag，tag == key 或 key 的祖先 | key | 收到 |
| 打了 tag，tag 在 key 之下 | key | **不收**（事件只向上流，不向下） |
| 任意 | `key === undefined`（无 scope 主体） | 只收未打 tag 的 |

`Scoped<T>` 是编译期品牌：scope 过滤事件的声明把 `this` 类型写成 `Scoped<Agent>`，拿着裸 service 派发会编译失败；载体本身不暴露 subject 属性，真实主体始终走事件参数。`carrierKeys` 这个 WeakMap 让 `isScopeCarrier` / `carrierKeyOf` 能回读路由 key。

## 在 agent/* 事件与 approval/request 中的作用域过滤

agent 生命周期事件全部声明为 scope 过滤（`packages/core/agent/src/runtime-types.ts`）：

```ts
'agent/pre-step'(this: Scoped<Agent>, payload: { agent: Agent; messages: UserMessage[]; ... }, next): Promise<PreStepDecision>
'agent/status'(this: Scoped<Agent>, payload: { agent: Agent; status: AgentStatus }): void
...
```

派发端由**融合派发器** `agentEvents`（`packages/core/agent/src/dispatch.ts`）保证 subject 与 scope key 永不分家：`agentCarrier(agent) = scopeTarget(agent, agent)` 造一次载体，每次派发都经它进行，并把 `agent` 注入 payload——调用方即使传了带 `agent` 字段的 payload 也会被展开顺序压住。注册端则很简单：`scope.ctx.on('agent/status', ...)` 注册的 listener 带着 scope tag，只收到这个 agent（及其后代）的事件。`dsh-subagent` 让一个常驻组合观察每个委派子 agent，靠的正是"祖先 tag 收后代事件"。

审批走同一条路（`packages/interaction/user-approval/src/index.ts`）：

```ts
scopeTarget(req.agent, req.agent), 'approval/request', req,
```

`approval/request` waterfall 用请求 agent 自己当 key 派发——审批策略插件通过某个 agent 的 `agent.ctx` 挂 listener，就只审批那个 agent 的请求；全局挂的 listener 审批所有 agent。`agent/request`、`agent/error` 等 waterfall/emit 同理。

注册表一侧的消费方式见 [store.md](store.md)：`system-prompt/assemble` 的声明也是 scope 过滤的（`this: Scoped<SystemPrompt>`，派发载体 `scopeTarget(this, scope)`），所以 scoped listener 只看到本 scope 的组装；而 `system-prompt/change` 这类"全局变更通知"故意**不**过滤。

## 与 Cordis 上下文的关系

| 概念 | Cordis 侧 | dsh-scope 侧 |
|---|---|---|
| 注册可见性 | `ctx.effect()` 挂在哪个上下文 | `scopeOf(ctx)` 读出的 tag 决定落层 |
| 生命周期 | fiber 销毁统一反注册 effect | `Scope.dispose()` = fiber 静止 |
| 事件过滤 | `Context.filter` 协议 | `scopeTarget` 组合 base filter + scope 谓词 |
| 依赖解析 | 铸造插件的 inject 链 | scope.ctx 继承（不收窄） |

局限也要认清（README "Known Limitations"）：只有**scope 感知的 API** 才隔离状态——一个普通服务被穿过 scoped 上下文调用时仍是全局的；一个上下文只携带最近的 scope tag，层级住在 key 级父链里而非上下文 tag 里；服务可达性来自铸造者，发出 `Scope.ctx` 即发出其依赖。

## 核心函数表

| 函数 | 位置 | 一句话 |
|---|---|---|
| `createScope(ctx, key, options?)` | `packages/core/scope/src/index.ts` | 铸造 scope：打 tag 的 ctx + 两条销毁路径 |
| `scopeOf(ctx)` | 同上 | 读上下文最近的 scope tag |
| `bindScopeParent(key, parent)` / `ScopeParentBinding.rebind` | 同上 | 一次性的父链绑定与受控重绑 |
| `scopeParentOf(key)` / `scopeChainOf(key)` | 同上 | 读父；最近优先的整条链 |
| `scopeTarget(base, key)` | 同上 | 造路由载体：未打 tag 全局放行，tag 命中 key 或其祖先放行 |
| `isScopeCarrier(value)` / `carrierKeyOf(value)` | 同上 | 载体判别与回读 key |
| `ScopedLayers` / `NamedEntries` / `AnonymousEntries` | `packages/core/scope/src/store.ts` | 注册表存储原语（见 [store.md](store.md)） |

## 与其他模块的关系

- **agent / agent-loop**：loop 用 `createScope(loopCtx, agent)` 造出 `agent.ctx`；`agentEvents` 融合派发器在所有 `agent/*` 事件上保证 subject 与 scope key 一致；驱动退出时销毁 scope。
- **system-prompt / tools**：`PromptLayer`、`ToolLayer`、`GuardLayer` 都实现 `ScopeLayer` 接口、用 `ScopedLayers` 分层；`agent.ctx` 注册的工具/段/限制只对该 agent 生效。
- **interaction（approval/questions）**：`approval/request` 等以 agent 为 key 派发，审批策略因此可以 per-agent。
- **subagent / preset**：委派与组合靠父链获得"祖先观察后代"的事件视图与"近层遮蔽远层"的注册视图。

## 小结

`dsh-scope` 用一个 200 行出头的源文件（`src/index.ts`，另有 267 行的 `store.ts` 存储层）给出三个正交原语：**不透明身份**（`ScopeKey`）、**拥有的注册上下文**（`createScope` 的 `Scope`，可见性与销毁所有权同源）、**路由载体**（`scopeTarget` + `Scoped<T>` 品牌与 `this` 类型约束）。一条父链关系支撑两个方向的语义：注册视图向下继承，事件准入向上延伸。它不引入沙箱或权限——它让"同一插件在不同 agent 手里有不同实例"成为注册表作者只要遵循 `scopeOf()` / `scopeTarget()` 两个约定就能免费获得的结构性质。
