# 核心引擎篇 — 逐文件精读

> 八个核心包的**逐文件**教程：每个源文件一篇，含真实代码走读、函数表与模块关系。这是全库的心脏，建议对照源码精读。

## 推荐阅读顺序

| 顺序 | 模块 | 内容 | 为什么在这个位置 |
|------|------|------|------------------|
| 1 | [Session](session/) | append-only 会话事件日志——事实唯一来源 | 数据先于逻辑 |
| 2 | [Agent Loop](agent-loop/) | 默认驱动：turn/step 状态机、请求构建、取消承诺 | 驱动消费数据 |
| 3 | [Tools](tools/) | 工具注册表与三段瀑布执行管线 | 模型的"手" |
| 4 | [Agent](agent/) | Agent 接口、活体注册表、agent/* 事件契约 | 契约与实现分离 |
| 5 | [Scope](scope/) | 按 agent 作用域注册的原语 | 支撑多 agent 树 |
| 6 | [System Prompt](system-prompt/) | 提示词段与工具 schema 组装 | 组装进请求 |
| 7 | [Executors](executors/) | native 与 PTC 双执行器 | 执行形态可换 |
| 8 | [LLM](llm/) | 消息/流词汇表、适配器接缝与 provider 族 | 出站到模型 |

每章首页（模块导览）含 ASCII 架构图、关键不变量与推荐文件阅读顺序；`index.ts` 的内容并入模块导览，其余每文件一篇。

**前置**：先读 [总览篇的 turn 生命周期](../01-overview/02-turn-lifecycle.md)，本篇逐文件展开它引用的机制。

**下一篇**：[能力缝篇 — 设计思想](../03-capability-seams/00-seam-design.md)
