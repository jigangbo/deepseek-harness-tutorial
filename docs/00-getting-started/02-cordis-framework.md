# 02 · Cordis 框架精讲


上一章我们建立了"Agent Harness = 模型之外的所有工程"的概念。本章讲解 DeepSeek Harness(下文简称 `dsh`)的地基——**Cordis 插件框架**。`dsh` 的每一个部分:模型适配器、工具注册表、会话日志、agent 循环本身,都是挂在 Cordis Context 上的插件。**读懂 Cordis,就读懂了整个项目的组织方式**;后面所有章节的源码阅读,都建立在本章的词汇之上。

Cordis 的源码以"固定版本拷贝"(vendored)的形式存放在主仓库 `vendor/cordis/` 下,本文所有源码引用均相对于主仓库根目录,例如 `vendor/cordis/src/events.ts`。你随时可以打开这些文件对照阅读。

本章依次讲:Cordis 是什么;Context 与插件生命周期;服务(Service);事件系统与 Waterfall 深讲;配置与 loader;三段关键源码走读;dsh 中的落地;常见误区清单。

---

## 1. Cordis 是什么

Cordis 是一个为**可组合的长期运行应用**设计的插件框架。它回答的问题是:当一个程序要长期运行、能力不断扩展、各个部分可以独立热替换时,如何组织代码,才不会退化成一团互相 import 的面条?

Cordis 的答案是三条核心机制:

- **插件向共享 Context 贡献能力**。每个插件把自己的功能挂到一个共享的 `ctx` 对象上(服务)、或者声明对某些事件的兴趣(监听器),插件之间不直接 import。
- **类型化事件做间接通信**。插件宣布"发生了什么",其他插件决定是否关心;调用方和监听方互不知道对方存在。
- **一切注册皆可逆 effect**。插件运行期间做的每一次注册——监听器、子插件、服务、工具——都记录在一个统一的回收列表里;插件卸载时,Cordis 自动按注册记录反向撤销,不留残余。

官方入门文档 `docs/cordis-primer.md` 把它总结为五个要点,值得背下来:

| # | 要点 | 一句话解释 |
|---|------|-----------|
| 1 | 插件是实现 Service 的对象 | 函数(带可选 `inject`/`apply` 字段)、`{ apply }` 对象、或 `Service` 子类,三种形态皆可 |
| 2 | Context 是服务的仓库 | 服务占一个稳定的 `ctx.<key>`,如 `ctx.tools`、`ctx.llm`;消费方按名字找人,不 import 具体实现 |
| 3 | 依赖用 `inject` 声明 | 写明需要哪些服务,Cordis 等它们就绪再启动插件——加载顺序由依赖决定,不由文件顺序决定 |
| 4 | 类型化事件做通信 | 服务通过 TypeScript 声明合并定义事件名,按 `emit`/`waterfall`/`parallel`/`serial`/`bail` 五种模式派发 |
| 5 | 注册即可逆 effect | 工具 schema、提示词段落、适配器、监听器都通过 `ctx.effect()`/`ctx.on()` 安装,热重载和卸载时可预测地回退 |

### 1.1 源码地图

`vendor/cordis/src/` 一共九个文件,核心是五个:`context.ts`(Context 类:根上下文与 `extend`/`isolate`/`intercept` 子上下文)、`events.ts`(事件总线:五种派发模式、`on`/`once` 注册)、`fiber.ts`(Fiber:插件运行实例,生命周期状态机、effect 回收、config 校验)、`registry.ts`(插件注册表:三种插件形态归一化、`inject` 依赖解析、`ctx.plugin()`)、`service.ts`(Service 基类:服务在 ctx 上的注册)。其余是支撑:`reflect.ts`(Context 的 Proxy handler、服务解析、mixin)、`logger.ts`(日志服务)、`utils.ts`(内部工具)、`index.ts`(汇总导出)。

配套的动手教程在 `docs/cordis-tutorial/`(七个章节,每章一个可运行的小例子),概念速查在 `docs/cordis-primer.md`。本章是这两者的精讲与内化。

---

## 2. Context 与插件生命周期

### 2.1 Context:一个被 Proxy 包住的服务仓库

`Context` 是整个框架的舞台。看它的构造函数(`vendor/cordis/src/context.ts`):

```ts
/** Create the root context and install the built-in services. */
constructor() {
  this[symbols.isolate] = Object.create(null)
  this[symbols.intercept] = Object.create(null)
  const self = new Proxy<this>(this, ReflectService.handler)
  this.root = self
  this.baseUrl = undefined
  this.fiber = new Fiber(self, {}, Object.create(null), null, () => [])
  this.reflect = new ReflectService(self)
  this.registry = new RegistryService(self)
  this.events = new EventsService(self)
  this.logger = new LoggerService(self)
  this.fiber._disposables.clear()
  return self
}
```

三个关键事实:

1. **Context 是一个 Proxy**(`new Proxy(this, ReflectService.handler)`)。普通属性读取会经过 `ReflectService.handler` 的 `get` 陷阱,也就是服务解析器——这正是"accessor 模式"的实现基础(第 3 节展开)。
2. **框架自带四个内置服务**:`ctx.reflect`(反射层)、`ctx.registry`(插件注册表)、`ctx.events`(事件总线)、`ctx.logger`(日志)。它们自己也是用第 3 节要讲的 Service 机制挂上来的——框架同样吃自己的狗粮。
3. `ctx.on`、`ctx.plugin`、`ctx.effect` 这些你最常用的方法,并不是 `Context` 类的成员,而是通过 `ctx.mixin()` 从内置服务**转发**到 ctx 上的(`vendor/cordis/src/reflect.ts`):

