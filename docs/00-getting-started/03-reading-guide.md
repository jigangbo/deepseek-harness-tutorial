# 阅读指南 — 三条学习路径

本教程共约 30 个主题文档 + 55 篇核心引擎逐文件精读。按你的目标选择路径。

## 路径一：快速上手（约 2 小时）

目标：能跑起来、说清楚 dsh 是什么、看懂一次对话的完整流程。

1. [什么是 AI Agent 与 Harness](01-what-is-an-agent-harness.md) — 领域背景，零基础可读
2. [全局架构与包地图](../01-overview/01-global-architecture.md) — 建立分层心智模型
3. [一次对话的完整生命周期](../01-overview/02-turn-lifecycle.md) — turn/step 主流程
4. [能力缝设计思想](../03-capability-seams/00-seam-design.md) — 理解"换 provider 换产品"
5. 动手：`npx @deepseek-ai/dsh web`，对照 UI 观察一次工具调用

## 路径二：完整学习（约 2-3 周，吃透源码）

目标：能独立修改核心引擎、设计新能力缝。按顺序：

| 阶段 | 章节 | 重点 |
|------|------|------|
| 1. 地基 | 入门篇全部；总览篇全部 | Cordis 的事件/服务/effect 模型必须吃透 |
| 2. 数据平面 | [Session 模块](../02-core-engine/session/index.md) 全部 9 篇 | append-only 日志；模型可见⟺已记录 |
| 3. 驱动 | [Agent Loop](../02-core-engine/agent-loop/index.md) 全部 5 篇 | agent.ts 是全库最核心的文件 |
| 4. 工具 | [Tools](../02-core-engine/tools/index.md) 8 篇 + [执行器](../02-core-engine/executors/index.md) | 三段瀑布管线 |
| 5. 契约 | [Agent](../02-core-engine/agent/index.md) 7 篇、[Scope](../02-core-engine/scope/index.md) | Agent 接口与作用域 |
| 6. 模型 | [LLM](../02-core-engine/llm/index.md) 12 篇 + [Providers](../02-core-engine/llm/providers.md) | 适配器接缝与流式结算 |
| 7. 能力 | 能力缝篇挑 4-5 篇精读（建议 fs、shell+沙箱、subagent、MCP） | 三角色模型 |
| 8. 数据 | 持久化篇全部 | 格式版本与迁移纪律 |
| 9. 产品 | 交互与产品形态篇挑 2-3 篇 | UI 如何从事件投影 |
| 10. 出师 | 精通篇全部 | 测试门 + 三篇扩展实战 |

每章配套源码都在 `packages/` 下，**务必对照源码阅读**；逐文件篇里的代码片段可以直接跳到源文件看上下文。

## 路径三：按主题查阅

- **我想写一个工具** → [扩展实战：添加一个工具](../06-mastery/02-recipe-new-tool.md)
- **我想接一个新模型** → [LLM 模块](../02-core-engine/llm/index.md) → [扩展实战：添加 LLM 适配器](../06-mastery/03-recipe-new-llm-adapter.md)
- **我想做沙箱/远程执行** → [命令执行与沙箱](../03-capability-seams/02-shell-subprocess-sandbox.md)
- **我想做 UI 集成** → [Web 前端架构](../05-product-surfaces/01-web-frontend.md)
- **我想懂会话文件格式** → [会话日志与存储后端](../04-persistence/01-session-log-and-backends.md)、[格式版本与迁移](../04-persistence/02-format-versions-and-migrations.md)
- **我想提效：学它的工程实践** → [测试体系](../06-mastery/00-testing.md)、[架构设计模式总结](../06-mastery/05-architecture-patterns.md)

## 阅读建议

1. **先图后码**：每章的 ASCII 图先看懂，再进代码。
2. **跟一条真实事件流**：最好的读法是打开 `dsh --profile web --dump-config` 看你自己机器上的插件树，再对照教程找对应源文件。
3. **Waterfall 事件**：看到 `agent/*`、`tools/*`、`fs/*` 瀑布事件，记住监听器必须调用 `next()`，否则链路短路。
4. **遇到术语**：回首页底部术语速查（入门篇第 1 章末尾有完整表）。
