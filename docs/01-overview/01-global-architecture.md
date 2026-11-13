# 全局架构与包地图

> 本章是全教程的地图页。读完你应当建立四个心智模型：
>
> 1. **一个 dsh 进程如何被组装出来** —— 启动器 → Profile/Bundle 叠层 → Cordis 插件树；
> 2. **组装进来的东西怎么分工** —— `packages/` 下 53 个包组的职责地图；
> 3. **它们靠什么通信** —— 三类事件域（Session events / Agent events / Capability events）；
> 4. **它们靠什么解耦** —— 能力缝（capability seam）的三角色模型。
>
> 本章蓝本是主仓库的 `docs/architecture.md` 与 `packages/README.md`。所有源码路径相对主仓库根目录。前置知识：TypeScript / Node.js；Cordis 框架本身见入门篇《Cordis 框架》一章或主仓库 `docs/cordis-primer.md`。

---

## 1. 总体分层架构

DeepSeek Harness（下称 dsh）是一个**全插件**的 agent harness：模型负责思考，harness 负责其余一切工程。它构建在 Cordis 插件框架之上——**没有特权核心**。模型适配器、工具注册表、会话日志、甚至 agent 循环本身，都是插件，都从同一个 `ctx`（Cordis context）上取，都可以从配置替换。

一个运行中的 dsh，是**启动时按有序叠层组合出来的插件树**。从双击桌面图标到模型吐出第一个 token，中间经过六层：

```text
      dsh --profile web ／ dsh web ／ SDK／Python 客户端 ／ dsh plugin
                          ▼  选定一个命名 profile
┌────────────────────────────────────────────────────────────────────
│ L1 启动器 Launcher
│    apps/cli 的 dsh bin —— 唯一支持的应用启动入口
│    apps/desktop + apps/desktop-host —— Electron 桌面，同样走 profile
│    python/ 运行时 —— 把 dsh CLI 打包成平台 wheel，启动 dsh --profile sdk
├─────────────────── ▼  app-boot 解析配置、逐层叠加 patch ───────────
│ L2 组合层 Composition
│    packages/boot/app-boot —— 读 cordis.yml、叠 patch 层、启动 Loader
│    packages/bundle/* —— dsh-base → web-app / headless / sdk-app / acp-app
│                          （sdk-minimal 单 bundle 自成一体，不套 base）
│    叠层顺序：profile bundles → cordis.patch.yml → home patch → --patch
├─────────────────── ▼  Loader 按最终配置装载插件树 ─────────────────
│ L3 Cordis 运行时
│    vendor/（vendored Cordis，固定源码副本）
│    插件贡献 Service ＋ 类型化事件 ＋ 可回退 effects —— 一切皆插件
├────────────────────────────────────────────────────────────────────
│ L4 核心引擎（packages/core/* ＋ packages/llm/llm）
│    ctx.sessions      append-only 会话事件日志（事实唯一来源）
│    ctx.systemPrompt  提示词段与工具 schema 组装
│    ctx.tools         作用域工具注册表 ＋ 受守卫执行管线
│    ctx.agents        Agent 接口与活体注册表（agent/* 事件）
│    ctx.agentLoop     默认循环驱动（可替换；扩展者只依赖 dsh-agent）
│    ctx.llm           模型适配器缝（消息与流词汇）
│    scope             每-agent 作用域注册原语（库，无 ctx 键）
├────────────────────────────────────────────────────────────────────
│ L5 能力缝家族（每个缝 = Service Definition + Provider + Consumer）
│    fs  shell  subprocess  sandbox  terminal  ptc-runtime  workflow
│    web  lsp  skill  subagent  team  jobs  interaction  mcp  …
│    （目录：packages/fs/ packages/shell/ packages/subprocess/ …）
├────────────────────────────────────────────────────────────────────
│ L6 产品面 Product Surfaces
│    Web GUI：packages/host ＋ packages/client ＋ packages/api
│    SDK：packages/sdk（JSON-RPC）     ACP：packages/acp（仅自动化）
│    桌面：apps/desktop-host 载入同一套 Web 资产 ＋ Node IPC 宿主
└────────────────────────────────────────────────────────────────────
```

### 1.1 L1 启动器：只有 dsh profile 能启动应用

主仓库有一条铁律（见 `AGENTS.md` 的 Application launch 一节）：**只有命名的 dsh profile 能启动受支持的 Node 应用**。包自带的 bin、demo、公开 SDK 的 argv 逃生口一律禁止；`scripts/verify-application-entrypoints.ts` 这个门禁把每个包 bin、可执行源码、根 demo、`start:web` / `dev:web` 脚本归入显式类别，并拒绝任何绕过 `dsh` 的 Node 应用路径。

出厂 profile 有五个：`web`、`headless`、`sdk`、`sdk-minimal`、`acp`，用 `dsh --profile <name>` 或短写 `dsh <name>` 选择。`plugin` 是插件管理命令；若你自己的 profile 恰好也叫 `plugin`，就得显式写 `--profile plugin`。

三个启动器共享同一条组装路径：