```ts
this.mixin('reflect', ['get', 'set', 'provide', 'accessor', 'mixin'])
this.mixin('fiber', ['runtime', 'effect'])
this.mixin('registry', ['inject', 'plugin'])
this.mixin('events', ['on', 'once', 'parallel', 'emit', 'serial', 'bail', 'waterfall'])
```

所以 `ctx.on(...)` 实际转发给 `ctx.events.on(...)`,`ctx.plugin(...)` 转发给 `ctx.registry.plugin(...)`。每个 mixin 也是 effect,所属服务卸载时会一并移除。

Context 还支持派生子上下文,三者都不修改父级:

- `ctx.extend(meta)`:原型继承父级全部属性,叠加自己的元数据(每个插件得到的 `ctx` 就是这样一份带 `fiber` 的子上下文);
- `ctx.isolate(name, label?)`:为名为 `name` 的服务开辟独立作用域——两个 `isolate` 之后的子树可以各自挂一个不同配置的 `shell` 提供者而互不干扰;
- `ctx.intercept(name, config)`:给某个服务的 per-plugin 配置注入一层合并项。

### 2.2 插件的三种形态

Cordis 接受三种插件形态(`vendor/cordis/src/registry.ts` 的 `Plugin` 类型):

```ts
// 1. 函数插件:最常见,Cordis 直接调用它
export function apply(ctx: Context, config: Config) {}

// 2. 对象插件:带 apply 方法的对象
export const objectPlugin = {
  name: 'object-plugin',
  apply(ctx: Context) {},
}

// 3. 类插件:Service 子类(要暴露服务时用,见第 3 节)
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myService')
  }
}
```

三者之外,插件还可以带这些声明式元数据(`Plugin.Base`):

| 字段 | 作用 |
|------|------|
| `name` | 诊断用的显示名 |
| `Config` | Standard Schema 校验器,在 `apply` 运行前校验配置(第 5 节) |
| `inject` | 需要的服务名列表,未就绪则插件等待 |
| `provide` | 本插件提供的服务名(供 loader 和 `Service` 基类读取) |
| `intercept` | 声明本插件消费哪些服务的 intercept 配置 |

### 2.3 ctx.plugin() 与 Fiber

`ctx.plugin(plugin, config?)` 把一个插件装载进当前上下文,返回一个 **Fiber**——"一个已装载插件实例"的运行时句柄。Fiber 同时是 PromiseLike 的,可以 `await ctx.plugin(P)` 等它稳定。

Fiber 的生命周期是一个状态机(状态枚举见 `vendor/cordis/src/fiber.ts` 的 `FiberState`):

```text
                ctx.plugin(P, config)
                        │
                        ▼
                  ┌───────────┐   inject 声明的服务尚未就绪:
        ┌────────▶│  PENDING  │   静静等待,不报错、不运行 apply
        │         └─────┬─────┘◀─────────────────────────┐
        │               │ 所有依赖就绪                     │
        │               ▼                                 │
        │         ┌───────────┐  apply 抛错/               │
        │         │  LOADING  │  配置校验失败  ┌────────┐  │
        │         └─────┬─────┘─────────────▶│ FAILED │  │
        │               │ apply 正常返回       └────────┘  │
        │               ▼      (fiber.update() 可清除错误)  │
        │         ┌───────────┐                           │
        │         │  ACTIVE   │  服务已发布、监听器已生效     │
        │         └─────┬─────┘                           │
        │               │ 卸载触发:配置变更 / 依赖消失 /     │
        │               │ 父级卸载 / HMR / 手动 dispose      │
        │               ▼                                  │
        │         ┌───────────┐  卸载结束但 epoch 重新活跃     │
        └─────────│ UNLOADING │  (依赖恢复/配置更新)→ 重新 LOAD ─┘
                  └─────┬─────┘
                        │ 彻底卸载(uid 被清空)
                        ▼
                  ┌───────────┐
                  │ DISPOSED  │  不可复活;再注册 effect 会抛错
                  └───────────┘
```

判读规则(`fiber.ts` 的 `_getState()`):`uid === null` 即 DISPOSED;有 `_error` 即 FAILED;epoch 活跃(依赖齐备)即 ACTIVE;否则 PENDING。一个插件"什么都不打印"最常见的原因就是卡在 PENDING——第 8 节讲诊断方法。

### 2.4 effect:一切注册的统一记账

Cordis 的核心纪律:**Registrations are effects(注册即 effect)**。任何"插件活着才该存在的东西",都通过 effect 登记,由 Cordis 统一回收。

`ctx.effect()` 的用法(示例改编自 `docs/cordis-tutorial/02-lifecycle-and-effects.md`):

```ts
export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => console.log('tick'), 200)
    // 返回的 disposer 会在插件卸载时被自动调用
    return () => {
      clearInterval(timer)
      console.log('heartbeat cleaned up')
    }
  })
}
```

effect 体立即执行;它返回的 disposer(可以是异步函数,也可以产出多个 disposer 的迭代器)被记录下来,在插件卸载时运行。你几乎从不需要手写 `ctx.effect()`,因为内置注册 API 全部是 effect:

- `ctx.on(event, listener)` —— 监听器随插件卸载自动移除;
- `ctx.plugin(child)` —— 子插件随父插件卸载(子 Fiber 的 dispose 本身就注册在父 Fiber 的 effect 列表里,标签叫 `ctx.plugin()`);
- 服务注册是 effect —— `super(ctx, 'name')` 内部调用 `ctx.reflect.provide(...)`,卸载时自动注销;
- dsh 的各注册表(如 `ctx.tools.register(...)`)也把返回的 disposer 挂到调用方的 Fiber 上,卸载即注销。

