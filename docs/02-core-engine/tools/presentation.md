# `presentation.ts` — 工具卡片的渲染意图词汇

`packages/core/tools/src/presentation.ts` 是纯粹的类型文件：没有一行运行时逻辑，全部内容是工具"希望自己的调用在 UI 里长什么样"的**渲染意图**（render intent）词汇。工具通过 `presentCall`/`presentResult` 返回这些 `card` 标签的判别联合，UI 桥（Host presenter、Web client）switch 在 `card` 上渲染 —— **工具描述自己，但不依赖任何客户端协议**。它与执行完全解耦：两个回调都是纯函数，可能在实时流式期间被调，也可能在**会话日志重放**时被调（此时只有 args 和落账的 `content/isError/meta` 可用）。

## 文件概览

| 类型组 | 类型 | 问答 |
|---|---|---|
| 共享词汇 | `ToolCallKind`、`FileLocation`、`FileDiff`、`ReadFileLine` | 卡片的图标分类与文件卡片素材 |
| pending 联合 | `ToolCallView` = generic / terminal / diff | "调用**正在**执行"时显示什么 |
| completed 联合 | `ToolResultView` = generic / terminal / diff / search / read / web | "调用**完成**后"显示什么 |
| web 词汇 | `WebSource`、`WebSearchResultView`、`WebFetchResultView` | 网络检索的结构化来源卡片 |

## 关键代码走读

### 1. 共享词汇：图标与文件卡片素材

```ts
export type ToolCallKind = 'read' | 'edit' | 'delete' | 'move' | 'search' | 'execute' | 'fetch' | 'other'

export interface FileLocation {
  /** The file the tool operated on (the model-facing path); `line` is an optional 1-based line to focus. */
  path: string
  line?: number
}

export interface FileDiff {
  path: string
  /** Prior content, or `null` for a new file / an overwrite (no prior content available at call time). */
  oldText: string | null
  /** Content after the change. */
  newText: string
}

/** One numbered line of a file, the unit a {@link ReadResultView} carries... */
export interface ReadFileLine {
  number: number
  text: string
}
```

`ToolCallKind` 是 generic 卡片上的图标选择（`other` 缺省）。`FileLocation` 服务编辑器"follow-along"：工具在跑时高亮/跳到被读写的文件与行。`FileDiff` 的 `oldText: null` 有精确语义 —— **新文件**没有前像可以 diff；**call 时的覆写也一样**，因为 pending presenter 拿不到文件的旧内容（只有执行完成后才能算出真正的 hunk）。

### 2. pending 联合：`ToolCallView`

```ts
export type ToolCallView = GenericCallView | TerminalCallView | DiffCallView

export interface GenericCallView {
  card: 'generic'
  /** Human-readable, always-visible label describing what THIS call does. */
  title: string
  /** Category for icon/treatment; defaults to `other` when omitted. */
  kind?: ToolCallKind
  /** The salient input to show in a detail/expanded view (e.g. a background job id). */
  rawInput?: unknown
  content?: ContentBlock[]
  /** Files this call reads/modifies, for editor follow-along. */
  locations?: FileLocation[]
}

export interface TerminalCallView {
  card: 'terminal'
  /** The command, shown as the terminal card's title / header line. */
  title: string
  description?: string
  /**
   * Working directory the command runs in... An ABSOLUTE path is used as-is; a RELATIVE
   * path is resolved by the UI bridge against the session workspace (the pure presenter
   * can't see the session cwd). Omit entirely to let the bridge use the session workspace.
   */
  cwd?: string
}

export interface DiffCallView {
  card: 'diff'
  /** Card header (e.g. `Write foo.txt`). */
  title: string
  /** One entry per file the call changes. */
  diffs: FileDiff[]
  locations?: FileLocation[]
}
```

三种 pending 卡片覆盖三类调用：**generic** 是默认（shell/文件之外的几乎所有工具；`rawInput` 提示"只放最醒目的一个输入，不是整个 args 对象"）；**terminal** 是"这个调用就是一条命令"（`bash`）—— cwd 的相对路径由 UI 桥解析，因为纯 presenter 看不见会话工作目录；**diff** 是"这个调用要写文件"（`write`/`edit`）—— call 时基于**参数**画预览 diff。返回 `undefined`（或不声明 `presentCall`）回退 generic 卡片：标题=工具名、输入=原始 args。

### 3. completed 联合：`ToolResultView` 的六个臂

```ts
export type ToolResultView = GenericResultView | TerminalResultView | DiffResultView | SearchResultView | ReadResultView | WebResultView
```

