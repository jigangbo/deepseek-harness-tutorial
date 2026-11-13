# `schema.ts` — 统一 schema DSL、类型推导与 `defineTool`

`packages/core/tools/src/schema.ts` 是工具作者的日常入口：一套覆盖全部无损 JSON 值的**作者级 schema DSL**（`ValueSchemaSpec`），把它编译成受约束的原始 JSON Schema（`json-schema.ts`），一套**编译期类型推导**（`InferValue`/`InferArgs`），以及把三者粘起来的 `defineTool()` —— 参数校验、args 类型推导、body 返回类型由 output schema 决定，全部自动完成。它还提供 `projectToolSchema()`，即注册表把定义投影成模型可见 `ToolSchema` 的白名单函数。

## 文件概览

| 成员 | 职责 |
|---|---|
| `ValueSchemaSpec` 及 9 个变体 | 作者级 schema 词汇（string/number/integer/boolean/null/array/object/json/oneOf） |
| `ParameterSchemaSpec` / `ParameterPropertySpec` | 隐式开放对象根的参数表，per-property `required: true` |
| `InferValue` / `InferArgs` | schema → TypeScript 类型的编译期推导（16 层容器上限） |
| `valueSchemaSpecToJsonSchema()` / `parameterSchemaSpecToJsonSchema()` | DSL → 原始 JSON Schema 编译（迭代、防循环、防 `__proto__`） |
| `ToolArgsError` / `validateArgs()` | 参数校验失败的结构化错误 |
| `DefineToolOptions` / `defineTool()` | 第一方工具定义：校验包装 + 类型窄化 + 展示软校验 |
| `projectToolSchema()` | 定义 → 模型可见 schema 的白名单投影 |

## 关键代码走读

### 1. 作者级 DSL：9 种节点

```ts
/** String value schema with type-correct literal constraints. */
export interface StringValueSchemaSpec extends ValueSchemaAnnotations {
  type: 'string'
  enum?: readonly string[]
  const?: string
}

/**
 * Explicit object value schema. Openness is mandatory so a nested or output
 * object never acquires an accidental JSON Schema default.
 */
export interface ObjectValueSchemaSpec extends ValueSchemaAnnotations {
  type: 'object'
  properties?: ParameterSchemaSpec
  additionalProperties: boolean
}

/** Author-only unconstrained lossless JSON node. */
export interface JsonValueSchemaSpec extends ValueSchemaAnnotations {
  type: 'json'
}

/** Exact-one union schema; at least two branches are required. */
export interface OneOfValueSchemaSpec extends ValueSchemaAnnotations {
  oneOf: readonly [ValueSchemaSpec, ValueSchemaSpec, ...ValueSchemaSpec[]]
}

/** One author-facing schema for any lossless JSON value root. */
export type ValueSchemaSpec =
  | StringValueSchemaSpec | NumberValueSchemaSpec | IntegerValueSchemaSpec
  | BooleanValueSchemaSpec | NullValueSchemaSpec | ArrayValueSchemaSpec
  | ObjectValueSchemaSpec | JsonValueSchemaSpec | OneOfValueSchemaSpec
```

设计取舍：`enum`/`const` 的字面量类型与节点类型**强绑定**（string 节点只能 `enum: readonly string[]`），错误在编译期暴露；显式对象节点**强制声明** `additionalProperties`，嵌套对象永远不会意外继承 JSON Schema 的"开放"默认；`type: 'json'` 是**作者专属**节点 —— 编译成只有注解、无任何约束的原始 schema（"任意无损 JSON"）；`oneOf` 用元组类型要求至少两个分支。

### 2. 参数表：隐式开放对象根

```ts
/** One implicit parameter-root property, optionally required. */
export type ParameterPropertySpec = ValueSchemaSpec & { required?: true }

/**
 * Tool parameter schema. The map itself is an implicit open object root;
 * requiredness remains a per-property `required: true` annotation.
 */
export type ParameterSchemaSpec = {
  [key: string]: ParameterPropertySpec
  [key: symbol]: never
}
```

参数表不是显式 object 节点，而是"属性映射即对象根"：每个属性自带 `required: true` 标记，没标记的就是可选。这就是 `defineTool` 里写 `parameters: { path: { type: 'string', required: true, ... }, offset: { type: 'number' } }` 的形态来源。symbol 键被显式禁止（运行时编译也会拒绝）。

### 3. 编译期推导：`InferValue` 的 16 层上限

