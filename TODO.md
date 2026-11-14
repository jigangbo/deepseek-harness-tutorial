# DeepSeek Harness 系统性教程 — 进度追踪

> 目标：从入门到精通，系统深入理解 DeepSeek-Harness 项目架构
> 创建：2026-11-12　完成：2026-11-14
> 源仓库：`G:\code\AISource\DeepSeek-Harness`（约 2,690 个 TS 源文件，56 个包组）

## 覆盖策略

- **逐文件精讲**：核心引擎（`packages/core/*`、`packages/llm/llm`）的每个源码文件（`index.ts` 并入模块导览）。
- **模块级深讲**：其余包组按能力域聚合，含关键源码走读。
- **工程实践**：测试体系、配置体系、扩展实战、设计模式总结。

## 进度总览

| 篇章 | 计划文档 | 完成 | 状态 |
|------|---------|------|------|
| 首页 index.md | 1 | 1 | ✅ |
| 00 入门篇 | 3 | 3 | ✅ |
| 01 总览篇 | 3 | 3 | ✅ |
| 02 核心引擎篇（逐文件） | 55 | 55 | ✅ |
| 03 能力缝篇 | 10 | 10 | ✅ |
| 04 持久化篇 | 7 | 7 | ✅ |
| 05 交互与产品形态篇 | 5 | 5 | ✅ |
| 06 精通篇 | 6 | 6 | ✅ |
| **合计** | **90** | **90** | ✅ |

## 文档清单

### 首页与导航（由主协调者撰写）
- [x] `docs/index.md` — 首页：架构总图 + turn 流程图 + 代码映射表 + 模块索引
- [x] `docs/.pages` 及各篇 `.pages` — 导航排序
- [x] `docs/00-getting-started/03-reading-guide.md` — 阅读指南（三条路径 + 全索引）

### 00 入门篇 (00-getting-started)
- [x] `01-what-is-an-agent-harness.md` — 什么是 AI Agent 与 Harness（零基础背景知识）〔A1〕
- [x] `02-cordis-framework.md` — Cordis 框架精讲：插件/上下文/事件/Waterfall/Effect 〔A2〕

### 01 总览篇 (01-overview)
- [x] `01-global-architecture.md` — 全局架构与包地图 〔A3〕
- [x] `02-turn-lifecycle.md` — 一次对话的完整生命周期（turn/step 逐步走读）〔A4〕
- [x] `03-boot-and-profiles.md` — 启动流程与 Profile/Bundle 组合 〔A5 重建实例〕

### 02 核心引擎篇 (02-core-engine，逐文件)
- [x] session/：`index.md` + `types.md` `known-event-types.md` `preparation.md` `fork.md` `repair.md` `request-header.md` `seq-ranges.md` `surface.md` `tool-history.md` 〔B1 重建实例〕
- [x] agent-loop/：`index.md` + `agent.md` `assistant-stream.md` `constants.md` `inbox.md` `runtime-context.md` 〔B2〕
- [x] tools/：`index.md` + `batch.md` `contracts.md` `events.md` `executor.md` `json-schema.md` `pipeline.md` `presentation.md` `schema.md` 〔B3〕
- [x] agent/：`index.md` + `types.md` `runtime-types.md` `dispatch.md` `consumed-work.md` `archive-admission.md` `model-selection.md` `projection.md` 〔B4〕
- [x] system-prompt/：`index.md` 〔B5 重建实例〕
- [x] scope/：`index.md` + `store.md` 〔B5 重建实例〕
- [x] executors/：`index.md` + `native.md` + `ptc.md`〔主协调者自写〕
- [x] agent-default-model/：`index.md`〔主协调者自写〕
- [x] agent-tool-presentation/：`index.md`〔主协调者自写〕
- [x] llm/：`index.md` + `types.md` `message.md` `content.md` `assembler.md` `assistant-stream.md` `call-config.md` `api-key.md` `adapter-failure.md` `error.md` `retry-policy.md` `attribution.md` `brand.md` 〔B7 重建实例〕
- [x] llm/providers.md — DeepSeek 适配器族 / pi-ai / llm-retry / token-meter〔主协调者自写〕

### 03 能力缝篇 (03-capability-seams)
- [x] `00-seam-design.md` — 能力缝设计思想（Service Definition / Provider / Consumer）〔主协调者自写〕
- [x] `01-filesystem.md` — 文件系统 + 交付物 + 工作区 〔主协调者自写〕
- [x] `02-shell-subprocess-sandbox.md` — 命令执行、子进程与沙箱 〔主协调者自写〕
- [x] `03-terminal-and-ptc.md` — 终端 + PTC 运行时（合并为一篇）〔主协调者自写〕
- [x] `05-web.md` — 网络搜索与抓取 〔主协调者自写〕
- [x] `06-browser-computer-use.md` — 浏览器与计算机操作 〔主协调者自写〕
- [x] `07-lsp-skills.md` — LSP 与技能系统 〔主协调者自写〕
- [x] `08-subagent-team.md` — 子代理与团队协作 〔主协调者自写〕
- [x] `09-jobs-workflow.md` — 后台任务与工作流 〔主协调者自写〕
- [x] `10-mcp-extensions.md` — MCP 外部工具与运行时自扩展 〔主协调者自写〕

### 04 持久化篇 (04-persistence)
- [x] `00-overview.md` — 持久化总览 〔主协调者自写〕
- [x] `01-session-log-and-backends.md` — 会话日志与存储后端 〔主协调者自写〕
- [x] `02-format-versions-and-migrations.md` — 会话格式版本与迁移 〔主协调者自写〕
- [x] `03-session-projections.md` — 会话投影 〔主协调者自写〕
- [x] `04-compaction-and-context.md` — 上下文压缩与请求上下文 〔主协调者自写〕
- [x] `05-session-query.md` — 会话检索 〔主协调者自写〕
- [x] `06-attachment-spill-storage.md` — 附件、溢出与通用存储 〔主协调者自写〕

### 05 交互与产品形态篇 (05-product-surfaces)
- [x] `00-interaction-and-approval.md` — 人机交互、审批与提问 〔主协调者自写〕
- [x] `01-web-frontend.md` — Web 前端架构 〔主协调者自写〕
- [x] `02-host-services.md` — Host 服务层 〔主协调者自写〕
- [x] `03-desktop-app.md` — 桌面应用架构 〔主协调者自写〕
- [x] `04-sdk-acp-api.md` — SDK、ACP 与远程 API 〔主协调者自写〕

### 06 精通篇 (06-mastery)
- [x] `00-testing.md` — 测试体系（快照/覆盖门/e2e）〔主协调者自写〕
- [x] `01-config-and-settings.md` — 配置体系与用户设置 〔主协调者自写〕
- [x] `02-recipe-new-tool.md` — 扩展实战：添加一个工具 〔主协调者自写〕
- [x] `03-recipe-new-llm-adapter.md` — 扩展实战：添加 LLM 适配器 〔主协调者自写〕
- [x] `04-recipe-new-package.md` — 扩展实战：添加一个包 〔主协调者自写〕
- [x] `05-architecture-patterns.md` — 架构设计模式总结 〔主协调者自写〕

## 完成标准

1. 本清单全部勾选；Phase 4 自查通过（完整性 + 抽样质量）。
2. `mkdocs build` 无致命错误。
3. 独立文档仓完成首次提交。
