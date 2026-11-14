# 第 17 章 能力缝设计思想

> Service Definition / Provider / Consumer


> 本章是能力缝篇的总纲。读懂它，你就理解了 dsh 为什么"换一个 provider，整个产品行为随之改变"。

## 什么是能力缝

**能力缝（capability seam）** 是 dsh 中"可替换能力"的标准形态：一个可以整块换掉的产品能力，通过 Cordis 服务暴露，模型或宿主通过统一接口使用它，而不知道背后是谁在实现。

一个完整的缝由**三个角色**组成，缺一不可：

| 角色 | 职责 | 在代码里的位置 |
|------|------|----------------|
| **Service Definition**（服务定义） | 声明接口与契约，注册为 `ctx` 上的服务 | 定义包（如 `packages/fs/fs`） |
| **Service Provider**（服务提供者） | 实现该接口，注册到同一个键上 | 实现包（如 `packages/fs/fs-local`、`fs-ssh`） |
| **Consumer**（消费者） | 通过接口使用能力，常常是模型可见的工具 | 消费包（如 `packages/fs/tool-fs`） |

```text
        Service Definition                Service Provider(s)
   ┌──────────────────────┐        ┌────────────────────────────┐
   │  packages/fs/fs      │        │  fs-local   (本机文件系统) │
   │  声明 ctx.fs 接口     │◄───────│  fs-sandbox (沙箱围栏)     │
   │  （契约与类型）       │ 注册   │  fs-ssh     (远程 SSH)     │
   └──────────┬───────────┘        └────────────────────────────┘
              │ 只依赖接口
   ┌──────────▼───────────┐
   │  Consumer            │
   │  packages/fs/tool-fs │  ← 模型可见的文件工具
   │  （读写/编辑/列表）  │
   └──────────────────────┘
```

三个关键纪律（来自 `packages/README.md`）：

1. **一个包可以身兼多角色，但单独一个角色不构成缝。** 只有定义、没有提供者，接口永远无人实现；只有实现、没有消费者，能力进不了产品。
2. **扩展插件只依赖 Service Definition，绝不依赖具体 provider。** 这是依赖方向纪律：`tool-bash` 依赖 `ctx.shell` 接口，不 import `bash-local`。因此把 `bash-local` 换成 `bash-sandbox` 或远程 SSH 执行器，工具代码一行不改。
3. **角色各自独立演化时才拆分。** 如果定义与实现永远同生共死，就不必拆包（如 `ctx.ssh` 由单一包拥有连接所有权与三个配对远程 provider）。

## 为什么"换 provider = 换产品"

同一个 `ctx.fs` 接口有三个实现：`fs-local`（直接本机 IO）、`fs-sandbox`（按沙箱模式围栏写操作）、`fs-ssh`（经 SSH 到远程主机）。模型面对的 `tool-fs` 工具完全不变——它只调 `ctx.fs`。

于是同一个产品形态获得了完全不同的安全/部署性质：

```text
fs-local:    模型 → tool-fs → ctx.fs → 本机磁盘（无围栏）
fs-sandbox:  模型 → tool-fs → ctx.fs → Landlock/bwrap/Seatbelt 围栏内写
fs-ssh:      模型 → tool-fs → ctx.fs → SSH 通道 → 远程主机
```

更妙的是**缝与缝共享一个执行世界**：`ctx.subprocess` 的 provider（local 或 ssh）决定 bash、PTY、LSP 全部跟着走——`bash-local`、`terminal-bash`、`lsp-stdio` 全部通过 `ctx.subprocess` 孵子进程。指向远程沙箱，Bash/PTY/LSP 一起搬家，不需要为任何 provider 写分叉代码。

## 全库缝地图（节选）

dsh 运行时上有几十个缝。下表挑出模型与产品最直接依赖的（完整图谱见主仓库 `docs/capability-seams.md`，由脚本生成）：