```ts
/** Add one schema-container level to bounded compile-time inference. */
type NextInferenceDepth<Depth extends unknown[]> = [unknown, ...Depth]

/** Infer one node without recursively checking it against the full author union. */
type InferValueAt<S, Depth extends unknown[]> =
  Depth['length'] extends 16 ? JsonValue :
    S extends { type: 'string' } ? InferScalar<S, string> :
      S extends { type: 'number' | 'integer' } ? InferScalar<S, number> :
        S extends { type: 'boolean' } ? InferScalar<S, boolean> :
          S extends { type: 'null' } ? null :
            S extends { type: 'array' }
              ? S extends { items: infer I } ? InferValueAt<I, NextInferenceDepth<Depth>>[] : JsonValue[]
              : S extends { type: 'object'; additionalProperties: boolean }
                ? InferObject<S, NextInferenceDepth<Depth>>
                : S extends { type: 'json' } ? JsonValue :
                  S extends { oneOf: readonly unknown[] }
                    ? InferValueAt<S['oneOf'][number], NextInferenceDepth<Depth>>
                    : never

/** Infer the TypeScript value accepted by an author-facing value schema. Exact
 * inference is bounded to 16 container levels, then falls back to `JsonValue`. */
export type InferValue<S> = InferValueAt<S, []>

/** Infer the TypeScript argument object for an implicit parameter schema. */
export type InferArgs<S> = InferProperties<S, []>
```

三个细节：

- **深度计数**用元组长度实现（`[unknown, ...Depth]`），到 16 层就放宽为 `JsonValue` —— 防止 TypeScript 类型实例化栈耗尽。**运行时校验永远走完整 schema**，放宽只影响编辑器里类型的具体程度。
- `InferScalar` 优先字面量：`const` > `enum` > 基础类型，所以 `{ type: 'string', enum: ['a','b'] }` 推导出 `'a' | 'b'`。
- `InferObject` 按声明的开放性决定：`additionalProperties: true` 会加 `& Record<string, JsonValue>`，`false` 则严格只认声明的属性。`InferArgs` 把 per-property 的 `required: true` 翻译成必选键、其余翻译成可选键。

### 4. 编译器：迭代任务图，不用递归

```ts
/** Install a compiled node without giving `__proto__` assignment semantics. */
function assignCompiledNode(destination: NodeDestination, node: JsonSchemaNode): void {
  switch (destination.kind) {
    case 'root': destination.holder.value = node; break
    case 'property':
      Object.defineProperty(destination.target, destination.key, {
        value: node, enumerable: true, configurable: true, writable: true,
      })
      break
    ...
  }
}

/** Execute an author-schema compilation task graph without recursive descent. */
function runSchemaCompiler(initial: CompileTask): void {
  const seen = new Set<object>()
  const tasks: CompileTask[] = [initial]
  for (let task = tasks.pop(); task !== undefined; task = tasks.pop()) {
    ...
    if (seen.has(input)) authorError(`${path} is circular`)
    seen.add(input)
    ...
  }
}
```

两个防御点：属性安装用 `Object.defineProperty` 而不是赋值 —— 用户把属性名叫 `__proto__` 时不会触发原型赋值语义；编译用显式任务栈而不是递归调用 —— 深层 schema 不会栈溢出，`seen` 集合拒绝循环引用。每个节点的 `assertAuthorKeys` 检查作者没写 DSL 词表之外的键（比如错把原始 schema 的 `minLength` 写进来会被点名拒绝，而不是被静默丢弃导致约束失效）。

### 5. 两个编译出口

```ts
export function valueSchemaSpecToJsonSchema(spec: ValueSchemaSpec): JsonSchemaNode {
  const schema = compileValueSchema(spec, 'schema')
  assertSupportedJsonSchema(schema)
  return schema
}

export function parameterSchemaSpecToJsonSchema(spec: ParameterSchemaSpec): ParameterJsonSchema {
  const compiled = compilePropertyMap(spec, 'parameters')
  const schema: ParameterJsonSchema = {
    type: 'object',
    properties: compiled.properties,
    ...(compiled.required === undefined ? {} : { required: compiled.required }),
  }
  assertSupportedJsonSchema(schema)
  return schema
}
```

编译后再过一遍 `assertSupportedJsonSchema`（双保险：编译器产出的就是子集，但断言边界留在原始 schema 入口处）。参数根是 `type: 'object'` 且**不写 `additionalProperties`** —— 按原始子集的默认保持开放（模型可以多给键，校验只检查声明过的）。

### 6. `defineTool()` — 参数校验与 args 类型推导的落点