**顺序语义**要记准:同一个 effect 内部收集的多个 disposer,按**注册逆序**逐个执行;而**不同** effect 之间的异步 disposer,在卸载时是**并发**执行的(`_unload` 用 `Promise.all` 同时等待所有 disposer)。如果几个清理步骤必须先后发生,把它们放进同一个 disposer 里顺序 `await`。

### 2.5 inject:依赖驱动的加载与卸载

`inject` 声明插件需要的服务名:

```ts
export const name = 'consumer'
export const inject = ['greeter']

export function apply(ctx: Context) {
  // 走到这里时 ctx.greeter 一定已就绪
  console.log(ctx.greeter.greet('world'))
}
```

Cordis 把插件扣在 PENDING 状态,直到 `inject` 列出的每个服务都存在;`apply` 内部才可以放心使用。因此 `cordis.yml` 里条目的书写顺序不保证加载顺序——**顺序由依赖关系决定**。

更重要的是,依赖追踪不是"启动时检查一次"就完事。Fiber 内部维护一个 epoch 字符串(第 6 节走读它的实现),每次依赖集合发生变化就重新计算:

- 某个依赖的服务**消失**(提供者被卸载/热替换)→ epoch 变为 INACTIVE → 本插件的所有 effect 被回收,回到 PENDING;
- 服务**回来**(或换了一个实现)→ epoch 重新活跃 → `apply` 重新运行。

这就是热更新(HMR)和服务替换能安全工作的底层保证:卸载 `dsh-bash-local`、换一个别的 `shell` 提供者,所有 `inject: ['shell']` 的插件会干净地针对新实现重启一遍,而不是拿着旧引用继续跑。

---

## 3. 服务(Service)

### 3.1 accessor 模式:`ctx.<key>` 是"按需解析"出来的

服务是"一个插件提供、其他插件通过 `ctx` 消费的能力"。`ctx.tools`、`ctx.llm`、`ctx.agents` 都是服务。

它的实现不是普通的对象属性赋值,而是 **accessor 模式**:Context 的 Proxy `get` 陷阱(`vendor/cordis/src/reflect.ts` 的 `ReflectService.handler`)在每次属性读取时动态解析服务实现,核心路径如下(节选):

```ts
static handler: ProxyHandler<Context> = {
  get: (target, prop, ctx: Context) => {
    if (isSpecialProperty(prop)) return Reflect.get(target, prop, ctx)
    if (Reflect.has(target, prop)) return getTraceable(ctx, Reflect.get(target, prop, ctx))

    const error = new Error(`cannot get property "${prop}" without inject`)
    // ... accessor 类型走自定义 get;服务类型走 internal/get waterfall:
    return ctx.events.waterfall('internal/get', ctx, prop, error, () => {
      // 沿 fiber 链向上查找名为 prop 的实现;没有 inject 就抛错
    })
  },
  // ...
}
```

含义:读 `ctx.tools` 时,如果 `tools` 既不是自有属性也不是 accessor,解析器就**沿着 fiber 父子链向上找**谁提供了名为 `tools` 的实现,并且只在你(或祖先)声明过对这个名字的依赖时才放行——没有 `inject` 而直接读,得到的是 `cannot get property "tools" without inject` 这样的错误。这把"用了没声明的服务"从静默的 `undefined` 变成了响亮的失败。

顺带一提,服务读取本身也要经过一个 waterfall 事件(`internal/get`),核心服务可以借它拦截、诊断每一次服务访问——框架自己的扩展点也是用第 4 节的事件机制搭的。

### 3.2 Service 基类:定义与注册

`vendor/cordis/src/service.ts` 的 `Service` 基类只有几十行,构造函数是关键:

```ts
constructor(protected ctx: Context, name: string) {
  name ??= this.constructor['provide'] as string

  let self = this
  const tracker: Tracker = { associate: name, property: 'ctx' }
  if (self[symbols.invoke]) {
    // 声明了 [Service.invoke] 的服务是可调用的(如 ctx.logger())
    self = createCallable(name, joinPrototype(Object.getPrototypeOf(this), Function.prototype), tracker)
  }
  self.ctx = ctx
  self.name = name
  defineProperty(self, symbols.tracker, tracker)

  // 把自己注册为名为 name 的服务;这一步是 effect,随所属 fiber 卸载自动注销
  self.ctx.reflect.provide(name, self, this[symbols.check])
  return self
}
```

要点:

- `super(ctx, 'greeter')` 一执行,实例就通过 `ctx.reflect.provide(name, self, check)` 注册到 ctx 上;第三个参数是可选的**可用性谓词**,决定这个实现对依赖方是否"算数"。
- 服务名活在**每应用一个的扁平命名空间**里。dsh 占用了 `tools`、`llm`、`sessions`、`agents` 这些朴素名字(见 `packages/core/tools/src/index.ts`、`packages/core/session/src/index.ts` 等处的 `super(ctx, '<name>')`);你自己的服务应该加独特前缀。
- **服务定义与实现分离**:消费方只认服务**名**,不认实现类。所以同一个 `shell` 服务,今天由本地执行器提供,明天可以换成远程执行器,消费方一行不改——这正是 dsh 能力接缝(capability seam:服务定义/服务提供者/消费方三角色)的类型学基础。

### 3.3 declare module 类型合并:让服务可被任何插件扩展

`ctx.greeter` 在运行时是动态解析的,但 TypeScript 怎么知道 `ctx.greeter` 的类型?答案是**声明合并**(declaration merging)。每个提供服务的插件在自己源码里写:

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    greeter: GreeterService
  }
}
```

这段代码不生成任何运行时逻辑,它只是把 `greeter` 这个属性"告诉" TypeScript 的 `Context` 接口。运行时由 `ctx.plugin(GreeterService)` 提供实体,编译时由 `declare module` 提供类型,两者配对。

dsh 里到处都是这个模式。真实例子,`packages/core/tools/src/index.ts`(工具注册表服务):

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    tools: ToolsService
  }
}
```

