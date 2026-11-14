# `json-schema.ts`

> 受支持 JSON Schema 子集的断言与值校验


`packages/core/tools/src/json-schema.ts` 是整个工具体系的校验底座：它定义**wire 级**的原始 JSON Schema 节点（`JsonSchemaNode`）—— 与子代理、工作流、MCP、PTC 共享的受约束子集 —— 并提供两个方向的能力：**schema 侧**（`assertSupportedJsonSchema` 拒绝子集之外的任何东西）和**值侧**（`validateJsonSchemaValue` 对任意候选值返回路径限定的 violations）。[schema.md](schema.md) 的作者 DSL 编译产物最终都落到这里校验。

## 文件概览

| 成员 | 职责 |
|---|---|
| `JsonSchemaNode` / `JsonSchemaType` / `JsonSchemaScalar` / `ObjectJsonSchema` | 子集的 wire 级类型 |
| `JsonSchemaError` | schema 越界的结构化错误（`UNSUPPORTED_SCHEMA`，收集全部违规路径） |
| `assertSupportedJsonSchema()` / `assertObjectJsonSchema()` | schema 断言：子集 / 子集+对象根 |
| `validateJsonSchemaValue()` | 值校验：显式帧迭代，对任意值全容错 |
| `isPlainJsonRecord` / `isPlainJsonArray` / `isJsonSchemaRecord` | realm 无关的普通对象/稠密数组判定 |

子集本身只有：单一 `type`、对象 `properties`/`required`/布尔 `additionalProperties`、数组 `items`、类型正确的标量 `enum`/`const`、exact-one `oneOf`、四个注解键（`description`/`title`/`default`/`examples`）。**不支持的关键词拒绝而不是忽略** —— 被"接受但不强制"的约束比被拒绝更危险。

## 关键代码走读

### 1. wire 级节点类型

```ts
/** Single-type keywords accepted by the enforced subset. */
export type JsonSchemaType = 'object' | 'array' | 'string' | 'number' | 'integer' | 'boolean' | 'null'

/**
 * One raw JSON Schema node in the enforced subset. The optional fields express
 * the external wire schema; {@link assertSupportedJsonSchema} rejects invalid
 * combinations before a caller treats the node as trusted.
 */
export interface JsonSchemaNode {
  /** Omit with no constraints for any JSON value, or use `oneOf`. */
  type?: JsonSchemaType
  /** Exactly one branch must validate; at least two branches are required. */
  oneOf?: JsonSchemaNode[]
  properties?: Record<string, JsonSchemaNode>
  /** Required property names; each must appear in `properties`. */
  required?: string[]
  /** `false` rejects undeclared keys; absent/`true` follows JSON Schema's open default. */
  additionalProperties?: boolean
  /** Item schema (`type: 'array'` only); absent accepts any JSON item. */
  items?: JsonSchemaNode
  enum?: JsonSchemaScalar[]
  const?: JsonSchemaScalar
  description?: string
  title?: string
  default?: JsonValue
  examples?: JsonValue
}
```

`type` 与 `oneOf` 互斥；不写 type 也不写 oneOf 的节点是**注解节点**，含义是"任意无损 JSON"（作者 DSL 的 `type: 'json'` 编译成它）。`additionalProperties` 缺省或 `true` 都按 JSON Schema 的开放默认处理。

### 2. 结构化错误：收集全部违规路径

```ts
export class JsonSchemaError extends HarnessError {
  /** Individual schema violations in walk order. */
  readonly violations: string[]

  constructor(violations: string[]) {
    super(`unsupported JSON schema: ${violations.join('; ')}`, 'UNSUPPORTED_SCHEMA')
    this.name = 'JsonSchemaError'
    this.violations = violations
  }
}
```

错误消息把**所有**越界路径一次报全（不是遇到第一个就停）—— 作者修一轮就能修完，而不是改一个错再发现下一个。

### 3. Realm 无关的普通值判定