```ts
export function defineTool<const S extends ParameterSchemaSpec, const O extends ValueSchemaSpec>(
  options: DefineToolOptions<S, O>,
): ToolDefinition {
  const userExecute = options.execute
  const userRender = options.output.render
  const userPresentCall = options.presentCall
  const userPresentResult = options.presentResult
  const userIsConcurrencySafe = options.isConcurrencySafe
  if (options.timeoutMs !== undefined && (!Number.isFinite(options.timeoutMs) || options.timeoutMs <= 0)) {
    throw new Error(`defineTool(${options.name}): timeoutMs must be a positive finite number`)
  }
  const parameters = parameterSchemaSpecToJsonSchema(options.parameters)
  const outputSchema = valueSchemaSpecToJsonSchema(options.output.schema)
  const validate = (args: unknown): string[] => validateJsonSchemaValue(parameters, args, '')
  const tool: ToolDefinition = {
    name: options.name,
    description: options.description,
    parameters: parameters as unknown as Record<string, unknown>,
    output: { schema: outputSchema, render(args: unknown, value) { return userRender(args as InferArgs<S>, value as InferValue<NoInfer<O>>) }, ... },
    ...(options.deferLoading === true ? { deferLoading: options.deferLoading } : {}),
    ...(options.timeoutMs !== undefined ? { timeoutMs: options.timeoutMs } : {}),
    async execute(args: unknown, exec: ToolRunContext): Promise<JsonValue> {
      const violations = validate(args)
      if (violations.length > 0) throw new ToolArgsError(violations)
      return userExecute(args as InferArgs<S>, exec) as Promise<JsonValue>
    },
  }
```

这就是"工具定义接口（name/description/schema/handler）"到"带校验的 `ToolDefinition`"的翻译过程：

- 两个泛型参数都带 `const` 修饰符，字面量（如 enum 数组）能被精确捕获；
- `DefineToolOptions` 里所有消费 value 的位置用 `InferValue<NoInfer<O>>` —— 先让 `output.schema` 定型 O，再推导值类型，避免回调签名反向影响 schema 推导；
- 编译期把 DSL 编译成原始 schema，**运行期**在 `execute` 外面套校验壳：违规抛 `ToolArgsError`（`INVALID_ARGS`，路径限定的 violations 拼进消息），通过后把 `unknown` 断言成 `InferArgs<S>` 交给用户函数 —— 用户拿到的 args 类型安全，因为校验刚通过；
- `timeoutMs` 在 defineTool 和注册表两处都检查（defineTool 报错更早、带工具名）；
- `deferLoading: true` 原样保留到定义上（请求延迟加载，是否真的延迟由 LLM 包决定）。

### 7. 显示路径软校验：畸形参数回退 undefined，不抛错

```ts
  // Presentation is display-only and may run on REPLAY of arbitrary logged args
  // (possibly from an older schema), so it must never throw: validate softly and
  // fall back to `undefined` (a generic UI presentation) on any mismatch, rather
  // than the hard `ToolArgsError` the execute path raises.
  if (userPresentCall) {
    tool.presentCall = (args: unknown): ToolCallView | undefined => {
      if (validate(args).length > 0) return undefined
      return userPresentCall(args as InferArgs<S>)
    }
  }
  if (userPresentResult) {
    tool.presentResult = (args: unknown, result: ToolResult): ToolResultView | undefined => {
      if (validate(args).length > 0) return undefined
      return userPresentResult(args as InferArgs<S>, result)
    }
  }
  if (userIsConcurrencySafe) {
    tool.isConcurrencySafe = (args: unknown): boolean => {
      if (validate(args).length > 0) return false
      return userIsConcurrencySafe(args as InferArgs<S>)
    }
  }
```

三个包装各自对应一条软/硬策略，全部源于**同一份预编译的 parameters schema**：

| 路径 | 校验失败时 | 为什么 |
|---|---|---|
| `execute` | 抛 `ToolArgsError`（硬） | 执行必须拒绝非法输入，且错误要告诉模型哪里错 |
| `presentCall` / `presentResult` | 返回 `undefined`（软）→ UI 回退 generic 卡片 | 展示可能在**日志重放**时跑在任意历史 args 上（可能是旧 schema 的产物），抛错会炸掉整个会话重放 |
| `isConcurrencySafe` | 返回 `false`（软）→ 独占执行 | fail-closed：拿不准就别并行 |

### 8. `projectToolSchema()` — 白名单投影

```ts
export function projectToolSchema(definition: ToolDefinition, detachParameters: boolean): ToolSchema {
  const { name, description, parameters, deferLoading } = definition
  const detached = detachParameters ? snapshotJsonValue(parameters) : parameters
  if (detached === undefined) throw new Error(`tool "${name}" parameters must be lossless JSON before schema projection`)
  return { name, description, parameters: detached, ...deferLoading === true ? { deferLoading } : {} }
}
```