以及事件类型的合并(`packages/core/tools/src/events.ts`,下一节详细讲):

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'tools/pre-execute'(...): Promise<PreToolDecision>
    /* ... */
  }
}
```

这个机制的本质是:**任何插件都可以在不修改框架源码的前提下,扩展 `ctx` 的类型面**。Cordis 的 `Context` 接口只声明了少数内置属性(`events`/`logger`/`reflect`/`registry` 等),血肉全部由插件用声明合并长上去。反过来,消费方要看到类型,只需 `import type {} from '@deepseek-ai/dsh-tools'` 把对方的声明合并拉进编译。

### 3.4 一个完整的提供方 + 消费方

把上面三块拼起来(示例来自 `docs/cordis-tutorial/03-services.md`;`declare module` 块与 3.3 相同,略):

```ts
// greeter.ts —— 提供方
export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')          // 运行时:注册为 ctx.greeter
  }
  greet(who: string) { return `Hello, ${who}!` }
}

export function apply(ctx: Context) {
  ctx.plugin(GreeterService)       // 类插件本身也是插件,这样装载
}
```

```ts
// consumer.ts —— 消费方
export const inject = ['greeter']  // 声明依赖;就绪前本插件保持 PENDING

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))
}
```

可选依赖则不要写 `inject`,在使用点探测:`ctx.get('greeter')` 在没有提供者时返回 `undefined`(`strict` 默认只返回活跃 fiber 提供的实现)。

---

## 4. 事件系统

服务支持直接调用;**事件**让一个插件宣布事情的发生,而无需知道谁在听。dsh 的工具结果、模型请求、审批决定,都走事件。

### 4.1 事件域:事件属于拥有它的服务

事件名是一个扁平字符串,但按照惯例(也是 `docs/cordis-primer.md` 的 Practical Rules),每个事件归属一个服务域:

- 工具管线的事件属于 `ctx.tools`(`tools/result`、`tools/pre-execute`);
- 模型流式请求的事件属于 `ctx.llm`;
- 存活 agent 协调的事件属于 `ctx.agents`。

从名字空间看,`vendor/cordis/src/events.ts` 的 `Events` 接口里有三类条目:

| 类别 | 例子 | 说明 |
|------|------|------|
| `internal/*` | `internal/status`、`internal/update`、`internal/get` | 框架内部事件:fiber 生命周期、配置更新、服务访问等,是核心服务之间的扩展点 |
| `namespace/action` | `tools/result`、`agent/pre-step`、`approval/request` | 业务域事件,由 dsh 各包通过声明合并添加,`@mode` 标签写进 JSDoc |
| symbol 键 | `[key: symbol]: (...args: any[]) => any` | 允许用 symbol 做私有事件名,避开公共命名空间 |

### 4.2 声明、发射、监听

事件类型同样走声明合并——`interface Events` 是 `interface Context` 的孪生:

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'stats/report'(name: string, count: number): void
  }
}
```

声明之后,`ctx.emit('stats/report', name, count)` 与 `ctx.on('stats/report', (name, count) => {...})` 的参数与返回值就全部获得类型检查。`ctx.on()` 返回一个 disposer,但因为它本身是 effect,监听器**随插件卸载自动移除**——永远不需要手写 `removeListener`。

`ctx.on` 还接受第三个参数 `options`:`{ prepend: true }`(或简写布尔值 `true`)把监听器插到队首;`{ global: true }` 让监听器无视上下文过滤器(作用域过滤)。`ctx.once()` 是"触发一次自动注销"的糖。

dsh 的事件 JSDoc 有硬性要求:必须写 `@mode` 标注派发模式、每个 payload 参数写 `@param`。生成的事件目录会核对声明与发射点。真实例子(`packages/core/tools/src/events.ts`,节选):

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    /**
     * Allow, deny, cancel, or ask before dispatch. `next()` delegates to allow; ...
     * @param exec - the pending call (name, parsed arguments, caller agent).
     * @mode waterfall
     */
    'tools/pre-execute'(this: Scoped<ToolsService>, exec: ToolExecution,
      next: () => Promise<PreToolDecision>): Promise<PreToolDecision>
    // 'tools/result' 同在此声明,@mode emit:观察冻结后的最终结果,监听器失败被隔离
  }
}
```

### 4.3 五种派发模式

派发模式是事件公开契约的一部分:它决定监听器能否返回值、是否并发、能否短路彼此(`vendor/cordis/src/events.ts`):

| 模式 | 调用 | 等待? | 顺序 | 返回值 |
|------|------|-------|------|--------|
| `emit` | `ctx.emit(name, ...args)` | 否 | 按注册顺序同步广播 | 无(返回的 Promise 不会被等待) |
| `waterfall` | `ctx.waterfall(name, ...args, next)` | 否 | 按注册顺序,外层包裹内层 | 最外层监听器的返回值 |
| `parallel` | `await ctx.parallel(name, ...args)` | 是 | 全部并发 | 无(任一失败抛 AggregateError) |
| `serial` | `await ctx.serial(name, ...args)` | 是 | 按注册顺序逐个 await | 第一个 bail 值 |
| `bail` | `ctx.bail(name, ...args)` | 否 | 按注册顺序同步 | 第一个 bail 值 |

"bail 值"的定义在 `events.ts` 第一行:

```ts
export function isBailed(value: any) {
  return value !== null && value !== false && value !== undefined
}
```

即:`null`/`false`/`undefined` 算"没有表态",任何其他返回值都会让 `serial`/`bail` 提前结束并返回该值。`serial` 是 `bail` 的异步版,`parallel` 用 `Promise.allSettled` 收集全部结果。

### 4.4 Waterfall 深讲:本项目最容易踩的坑

Waterfall 是 **around 中间件**(环绕中间件),dsh 用它做一切"可拦截的决策"。每个监听器收到 `(…args, next)`,调用 `next()` 就把控制权交给下一个监听器(最终是内层默认实现);**不调用 `next()` 直接返回,就是否决(veto)——包括默认实现在内的整条右链都不会运行**。

先看实现(`vendor/cordis/src/events.ts`,全文仅十几行):

```ts
waterfall(...args: any[]) {
  const [thisArg, callbacks] = this._resolve('waterfall', args)
  const inner = args.pop()               // 最后一个参数是内层默认实现 next
  const dispatch = () => {
    const callback = callbacks.shift()   // 按注册顺序,最先注册的最外层
    if (!callback) return inner()        // 监听器用尽 → 执行默认实现
    let called = false
    const next = () => {
      if (called) throw new Error('next() called multiple times')
      called = true
      return dispatch()                  // next() 就是"继续派发下一个"
    }
    return Reflect.apply(callback, thisArg, [...args, next])
  }
  return dispatch()
}
```

数据流(一次三段链的完整往返):

```text
调用方:await ctx.waterfall('tools/execute', exec, 默认执行 inner)
                                        │
            ┌───────────────────────────┘
            ▼