```ts
/* jscpd:ignore-start -- this realm boundary mirrors the session-owned lossless-JSON intrinsic test */
/** Whether a realm-owned intrinsic prototype is backed by its native constructor. */
function hasIntrinsicConstructor(prototype: object, name: 'Array' | 'Object'): boolean {
  const descriptor = Object.getOwnPropertyDescriptor(prototype, 'constructor')
  const constructor: unknown = descriptor?.value
  if (typeof constructor !== 'function') return false
  try {
    return constructor.name === name
      && constructor.prototype === prototype
      && Function.prototype.toString.call(constructor) === `function ${name}() { [native code] }`
  } catch {
    return false
  }
}

/**
 * Test for a realm-agnostic plain JSON record without accepting arrays or exotic objects.
 */
export function isPlainJsonRecord(value: unknown): value is Record<string, unknown> {
  if (typeof value !== 'object' || value === null || Array.isArray(value)) return false
  try {
    const prototype: unknown = Object.getPrototypeOf(value)
    return prototype === null
      || typeof prototype === 'object' && isIntrinsicObjectPrototype(prototype)
  } catch {
    return false
  }
}

/** Test for a dense ordinary array with no JSON-invisible decorations. */
export function isPlainJsonArray(value: unknown): value is unknown[] {
  if (!Array.isArray(value)) return false
  try {
    if (!hasPlainArrayPrototype(value) || Reflect.ownKeys(value).length !== value.length + 1) return false
    for (let index = 0; index < value.length; index++) {
      if (!Object.hasOwn(value, index)) return false
    }
    return true
  } catch {
    return false
  }
}
```

为什么这么较真？工具 schema 和值可能来自**另一个 JavaScript realm**（worker、vm、测试框架的沙箱）—— 跨 realm 的 `instanceof Array` 会失败，普通 `typeof` 检查又挡不住带私有字段的 exotic 对象。这里的判定改为检查原型链是否是**本征构造函数** backing 的（构造函数名 + 原型同一性 + native code 源码三重验证），数组还要求**稠密且无额外自有键**（`Reflect.ownKeys` 长度恰好 = length + 1）。`isJsonSchemaRecord` 在此之上再要求"只有可枚举的自有字符串键"—— symbol 键、不可枚举键都会让 schema 被拒，因为它们过不了 JSON 投影。

### 4. schema 断言：迭代走查整个树

```ts
/** Validate object-only fields after its property schemas have been visited. */
function checkObjectSchemaTail(node, path, properties, violations): void {
  const hasRequired = Object.hasOwn(node, 'required')
  ...
  if (hasRequired) {
    if (!isPlainJsonArray(required) || required.some(entry => typeof entry !== 'string')) {
      violations.push(`${path}.required must be an array of strings`)
    } else {
      const declared = isJsonSchemaRecord(properties) ? properties : {}
      for (const key of required as string[]) {
        if (!Object.hasOwn(declared, key)) violations.push(`${path}.required names "${key}" which is not in properties`)
      }
    }
  }
  if (Object.hasOwn(node, 'additionalProperties') && typeof node.additionalProperties !== 'boolean') {
    violations.push(`${path}.additionalProperties must be a boolean`)
  }
}

/** Collect every violation for one raw schema tree without using the JavaScript call stack. */
function checkSchemaNode(root: unknown, rootPath: string, violations: string[], seen: Set<object>): void {
  const tasks: SchemaWalkTask[] = [{ kind: 'enter', node: root, path: rootPath }]
  for (let task = tasks.pop(); task !== undefined; task = tasks.pop()) {
    ...
    for (const key of Object.keys(node)) {
      if (CONSTRAINT_KEYWORDS.has(key)) continue
      if (ANNOTATION_KEYWORDS.has(key)) { ...check lossless JSON...; continue }
      violations.push(`${path}.${key} is not a supported keyword (subset: type/oneOf/properties/required/additionalProperties/items/enum/const + annotations)`)
    }
    ...
    if (hasType && hasOneOf) {
      violations.push(`${path} cannot declare both type and oneOf`); continue
    }
    if (!hasType && !hasOneOf) {
      for (const key of ONE_OF_SIBLING_KEYWORDS) {
        if (Object.hasOwn(node, key)) violations.push(`${path}.${key} requires type or oneOf`)
      }
      continue
    }
```