- **CLI**（`apps/cli`）：`dsh` bin。比如 `pnpm dsh --profile headless "task"` 就是从源码跑一个一次性任务。
- **桌面**（`apps/desktop` + `apps/desktop-host`）：Electron 以 Electron Node 模式启动私有的 Desktop Host；Host 调用**共享的 CLI profile runner** 与完整 Web 应用，窗口立即加载打包好的 Web 资产，等 boot 注入完成后在同一文档里激活 client 插件。RPC 与流由 Web 拥有，桌面载体只负责把本地页面连到已认证的 Host；Node IPC 传递 boot 注入、就绪信号、致命错误与关机。桌面拥有保留的 `$DSH_HOME/profiles/desktop`，与 npm CLI 共享产品数据但分开包、激活与锁文件。
- **Python SDK**（`python/`）：同一套应用架构。运行时 wheel 把正常的 `dsh` CLI 打包为 `deepseek-harness-sdk-runtime-<platform>-<arch>`，客户端默认以显式 Harness home 启动 `dsh --profile sdk`。Python 只暴露 profile 选择与有序 patch 文件，而不是完整的 Cordis 树。

### 1.2 L2 组合层：Profile 与 Bundle

两个核心概念（定义出自 `docs/architecture.md`）：

- **Profile（配置档）**：存放在 Harness home 的**命名组合**。它列出自己堆叠哪些 bundle、安装哪些树外插件，并保存用户自己的 `cordis.patch.yml`。
- **Bundle（捆绑包）**：Cordis 配置行及其所挂代码的**分发格式**——关键性质是，它插入的任何行，对它之上的层来说**始终可以被 patch**。

两者都在各自 `package.json` 的 `dsh` 字段里声明：`dsh.profile` 列出 profile 堆叠的 bundles；`dsh.bundle` 指向 bundle 的 patch 文件。

出厂 bundle 的堆叠关系（详见 `packages/bundle/README.md`）：

| Bundle | 作用 |
|---|---|
| `dsh-base` | `web` / `headless` / `sdk` / `acp` 的公共首层：模型适配器、工具、持久化、沙箱与审批策略、设置、凭据、遥测；还包含供 Web 与 agent 使用的 Plugin Manager |
| `dsh-web-app` | 在 base 之上加浏览器应用 |
| `dsh-headless` | 在 base 之上加无服务器的单次任务 runner |
| `dsh-sdk-app` | 在 base 之上加 SDK JSON-RPC 服务器 |
| `dsh-acp-app` | 在 base 之上加仅自动化的 ACP 服务器 |
| `dsh-sdk-minimal` | **刻意例外**：一个 bundle 拥有完整显式 SDK 树，不应用 `dsh-base` |

**叠层顺序**（作用于空的入口列表）：

```text
profile 声明的有序 bundles → 该 profile 的 cordis.patch.yml → home patch → --patch 命令行覆盖
```

一条 patch 按 id 定位某一行，**整行替换**它的全部配置，或插入新行。`preset` patch 则在该声明自己的 `config.plugins` 内部套用同样的层优先级。

YAML 同时控制 HMR：base 层启用仅配置的 `dsh-hmr`；headless、SDK、ACP 关闭它；`sdk-minimal` 干脆不含。想看你机器上真实启动的插件树：

```sh
dsh --profile web --dump-config
```

它打印的任何一行，都可以被你自己的一条 patch 替换——这是理解本架构最直接的一把钥匙。组合机制的细节在 `packages/boot/app-boot/README.md`（profiles 一节）；配置字段清单在 `docs/config-catalog.md`。

### 1.3 L3 Cordis 运行时：一切皆插件

`vendor/` 是 Cordis 框架的固定源码副本（见 `vendor/README.md`）。Cordis 做三件事：插件向共享 context 贡献 **Service**（服务）、**类型化事件**与**可回退的 effects**（注册即 effect，插件卸载时自动回收）。

对架构理解最重要的推论是：**没有需要打补丁的特权核心**。扩展 dsh = 在其他插件旁边挂一个自己的插件。整个产品的每个部件——模型适配器、工具注册表、会话日志、agent 循环——都是插件，因此每一个都能从配置替换。

Cordis 的瀑布事件语义、Service 生命周期等机制，入门篇《Cordis 框架》一章有专讲；本章只约定一句话：**瀑布监听者必须调用 `next()` 向下委托，直接 return 会短路整条链**（这是 `AGENTS.md` 明文规则）。

### 1.4 L4 核心引擎

核心包一览（出自 `docs/architecture.md` 与 `packages/core/README.md`）：

| 包 | 职责 | ctx 键 |
|---|---|---|
| `packages/core/session` | append-only 的 `SessionEvent` 日志与内存存储 | `ctx.sessions` |
| `packages/core/system-prompt` | 提示词段与工具 schema 的组装 | `ctx.systemPrompt` |
| `packages/core/tools` | 作用域工具注册表与受守卫执行管线 | `ctx.tools` |
| `packages/core/agent` | `Agent` 接口、活体注册表与 `agent/*` 事件 | `ctx.agents` |
| `packages/core/agent-loop` | 实现该接口的默认驱动 | `ctx.agentLoop` |
| `packages/core/scope` | 每-agent 作用域注册原语 | 库，无 ctx 键 |
| `packages/llm/llm` | 消息与流词汇 + 适配器缝 | `ctx.llm` |

注意 `agent` 与 `agent-loop` 的拆分：`agent` 拥有公开的 `Agent` 契约，`agent-loop` 只是它的默认实现。**扩展插件依赖 `dsh-agent`，驱动本身保持可替换**——这是第 5 节依赖纪律的头号案例。

两个贯穿全库的术语（完整定义见主仓库 `docs/glossary.md`）：

- **step（步）**：一次模型请求 + 它引发的工具执行。
- **turn（回合）**：零或多个 step——在第一条输入被认领前开启，在"没有欠着的东西"时关闭。