┌─────────────────────────────────┐
│ 监听器 1(最先注册,最外层)        │
│   const result = await next() ──┼──▶ ┌─────────────────────────────────┐
│   return 上报指标(加工 result)   │    │ 监听器 2                          │
└─────────────────────────────────┘    │   if (被否决) return 替代结果      │
                                       │   return next() ────────────────┼──▶ inner()
                                       └─────────────────────────────────┘    (默认实现)
返回值沿调用栈原路回传:inner → 监听器 2(可替换)→ 监听器 1(可再加工)→ 调用方
任何一层不调用 next(),其右侧的整条链(含 inner)都不执行
```

两个典型用法,来自 `docs/cordis-tutorial/04-events.md` 的可运行示例:

```ts
// 监听器 1:包裹下游结果(协作型,必须调用 next())
ctx.on('demo/transform', async (input, next) => {
  const downstream = await next()
  return downstream.toUpperCase()
})

// 监听器 2:拥有决策权时短路(否决型,故意不调用 next())
ctx.on('demo/transform', async (input, next) => {
  if (input.includes('blocked')) return '** blocked **'
  return next()
})
```

对 `'blocked words'`,监听器 1 先运行并调用 `next()`,进入监听器 2;监听器 2 看到 `blocked` 直接返回替代值——内层默认实现根本没有运行——监听器 1 再把替代值转成大写返回。最终输出 `** BLOCKED **`。

由此得出本仓库的铁律(**AGENTS.md 明文规则**):

> **Waterfall 监听器必须调用 `next()` 来委托;返回而不调用 `next()` 会短路整条链。**

- 只观察/只注记的监听器(打日志、记指标)**必须**调用 `next()`,否则它之后的所有监听器和默认行为都被它静默吞掉——这是本框架最容易犯、也最难排查的错误;
- 对于"单一决策"事件,短路就是设计:策略监听器拥有决策权时故意不 `next()`,比如 `agent/pre-step` 允许插件否决一个 step、`approval/request` 允许策略代替用户回答;
- `prepend: true` 只在监听器必须先于普通注册者运行时使用。

dsh 真实的 waterfall 事件链(工具执行管线,`packages/core/tools/src/events.ts`):

```text
ctx.tools.execute(...)
  │
  ├─ tools/pre-execute   (waterfall) 允许 / 拒绝 / 取消 / 转人工审批
  │
  ├─ tools/execute       (waterfall) 环绕真正的派发:超时、重试、指标
  │
  ├─ tools/post-execute  (waterfall) 接受 / 替换 / 增强 / 阻断规范化结果
  │
  └─ tools/result        (emit)     观察最终冻结结果(监听器失败被隔离,
                                     不影响管线)——这也是教程第 7 章
                                     tool-logger 插件监听的事件
```

前三个是"可改写决策"的 waterfall,最后一个是纯观察的 `emit`——**用哪种模式,取决于监听器被允许做什么**,这个选择本身就是事件契约的一部分。

---

## 5. 配置与 loader

### 5.1 cordis.yml:配置即应用

`cordis.yml` 是一个插件条目列表,loader 逐条装载。一个条目可以携带的元数据:

```yaml
- id: greeter           # 稳定身份:loader 用它区分"改了现有条目"和"删了再加"
  name: './greeter.ts'  # 模块标识:相对路径或 npm 包名
  disabled: true        # 保留条目但不装载;翻回 false 即恢复
  config:               # 传给插件的配置,先过 schema 校验
    greeting: 你好
- name: '@deepseek-ai/cordis-plugin-timer'   # npm 包形态
```

条目**并发启动**,所以列表顺序不保证加载顺序(顺序由 `inject` 决定)。条目可以被分组(group)嵌套为一整个装载/卸载单元;group 上的 `isolate` 给子列表一个独立的服务作用域。

### 5.2 Config schema:先校验,后启动,坏配置响亮失败

插件导出一个 `Config`,它既是 TypeScript 类型又是运行时校验器(dsh 用 Schemastery;Cordis 本身接受任何 Standard Schema 校验器):

```ts
import Schema from '@deepseek-ai/schemastery'

