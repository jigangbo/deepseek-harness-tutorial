# DeepSeek Harness 架构教程

> **从入门到精通**：系统深入学习 [DeepSeek Harness (dsh)](https://github.com/deepseek-ai/deepseek-harness) 的插件化 Agent 架构。
> 本教程是独立的第三方学习文档，源码路径均相对主仓库根目录（如 `packages/core/agent-loop/src/agent.ts`）。

DeepSeek Harness 是 DeepSeek AI 开源的 Agent Harness（代理马具）：模型负责思考，harness 负责其余一切工程——会话日志、工具执行、权限审批、文件/终端/浏览器能力、UI 与持久化。它构建于 **Cordis** 插件框架之上，采用 **一切皆插件（everything-is-a-plugin）** 架构：模型适配器、工具注册表、会话日志、甚至 agent 循环本身都是可从配置替换的插件。

---

## 架构总览

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        产品面 (Product Surfaces)                     │
│   apps/web (浏览器 UI)    apps/desktop (Electron)    SDK (JSON-RPC)   │
│   packages/client/*      packages/host/*            packages/sdk     │
├─────────────────────────────────────────────────────────────────────┤
│                     启动与组合层 (Boot & Composition)                 │
│   dsh CLI ──> Profile (web/headless/sdk/acp)                         │
│                 └─> Bundle 叠层 (dsh-base → dsh-web-app → …)         │
│                       └─> cordis.yml + patch 叠加 → Cordis 插件树    │
│   packages/boot/*                    packages/bundle/*               │
├─────────────────────────────────────────────────────────────────────┤
│                        核心引擎 (Core Engine)                        │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────────┐  │
│  │ agent-loop  │  │ system-prompt│  │ tools (注册表+执行管线)     │  │
│  │ (默认驱动)  │  │ (提示词组装) │  │   tools/pre→execute→post   │  │
│  └──────┬──────┘  └──────────────┘  └────────────────────────────┘  │
│         │ prepareCall                        │ tool/call             │
│  ┌──────▼──────┐                    ┌────────▼────────┐              │
│  │ llm (适配器 │                    │ 能力缝 (Seams)  │              │
│  │ seam+流词汇)│                    │ fs shell sandbox│              │
│  └──────┬──────┘                    │ web subagent …  │              │
│         │                           └────────┬────────┘              │
│  ┌──────▼──────────────────────────────────▼───────┐               │
│  │  session (append-only 会话事件日志 = 事实唯一来源) │               │
│  │  packages/core/session + packages/session/*      │               │
│  └─────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
          │                                    │
   ┌──────▼──────┐                      ┌──────▼──────┐
   │ LLM Provider│                      │ 持久化后端   │
   │ deepseek 等 │                      │ JSONL+zstd  │
   └─────────────┘                      └─────────────┘
```

三条铁律贯穿全库：

1. **一切皆插件**：没有特权核心，扩展 = 在插件树旁挂一个新插件；注册即 effect，插件卸载时自动回收。
2. **能力缝（Seam）**：每个可替换能力由三角色组成——Service Definition（接口）、Service Provider（实现）、Consumer（消费者）。换一个 provider，整个产品行为随之改变。
3. **模型可见 ⟺ 已记录（Model-visible ⟺ Logged）**：任何进入模型请求的内容都必须能从会话日志重建；会话日志是事实的唯一来源。

---

## 主工作流：一次对话的生命周期

一次用户输入触发一个 **turn**；每个 turn 由 0 或多个 **step** 组成，一个 step = 一次模型请求 + 它调用的工具。

```text
 (1) 用户输入 / agent.inject() ──> inbox
      │
 (2) turn/start ──► claim 下一步输入
      │
 (3) agent/pre-step (waterfall)：接受 / 改写 / 拒绝消息
      │        拒绝或首条声明为空 ⇒ 无 step 关闭 turn
 (4) 组装：system prompt 段 + step 准入工具 schema（快照模型选择）
      │
 (5) step/start
      │
 (6) agent/request → prepareCall()：绑定适配器；此阶段取消则什么都不提交
      │
 (7) reconcile system/message，user/message 入日志，记录 request/header、request/context
      │
 (8) 从日志冻结 (freeze) 模型历史  ◄── 模型可见 ⟺ 已记录
      │
 (9) 流式请求 → llm/stream → agent/assistant-stream (start → chunk* → end)
      │
(10) assistant/message 入日志（失败的尝试记为 assistant/attempt，不进历史）
      │
(11) tool/call* → tools/pre-execute (审批/守卫)
      │        → tools/execute (并发限制、native/PTC 执行器)
      │        → tools/post-execute → tool/result* 入日志
      │
(12) step/end
      │
 (13) 工具还欠一次请求？或新输入已排队？ ── 是 ──► 回到 (2) 下一个 step
      │ 否
(14) agent/turn-stopping (串行，无 next())
      │
(15) turn/end
```

### 流程步骤 → 源码 → 教程章节映射表

| 步骤 | 机制 | 核心源码 | 教程章节 |
|------|------|----------|----------|
| 1 输入排队 | inbox 投递 | `packages/core/agent-loop/src/inbox.ts` | 核心引擎篇 · agent-loop |
| 2-3 turn 开启与预检 | 瀑布事件 | `packages/core/agent-loop/src/agent.ts` | 总览篇 · 生命周期 / 核心引擎篇 · agent-loop |
| 4 提示词组装 | section + schema 准入 | `packages/core/system-prompt/src/index.ts` | 核心引擎篇 · system-prompt |
| 5-7 请求准备 | prepareCall / 头记录 | `packages/core/agent-loop/src/agent.ts` | 核心引擎篇 · agent-loop |
| 8 冻结历史 | deriveMessages | `packages/core/session/src/index.ts` | 核心引擎篇 · session |
| 9-10 流式与结算 | 适配器流 | `packages/llm/llm/src/assistant-stream.ts` | 核心引擎篇 · llm |
| 11 工具管线 | 三段瀑布 | `packages/core/tools/src/pipeline.ts` | 核心引擎篇 · tools |
| 11 执行器 | native / PTC | `packages/core/tool-executor-native`、`tool-executor-ptc` | 核心引擎篇 · executors |
| 13-15 turn 收尾 | 串行停止事件 | `packages/core/agent-loop/src/agent.ts` | 总览篇 · 生命周期 |

---

## 模块索引

| 篇章 | 内容 | 适合谁 |
|------|------|--------|
| [入门篇](00-getting-started/01-what-is-an-agent-harness.md) | 什么是 Agent 与 Harness、Cordis 框架精讲、阅读指南 | 所有人从这里开始 |
| [总览篇](01-overview/01-global-architecture.md) | 全局架构与包地图、turn 生命周期、启动与 Profile | 建立全局心智模型 |
| [核心引擎篇](02-core-engine/session/index.md) | session / agent-loop / tools / agent / scope / system-prompt / executors / llm **逐文件精讲** | 吃透主循环与数据流 |
| [能力缝篇](03-capability-seams/00-seam-design.md) | fs、shell+沙箱、终端、PTC、web、浏览器与计算机、LSP+技能、子代理+团队、任务+工作流、MCP+自扩展 | 理解"换 provider 换产品" |
| [持久化篇](04-persistence/00-overview.md) | 会话日志与存储后端、格式版本迁移、投影、压缩、检索、附件 | 掌握数据平面 |
| [交互与产品形态篇](05-product-surfaces/00-interaction-and-approval.md) | 审批与提问、Web 前端、Host 服务、桌面应用、SDK/ACP/API | 了解产品如何组装 |
| [精通篇](06-mastery/00-testing.md) | 测试体系、配置体系、扩展实战三篇、架构模式总结 | 独立开发插件 |

## 快速开始

- **只想跑起来**：`npx @deepseek-ai/dsh web`，浏览器打开 `http://127.0.0.1:3080`。
- **2 小时速览**：入门篇 → 总览篇 → 挑一个能力缝章节。
- **完整学习**：按篇章顺序通读，核心引擎篇对照源码精读。
- **动手写插件**：直接跳精通篇扩展实战。

## 约定

- 源码路径相对主仓库根：`packages/<group>/<pkg>/src/...`。
- 教程基于主仓库 2026-11 前后的 master 分支；项目处于 developer preview，API 可能变化。
- 本教程由 AI 辅助生成，欢迎对照源码勘误。