turn 内部的完整事件流（`agent/pre-step` → `step/start` → `agent/request` → 流式 → 工具三段瀑布 → `step/end` → `agent/turn-stopping` → `turn/end`）是下一章《一次对话的完整生命周期》的主题；核心引擎篇随后对 `session` / `agent-loop` / `tools` / `agent` / `scope` / `system-prompt` / `llm` 逐文件精讲。

### 1.5 L5 能力缝家族

L4 之下、L6 之上，是数量最多的一层：**能力缝家族**。文件系统（`packages/fs/`）、命令执行（`packages/shell/` `packages/subprocess/`）、沙箱（`packages/sandbox/`）、网络访问（`packages/web/`）、子代理（`packages/subagent/`）……每个家族都按同一个三角色模板拆包：一个包声明接口（Service Definition），若干包提供实现（Service Provider），若干包消费接口（Consumer，通常是面向模型的工具）。

"换一个 provider 就换掉整个产品行为"是这套架构的招牌戏法——第 4 节用 shell 与 fs 两个真实家族拆解它。

### 1.6 L6 产品面

最上层是四类产品形态，它们**共享同一个插件树**，只是挂载的顶层插件不同：

- **Web GUI**：`packages/host/`（宿主服务：webserver、目录选择、插件清单、产品遥测）+ `packages/client/`（浏览器半边：shell、wire、对象服务、slots、`ui-*` 插件）+ `packages/api/`（远程 BFF 与 Typert RPC 网关）。由 `dsh-web-app` bundle 挂载。
- **SDK**：`packages/sdk/` 的 JSON-RPC 协议与 TypeScript client/server，由 `dsh-sdk-app` 挂载为 stdio 服务器。
- **ACP**：`packages/acp/`，仅自动化的 Agent Client Protocol 服务器。
- **桌面**：`apps/desktop-host` 载入同一套 Web 资产——桌面不是另一套 UI，而是 Web 的载体。

一个提醒：这六层是**逻辑分层**而非目录强制。bundle 包属于 L2，但运行时它们就是插件树里的普通插件；产品面包与能力缝包同处一个 context。分层描述的是"谁组合谁"，不是 import 方向的硬边界——真正的依赖纪律在第 5 节。

---

## 2. 包组地图：53 个组

`packages/` 下每个包（scope 统一为 `@deepseek-ai/dsh-*`）**恰好属于一个组**，新包加入既有组，新组须同时更新自己的 README 与 `packages/README.md` 的分组表。以该表为准，目前共 **53 个组**。下面按主题域重排为 10 张中文表；"教程章节"列给出本教程计划讲解它的篇章名（无链接，对应教程首页的篇章划分）。

### 2.1 核心脊柱与远程 API

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `core/` | 产品 API 脊柱：会话、提示词、工具、agent 服务与具体循环 | `session` `agent-loop` `tools` `agent` `scope` `system-prompt` | 核心引擎篇各模块逐文件篇 |
| `api/` | 远程 BFF 组装与 Typert RPC 网关 | `session-controller` `gateway` `settings-controller` | 交互与产品形态篇 · SDK/ACP/API |
| `typert/` | 类型图生成、artifact 加载与运行时注册表 | `typert-registry` `typert-loader` `typert-protocol` | 交互与产品形态篇 · SDK/ACP/API |

### 2.2 模型接入与上下文

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `llm/` | LLM 能力家族：抽象服务 + provider 适配器 | `llm` `llm-deepseek` `llm-pi-ai` `llm-retry` | 核心引擎篇 · LLM |
| `context/` | 模型可见请求上下文：工作区指令、时间上下文、引用 | `working-directory` `time-context` `session-reference` | 核心引擎篇 · system-prompt（上下文家族） |
| `compaction/` | 压缩能力家族：Service Definition + 基础 provider + 命令 Consumer | `compaction` `compaction-basic` | 持久化篇 · 压缩 |

### 2.3 执行世界：文件、进程、沙箱、终端

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `fs/` | 文件系统能力家族：缝、本地实现、面向模型的文件工具与发现工具 | `fs` `fs-local` `fs-sandbox` `tool-fs` `tool-fs-search` | 能力缝篇 · fs |
| `shell/` | Bash 能力家族：执行器缝、本地实现、面向模型的工具 | `shell` `bash-local` `bash-sandbox` `tool-bash` | 能力缝篇 · shell+沙箱 |
| `subprocess/` | 子进程能力家族：Service Definition + 本地进程树 provider | `subprocess` `subprocess-local` | 能力缝篇 · shell+沙箱 |
| `sandbox/` | 进程约束缝；bwrap / Landlock / Seatbelt 后端 | `sandbox` `sandbox-local` `sandbox-policy` | 能力缝篇 · shell+沙箱 |
| `ssh/` | POSIX 远程连接，配套成对的 fs / subprocess / sandbox provider | `ssh` `fs-ssh` `subprocess-ssh` `sandbox-ssh` | 能力缝篇 · shell+沙箱 |
| `terminal/` | 持久 PTY 服务（owner 作用域会话）与本地 shell 后端 | `terminal` `terminal-bash` | 能力缝篇 · 终端 |
| `ptc-runtime/` | PTC 执行能力家族：Service Definition + 沙箱化 Node provider + PTC 模式 Consumer | `ptc-runtime` `ptc-runtime-node` | 能力缝篇 · PTC |