export interface Config {
  greeting: string
  targets: string[]
}
export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  // apply 收到的永远是校验并补全默认值之后的完整配置
}
```

校验发生在 `apply` 运行之前(`vendor/cordis/src/fiber.ts` 的 `resolveConfig`),不通过就抛 `ValidationError`,fiber 进入 FAILED,插件**绝不带病启动**;错误信息精确到字段路径:

```text
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

另一类"配置改一改就生效、无需重载插件"的字段用 `.volatile()` 标记:变化只更新稳定引用,不重新挂载插件(细节见 `docs/cordis-tutorial/05-config.md`)。

### 5.3 loader 配置里的 `!!js`

需要"装载时计算"的配置值用 `!!js` 标签:

```yaml
- name: '@deepseek-ai/dsh-session-persistence-jsonl'
  config:
    root: !!js dshHomePath('sessions')
- name: '@deepseek-ai/dsh-bash-sandbox'
  disabled: !!js process.platform === 'win32'
```

作用域有硬性限制(见 `docs/cordis-primer.md` 的 Loader Configuration):`!!js` **只允许出现在条目的 `config` 和 `disabled` 两个字段里**。`config` 里的表达式在该插件上下文激活后求值;`disabled` 在每次装载决策时对 loader 上下文求值(用于按平台/环境关停一行)。其余元数据(`name`、`id`、`inject`……)一律是字面量。需要按环境切换插件组合时,用 overlay,不要在别的字段里塞表达式。

### 5.4 overlay 与 patch:分层组合

dsh 的应用组装是**分层 patch**:基础层声明全部行,后续层(模式 bundle、用户的 profile `cordis.patch.yml`、命令行 `--patch` overlay)按 `id` 寻址覆写,每行最后写入者胜出;patch 替换目标行的**整个 `config`**,不是合并。看 `packages/bundle/base/cordis.patch.yml` 的开头(节选):

```yaml
- insert:
    - id: timer
      name: '@deepseek-ai/cordis-plugin-timer'

    - id: llm
      name: '@deepseek-ai/dsh-llm'

    - id: session
      name: '@deepseek-ai/dsh-session'

    - id: agent
      name: '@deepseek-ai/dsh-agent'

    - id: tool-bash
      name: '@deepseek-ai/dsh-tool-bash'
      disabled: !!js process.platform === 'win32'

    - id: tool-pwsh
      name: '@deepseek-ai/dsh-tool-pwsh'
      disabled: !!js process.platform !== 'win32'
    # ...(insert 列表共约 90 个条目,即一个完整 agent 应用的插件树)
```

注释里的设计要点值得读原文:行顺序**没有装载语义**(装载由服务可用性驱动),分组只是为了读者;模式相关的值不放在这一层,避免任何一行被多层夹着改。

### 5.5 热更新(HMR)

effect 回收 + 依赖驱动装载,让"卸载再装载"成为安全的原子操作,`@deepseek-ai/dsh-hmr` 插件监听文件变化并执行它;编辑 `cordis.yml` 本身也会被捕捉,loader 按 `id` 差异化装载/卸载/重配。**每个条目都应写显式 `id`**:没有 `id` 的条目每次读取都会得到新生成的 id,于是任何一次配置文件编辑都会把它当成"删除 + 新增"而重载。

---

## 6. 源码走读:三个关键函数

下面挑三段 vendored 源码,把前文的机制落到实处。

### 6.1 `EventsService.waterfall` —— 环绕中间件如何合成

已在 4.4 节逐行讲过,这里补一个容易忽略的细节:所有模式共享的入口 `_resolve` 做了三件事(`vendor/cordis/src/events.ts`):

1. **识别 `thisArg`**:第一个参数若是对象/函数,则弹出作为监听器的 `this`;
2. **发诊断事件**:非 `internal/` 事件派发前,先 `emit('internal/dispatch', mode, name, args, thisArg)`——事件总线自身的可观测性也是一个普通事件;
3. **作用域过滤**:按 `thisArg[Context.filter]` 过滤监听器(`hook.global` 的监听器豁免)——dsh 的 agent 作用域工具事件(如"agent 专属监听器只收到该 agent 的调用")就建立在这个钩子上。

`ctx.on()` 的注册路径同样值得看:

```ts
on(name: string | symbol, listener: (...args: any) => any, options?: boolean | EventOptions) {
  if (typeof options !== 'object') options = { prepend: options }
  this.ctx.fiber.assertActive()               // 已卸载的 fiber 再注册 → INACTIVE_EFFECT
  listener = this.ctx.reflect.bind(listener)
  const result = this.bail(this.ctx, 'internal/listener', name, listener, options)
  if (result) return result                   // 拦截器可整体接管注册(internal/update 由此特判)
  const label = `ctx.on(${typeof name === 'string' ? JSON.stringify(name) : name.toString()})`
  return this.register(label, name, listener, options)
}
```

`register` 最终把"登记 + 反注销"包成一个 effect 挂到当前 fiber:

```ts
private register(label: string, name: string | symbol, callback: any, options: EventOptions): () => void {
  const method = options.prepend ? 'unshift' : 'push'
  return this.ctx.fiber.effect(() => {
    const hooks = this._hooks[name] ??= []
    hooks[method]({ ctx: this.ctx, callback, ...options })
    return () => this.unregister(name, callback)   // 卸载时反向移除
  }, label)
}
```

注意 effect 的 `label`:`ctx.on("tools/result")` 这样的字符串会进入 `fiber.getEffects()` 的诊断树,排查"这个插件到底注册了什么"时非常有用。

### 6.2 `Fiber.effect` 与 `_unload` —— 自动回收的实现

`fiber.effect()`(`vendor/cordis/src/fiber.ts`)的主干:

