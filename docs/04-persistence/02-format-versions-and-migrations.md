# 会话格式版本与迁移 — 世代文件与邻接迁移链

> 用户会话数据一旦发布就是**不可丢弃的资产**——连 GitHub 的 prerelease 标记都不能让持久化用户数据变成可抛弃的。dsh 为此建立了一套严格到几乎偏执的格式版本纪律。核心源码：`packages/session/session-format*` 六个包。

## 三个事实来源（谁说了算）

| 权威 | 位置 | 拥有什么 |
|------|------|----------|
| **checkout 写入者** | `SESSION_FORMAT_VERSION`（`packages/core/session/src/types.ts`） | 代码中唯一手工维护的"当前写入版本号"；目录生成器从它推导 codec 排序并检查邻接迁移能否到达它 |
| **最终化记录** | `docs/persistence-changes/finalized/` | 已接受的兼容基线（当前：`latestFinalizedVersion: 4`） |
| **发布记录** | `docs/session-format-status.md` 的 yaml 记录 | 已验证的**已发布**格式（当前：`latestReleasedVersion: 4`，证据 tag `dsh-v0.2.0-rc.2`） |

规则：

- 包版本号、codec 导出名、fixture 文件名、投影缓存版本**都不是**写入版本权威。
- alpha/beta/rc 发布同样确立已发布的会话格式义务。
- 发布记录在开发主干上**永不降低**；结构、双语一致性、证据 tag 与写入者路径一致性由 `scripts/doc-standard.spec.ts` 门禁检查。

## 命名即契约：世代文件

```text
v0:   session.jsonl[.zstd]            ← 初代，无版本后缀
v1+:  session.vN.jsonl[.zstd]         ← 小写 .vN，文件名必须与 header 一致
```

铁律：

1. **已提交的世代路径永不改名、替换或删除**——迁移产生新文件，旧文件原样保留。
2. 消费者只认识"当前逻辑格式"；header-only 的 `stat`/`list` 重扫每个会话目录，选择**数值最高的规范世代**，翻译受支持的历史 header，不加载事件。
3. 打开存储会话时：选择同一世代 → 未来版本拒绝 → 否则解码并组合**静态邻接迁移链**一次，返回校验过的当前逻辑事件。

## 邻接迁移链：每代一包

```text
v0 ──► v1 ──► v2 ──► v3 ──► v4
│      │      │      │
└─ session-format-v0-to-v1     每个包恰好拥有一步 vN → vN+1
       ├─ session-format-v1-to-v2
       │     ├─ session-format-v2-to-v3
       │     │     ├─ session-format-v3-to-v4
       │     │     │     └─ session-format + session-format-catalog
       ▼     ▼      ▼      ▼
        组合成一条静态链（启动前一次性组合，非运行时解释）
```

- **每个迁移包恰好拥有一个 `vN -> vN+1` 步骤**；不相邻版本之间没有跳跃转换。
- 目录生成器（`scripts/gen-session-format-catalog.ts`）检查邻接迁移链能到达 `SESSION_FORMAT_VERSION`——断链在 CI 就炸。
- **相邻迁移可能添加以版本命名的后继**，但绝不移动、覆盖或删除已提交的世代；前代存在既不暗示回退支持也不暗示降级支持。

## 读打开 vs 写打开

```text
读打开 (open(id, 'read')):
  选世代 → 解码 → 组合迁移链（内存一次）→ 返回校验过的当前逻辑事件
  不发布后继文件——源路径、字节、inode 一个都不动

写打开 (open(id, 'write')):
  持有单写者声明与文件租约 → 复用准备好的迁移结果
  → 编码、校验 → 独占地发布最终版本命名的后继文件（在未改动的源旁）
  → 然后才返回可写 handle
```

- **修复与迁移是两回事**：普通修复未封口的打断尾部是 handle 消费者的职责；迁移只为"已被后续 `turn/start` 封口的、有界的已发布重启"插入缺失的打断 `turn/end`。
- 当前格式恢复保留已安装扩展与带 `ignorable: true` 的未知事件；**历史 v0/v1/v2 迁移遇到未知类型即拒绝**——即使标了 ignorable。
- 版本 bump 决策需要更高的写入者版本和自己的 header 转换；**不能复用已接受的 3→4 转换**。

## 同版本变更的纪律

不是所有 schema 变化都要 bump 版本。在当前版本内继续演化 V4 的条件（读者兼容规则）：

1. 变化对**安全的读者行为**无害：判别标签（discriminator）足以防止误解；新成员 required-on-read 除非信封带 `ignorable: true`。
2. 记录了所需的类型变更确认（typed acknowledgements review）。
3. `SESSION_FORMAT_VERSION` 只在**现有判别标签无法阻止不安全解读**时才 bump。

用词纪律：一般行为用"当前格式 / 下一邻接版本"；**显式数字只用于**固定的迁移输入输出、wire schema、历史证据及其测试。

## 会话格式演化史（v0 → v4 概览）

| 版本 | 文件名 | 关键变化（概览，细节见各迁移包 README） |
|------|--------|----------------------------------------|
| v0 | `session.jsonl[.zstd]` | 初代词汇表 |
| v1 | `session.v1.jsonl[.zstd]` | 词汇表重组，文件名开始携带版本 |
| v2 | `session.v2.jsonl[.zstd]` | 继续演化（以各代迁移包文档为准） |
| v3 | `session.v3.jsonl[.zstd]` | 引入嵌入压缩 Assistant 流等（以 v3→v4 迁移文档为准） |
| v4 | `session.v4.jsonl[.zstd]` | 当前格式：`MessageSourceMap` 化的 source 声明，拒绝旧 `{ kind: 'plugin', plugin }` 包装 |

> 每代的精确 schema 见 `docs/persistence-changes/historical-formats/`——从 0 到 checkout 写入者版本的每个整数都有历史 schema 与现行目录。

## 历史格式引用与测试

- `docs/persistence-changes/historical-formats/` 记录每个整数版本。
- 显式 `sessionFormat` 的 owner 保留迁移覆盖（测试政策）；历史 fixture 保留其发布时的表示，不追新。
- 快照体系的父/子文件命名（`session[.vN].jsonl` / `session.<ordinal>[.vN].jsonl`）与这里的世代命名完全同构——见测试体系章。

## 设计启发

1. **不可变世代 + 只增迁移**：像 Git 对象一样对待用户数据；任何时刻都能从原始字节重建任何历史版本。
2. **邻接链而非跳变**：每步迁移小而可验证，N 版本跨度由链组合完成，测试矩阵是线性的。
3. **权威的唯一化**：一个常量 + 两个机器可检查的记录，门禁保证它们不漂移。
4. **拒绝优先于猜测**：读不了就响亮拒绝（`SessionFormatUnsupportedError`），永不"尽力解读"用户数据。
