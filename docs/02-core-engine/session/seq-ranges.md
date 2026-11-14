# `seq-ranges.ts` — 序号区间压缩
> `sourceEventSeqs` 的无损范围编码（JSONL 存储格式）


> 源码：`packages/core/session/src/seq-ranges.ts`（73 行，模块注释："Lossless range encoding for JSONL `sourceEventSeqs` arrays."）

## 文件概览

表面事件（尤其 compaction 的 `replace`）用 `sourceEventSeqs` 完整引用它遮蔽/来源的事件 seq 列表。一个 compaction 替换遮蔽几百个连续节点时，裸数组会写成 `[100, 101, 102, …, 399]` 几百个数字。本文件把"严格递增的连续段"编码成包含式区间对 `[start, end]`，例如 `[100, [100, 399]]`——存储体积大幅缩小，语义完全无损。

导出两个方向相反的函数：

- `encodeSeqRanges`：内存中的已验证 seq 数组 → JSON 存储形式。
- `decodeSeqRanges`：解析出的存储值 → 内存 seq 数组（逐项构造 `SessionSeq` 品牌，带容量上限防御）。

使用者是**格式迁移链**：`packages/session/session-format-v0-to-v1/src/codec.ts`、`session-format-v1-to-v2/src/codec.ts` 在搬移旧日志时编解码此字段；`SessionStore` 从 `index.ts` 再导出两者。

## 关键代码走读

### 1. 存储形式与编码：只对"有利可图"的连续段配对

```typescript
/** A stored source sequence or inclusive consecutive range. */
export type EncodedSeq = number | [number, number]

function isStrictlyIncreasing(values: readonly SessionSeqType[]): boolean {
  return values.every((value, index) => index === 0 || value > (values[index - 1] as SessionSeqType))
}

/**
 * Replace profitable consecutive runs with inclusive pairs.
 * @param values - validated in-memory source sequences.
 * @returns a lossless JSON storage form.
 */
export function encodeSeqRanges(values: readonly SessionSeqType[]): EncodedSeq[] {
  if (!isStrictlyIncreasing(values)) return [...values]
  const encoded: EncodedSeq[] = []
  for (let start = 0; start < values.length;) {
    let end = start
    while (end + 1 < values.length && values[end + 1] === (values[end] as number) + 1) end += 1
    if (end - start >= 2) encoded.push([values[start] as number, values[end] as number])
    else for (let index = start; index <= end; index += 1) encoded.push(values[index] as number)
    start = end + 1
  }
  return encoded
}
```

编码规则的三层判断：

- **前置卫兵**：输入不是严格递增（`isStrictlyIncreasing` 失败）就原样复制返回——**编码永不改变语义**，不合条件的输入不投机。
- **`end - start >= 2` 才配对**：连续段至少 3 个元素（如 `[100,101,102]` → `[100,102]`，3 换 1 对）才"有利可图"；两个元素的连续段 `[100,101]` 写成对 `[100,101]` 体积相同还多了嵌套，所以保留裸数字。
- **贪心扩段**：内层 while 一直吞到不再连续为止，段间无缝拼接。

JSDoc 强调输入是"validated in-memory source sequences"——编码器不做验证，信任调用方（append 路径已在 `surface.ts` 验证过唯一性、更早性）。

### 2. 解码：验证边界，容量有上限

```typescript
export function decodeSeqRanges(value: unknown, maxEntries = Number.MAX_SAFE_INTEGER): SessionSeqType[] {
  if (!Array.isArray(value)) throw new TypeError('sourceEventSeqs must be an array')
  const decoded: SessionSeqType[] = []
  let hasRange = false
  for (const entry of value) {
    if (typeof entry === 'number') {
      assertSeq(entry)
      if (decoded.length >= maxEntries) throw new TypeError('sourceEventSeqs exceeds its event sequence')
      decoded.push(SessionSeq(entry))
      continue
    }
    if (!Array.isArray(entry) || entry.length !== 2) {
      throw new TypeError('sourceEventSeqs range entries must be [start, end] pairs')
    }
    const start: unknown = entry[0]
    const end: unknown = entry[1]
    assertSeq(start)
    assertSeq(end)
    if (end < start) throw new TypeError('sourceEventSeqs ranges require start <= end')
    const length = end - start + 1
    if (length > maxEntries - decoded.length) {
      throw new TypeError('sourceEventSeqs range exceeds its event sequence')
    }
    for (let seq = start; seq <= end; seq += 1) decoded.push(SessionSeq(seq))
    hasRange = true
  }
  if (hasRange && !isStrictlyIncreasing(decoded)) {
    throw new TypeError('sourceEventSeqs ranges must be strictly increasing')
  }
  return decoded
}
```