模型可见字段只有四个：`name`/`description`/`parameters`/`deferLoading`。`output`、`execute`、`projectContent`、`finalizeContent`、`timeoutMs`、`isConcurrencySafe`、`presentCall`、`presentResult` **按结构永远不会漏上 wire** —— 投影是显式解构而非展开。`detachParameters: true` 时深拷贝参数 schema，调用方可以安全持有/改写这份投影而不影响注册表里的定义。

### 9. `validateArgs()` 与 `ToolArgsError`

```ts
/** Invalid model-generated arguments for a typed tool. */
export class ToolArgsError extends HarnessError {
  /** Individual violations in schema-walk order. */
  readonly violations: string[]

  constructor(violations: string[]) {
    super(`invalid arguments: ${violations.join('; ')}`, 'INVALID_ARGS')
    this.name = 'ToolArgsError'
    this.violations = violations
  }
}

export function validateArgs(spec: ParameterSchemaSpec, args: unknown): string[] {
  return validateJsonSchemaValue(parameterSchemaSpecToJsonSchema(spec), args, '')
}
```

注意 `validateArgs` 每次调用都重新编译 schema —— 它是给外部一次性校验用的；`defineTool` 内部只编译一次并把编译产物闭包给 `validate`。

## 核心类型与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ValueSchemaSpec` | 9 节点联合 | 作者级 schema 词汇，标量字面量约束类型安全 |
| `ParameterSchemaSpec` | `{ [key]: Spec & { required?: true } }` | 隐式开放对象根的参数表 |
| `InferValue<S>` | 条件类型，16 层上限 | schema → 值类型，超深放宽为 `JsonValue` |
| `InferArgs<S>` | `InferProperties<S, []>` | 参数表 → 必选/可选键对象 |
| `valueSchemaSpecToJsonSchema()` | `(spec) => JsonSchemaNode` | 输出 DSL 编译 + 子集断言 |
| `parameterSchemaSpecToJsonSchema()` | `(spec) => ParameterJsonSchema` | 参数表编译成开放对象根 |
| `runSchemaCompiler()`（内部） | 迭代任务图 | 防循环、防原型注入、防栈溢出的编译 |
| `ToolArgsError` | `HarnessError('INVALID_ARGS')` | 参数校验失败，携带按序 violations |
| `validateArgs()` | `(spec, args) => string[]` | 一次性参数校验（每次重编译） |
| `DefineToolOptions<S, O>` | interface | defineTool 输入：定义 + 推导回调 |
| `defineTool()` | `<const S, const O>(options) => ToolDefinition` | 校验壳包装 + 类型窄化 + 展示软校验 |
| `projectToolSchema()` | `(definition, detachParameters) => ToolSchema` | 白名单投影，回调永不入 wire |

## 与其他模块的关系

- `json-schema.ts`：编译产物交给它断言（`assertSupportedJsonSchema`）、值交给它校验（`validateJsonSchemaValue`）；`JsonSchemaError` 是作者错误与原始子集错误的共同类型。
- `contracts.ts`：`defineTool` 产出 `ToolDefinition`；`ToolRunContext`/`ToolExecution`/`ToolResult` 是各回调的参数类型。
- `presentation.ts`：`presentCall`/`presentResult` 的返回类型 `ToolCallView`/`ToolResultView`；本文件的软校验保证展示回调只在合法 args 上被调。
- `index.ts`：注册时再次 `assertSupportedJsonSchema(output.schema)` 并校验 `timeoutMs`；`schemas()` 调 `projectToolSchema(definition, true)`。
- `pipeline.ts`：`createSuccessResult` 用**编译后的原始 schema**（`tool.output.schema`）校验正文值 —— 与参数校验共享同一套底层校验器。

## 小结

`schema.ts` 用一个 DSL 覆盖"参数声明、输出声明、类型推导、运行校验"四件事：作者写一次 `ValueSchemaSpec`，编译期得到精确到字面量的 TS 类型（16 层深度内）、运行期得到子集断言过的 JSON Schema 和全路径校验。`defineTool` 把校验壳做成分层策略 —— 执行硬校验抛 `ToolArgsError`、展示软校验回退 `undefined`、并发分类 fail-closed 返回 `false` —— 让"畸形参数"在每条路径上都有安全的默认行为。而 `projectToolSchema` 的显式白名单从结构上杜绝了执行回调与调度元数据泄漏到模型请求里。