### 2.4 模型可见能力（工具面）

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `web/` | Web 能力家族：缝、搜索/抓取 provider、面向模型的 web 工具 | `web` `web-fetch-http` `tool-web` | 能力缝篇 · web |
| `lsp/` | LSP 能力家族：缝、通用 stdio provider、`lsp` 工具 | `lsp` `lsp-stdio` `tool-lsp` | 能力缝篇 · LSP+技能 |
| `skill/` | 技能能力家族：provider 注册表、本地 provider、面向模型的目录/加载器 | `skill` `skill-filesystem` `tool-skill` | 能力缝篇 · LSP+技能 |
| `subagent/` | 子代理能力家族：provider 注册契约与面向模型的委派工具 | `subagent` `tool-subagent` `subagent-spawn-in-process` | 能力缝篇 · 子代理+团队 |
| `team/` | 团队协作：共享任务板与面向模型的任务工具 | `agent-team` `tool-agent-team` | 能力缝篇 · 子代理+团队 |
| `jobs/` | 通用后台任务运行时与面向模型的 job 控制工具 | `jobs` `jobs-local` `tool-jobs` | 能力缝篇 · 任务+工作流 |
| `workflow/` | workflow 缝、PTC 流程引擎与面向模型的 `workflow` 工具 | `workflow` `workflow-ptc` `tool-workflow` | 能力缝篇 · 任务+工作流 |
| `mcp/` | 外部 Model Context Protocol 服务器暴露为原生工具 | `mcp-client` `mcp-manager` `mcp-resources` | 能力缝篇 · MCP+自扩展 |
| `todo/` | 面向模型的 `todo_write` 工具 | `tool-todo` | 能力缝篇 · 任务+工作流 |
| `computer-use/` | 独占的命名桌面 provider 注册 | `computer-use` | 能力缝篇 · 浏览器与计算机 |
| `browser-use/` | 独占的命名浏览器 provider 注册 | `browser-use` | 能力缝篇 · 浏览器与计算机 |

### 2.5 会话数据面（持久化）

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `session/` | 持久会话数据平面：持久化缝 + 后端、投影缝、日志标题、会话报告 | `session-persistence` `session-persistence-jsonl` `session-projection` | 持久化篇 · 会话日志与存储后端 |
| `session-query/` | 会话检索家族：逻辑语料、有界读取、谱系、语义过滤、SQLite 全文搜索 | `session-query` `session-query-sqlite` | 持久化篇 · 检索 |
| `attachment/` | 持久附件身份、校验、本地内容寻址存储 | `attachment` `attachment-local` | 持久化篇 · 附件 |
| `spill/` | spill 能力家族：存储缝、本地实现、工具结果溢出策略 | `spill` `spill-local` `spill-policy` | 持久化篇 · 压缩与溢出 |
| `storage/` | 非会话存储 hub + 后端 + 域形态 | `storage` `storage-json` `storage-sqlite` `storage-domain` | 持久化篇 · 存储后端 |
| `deliverables/` | turn 交付物：显式文件交付与记录的工作区变更 | `workspace-changes` | 持久化篇 · 投影 |

### 2.6 会话行为与策略

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `goal/` | 同会话目标的持久化与生命周期 | `goal` `goal-round-driver` `command-goal` | 能力缝篇 · 任务+工作流 |
| `plan/` | 计划协作状态，带直接入口命令与受审退出 | `plan-mode` | 能力缝篇 · 任务+工作流 |
| `preset/` | 基于 preset `cordis.yml` 的每会话 agent 组合 | `agent-preset-registry` | 总览篇 · 启动与 Profile |
| `guard/` | 循环卫生守卫：建议性重复调用提醒 + `tools/execute` 期限强制 | `repeat-tool-reminder` `timeout-policy` | 核心引擎篇 · tools（执行管线守卫） |
| `extensions/` | agent 运行时自修改：活体插件/服务检查与模型写入的挂载/卸载 | `cordis-host-runner` `tool-cordis` | 能力缝篇 · MCP+自扩展 |

### 2.7 人机交互与凭据

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `interaction/` | 人机协作平面：审批/交互缝、权限 preset、命令、ask-user 工具 | `user-approval` `permission-presets` `commands` `user-questions` | 交互与产品形态篇 · 审批与提问 |
| `credentials/` | 凭据引用与凭据记录缝 + env-over-`.env` provider + 向人提问的授权流程 | `credentials` `credentials-local` `authorization` | 交互与产品形态篇 · 审批与提问 |
| `feedback/` | 人工反馈采集与命令 | `message-feedback` `command-feedback` | 交互与产品形态篇 |
| `identity/` | 共享匿名身份 | `identity` | 交互与产品形态篇 |
| `telemetry/` | 共享 Cordis OTel 上报通道 | `otel` | 交互与产品形态篇 · Host 服务 |

### 2.8 产品面组装

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `bundle/` | 可安装的 `dsh --profile` patch 层 | `base` `web-app` `headless` `sdk-app` `acp-app` `sdk-minimal` | 总览篇 · 启动与 Profile |
| `boot/` | 共享 app bin 启动胶水 | `app-boot` `cmdline` `hmr` `config-editor` `plugin-manager` | 总览篇 · 启动与 Profile |
| `host/` | Web GUI 宿主服务、目录选择、应用启动、插件清单、产品遥测 | `webserver` `directory-picker` `frontend-static` | 交互与产品形态篇 · Host 服务 |
| `client/` | Web GUI 浏览器半边：shell、wire、对象服务、slots、`ui-*` 插件 | `connection` `modules` `store` `ui-chat` | 交互与产品形态篇 · Web 前端 |
| `sdk/` | 进程外 SDK：JSON-RPC 协议与 TypeScript 客户端/服务器 | `sdk-protocol` `sdk-client` `sdk-jsonrpc-server` | 交互与产品形态篇 · SDK/ACP/API |
| `acp/` | 仅自动化的 Agent Client Protocol 服务器 | `acp` | 交互与产品形态篇 · SDK/ACP/API |
| `workspace/` | 工作区实体 | `workspace` | 交互与产品形态篇 · Host 服务 |
| `schedule/` | Host 拥有的定时跟进 | `schedule` `tool-schedule` | 交互与产品形态篇 · Host 服务 |
| `document/` | 共享 Host Office→PDF 转换 | `office-to-pdf` | 交互与产品形态篇 · Host 服务 |

