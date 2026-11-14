# 持久化篇 — 数据平面

> 会话日志如何落盘、如何演进格式、如何折叠成领域状态、如何被检索。核心不变量：**事件从 seq 0 连续、永不重写；flush 才是持久性屏障**。

| 章节 | 内容 |
|------|------|
| [持久化总览](00-overview.md) | 五组职责地图 + 五条核心不变量 |
| [会话日志与存储后端](01-session-log-and-backends.md) | SessionHandle、write-behind 窗口、崩溃恢复（不修剪、读者修复）、JSONL 后端 |
| [格式版本与迁移](02-format-versions-and-migrations.md) | 三个权威来源、世代文件命名、v0→v4 邻接迁移链、读/写打开的差异 |
| [会话投影](03-session-projections.md) | 纯同步折叠单元、水位线一致性、Object.is 引用纪律、持久投影缓存 |
| [压缩与请求上下文](04-compaction-and-context.md) | log-only 事件锁、surface replace、工具结果修剪、请求上下文注入 |
| [会话检索](05-session-query.md) | 精确读/过滤/追踪、revision 键控冷读缓存、SQLite 全文搜索 |
| [附件、溢出与通用存储](06-attachment-spill-storage.md) | 内容寻址附件、工具结果溢出、多后端 KV 枢纽 |

**上一篇**：[能力缝篇](../03-capability-seams/)　**下一篇**：[交互与产品形态篇](../05-product-surfaces/)
