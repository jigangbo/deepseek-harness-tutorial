# `content.ts`

> 内容块投影:文件/图片占位与工具更新


> 源码:`packages/llm/llm/src/content.ts`(约 500 行)。请求装配阶段的全部"投影"逻辑都住在这里:持久引用如何在**不重写追加式会话历史**的前提下变成每个路由各得其所的请求版本。

## 文件概览

投影规则一句话版:

- **文件无条件投影**——没有任何 provider 原生接收文件块;每个 `FileBlock` 出现(含 tool 角色结果里的)都变成确定性 handle 文本。
- **图片按路由能力投影**——image-capable 路由保留图片;文本模型收到确定性占位;被 offload 的出现对所有路由都是占位文本。
- **工具更新按路由模式投影**——路由声明的 `toolUpdate` 模式决定 developer 消息里的增删块哪些能进请求。

主要导出:`ImageAttachmentAccess`/`resolveImageAttachmentAccess`、四个占位文本函数、`contentHasImage`/`contentHasFile`、`projectFilesToText`、`projectOffloadedImages`、`requiredImageOffload`、`projectImagesForTextModel`、`projectToolUpdates`。

## 关键代码走读

### 占位文本的确定性

```typescript
// packages/llm/llm/src/content.ts
export function textOnlyImageText(ref: ImageAttachmentRef): string {
  const digest = String(ref.attachmentId).slice('sha256:'.length, 'sha256:'.length + 8)
  return `[image omitted because this model accepts text only; attachment sha256:${digest}]`
}

export function fileHandleText(ref: FileAttachmentRef, readonlyPath: string | undefined): string {
  const digest = String(ref.attachmentId).slice('sha256:'.length, 'sha256:'.length + 8)
  const identity = `File ${quoted(ref.name)} (${ref.bytes} bytes, sha256:${digest})`
  if (readonlyPath === undefined) {
    return `[${identity} was uploaded, but the current execution environment cannot access a readable path. Report that limitation if its contents are needed; do not claim to have read it.]`
  }
  return `[${identity}: verbatim read-only copy saved at ${quoted(readonlyPath)}. Read that path with your file tools when its contents are needed; …]`
}
```

- 身份文本从**出现自身**的持久引用派生(名字、字节数、sha256 前 8 位),同一文件在每次请求中产生相同文本——这是 KV cache 前缀稳定的必要条件。
- 路径不可达时不是省略,而是明确告诉模型"报告该局限,不要声称读过"。防幻觉写进了占位文本本身。
- `requestImageHandleText`(image-capable 路由的请求预览)与 `offloadedImageText`(offload 占位)同构,后者在无本地路径时让模型"请用户重新附件"。

### resolveImageAttachmentAccess:把宿主路径映射进工具执行世界

```typescript
// packages/llm/llm/src/content.ts
export function resolveImageAttachmentAccess(
  attachments: AttachmentStore,
  mapHostPath: (hostPath: string) => string | undefined,
  ref: ImageAttachmentRef,
): ImageAttachmentAccess | undefined {
  const hostPath = attachments.imageHostPath(ref)
  if (hostPath === undefined) return undefined
  const readonlyPath = mapHostPath(hostPath)
  return readonlyPath === undefined ? undefined : { readonlyPath }
}
```

附件 provider 只知道宿主路径;工具执行文件系统(可能是沙箱/SSH 世界)由消费者提供映射函数。两个定义互不依赖,这个函数只是把两者接起来。结果只对当次请求有效,不参与 `variantId`。

### projectFilesToText:延迟拷贝的逐块替换

```typescript
// packages/llm/llm/src/content.ts
function replaceFilesWithHandles(
  blocks: readonly ContentBlock[],
  resolvePath: (ref: FileAttachmentRef) => string | undefined,
): ContentBlock[] {
  let next: ContentBlock[] | undefined
  for (const [index, block] of blocks.entries()) {
    if (block.type === 'file') {
      next ??= blocks.slice(0, index)          // 首次替换时才拷贝前缀
      next.push({ type: 'text', text: fileHandleText(block.attachment, resolvePath(block.attachment)) })
      continue
    }
    next?.push(block)
  }
  return next ?? blocks as ContentBlock[]      // 无文件 → 原数组原样返回
}

export function projectFilesToText(
  messages: readonly RequestMessage[],
  resolvePath: (ref: FileAttachmentRef) => string | undefined,
): readonly RequestMessage[] {
  if (!messages.some(message => contentHasFile(message.content))) return messages
  return messages.map((message) => {
    const content = replaceFilesWithHandles(message.content, resolvePath)
    return content === message.content ? message : { ...message, content }
  })
}
```

统一的替换模式在文件、offload 图片、text-only 图片三处复用:`next` 未赋值说明一个都没替换,直接返回**原数组**(引用相等);一旦有替换,从首个替换点切片拷贝。消息层同理——没动的消息保持原引用。整条链路上"没变的引用不变",比较与缓存都因此可靠。

注意 `projectFilesToText` 有两个重载:`Message[]` 与 `RequestMessage[]`(混合持久消息与请求专用输入);文件检测每次调用都读当前内容,**不缓存**消息身份或冻结状态。

### requiredImageOffload:预算超限时的确定性推进

```typescript
// packages/llm/llm/src/content.ts
export function requiredImageOffload(
  messages: readonly RequestMessage[],
  budget: Pick<LlmImageRequestBudget, 'representation' | 'maxBytes' | 'maxImages' | 'byteQuantum' | 'countQuantum'>,
  versionBytes: (block: ImageBlock) => number,
): number {
  const lengths: number[] = []
  for (const message of messages) {
    visitImageBlocks(message.content, (block) => {
      if (block.offloaded === true) return     // 已 offload 的不算
      const bytes = versionBytes(block)
      lengths.push(budget.representation === 'base64' ? base64Length(bytes) : bytes)
    })
  }
  return offloadedImagePrefixCount(lengths, budget)
}
```