### 2.9 基础设施

| 组 | 角色 | 代表包 | 教程章节 |
|---|---|---|---|
| `experimental/` | 评估中的能力；状态与支持政策见其 README | 各 `experimental-*` 包 | 各篇章穿插提及 |
| `test-support/` | 测试基础设施（testkit、回放、Loader 冒烟） | `agent-loop-testkit` `llm-replay` `remote-mock` | 精通篇 · 测试体系 |
| `util/` | 跨组共享的低层工具与纯数据清单（`Branded<B>`、home/path 帮手、超时、保留策略、官方 bundle 清单） | `brand` `home-paths` `timeout` | 精通篇 · 架构模式总结 |

两个阅读提示：

- 每个组的 README 是该家族**权威的包清单**；每个包 README 固定覆盖用途、配置、扩展点与 Model Experience（面向模型使用者的体验说明），并带有 `## Known Limitations and Deferred Work` 一节（或使用允许清单豁免）。
- `experimental/` 组的实验状态与"是否发布/安装/可选交付"互相独立；非实验运行时包必须有产品用途或明确的分类声明（见 `scripts/product-package-policy.ts`）。

---

## 3. 三类事件域

dsh 的扩展点主要由**类型化事件**构成，用 TypeScript declaration merging 挂在 Cordis `Events` 接口上（每个事件声明带 `@mode` 与 payload `@param` 的 JSDoc——`packages/fs/fs/src/index.ts` 的 `fs/write-intent` 是标准范例）。`docs/architecture.md` 把事件划成三个域，**选错域是新手最常见的架构错误**，所以判据要背下来：

### 3.1 Session events：持久事实

**判据：这条事实必须活过一次重载。** Session events 被 append 到会话日志，并通过 `session/event` 广播给所有监听者（持久化后端据此落盘、Web 的 session-controller 据此推流、投影单元据此折叠状态）。

持久的会话事件词汇包括：`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*`。三条关键性质：

- **模型可见 ⟺ 已记录**：任何进入模型请求的内容都必须能从日志重建；新的模型可见输入必须引入新的 session 事件。这是全库最重要的不变量。
- 每条 `assistant/message` 内嵌产生它的完整紧凑计时流；`assistant/attempt` 保留失败/重试/取消/流错误的尝试但**不进入模型历史**。fork、resume、transcript、遥测、持久化全部从这些持久结算派生；活 UI 的增量来自 `agent/assistant-stream`（见下）。
- 消费者只认"当前逻辑格式"；版本迁移是相邻单步链（`vN → vN+1`），已提交的世代路径永不重命名、替换或删除——详见持久化篇《格式版本与迁移》。

### 3.2 Agent events：`agent/*`，观察与拦截在途工作

**判据：我要在"正在进行"的工作上挂一嘴。** `agent/*` 事件携带活的 `Agent` 对象，覆盖 inbox、step、status、request、validation、continuation 全程：

| 事件 | 模式 | 用途 |
|---|---|---|
| `agent/created` | serial | agent 创建完成（循环回滚失败的初始化、串行等待它之后才处理排队工作） |
| `agent/pre-step` | waterfall | **每步准入闸**：接受、改写或拒绝已认领消息；拒绝或首条声明为空 → 无 step 关闭 turn |
| `agent/request` | waterfall | 已弃用的请求前钩子：只调控制参数，不改 provider/model/effort 选择 |
| `agent/request-error` | waterfall | 请求失败（压缩、重试等消费者挂在这里） |
| `agent/assistant-stream` | emit | 进程内的 start / transient chunk / end 三帧；循环在提交 end 帧前把完整紧凑流结算为一条消息或仅日志尝试。Web 的 Session-follow 适配器是它的唯一远程消费者 |
| `agent/turn-stopping` | serial（无 `next()`） | 回合即将终止的串行通知 |
| `agent/status` `agent/error` `agent/disposed` `agent/inbox/*` | emit | 状态、错误、销毁与收件箱变化 |

注意两种模式差异：`agent/pre-step` 这类 **waterfall** 监听者必须调 `next()` 委托；`agent/turn-stopping` 是 **serial**、没有 `next()`。

### 3.3 Capability events：给缝挂策略与适配器

**判据：我要给某个能力缝附加策略或适配器，而且不想 import 循环。** 这类事件以缝的域名为前缀，代表是 `fs/*`、`tools/*`、`telemetry/*`：

- **`tools/*`**：`tools/pre-execute` → `tools/execute` → `tools/post-execute` 三段 waterfall 是工具执行管线（审批、守卫、超时、hook 桥都挂在这条链上）；`tools/change`（注册表变化）与 `tools/result`（结果广播）是 emit。
- **`fs/*`**：`fs/write-intent` / `fs/edit-intent` / `fs/delete-intent` 是**单槽决策** waterfall（详见第 4.3 节），`fs/observed` 是 emit（记录正/负观察事实）。
- **遥测侧**：`session-telemetry/record` waterfall 把脱敏后的会话记录交给唯一后端，输出直接离开进程。