各臂的字段契约（省略所有臂都有的可选 `title?` —— 换标题用，缺省沿用 pending 标题）：

- **`GenericResultView`**（`card: 'generic'`）：可选 `content?` 重排模型可见内容；什么都不给就原样渲染 raw content。
- **`TerminalResultView`**（`card: 'terminal'`）：`output?` 捕获输出、`exitCode?` 与 `signal?` 互斥（退出码 vs 被信号杀死）。UI 桥负责从 `output` 推导 ```console 围栏兜底 —— 工具不做双重编码。
- **`DiffResultView`**（`card: 'diff'`）：完成后的文件变更。注释里写了为什么"重复"是设计而非冗余：completed 内容会**替换** pending 卡片内容，变异工具必须再返回 diff，否则 raw 结果文本会顶掉 diff 展示；hunk 从前后内容计算，没有前像（create）就整文件 diff。
- **`SearchResultView`**（`card: 'search'`，`shape` 判别）：两个变体 —— `shape: 'matches'`（grep：按文件分组的匹配行）与 `shape: 'paths'`（glob：扁平路径列表）。都用 `truncated` + `total` 报告内联结果是否被截断，**UI 永远不把部分结果呈现成完整结果**。注意判别符是 `shape` 而不是 `kind` —— `kind` 已被 `ToolCallKind` 占用（图标桥会读它），两个值域不能混淆。视图**不带结果文本**：没有搜索卡片的 UI 回退渲染 raw content。
- **`ReadResultView`**（`card: 'read'`）：行号代码视图 —— `path`、1-based `offset`（窗口越界导致 `lines` 为空时也保留，UI 因此知道续读位置）、`lines: ReadFileLine[]`（保留文件自身行号）、精确的 `totalLines`、按扩展名推断的 `lang?` 高亮提示、以及给不支持 read 卡片的 UI 用的 `content?`（剥掉信封的模型可见文本）。
- **`WebResultView`**（`card: 'web'`，`kind` 判别）：`kind: 'search'` 带 `sources: WebSource[]` / `answer?` / `truncated`；`kind: 'fetch'` 带 `url` / `statusCode` / `truncated`。两种形态合并成一个联合，因为 UI 用同一组件族渲染；这里的 `kind` 是本联合自己的判别符，不是 `ToolCallKind`（值刻意对齐工具的 pending 图标，但新增臂只是"联合编辑 + 消费者分支"，不接受任意 `ToolCallKind` 值）。**视图不带正文** —— fetch 的 markdown 正文已在 raw content 里，卡片只带检索摘要。

### 4. `ReadResultView` 与 `presentationMeta` 的持久化链路

```ts
/**
 * A completed file read rendered as a line-numbered, optionally syntax-highlighted
 * code view by a capable UI. ... The structured
 * `lines`/`path`/`lang`/`totalLines` fields cannot be reconstructed from the
 * model-facing result text alone, so the read tool projects them through its
 * `output.presentationMeta` (persisted with the session log) and `presentResult`
 * narrows that metadata back into this view on live and replay paths alike.
 */
export interface ReadResultView {
  card: 'read'
  ...
}
```

注释里的这句话就是"presentationMeta 持久化卡片元数据"机制的全貌，串起来是四步：

1. **投影**：工具的 `output.presentationMeta(args, value)`（纯函数）在管线 `createSuccessResult` 里被调用，产出 JSON 元数据（read 工具放 `lines/path/totalLines/lang`，web 工具放 `sources`）；
2. **落账**：`batch.ts` 的 `commit()` 把 `result.meta` 写进 `tool/result`（PTC 子调用则写进 `tool/ptc-dispatch`）—— session 日志因此携带卡片元数据；
3. **收窄**：工具的 `presentResult(args, result)` 拿到的 `ToolResult` 就带 `meta?: JsonValue`，工具自己把它窄化成 `ReadResultView` 等结构化视图；
4. **重放**：UI 从落账的 `meta` 重建视图 —— 实时与重放走同一条路。

为什么必须这样？`lines`/`totalLines`/`statusCode` 这些结构化字段**无法从模型可见文本无损重建**（文本被截断、信封被剥离），而 web 卡片的 `sources` 连 faithful 投影都做不到 —— 这正是 `WebSource` 注释强调的：

```ts
/**
 * One citeable source in a completed {@link WebSearchResultView}, the faithful
 * projection of one web-search source. The presentation projection of `dsh-web`'s
 * `WebSearchSource`: that Service Definition type is authoritative (core cannot depend
 * on the web Service Definition, so the two are declared separately and MUST evolve together).
 */