- 字节数来自 `versionBytes` 回调(路由自己的请求版本字节数,适配器经 `imageRequestTarget` 提供),`base64` 表示按 `Math.ceil(bytes/3)*4` 展开为编码长度。
- 内部 `offloadedImagePrefixCount` 只依赖表示长度,所以**每个路由对同一组图片算出同样的个数**;`byteQuantum`/`countQuantum` 让"一次 offload 多少个"成为路由的确定性步长(不足一个量子就向上取整)。
- 返回值进 `LlmFailure.offloadImages`,路由**不自行 offload**——失败带数,由 `dsh-compaction-image-offload` 记录 `image/offload` 事件后重试(模型可见 ⟺ 已记录)。

### projectToolUpdates:按路由模式投影工具声明

这是文件里最复杂的函数,签名:

```typescript
// packages/llm/llm/src/content.ts
export function projectToolUpdates(
  messages: readonly RequestMessage[],
  tools: readonly ToolSchema[] | undefined,
  toolUpdate: ToolUpdate | undefined,   // 'in-history' | 'addition-only' | undefined
  history?: ToolHistory,
): ProjectedToolUpdates
```

三个分支:

```typescript
// packages/llm/llm/src/content.ts
if (toolUpdate === undefined) {
  // 不支持更新历史的路由:剥掉 deferLoading,立即声明全部工具,丢掉 developer 消息
  let immediateTools = tools
  if (tools?.some(tool => tool.deferLoading === true)) {
    immediateTools = tools.map(({ deferLoading: _loading, ...tool }) => tool)
  }
  return { messages: withoutDeveloperMessages(messages), tools: immediateTools }
}

if (history === undefined) {
  // 只有当前 schema,无法解析历史更新引用的定义 → 同样走"无 developer 消息"
  return { messages: withoutDeveloperMessages(messages), tools }
}
```

第三分支(支持更新且有完整历史)先做完整性检查:请求里每个被 `history.updates` 引用的 developer 消息必须存在——辅助调用的请求前缀可能缺更新,缺了就整条退回上面第二条路。然后:

- `addition-only` 模式先跑 `rebaseAdditionOnlyHistory`:当出现 `tool-removal`(或向空集合添加)导致"请求前缀会变"时,重建基线声明并把已激活的定义变为立即声明,保留其后的更新序列;
- `toolDeclarations` 把初始 tools + 各次 additions 折成声明 map(尚未激活的加 `deferLoading: true`);
- 遍历消息投影 developer 块:`tool-addition` 只保留"已声明且尚未 offered"的(被取代的块省略,同消息里无关内容保留);`tool-removal` 仅 `in-history` 路由保留(`addition-only` 不能经历史去激活);
- 最终若逐条引用相等,返回原 messages 数组——依然是"没变就不拷贝"。

## 核心类与函数

| 名称 | 签名概要 | 职责 |
|------|----------|------|
| `resolveImageAttachmentAccess` | `(attachments, mapHostPath, ref) => {readonlyPath} \| undefined` | 把附件宿主路径映射进当前工具执行世界 |
| `textOnlyImageText` | `(ref) => string` | 文本模型图片占位(确定性) |
| `requestImageHandleText` | `(ref, version, access?) => string` | image 路由的请求预览 handle 文本 |
| `offloadedImageText` | `(ref, access?) => string` | offload 占位文本 |
| `fileHandleText` | `(ref, readonlyPath?) => string` | 文件 handle 文本(provider 能收到的唯一表示) |
| `contentHasImage` / `contentHasFile` | `(content) => boolean` | 共享的内容走查(所有图片/文件策略共用) |
| `projectFilesToText` | `(messages, resolvePath) => messages` | 文件块 → handle 文本(无条件) |
| `projectOffloadedImages` | `(messages, placeholder) => messages` | offloaded 图片出现 → 占位文本(所有路由同一集合) |
| `requiredImageOffload` | `(messages, budget, versionBytes) => number` | 预算超限时还要 offload 多少个最旧出现 |
| `projectImagesForTextModel` | `(messages) => messages` | 文本模型的图片 → 占位 |
| `projectToolUpdates` | `(messages, tools, toolUpdate, history?) => {messages, tools}` | 工具声明/更新按路由模式投影 |

## 与其他模块的关系

- `index.ts` 的 `adapterStream` 在派发前依次调用 `projectFilesToText`(有文件时)、`projectImagesForTextModel`(文本模型且有图时)、`projectToolUpdates`(总是);`fileRequestText()` 暴露的正是 `fileHandleText` 的同步版本。
- `dsh-attachment` 提供 `AttachmentStore`/引用类型;`@deepseek-ai/dsh-util-values` 提供 `assertNever`(media type switch 收尾)。
- 图片预算失败 → `error.ts` 的 `IMAGE_OFFLOAD_REQUIRED` 码 → `dsh-compaction-image-offload` 消费。
- developer 消息的 tool-addition/removal 块由 `message.ts` 的角色体系承载,本文件决定它们在每条路由上"看得见什么"。

## 小结

- 投影的全部价值在两个不变量:**确定性**(同一输入永远同样的文本)与**追加历史的不可重写**(投影发生在请求装配,持久日志不动)。
- 替换函数统一采用"没变返回原引用"的懒拷贝模式;引用相等是下游优化与比较的地基。
- 预算机制是"失败带数、由记录者修复"的典型分工:适配器拒绝并命名个数,compaction 记录决策并重试——不让投影层悄悄改写模型可见输入。