其他缝也沿用同一模式：`approval/request`（审批瀑布）、`subagent/*`（子代理生命周期）、`workflow/*`、`skills/change`、`system-prompt/assemble` 等。哪些包派发/监听哪个事件，主仓库有一份生成的矩阵 `docs/event-producer-consumer.md`——查"这个事件谁发谁听"时先翻它。

**一句话总结三域**：要活过重载 → session 事件；要碰在途工作 → `agent/*`；要给某个能力挂策略 → 该缝的 capability 事件。

---

## 4. 能力缝：三角色模型

### 4.1 定义

主仓库 `docs/glossary.md` 对 **seam** 的定义值得全文引用（意译）：

> seam 是一个**可替换能力**，由三个角色组成：**Service Definition**（拥有 `ctx.<key>` 与词汇类型的 Cordis `Service`——通常是抽象类如 `ShellExecutor`，或具体注册表如 `WebRuntime`，**绝不是 TypeScript `interface`**）、一个或多个 **Service Provider**、以及一个或多个注入该服务的 **Consumer**。

三条补充规则：

- 一个包可以身兼多角，但**单一个角色不构成缝**；加一个能力意味着把三个角色一起设计出来（`docs/architecture.md`）。
- 角色是否拆包看演化节奏：**独立演化就分包**（`packages/shell` 是正典）；同一关切就同包（`dsh-user-approval` 一个包同时拥有审批缝的 Definition 与实现）。
- 术语纪律：**"缝"指完整的三角色能力**，谈论成员时叫它的角色名（Service Definition / provider / consumer）或类名、服务名、扩展点名。

### 4.2 例一：shell（命令执行）

`docs/capability-seams.md` 生成的服务图给出的完整角色表：

| 角色 | 包 | 位置 |
|---|---|---|
| Service Definition | `dsh-shell`（`ctx.shell`） | `packages/shell/shell/` |
| Service Provider | `dsh-bash-local` / `dsh-bash-sandbox` / `dsh-pwsh-local` / `dsh-pwsh-sandbox` | `packages/shell/` |
| Consumer | `dsh-tool-bash` / `dsh-tool-pwsh`（注册到 `ctx.tools`）、hook 桥 `dsh-hooks-claude-code` / `dsh-hooks-codex` | `packages/shell/`、`packages/experimental/` |

配角：本地执行器通过 `ctx.subprocess` 缝派生进程；沙箱执行器通过 `ctx.sandbox` + `ctx.sandboxPolicy` 约束；`ctx.shellEnv` 供给受管的 `DSH_*` 环境；后台句柄的 id/所有权/轮询归 `ctx.jobs`——执行器因此保持与会话无关。

Service Definition 是一个真正的抽象类（摘自 `packages/shell/shell/src/index.ts`）：

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    shell: ShellExecutor
  }
}

/**
 * Abstract bash execution service. Subclass, implement the abstract methods,
 * and load the subclass as a plugin — it registers as `ctx.shell` (one
 * implementation per context; loading a second throws, which is cordis'
 * standard duplicate-service behavior).
 */