```ts
effect(execute: () => Effect, label = 'anonymous'): any {
  this.assertActive()                        // DISPOSED 的 fiber 直接抛 CordisError
  if (this.state === FiberState.UNLOADING) throw new CordisError('INACTIVE_EFFECT')

  const disposables: Disposable[] = []
  let disposing = false
  const dispose = () => {
    if (disposing) return disposalTask       // 幂等:重复调用返回同一个进行中的任务
    disposing = true
    let task!: void | Promise<void>
    for (const disposable of disposables.splice(0).reverse()) {   // 逆序
      if (task) {
        task = task.then(() => runDisposable(disposable))        // 串行链
      } else {
        const result = runDisposable(disposable)
        if (isObject(result) && 'then' in result) task = result as any
      }
    }
    return disposalTask = task
  }
  // ...executor 立即执行 execute(),把返回的 disposer 收进 disposables 并挂到 fiber 上
}
```

三个设计点:

- **幂等**:disposer 只执行一次,后来者拿到同一个 in-flight promise;多个调用方(显式调用、owner 卸载、setup 失败回滚)可以安全地竞争同一次清理;
- **逆序串行**:单个 effect 内部的多个 disposer 按注册逆序串行执行;
- **setup 失败即回滚**:`execute()` 本身抛错时,已收集的 disposer 立即反向执行——不存在"半个 effect"。

插件真正卸载走 `_unload()`:

```ts
private async _unload() {
  // 先等:本 fiber 提供的服务,其消费方( dependents )都卸载完才继续,
  //       因为消费方在自己的 teardown 期间可能还要访问这些服务
  while ([...this.provisions].some(impl => impl.dependents.size)) {
    await new Promise<void>((resolve) => this._release = resolve)
  }
  this._release = undefined
  // 再回收:本 fiber 的所有 disposer 并发执行,单个失败只记日志、不阻断其他
  await Promise.all(this._disposables.clear().map(async (dispose) => {
    try {
      /* ... */
      await runDisposable(dispose)
    } catch (reason) {
      this.ctx.logger.error(reason)
    }
  }))
  // 最后:解除本 fiber 对所依赖实现的占用,唤醒等待它们的提供者
  for (const impl of Object.values(this.committed ?? {})) {
    if (!impl.dependents.delete(this) || impl.dependents.size) continue
    impl.release?.()
    impl.ctx.fiber._release?.()
  }
  /* 状态收尾:epoch 仍活跃则转 LOADING 重载,否则停回 PENDING */
}
```

对照 2.4 节的顺序语义,这段代码就是出处:**effect 之间并发(`Promise.all`)、effect 内部逆序串行、单个 disposer 失败被日志隔离**。而"先等消费方退场,再撤自己的服务"这两段式等待,保证了依赖图上相邻的插件在卸载期间看到的始终是仍可用的服务。

### 6.3 epoch 机制 —— 依赖追踪如何驱动装载/卸载

Fiber 用一个字符串 epoch 表示"当前依赖组合"(`vendor/cordis/src/fiber.ts`):

```ts
_refresh() {
  let epoch = `${this._revision}`
  for (const name of Object.keys(this.inject)) {
    const impl = this._store[name]
    if (!impl) {
      epoch = INACTIVE          // 任一依赖缺失 → 整体不活跃
      break
    }
    epoch += ':' + impl.uid     // 依赖齐备 → epoch 由各实现的 uid 拼成
  }
  this._setEpoch(epoch)
}

private _setEpoch(epoch: string) {
  const oldEpoch = this._runner.epoch
  if (epoch === oldEpoch) return          // 依赖组合没变 → 什么都不做
  if (this._error) return                 // FAILED 的 fiber 只能由 update() 复活
  this._runner.epoch = epoch
  if (this.inertia) return                // 已有进行中的迁移,让它自己接力
  this._updateState(() => {
    if (epoch !== INACTIVE && oldEpoch === INACTIVE) {
      this.inertia = this._reload()       // INACTIVE → 活跃:装载
      return FiberState.LOADING
    } else {
      this.inertia = this._unload()       // 活跃 → INACTIVE:卸载
      return FiberState.UNLOADING
    }
  })
}
```

读懂它,很多行为就"显然"了:

- **epoch 是指纹**:它把"依赖了哪些服务的哪些具体实现"编码成字符串。依赖集合不变 → epoch 不变 → 什么都不发生;任何一个服务的实现换了 uid → epoch 变化 → 自动 `_unload` + `_reload`,插件针对新实现重新运行;
- **缺少依赖 ≠ 错误**:`epoch = INACTIVE` 让 fiber 回到 PENDING 而不是 FAILED——提供者可能稍后才装载;
- **FAILED 需要显式恢复**:只有 `fiber.update()`(清除 `_error`)能复活失败 fiber,避免"半死不活地自动重启"。

`_reload()` 里还有一处防御:装载完成后比对 `this._runner.epoch === oldEpoch`,若期间 epoch 又变了(依赖又动了),就放弃本次结果交给 `_unload` 处理——快速连续的依赖抖动最终总会收敛到最新状态。

---

## 7. 在 dsh 中的落地

现在把镜头拉远。dsh 没有"特权核心":**模型适配器、工具注册表、会话日志、agent 循环,统统是 Cordis 插件**,彼此用服务名与事件相连。对照 `packages/bundle/base/cordis.patch.yml`(第 5.4 节节选过)可以逐行验证:

