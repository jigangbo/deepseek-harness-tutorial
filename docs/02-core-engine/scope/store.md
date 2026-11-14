# `store.ts`

> 作用域存储实现


`packages/core/scope/src/store.ts`（267 行）是 scope 感知注册表的存储层。它导出四个东西：`ScopeLayer` 接口（一层的空判约定）、`NamedEntries`（具名插入有序表）、`AnonymousEntries`（匿名插入有序表）、`ScopedLayers`（全局层 + 惰性 scoped 层的持有者）。`packages/core/scope/src/index.ts` 从这里重导出它们。注册型服务作者主要跟这三个类打交道——系统提示词的 `PromptLayer`、工具注册表的 `ToolLayer` 与 `GuardLayer` 都是照这个模板搭的。

## 概览

设计约束来自两个事实：

- **读不能有副作用**。任意多次读取（每个模型 step 的组装都会读好几遍）不得创建 scoped 层，否则一次诊断性读取就会留下残留状态；
- **可见性与所有权必须同源**。注册时拿到的 Cordis 上下文同时决定贡献落在哪一层、归谁销毁——不能让"可见于 A scope、销毁于 B scope"的组合存在。

| 成员 | 职责 |
|---|---|
| `ScopeLayer` | 一层对注册表的完整聚合贡献；只约定 `isEmpty()` |
| `NamedEntries<V>` | 具名条目：重名抛错（错误由调用方定制）、插入有序、精确撤销 |
| `AnonymousEntries<V>` | 匿名条目：每次 append 独立身份，等值不合并 |
| `ScopedLayers<L>` | 持有急切构建的全局层 + 惰性 exact-scope 层；`peek` / `chainLayers` / `merge` / `effect` |

## 代码走读

### 1. ScopeLayer：整层空判

```ts
/** One scope's aggregate contribution to a registry. */
export interface ScopeLayer {
  /** Whether every table in this layer is empty. */
  isEmpty(): boolean
}
```

一个具体层往往聚合多张表（工具层 = 具名 tools + 匿名 restrictions；提示词层 = 五张表）。`ScopedLayers` 回收 scoped 层时要求**整层**为空——不能因为一张表清了就把还有兄弟表的层删掉。

### 2. NamedEntries：具名条目与"代"（generation）

```ts
export class NamedEntries<V> implements EntryValues<V> {
  private data = new Map<string, V>()

  constructor(
    private readonly duplicateError: (name: string) => Error,
  ) {}

  insert(name: string, value: V): () => void {
    const data = this.data
    if (data.has(name)) throw this.duplicateError(name)
    data.set(name, value)
    let active = true
    return () => {
      if (!active) return
      active = false
      data.delete(name)
      if (data.size === 0 && this.data === data) this.data = new Map()
    }
  }
  ...
}
```

三个值得注意的细节：

- **重复错误由调用方定制**。构造时传入 `duplicateError(name) => Error`，于是全局层和 scoped 层能给出不同文案——系统提示词的全局重名会说"per-agent override 请走该 agent 的 `agent.ctx`"（见 `packages/core/system-prompt/src/index.ts` 的 `PromptLayer` 构造函数），工具注册表同理。
- **撤销是幂等且精确的**。`active` 标志让二次调用变成 no-op；只删这一次插入的那个名字，不动其他同名尝试。
- **排水即换代**。撤销后表空时 `this.data = new Map()` 换一个新 Map——此前借出的 live 迭代器（`keys()`/`entries()`/`values()` 直接返回原生 Map 迭代器）从此**脱离**这张表，之后的插入不再出现在旧迭代器里。这保证"迭代期间注册表被清空"不会让旧迭代器看见新代的数据；只要表非空，迭代器在当前代内是活的。`AnonymousEntries` 有完全相同的换代逻辑。

`get` / `has` / `keys` / `entries` / `values` 都是直接借用 `Map` 的对应方法，迭代顺序即插入顺序。

### 3. AnonymousEntries：symbol 键的独立身份

```ts
export class AnonymousEntries<V> implements EntryValues<V> {
  private data = new Map<symbol, V>()

  append(value: V): () => void {
    const data = this.data
    const key = Symbol()
    data.set(key, value)
    let active = true
    return () => { ... data.delete(key) ... }
  }
```

没有名字的贡献（工具 schema provider、运行时上下文压制器、restriction、guard）每次 append 拿到独立的 `Symbol()` 键：**相等的值仍是两条独立注册**，各有各的撤销句柄。这与 `Map` 对键做 `SameValueZero` 合并的行为刻意不同——两条相同内容的压制器应当独立解除。

### 4. ScopedLayers：全局层 + 惰性 scoped 层

```ts
export class ScopedLayers<L extends ScopeLayer> {
  readonly global: L
  private readonly scoped = new Map<ScopeKey, L>()

  constructor(
    private readonly createLayer: (scope: ScopeKey | undefined) => L,
    private readonly onChange: () => void,
  ) {
    this.global = createLayer(undefined)
  }
```

构造函数立即建全局层（大多数注册落在这里，不值得惰性），scoped 层等第一条 scoped 注册出现才创建。

**读取三件套**（都不创建层）：

```ts
peek(scope)        // 精确取该 scope 自己的层；undefined 表示无覆盖。
                   // 刻意"链盲"：查看某 scope 自己的贡献（restrictions、guards）
                   // 时不能悄悄捡到祖先的——要继承语义请用 chainLayers。

chainLayers(scope) // scopeChainOf(scope).reverse()：最远祖先在前、精确 scope 在后。
                   // 调用方按序叠层，近层自然获得最终发言权。

merge(scope, pick) // 先拷贝全局具名表，再按 chainLayers 逐层 Map.set 覆盖。
                   // 产出"全局 + scope 链遮蔽"的有效具名视图，近层同名胜出。
```