检查器与编译器（`schema.ts`）同款套路：显式任务栈代替递归、`seen` 拒绝循环、`enter`/`leave`/`*-tail` 任务分离"访问子节点前"与"访问完后"的检查（`required` 是否都出现在 `properties` 里需要先走完 properties 才能判，所以是 object 的 tail 任务）。关键规则一览：

- **未知关键词 = 违规**（错误消息里直接列出子集词表）；
- 注解键的值必须是**无损 JSON**（getter 抛错也判违规）；
- `type` 数组（JSON Schema 的 `type: ['string','null']`）明确不支持，报错文案单独点名；
- 关键词与 type 的归属错位（如 `items` 写在 string 节点上、`properties` 写在 array 上）逐个报；
- `oneOf` 旁边不许放 `properties`/`enum` 等兄弟关键词（子集没有定义它们的组合语义）；
- 标量节点：`enum` 必须非空、每项类型匹配；`const` 类型匹配，且与 `enum` 同用时必须是 enum 的成员。

两个公开断言：

```ts
export function assertSupportedJsonSchema(schema: unknown): asserts schema is JsonSchemaNode {
  const violations: string[] = []
  checkSchemaNode(schema, 'schema', violations, new Set())
  if (violations.length > 0) throw new JsonSchemaError(violations)
}

export function assertObjectJsonSchema(schema: unknown): asserts schema is ObjectJsonSchema {
  ...
  if (violations.length === 0
    && (!isJsonSchemaRecord(schema) || !Object.hasOwn(schema, 'type') || schema.type !== 'object')) {
    violations.push('schema.type must be "object" (structured output is object-rooted)')
  }
  ...
}
```

后者是子代理/工作流**结构化输出**的消费级约束：共享词表允许任意 JSON 根，但要求对象根的消费者自己加 `assertObjectJsonSchema` —— 子集不做不必要的收紧。

### 5. 值校验：显式帧、全容错、exact-one oneOf

```ts
/** Explicit call frame for stack-safe schema-value validation. */
interface ValueFrame {
  readonly node: JsonSchemaNode
  readonly value: unknown
  readonly path: string
  catches: boolean
  phase: 'start' | 'children'
  kind?: 'oneOf' | 'object' | 'array'
  children: ValueChild[]
  childIndex: number
  violations: string[]
  tailViolations: string[]
  matches: number
}

/** Validate one trusted schema/value pair with explicit frames rather than recursive calls. */
function checkValue(schema: JsonSchemaNode, value: unknown, path: string): string[] {
  const frames: ValueFrame[] = [valueFrame(schema, value, path)]
  ...
  while (frames.length > 0) {
    const frame = frames.at(-1)
    try {
      if (frame.phase === 'children') { ...consume children / finish oneOf & container... }
      ...
      switch (nodeType) {
        case 'object': {
          if (!isPlainJsonRecord(frame.value)) { finish([`"${diagnosticPath(frame.path)}" must be an object`]); break }
          const properties = ...
          const violations: string[] = []
          const required = ...
          for (const key of required) {
            if (!Object.hasOwn(frame.value, key) || frame.value[key] === undefined) {
              violations.push(`missing required property "${propertyPath(frame.path, key)}"`)
            }
          }
          ...
          if (Object.hasOwn(frame.node, 'additionalProperties') && frame.node.additionalProperties === false) {
            for (const key of Object.keys(frame.value)) {
              if (!Object.hasOwn(properties, key)) {
                tailViolations.push(`"${propertyPath(frame.path, key)}" is not a declared property (additionalProperties: false)`)
              }
            }
          }
          ...
        }
        ...
      }
    } catch (error) {
      let failed = frames.pop()
      while (failed !== undefined && !failed.catches) failed = frames.pop()
      if (failed === undefined) throw error
      receive(losslessValueViolation(failed.path))
    }
  }
```

设计要点：

