# 01 · 持久化总览

> 数据平面地图


> dsh 的持久化哲学：**内存中的 append-only 会话日志是事实的唯一来源；持久化层负责让这份日志在崩溃后依然存在且可信。** 本章是持久化篇的地图，后续章节逐个展开。

## 数据平面的五组职责

```text
┌────────────────────────────────────────────────────────────────┐
│ 1. 会话日志与存储后端                                            │
│    ctx.sessions (内存) ──► ctx.sessionPersistence (缝)          │
│    packages/session/session-persistence + -jsonl               │
├────────────────────────────────────────────────────────────────┤
│ 2. 会话格式版本与迁移                                            │
│    session-format / catalog / v0→v1→v2→v3→v4 逐代迁移包         │
├────────────────────────────────────────────────────────────────┤
│ 3. 会话投影 (Projection)                                        │
│    ctx.sessionProjections：从日志折叠出的领域状态 + 持久缓存     │
├────────────────────────────────────────────────────────────────┤
│ 4. 上下文管理                                                   │
│    compaction (压缩) + context (请求上下文) + token-meter       │
├────────────────────────────────────────────────────────────────┤
│ 5. 检索与外围存储                                               │
│    session-query (检索) / attachment (附件) / spill (溢出)      │
│    / storage (非会话存储) / workspace / session-title          │
└────────────────────────────────────────────────────────────────┘
```

## 核心不变量（贯穿全篇）

1. **没有平行的持久化事件类型**：`SessionPersistence` 直接持久化 `SessionEvent` 词汇表，不发明第二套存储事件——内存态与磁盘态永远同构。
2. **事件从 seq 0 连续、永不重写**：已提交事件不可改；撕裂的物理尾巴永远不会返回给读者，写入路径在第一次 append 前截断它。
3. **append 是尽力而为，flush 才是持久性屏障**：flush 解析后，崩溃也能恢复；崩溃前从未物化的会话等于不存在。
4. **单写者所有权**：每个会话同时最多一个写 handle；第二个 `open(id, 'write')` 拒绝。
5. **格式只进不退**：识别不了的更高版本拒绝，而不是猜测；已提交的世代文件永不改名/替换/删除。

## 各章导航

| 章节 | 内容 | 核心包 |
|------|------|--------|
| [会话日志与存储后端](01-session-log-and-backends.md) | SessionHandle、flush 检查点、崩溃恢复、JSONL 后端 | `session-persistence`、`session-persistence-jsonl` |
| [格式版本与迁移](02-format-versions-and-migrations.md) | SESSION_FORMAT_VERSION、逐代迁移链、拒绝语义 | `session-format*` 五个包 |
| [会话投影](03-session-projections.md) | 折叠器注册、水位线、投影缓存 | `session-projection`、`session-projection-cache` |
| [压缩与请求上下文](04-compaction-and-context.md) | 压力触发压缩、工具结果修剪、请求上下文注入 | `compaction*`、`context/*` |
| [会话检索](05-session-query.md) | 冷读缓存、全文检索、游标 | `session-query*` |
| [附件、溢出与通用存储](06-attachment-spill-storage.md) | 内容寻址附件、工具结果溢出、KV 存储枢纽 | `attachment*`、`spill*`、`storage*` |

## 与核心引擎的关系

- agent-loop 是持久化层最重要的消费者：turn 结束前用 `session/flush` 作为排序与错误观察检查点；恢复时通过写 handle 补记 `interrupted` 关闭事件。
- 「模型可见 ⟺ 已记录」成立的前提是日志先落盘——flush 检查点保证模型请求发出前，构成请求的事件已经在持久化管线上。
- 会话投影是 UI 与领域服务读日志的标准方式（见交互与产品形态篇）。