`peek` 与 `chainLayers` 的分工写在 JSDoc 里：一个是"**自己的**"语义（工具注册表的 restriction 检查用它——某层限制只约束该层自己），一个是"**继承的**"语义（可见性合并用它）。

### 5. effect：一次注册，三件事

`effect()` 是唯一的写入口，把"层选择、变动、通知"焊在一个 Cordis effect 里：

```ts
effect(
  ctx: Context,
  action: (layer: L) => () => void,
  options: { label: string; notify?: boolean },
): () => void {
  const scope = scopeOf(ctx)
  const notify = options.notify ?? true
  const dispose = ctx.effect(function* (this: ScopedLayers<L>) {
    let layer: L
    let created = false
    if (scope === undefined) {
      layer = this.global
    } else {
      const existing = this.scoped.get(scope)
      if (existing === undefined) {
        layer = this.createLayer(scope)
        this.scoped.set(scope, layer)
        created = true
      } else {
        layer = existing
      }
    }

    let undo: () => void
    try {
      undo = action(layer)
    } catch (error) {
      if (scope !== undefined && created && layer.isEmpty()) this.scoped.delete(scope)
      throw error
    }

    yield () => {
      undo()
      if (scope !== undefined && layer.isEmpty()) this.scoped.delete(scope)
      if (notify) this.onChange()
    }
    if (notify) this.onChange()
  }.bind(this), options.label)
  return dispose
}
```

逐段读：

1. **`scopeOf(ctx)` 先行**——注册上下文的 scope tag 在 effect 安装那一刻定格，决定落层；`ctx.effect()` 同时把这个变动挂进该上下文的 fiber，插件卸载/agent 销毁时 `yield` 出的撤销函数自动执行。可见性与所有权同源就是在这里实现的。
2. **惰性建层 + 失败不留残渣**。`action` 抛错时，若这个 scope 层是本次新建且仍是空的，立即从 `scoped` 删除——半初始化的层不会被 `peek` 看见。
3. **先收 undo，后通知**。`action` 返回同步撤销函数，注册成功后（以及将来撤销后）各触发一次 `onChange()`。调用方通常在这里 emit `tools/change` / `system-prompt/change` 这类"注册表变了"通知；`notify: false` 可关掉（例如注册内部还要继续做别的时）。
4. **整层空才回收**。撤销后层空即从 `scoped` 删除，弱引用式的生命周期：最后一个 agent 级工具反注册后，该 agent 的层不复存在。
5. **返回值是 Cordis 的精确 disposer**（带一行 oxlint 注释解释为何保留同步 disposer 身份），所以嵌进有序复合 effect 不会破坏撤销顺序。

## 函数表

| 成员 | 签名要点 | 一句话 |
|---|---|---|
| `ScopeLayer.isEmpty()` | 接口 | 整层空判，层回收的判据 |
| `NamedEntries.insert(name, value)` | 抛调用方定制的重复错误；返回幂等撤销 | 具名注册 |
| `NamedEntries.get/has` | 读 | 精确查询（不沿链） |
| `NamedEntries.keys/entries/values` | live 迭代器 | 插入有序；表排空后旧迭代器脱离 |
| `AnonymousEntries.append(value)` | 返回幂等撤销 | 独立身份追加，等值不合并 |
| `ScopedLayers.peek(scope)` | 不创建层，链盲 | 该 scope **自己**的层 |
| `ScopedLayers.chainLayers(scope)` | 远祖先在前、精确 scope 在后 | 继承遍历的层序列 |
| `ScopedLayers.merge(scope, pick)` | 全局 + 链上遮蔽 | 有效具名视图，近层胜出 |
| `ScopedLayers.effect(ctx, action, options)` | 返回精确 disposer | 唯一写入口：落层 + 挂 effect + 通知 |

## 关系

- **index.ts**：`createScope` 产出带 tag 的上下文；`scopeOf` 是 `effect()` 落层的判据；`scopeChainOf` 是 `chainLayers` 的底座。父链语义见 [index.md](index.md)。
- **system-prompt**（`packages/core/system-prompt/src/index.ts`）：`PromptLayer implements ScopeLayer`，五张表（sections/contexts 具名，suppressors/toolProviders 匿名，variables 具名）；`assemble()` 用 `merge()` 求有效段/上下文，用 `chainLayers()` 收集匿名 provider 与压制器。
- **tools**（`packages/core/tools/src/index.ts`）：`ToolLayer`（具名 tools + 匿名 restrictions）与 `GuardLayer`（匿名 guards）各挂一个 `ScopedLayers`；`peek()` 的链盲语义被 `view()` 用来看"本层自己的 restriction"。
- **其他消费方式**：任何"全局默认 + per-agent 覆盖"的注册表都可以直接套这个模板——实现 `ScopeLayer`、给 `ScopedLayers` 传层工厂与 change 通知、写入口走 `effect()`。

## 小结

`store.ts` 把 scoped 注册表的存储问题压缩成三件事：**具名表**（重名响亮失败、插入有序、精确幂等撤销、排水换代）、**匿名表**（symbol 身份，等值不合并）、**层持有者**（读三件套不建层，`effect()` 唯一写入口同时完成落层、挂 effect、通知）。全部机制服务一条不变式：注册上下文同时决定可见性与销毁所有权，任何一层在完全变空后不留痕迹。