- **显式帧代替递归**：深层嵌套值不会栈溢出；`matches` 计数器实现 exact-one 语义 —— `oneOf` 的子分支全部试完后，恰好 1 个匹配才算过，否则报 `must match exactly one oneOf branch (matched N)`。
- **对象值里显式 `undefined` 的属性按缺失处理**（`value[key] === undefined` 与 `!Object.hasOwn` 同判）—— 对齐 JSON 的语义（JSON 里没有 undefined）。
- **异常包容**：值对象上带 getter 的属性读取抛错时，帧的 `catches` 标志决定向哪个方向收 —— 合法 schema 节点上的异常收进"必须是 lossless JSON 值"的诊断，而不是让校验器自己崩溃（校验器对**任意值**是全函数/total）。
- 容器收尾时还要做 `safelyIsJsonValue` 复查：对象必须是普通 record、数组必须是稠密无损数组（与 schema 断言用同一套 realm 判定）。
- 诊断路径对根友好：根路径 `''` 显示为 `arguments`（`defineTool` 的参数校验以 `''` 为根调用），嵌套用 `a.b[0].c` 风格。

### 6. 公开入口

```ts
export function validateJsonSchemaValue(schema: JsonSchemaNode, value: unknown, path = 'value'): string[] {
  return checkValue(schema, value, path)
}
```

返回值约定：**空数组 = 有效**，非空 = 按走查顺序排列的路径限定违规。`pipeline.ts` 用它校验工具正文值（`validateJsonSchemaValue(tool.output.schema, detached, 'value')`），`schema.ts` 用它校验参数（根路径 `''`）。注意调用前提：schema 必须先通过 `assertSupportedJsonSchema` —— 校验器信任 schema（同进程可信边界），只对**值**设防。

## 核心类型与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `JsonSchemaNode` | 子集节点 interface | wire 级 schema 类型；注解节点 = 任意无损 JSON |
| `ObjectJsonSchema` | `JsonSchemaNode & { type: 'object' }` | 对象根约束的结构化输出 schema |
| `JsonSchemaError` | `HarnessError('UNSUPPORTED_SCHEMA')` | 收集全部违规路径的 schema 错误 |
| `assertSupportedJsonSchema()` | `asserts schema is JsonSchemaNode` | 拒绝子集之外的 schema（未知关键词、组合错位、循环） |
| `assertObjectJsonSchema()` | `asserts schema is ObjectJsonSchema` | 子集断言 + 对象根（子代理/工作流用） |
| `validateJsonSchemaValue()` | `(schema, value, path?) => string[]` | 全容错的值校验，空数组即有效 |
| `isPlainJsonRecord()` | type guard | realm 无关的普通对象判定 |
| `isPlainJsonArray()` | type guard | 本征原型、稠密、无装饰的数组判定 |
| `isJsonSchemaRecord()` | type guard | 普通对象 + 仅可枚举自有字符串键 |

## 与其他模块的关系

- `schema.ts`：编译产物由 `assertSupportedJsonSchema` 断言；`validateArgs` 与 `defineTool` 的 `validate` 闭包都调 `validateJsonSchemaValue`。
- `pipeline.ts`：`createSuccessResult` 用它校验正文返回值与 post-execute 替换值（违反抛 `ToolOutputError`）。
- `index.ts`：注册时对 `output.schema` 断言子集。
- 消费方（子代理 `dsh-subagent`、工作流 `dsh-workflow`、MCP `dsh-mcp`、PTC SDK 生成）：共享这个子集与 `assertObjectJsonSchema` 的对象根模式。
- `dsh-util-values` 的 `isJsonValue`：注解键与容器值的 lossless JSON 复查。

## 小结

`json-schema.ts` 的立场是"**少而严**"：只支持七种单类型 + oneOf + 三个对象关键词 + 类型绑定的字面量约束，任何子集外的表达都显式拒绝（带完整路径清单），绝不出现"声明了但没人强制"的约束。实现上把两个易碎点做成了硬保证：跨 realm 的普通对象/稠密数组判定（构造函数三重验证 + ownKeys 计数），以及全容错的值校验（显式帧、getter 异常收进诊断、对任意值 total）。上层一切校验 —— 参数（`ToolArgsError`）、输出（`ToolOutputError`）、post-execute 替换值 —— 都站在这一层上。