解码是**durable/file 边界**（持久化 JSON 可能来自旧版本写入或被篡改的存储），所以验证完备：

- **逐项类型检查**：数字走 `assertSeq`（非负 safe integer），其余必须是恰好两元素的数组对；`end < start` 拒绝。
- **`maxEntries` 容量防御**：默认 `Number.MAX_SAFE_INTEGER`，但事件所有者可以传入该事件允许的最大引用数（"largest list permitted by the owning event"）。检查在**扩段之前**（`length > maxEntries - decoded.length`），一个 `[0, 2^53]` 的区间不会先撑爆内存再报错——拒绝在分配之前发生。
- **整体有序性回检**：出现区间时（`hasRange`）最后统一验证解码结果严格递增。编码端只在递增输入上产出区间，所以**含区间的解码结果必然递增**——这条检查是对恶意构造存储（如 `[0, 5], [3, 9]` 交叠区间）的防御，而不是对合法编码的重复劳动。
- **逐项 `SessionSeq(seq)`**：解码结果直接带上品牌类型，调用方拿到的是已验证的 `SessionSeqType[]`。

### 3. 配套的 `assertSeq`

```typescript
function assertSeq(value: unknown): asserts value is number {
  if (!Number.isSafeInteger(value) || (value as number) < 0) {
    throw new TypeError('sourceEventSeqs must contain non-negative safe integers')
  }
}
```

注意它与 `SessionSeq` 构造器的差异：这里**不拒绝 `-0`**（`value < 0` 对 `-0` 为 false）。边界由上游的 `SessionSeq` 构造器兜底（它显式 `Object.is(value, -0)` 拒绝），本函数只做廉价的预检。两道验证分工：`assertSeq` 给出关于 `sourceEventSeqs` 字段的错误消息，`SessionSeq` 保证品牌不变量。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `EncodedSeq` | `type = number \| [number, number]` | 存储形式：裸 seq 或包含式区间对 |
| `encodeSeqRanges` | `(values) => EncodedSeq[]` | 连续段（≥3 元素）配对压缩；非严格递增输入原样复制 |
| `decodeSeqRanges` | `(value, maxEntries?) => SessionSeqType[]` | 存储值 → 已验证品牌 seq 数组；逐项验证 + 容量防御 + 递增回检 |
| `assertSeq` | 模块私有 asserts | 非负 safe integer 预检（`-0` 由 `SessionSeq` 构造器拒绝） |
| `isStrictlyIncreasing` | 模块私有 | 严格递增判定（编码前置卫兵与解码回检共用） |

## 与其他模块的关系

- **格式迁移链**（`packages/session/session-format-v0-to-v1`、`session-format-v1-to-v2`）：codec 在版本转换时对 `sourceEventSeqs` 编码/解码——v1 起的存储格式采用区间压缩。
- **`index.ts`**：`export { decodeSeqRanges, encodeSeqRanges } from './seq-ranges.ts'` 供持久化与快照工具使用（如 `test-support/session-snapshot`）。
- **`types.ts` 的 `SessionSeq`**：解码逐项经过品牌构造器；本文件是"构造器只接受已验证值"约定的下游受益者。
- **`surface.ts`**：内存中 `sourceEventSeqs` 的语义（完整、非空、唯一、更早、覆盖被遮蔽节点）在这里只保留"递增"一半——事件级语义校验发生在 append/restore，本文件只管存储表示。

## 小结

一个纯存储优化件：把"严格递增的连续 seq 段"无损压缩为区间对，配对阈值（≥3）保证编码永不增大体积；解码端则以 durable 边界标准对待输入——逐项验证、先算容量再分配、递增回检。它让 compaction 大范围替换在 JSONL 上的 `sourceEventSeqs` 开销从 O(遮蔽节点数) 降到 O(连续段数)。