| ctx 键 | 缝 | 定义包 | 已知实现 | 直接消费者 |
|--------|-----|--------|----------|------------|
| `ctx.llm` | LLM 适配器 | `packages/llm/llm` | `llm-deepseek`、`llm-pi-ai`、`llm-replay` | `agent-loop`、`compaction-basic` |
| `ctx.fs` | 文件系统 | `packages/fs/fs` | `fs-local`、`fs-sandbox`、`fs-ssh` | `tool-fs` |
| `ctx.shell` | Bash 执行器 | `packages/shell/shell` | `bash-local`、`bash-sandbox`、`pwsh-local` | `tool-bash`、`tool-pwsh`、hooks 桥 |
| `ctx.subprocess` | 子进程 | `packages/subprocess/subprocess` | `subprocess-local`、`subprocess-ssh` | bash 执行器、PTY、LSP、外部 subagent 后端 |
| `ctx.sandbox` | 进程沙箱 | `packages/sandbox/sandbox` | `sandbox-local`、`sandbox-ssh` | `bash-sandbox`、`terminal-bash` |
| `ctx.web` | 网络访问 | `packages/web/web` | `web-search-deepseek`、`web-fetch-http` | `tool-web` |
| `ctx.subagents` | 子代理 | `packages/subagent/subagent` | `spawn-in-process`、`fork-in-process`、`acp`、`codex`、`claude-code`、`dsh-sdk` | `tool-subagent` 等 |
| `ctx.skills` | 技能 | `packages/skill/skill` | `skill-filesystem`、`skill-office` … | `tool-skill` |
| `ctx.terminals` | 持久终端 | `packages/terminal/terminal` | `terminal-bash` | `tool-terminal` |
| `ctx.sessionPersistence` | 会话持久化 | `packages/session/session-persistence` | `session-persistence-jsonl` | `agent-loop`、`session-query` … |
| `ctx.approval` | 审批 | `packages/interaction/user-approval` | 监听器即应答者 | `tools`、`tool-bash`、`acp` |
| `ctx.userQuestions` | 向人提问 | `packages/interaction/user-questions` | UI 前端注册 | `tool-ask-user` |
| `ctx.compaction` | 上下文压缩 | `packages/compaction/compaction` | `compaction-basic` | （自消费，后端驱动） |
| `ctx.workflowEngine` | 工作流引擎 | `packages/workflow/workflow` | `workflow-ptc` | `tool-workflow` |
| `ctx.jobs` | 后台任务 | `packages/jobs/jobs` | `jobs-local` | `tool-jobs`、bash/PTY 工具 |

两类特殊形态值得注意：

- **互斥命名注册**：`ctx.browserUse` 与 `ctx.computerUse` 每个服务实例只允许一个 provider 占一个名字，provider 自带工具与浏览器资源，服务本身没有通用操作 API——因为不同浏览器后端的能力差异太大，强行统一接口反而失真。
- **监听器即提供者**：`ctx.approval` 的"实现"是瀑布事件 `approval/request` 上的监听器；没有任何监听器时请求**失败关闭（fail closed）**为 `unavailable`。策略以监听器形式叠加。

## 设计一个新缝的检查清单

1. **先写三个角色，而不是一个接口**：定义（契约+类型）、至少一个可用 provider、一个进入模型的 consumer（工具或命令）。
2. **接口最小化**：如 `ctx.lsp` 只暴露四种归一化操作，明确"不提供协议逃生门"——后端把私有协议翻译进归一化请求，而不是让接口长满 provider 特有的口子。
3. **配置显式化**：provider 间差异用 `Config` 字段表达（如沙箱模式、远端地址），禁止在代码里硬编码部署差异；配置缺失要"响亮失败"。
4. **策略与机制分离**：`ctx.sandboxPolicy` 是模式+根目录的唯一家；执行器家族（bash、fs）都读它，保证不会围到不同的根。
5. **失败语义**：定义包写清故障归属、是否可重试、对模型的呈现；审批缺席=拒绝，而不是放行。
6. **文档三件套**：README 写 purpose / configuration / extension points / Model Experience；能力缝图谱由 `pnpm run gen-doc-graphs` 自动收录（角色分类在 `scripts/gen-doc-graphs.ts` 有完备性守卫）。

## 与其他篇章的关系

- 核心引擎篇的 `ctx.llm`、`ctx.tools` 是两个最重要的缝：一个换模型，一个给工具执行插策略。
- 持久化篇的 `ctx.sessionPersistence` 展示了"缝 + 单一官方后端 + 严格格式纪律"的形态。
- 精通篇的扩展实战会亲手走一遍三角色流程。