export interface WebSource {
  url: string
  title?: string
  snippet?: string
  publishedAt?: string
}
```

`core` 包不能依赖 `dsh-web` 的 Service Definition，所以词汇在两处各声明一次、**必须同步演化** —— 这是依赖方向约束下的显式重复，不是疏忽。

### 5. 与执行解耦的边界

贯穿全部类型的两条纯度规则（写在 `contracts.ts` 的 `presentCall`/`presentResult` JSDoc，落实在 `schema.ts` 的 `defineTool` 包装）：

- **纯函数**：只许依赖 `args`（pending）或 `args + content/isError/meta`（completed），不许碰文件系统、Session 或任何可变状态 —— 因为调用时机不可预测（实时 or 重放）；
- **软校验**：`defineTool` 包装的 presenter 在 args 不合法时返回 `undefined` 回退 generic 卡片，绝不抛错 —— 重放的 args 可能来自旧 schema。

另外注意"哪些视图没有 call 时形态"：search、read、web 的**结果视图**都没有对应 pending 卡片 —— pending 状态下结构化数据尚不存在（匹配行、文件行、检索来源都要等 `execute` 返回），所以它们的 pending 卡是 generic（`kind: 'search'`/`'read'`/`'fetch'` 只影响图标）；diff 相反，pending 与 completed 都有卡片（completed 替换 pending）。

## 核心类型与函数

| 名称 | 签名概要 | 职责 |
|---|---|---|
| `ToolCallKind` | 8 值字符串联合 | generic 卡片图标分类 |
| `FileLocation` | `{ path, line? }` | 编辑器 follow-along 的文件定位 |
| `FileDiff` | `{ path, oldText, newText }` | 单文件变更；`oldText: null` = 无前像 |
| `ReadFileLine` | `{ number, text }` | 1-based 行号的读取窗口行 |
| `ToolCallView` | generic/terminal/diff 联合 | pending 状态渲染意图（`presentCall` 返回） |
| `ToolResultView` | 6 臂联合 | completed 状态渲染意图（`presentResult` 返回） |
| `SearchResultView` | `shape: 'matches' \| 'paths'` 判别 | grep/glob 结果卡片 + truncated/total |
| `ReadResultView` | `{ path, offset, lines, totalLines, lang?, content? }` | 行号代码视图；结构化字段靠 presentationMeta |
| `WebResultView` | `kind: 'search' \| 'fetch'` 判别 | 网络检索摘要卡片；正文不复制 |
| `WebSource` | `{ url, title?, snippet?, publishedAt? }` | web 来源的忠实投影，与 dsh-web 类型同步演化 |

## 与其他模块的关系

- `contracts.ts`：`ToolDefinition.presentCall`/`presentResult` 的返回类型来自这里；`ToolResult`（`{ content, isError, meta? }`）是 `presentResult` 的第二参数。
- `schema.ts`：`defineTool` 给 presenter 套软校验包装（非法 args → `undefined`）。
- `pipeline.ts`：`output.presentationMeta` 在 `createSuccessResult` 被调用，`meta` 经 `materializeFinalResult` 快照冻结。
- `batch.ts`：`meta` 随 `tool/result` 事件落账 —— 重放 UI 的数据源。
- 消费方：Host presenter（`packages/host`）直接调用 `presentCall`/`presentResult` 渲染；内置 Web Client **不消费**这两个方法 —— 它从 raw 事件与落账 `meta` 自行推导卡片（transport split，见 `packages/core/tools/README.md` 的 Host presentation descriptors 一节与 agent-tool-presentation 教程）。
- `dsh-web` 的 `WebSearchSource`：`WebSource` 的权威对应物。

## 小结

`presentation.ts` 把"UI 想怎么显示一次工具调用"建模成 provider-neutral 的渲染意图联合：pending 只有 generic/terminal/diff 三种（search/read/web 的结构化形态在执行前不存在），completed 六种，判别符 `card` 统一、二级判别符各归其位（`shape` 避让图标用的 `kind`）。它通过两条规则与执行解耦 —— presenter 是只依赖输入的纯函数（live 与 replay 同路径），结构化卡片数据无法从模型文本重建的部分由 `presentationMeta` 投影、随 session 日志持久化、重放时窄化回视图。工具因此只需描述"我长什么样"，永远不需要知道哪个客户端在渲染它。