| dsh 的部分 | 对应插件(条目 id → 包名) | 提供的服务 / 关键事件 |
|------------|---------------------------|----------------------|
| 模型适配层 | `llm` → `@deepseek-ai/dsh-llm`;`llm-deepseek` → `@deepseek-ai/dsh-llm-deepseek-api-key` | `ctx.llm`;路由按 settings 每请求解析密钥 |
| 工具注册表 | `tools` → `@deepseek-ai/dsh-tools`;各 `tool-*` 插件注册具体工具 | `ctx.tools`;`tools/pre-execute`/`tools/execute`/`tools/post-execute`/`tools/result` |
| 会话与日志 | `session` → `@deepseek-ai/dsh-session`;`session-persistence-jsonl` → 持久化后端 | `ctx.sessions`;追加式 JSONL 会话日志 |
| agent 循环 | `agent-loop` → `@deepseek-ai/dsh-agent-loop` | 装载启动时创建的 agents;`agent/pre-step` 等 waterfall 暴露每一步的拦截点 |
| 审批/沙箱 | `approval`、`permission`、`sandbox-policy` | 审批 waterfall、权限预设、文件效果边界 |
| 压缩/子代理/计划模式 | `compaction-basic`、`subagent`、`plan-mode` | 各自的服务与事件,同机制接入 |

这个列表本身就是一份"用配置写成的应用"。它的可读含义是:

- **没有特权核心**。agent 循环不比一个工具插件更"内置"——它同样从 `cordis.yml` 装载、同样 `inject` 依赖、同样被卸载回收。框架(`vendor/cordis`)只提供机制,不提供策略。
- **扩展 = 在旁边挂一个插件**。想给所有工具调用加审计?写一个 `ctx.on('tools/result', ...)` 的监听插件,`--patch` 挂上去即可,不需要改 agent 循环。想替换 shell 执行器?换掉 `shell` 服务的提供者条目,所有 `inject: ['shell']` 的插件自动对新实现重启(2.5 节的 epoch 机制保证这一点)。
- **组合即部署**。同样的插件树,换个 overlay 就是另一个产品形态:headless CLI、Web、桌面端共享 base 层,各自只 patch 差异行。

动手路径:照着 `docs/cordis-tutorial/` 七章做一遍(全部免 API key),第 7 章"Into the harness"会带你把一个真实工具注册进 `ctx.tools` 并观察 `tools/result` 事件——那就是本章概念的最小完整闭环。

---

## 8. 常见误区与陷阱清单

按"踩到的概率"排序:

1. **Waterfall 监听器忘记调用 `next()`**。只观察的监听器(日志、指标)不调用 `next()`,会短路它之后的所有监听器和默认行为,而且**不报任何错**。规则:观察到的东西是"路过",必须 `await next()` 并(视需要)返回其结果;不 `next()` 只属于"我拥有这个决策"的否决型监听器。
2. **插件静默卡在 PENDING**。`inject` 的服务没人提供时,插件永远等待、不打印、不报错(PENDING 是合法状态)。诊断:遍历 `ctx.registry` 检查 fiber 状态(`docs/cordis-tutorial/06-composition-and-hmr.md` 有现成的 `diagnose.ts`)。另一个相关坑:PENDING 的 fiber 不维持 Node 事件循环,整棵树没别的东西在跑时进程会静默退出。
3. **依赖 `cordis.yml` 的书写顺序**。条目并发启动,顺序不保证;依赖顺序只能用 `inject` 表达。
4. **以为卸载是按注册顺序的**。不同 effect 的异步 disposer **并发**执行;需要先后关系的清理要放进同一个 disposer 里顺序 `await`。
5. **在卸载后注册**。对 DISPOSED/UNLOADING 的 fiber 调用 `ctx.on()`/`ctx.effect()` 抛 `CordisError('INACTIVE_EFFECT')`;异步回调里注册前先确认插件还活着。
6. **条目不写 `id`**。任何一次 `cordis.yml` 编辑都会让无 `id` 的条目被当成"删除 + 新增"而重载——哪怕它的行一个字没改。
7. **新增条目"毫无反应"**。模块标识拼错(路径/包名打错)的失败走日志服务上报,而启动早期日志导出器可能还没就绪,报告会丢失。先检查拼写。
8. **`Config` 导出普通对象**。Cordis 要的是 Standard Schema 校验器(如 Schemastery),纯 TS interface 没有运行时校验,插件不会半配置启动——它会拒绝启动。
9. **把可选依赖写进 `inject`**。`inject` 是硬依赖,缺了就 PENDING。可有可无的能力用 `ctx.get('name')` 在使用点探测。
10. **在 `config`/`disabled` 之外使用 `!!js`**。其余条目元数据保持字面量,表达式在那里就是普通字符串数据;按环境选插件请用 overlay。
11. **破坏服务名的扁平命名空间**。`ctx.greeter` 这种朴素名字要留给框架/产品;自己的服务加独特前缀,否则迟早撞名(同名校验是响亮失败,但撞上 dsh 的保留名就是行为冲突)。
12. **对 `ctx` 做解构缓存**。`ctx` 是 Proxy,服务按需解析、随生命周期出入;把 `ctx.tools` 解构出来长期持有,可能在依赖重启后拿到失效引用。要长期引用,就声明 `inject` 并接受"依赖消失时整个插件被回收重载"的语义。

---

## 小结

- Cordis 的三个动词:**挂载**(插件向 Context 贡献服务与监听)、**通信**(类型化事件,五种派发模式)、**回收**(一切注册皆 effect,卸载自动反向撤销)。
- 生命周期由**依赖**驱动:`inject` 决定何时装载;epoch 指纹决定依赖变化时如何重载。
- Waterfall 是 dsh 一切拦截点的载体,**监听器必须调用 `next()` 委托**,这是全仓库最重要的单条纪律。
- dsh 没有特权核心:理解了"服务名 + 事件 + effect",`packages/bundle/base/cordis.patch.yml` 就是可读的应用定义,任何新能力都是"再挂一个插件"。

下一章(03-reading-guide.md)将给出整个仓库的阅读路线,带你把这套机制映射到具体的 dsh 包上。