export abstract class ShellExecutor extends Service {
  constructor(ctx: Context) {
    super(ctx, 'shell')
  }
  // ...
}
```

两个值得注意的设计事实（出自 `packages/shell/README.md`）：

- **每个组合恰好挂一个执行器实现**：win32 层把 POSIX 行换成 pwsh 行；同时挂两个会在加载时因 Cordis 的重复 Service 注册**响亮地失败**——错配置立即暴露，绝不静默择一。
- 模型工具（`tool-bash` 等）只认 `ctx.shell` 缝，"沙箱化、远程或 PowerShell 执行器替换 bash-local 时，它们一行都不用改"。

### 4.3 例二：fs（文件系统）

角色表（出自 `packages/fs/README.md`）：

| 角色 | 包 | 说明 |
|---|---|---|
| Service Definition | `dsh-fs`（`ctx.fs` + `fs/*` 事件） | 一个执行世界的文件契约 |
| Service Provider | `dsh-fs-local` / `dsh-fs-sandbox` / `dsh-fs-ssh` | 本地 / 按沙箱模式设栏 / SSH 远程 |
| Consumer | `dsh-tool-fs`（`read` `read_image` `write` `edit` `apply_patch`）、`dsh-tool-str-replace-editor` | 注册到 `ctx.tools`，注入 `ctx.fs` |
| 伴随策略插件 | `dsh-fs-observation-policy` | "先读后写"门，挂在 `fs/*` 事件上 |

fs 的 Service Definition 同时声明了能力事件（摘自 `packages/fs/fs/src/index.ts`）：

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    fs: FileSystem
  }

  interface Events {
    /**
     * Single-slot decision for the next FileSystem.writeText. Calling
     * `next()` yields the bare provider's unconditional write; the first listener
     * that returns an intent owns the decision rather than composing with peers.
     * @mode waterfall
     */
    'fs/write-intent'(target: FsTarget, actor: object | undefined,
      next: () => FsWriteIntent | undefined | Promise<FsWriteIntent | undefined>):
      Promise<FsWriteIntent | undefined>
    // 'fs/edit-intent' / 'fs/delete-intent' 同为单槽决策 waterfall …
    /** @mode emit */
    'fs/observed'(target: FsTarget, observation: FsObservation, actor: object | undefined): void
  }
}
```

这里有一个**瀑布语义的刻意变体**：`fs/write-intent` 是"单槽决策"——调 `next()` 得到的是裸 provider 的无条件写入；**第一个返回 intent 的监听者独占决策权**，而不是与同侪组合。这和 `agent/pre-step` 那种层层改写的瀑布不同，读代码时别混。

策略插件的角色也值得记住：`fs-observation-policy` **是插件，不是工具注入的服务**——把它摘掉，工具得到的是裸 provider 的无条件写入行为，而不是功能损坏。它和 `fs-sandbox` 的模式栏可以叠加。

### 4.4 为什么"换一个 provider 就换掉整个产品行为"

`docs/architecture.md` 的原话是：**"Seams are why one provider swap changes the whole product."（缝就是为什么换一个 provider 就换掉整个产品。）** 机制上有两条：

**第一，消费者只认 Service Definition。** `tool-bash` 注入的是 `ctx.shell`，`tool-fs` 注入的是 `ctx.fs`；它们对 provider 是谁零感知。于是 profile 层换一行配置（换成 `bash-sandbox`、`pwsh-local` 或远程执行器），模型看到的工具名、参数、行为面完全不变，变的只是背后那个执行世界。

**第二，配对的 provider 共享一个执行世界。** 文件系统 provider 与子进程 provider 刻意共享同一个"执行世界"概念——把本地三件套（`fs-local` + `subprocess-local` + `sandbox-local`）换成 SSH 三件套（`fs-ssh` + `subprocess-ssh` + `sandbox-ssh`，全部配对挂在 `ctx.ssh` 拥有的同一条已认证 OpenSSH 连接上），**Bash、PTY、LSP、文件工具就整体搬到了远程机器上**，不需要为任何 provider 写分支代码。反过来说，如果一个工具绕过缝直接 import 了某个 provider 包，这个戏法就失效——这正是第 5 节依赖纪律要禁止的事。

子代理（`ctx.subagents`）是同一思想的极端案例：同一个接口后面，provider 可以是本进程新起的子 agent，也可以是另一个产品里的一次委派 turn（`subagent-acp` / `subagent-codex` / `subagent-claude-code` / `subagent-dsh-sdk`）。

---

## 5. 依赖纪律

包图的权威版本是生成的 `docs/module-graph.md`（`pnpm run gen-module-graph` 重新生成，CI 有新鲜度门禁）。它只画 **peer 依赖**——一条边 `a --> b` 意味着 a 把 b 声明为 peer，即 **a 要求与 b 共享同一个运行实例**（这正是 Cordis 服务的语义；普通运行时依赖与仅开发依赖不画）。

`packages/README.md` 的 Dependencies 一节给出三条硬规则：

1. **扩展插件只依赖 Service Definition，永不依赖具体 provider。** 例证（可到 `docs/module-graph.md` 的表里验证）：`tool-subagent` 的 peer 列表里有缝定义 `subagent` 和核心脊柱（`agent` `session` `system-prompt` `tools`），但没有任何 `subagent-*` provider；`tool-bash` 消费 `ctx.shell` 缝，peer 里没有 `bash-local`。
2. **`dsh-agent-loop` 可替换；UI、hook、工具插件一律用 `dsh-agent`。** `docs/capability-seams.md` 对 `ctx.agentLoop` 的注记原话："extension packages depend on dsh-agent events and services, not on this package." 连默认驱动都按"实现可换"对待。
3. **组合 bundle 可以依赖脊柱插件。** 组合层（`packages/bundle/*`）的职责就是"把正确的插件树拼出来"，所以 `dsh-headless` 的 peer 列表里合法地出现 `agent` `session` `fs` `llm` `subagent` `session-query` `user-approval` `permission-presets` `sandbox-policy` 一整排脊柱与缝定义。

配套的两条工程约束：

- **角色拆分跟随演化节奏**：能力家族在角色独立演化时拆成 Definition / Provider / Consumer 三包；同一关切时允许一个包身兼多角（如 `dsh-user-approval`）。
- **workspace 依赖写法**：DSH 包之间用 `workspace:*`，vendor/native 用 `workspace:~`（`AGENTS.md` 约定；发布序列规则见 `.agents/notes/implemented/process/2026-08-10-npm-release-sequences.md`）。

一个自检问题："我要新写的这个插件，卸掉它之后产品是否仍按缝的语义工作？"如果它依赖了某个具体 provider 的存在，答案就是否——把依赖改到 Definition 上。

---

## 6. 「新行为放哪里」速查表

`docs/architecture.md` 的 "Where new behavior goes" 一表是全库扩展点的索引。原表有一句总纲：**新行为挂到文档化的扩展点上；改循环本身（agent-loop）就必须同步更新这张表**——即"改 loop"永远是最后选项，且要付文档税。以下为中文整理：

| 目标 | 机制 |
|---|---|
| 新增模型 provider | 在 `ctx.llm` 上注册其适配器 |
| 新增面向模型的能力 | 在 `ctx.tools` 上注册；其 schema 自动加入提示词组装 |
| 给单个会话不同的能力集 | 组合 agent preset；其中的 service 行需要 `isolate` realm |
| 新增 shell 执行 | 注册一个 `ctx.shell` 后端；本地实现通过 `ctx.subprocess` 派生进程 |
| 新增持久终端执行 | 注册 `ctx.terminals` 后端，并加装 `dsh-tool-terminal` |
| 新增人工命令（slash 命令） | 在 `ctx.commands` 上注册；不经模型回合直接派发 |
| 管理后台任务 | 在 `ctx.jobs` 上注册；`job_*` 工具读取或停止任务 |
| 从外部 webhook 启动会话 | 在 `ctx.webhookRuntime` 注册受信任规则，并挂载 provider 适配器 |
| 新增文件系统访问或策略 | 注册一个 `ctx.fs` provider，或监听 `fs/*` 事件 |
| 约束派生进程 | 使用 `ctx.sandbox` 后端；消费者在派生前包装 argv |
| 拦截请求、工具或回合 | 用对应的 `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 可终止回合 |
| 新增模型可见上下文 | 调 `agent.inject()`；它落入下一个被准入的请求 |
| 新增 UI 或编辑器集成 | 驱动 `ctx.agents`，并从 `session/event` 渲染 |
| 新增 Web Client Chat 节点 | 注册 `ConversationNodeDefinition` + 键控渲染器 |
| 新增持久会话状态 | 扩展 `SessionEventMap`；从日志渲染与回放 |
| 生成会话标题 | 注册唯一的 `ctx.sessionTitle` provider |
| 管理同会话目标 | 使用 `ctx.goals`；通过 `agent/*` 持续推进 |
| 在回合边界 fork | `ctx.agents.create({ sessionId, seed, inheritedEventCount, meta: { isSeeded: true } })`——只有 agent-loop 发布的会话会持久化 |
| 用新后端存会话 | 实现 `SessionPersistence`（`create`/`open`/`stat`/`list`/`export`），基于共享 handle 脚手架 |
| 把注册限定到单个 agent | 使用该 agent 的 `agent.ctx` |

配套阅读：扩展手册 `docs/cookbook/extension-cookbook.md` 把特性映射到能力，并索引了添加包、添加工具、添加 LLM 适配器、添加设置页四篇分步指南——精通篇的扩展实战三章将带你走完其中三条。

---

## 7. 源码阅读入口

主仓库文档分四类：**手写总纲**、**手写子系统参考**、**生成图**（脚本产物，勿手改）与**包 README**。建议按下面的顺序建立第一遍心智模型：

**第一步：总纲与词汇（半天）**

1. `docs/architecture.md` —— 本章蓝本，全文精读；先只读 Profiles/bundles、Events、Turn flow、Capability seams 四节，"Where new behavior goes" 当字典。
2. `docs/glossary.md` —— 很短；seam / scope / turn / step / round / exchange 的正典定义都在，避免被社区同义词带偏。
3. `docs/cordis-primer.md`（或 `docs/cordis-tutorial/`）—— 不懂 Cordis 先补这一课，重点看瀑布语义与 Service 生命周期。

**第二步：从组 README 到包 README（一天）**

4. `packages/README.md` —— 本章第 2 节的原文；用它定位组。
5. `packages/bundle/base/README.md` —— 默认产品组合（`dsh-base`）挂了什么，是"最小完整样本"。
6. 逐组读 `packages/<group>/README.md`，再跳感兴趣的 `packages/<group>/<pkg>/README.md`。每个包 README 固定含配置、扩展点与 Model Experience，可直接当 API 文档用。

**第三步：子系统参考与生成图（两天）**

7. `docs/subsystems/` 下的手写参考：`core.md`（逐包循环地图与 Agent 句柄）、`session.md`（事件词汇与派生历史）、`tools.md`、`system-prompt.md`、`scope.md`、`llm-streaming.md`、`shell.md`、`filesystem.md`、`sandbox.md`、`subagent.md`、`agent-team.md`、`working-directory.md`、`boot.md`、`conversation.md`。
8. 三张生成图（改内容要跑脚本再生成，别手编）：
   - `docs/capability-seams.md` —— 全部 ctx 服务的 Definition/Provider/Consumer 角色表；
   - `docs/module-graph.md` —— peer 依赖图（共享实例语义）；
   - `docs/event-producer-consumer.md` —— 每个事件谁派发、谁监听。

**第四步：动手验证**

9. `pnpm dsh --profile web --dump-config` —— 打印你机器上真实的插件树；对照 `apps/cli/composition.md`（每个出厂 profile 的精确组合）。
10. `docs/cookbook/` —— `extension-cookbook.md` + 添加包/工具/LLM 适配器/设置卡四篇；边读边在测试分支上做一个最小插件。
11. 工程约束两篇：`docs/development.md`（TypeScript 项目布局、源码面与产物面分离）、`docs/testing.md`（测试政策与快照要求）。
12. 历史决策：`.agents/notes/implemented/`（已归档笔记是冻结的历史依据，不是现行权威——查"当初为什么"用它，查"现在是什么"永远回到 README 与 docs）。

---

## 小结

把本章压缩成五句话：

1. **一个 dsh 进程 = 启动器选定 profile，app-boot 按四层顺序叠 patch，Loader 装出 Cordis 插件树**；启动后没有特权核心，一切皆插件、皆可从配置替换。
2. **`packages/` 的 53 个组**按"脊柱 → 模型 → 执行世界 → 工具 → 数据面 → 交互 → 产品面 → 基础设施"分工；每组 README 是权威包清单。
3. **事件分三域**：要活过重载进 session 日志；要观察/拦截在途工作挂 `agent/*`；要给能力挂策略挂该缝的 capability 事件。
4. **能力缝 = Service Definition + Service Provider + Consumer 三角色**；消费者只认 Definition，配对 provider 共享执行世界——所以换一个 provider 就换掉整个产品行为。
5. **依赖纪律只有一条主线**：扩展插件依赖 Service Definition（含 `dsh-agent`），组合 bundle 才可以依赖脊柱。

下一章《一次对话的完整生命周期》把第 1.4 节的 turn/step 流程图逐帧展开；之后核心引擎篇进入逐文件精读。
