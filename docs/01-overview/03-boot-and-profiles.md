# 启动流程与 Profile/Bundle 组合

DeepSeek Harness 的所有受支持 Node 应用都由同一个启动器 `dsh` 拉起：`dsh web`（浏览器 GUI）、`dsh --profile headless "task"`（一次性任务）、`dsh --profile sdk`（JSON-RPC stdio）、`dsh --profile sdk-minimal`、`dsh --profile acp`。它们不是五个可执行文件，而是**同一个二进制按名字装配出的五份插件树**。装配的单位是 profile（目录）和 bundle（可安装的补丁层），装配的语言是 YAML patch。

本文以 `dsh web` 回车为起点，走完从 argv 解析到插件树就绪的完整序列，再展开 Profile / Bundle / patch / HMR 四个机制，最后说明"只有 dsh profile 能启动应用"这条纪律的设计动机。所有路径均相对主仓库根目录。

---

## 1. 从 `dsh web` 回车到插件树就绪

### 1.1 启动时序图

```text
dsh web --no-open
  │
  ▼
[1] runCli()                          apps/cli/src/bin.ts
  │   getDshRuntimeVersion() → parseDshArgs()
  │   args.ts 把首词 web 展开为 --profile web（plugin 除外）
  │   启动器只解析自己的旗标（--profile/--patch/--dump-config/...），
  │   第一个不认识的 token 之后全部原样交给被启动的应用
  ▼
[2] loadLayeredEnv('dsh')             分层 .env：继承环境 > 调用目录 .env > $DSH_HOME/.env
  │   PATH / DSH_* / XDG_* 等启动敏感变量拒绝从文件读取
  ▼
[3] runProfile()                      apps/cli/src/profile-boot.ts
  │   installProxyFromEnvironment()   在任何插件挂载之前安装代理
  │   注册 SIGTERM(exit 0) / SIGINT(exit 130)、installFailLoud('dsh')
  ▼
[4] composeProfile() → prepareProfile()
  │   4a  resolveProfileDir('web') → $DSH_HOME/profiles/web
  │   4b  目录缺失 → initProfile() 写入 package.json（含 dsh.profile.bundles）、
  │       空的 cordis.patch.yml 模板、pnpm-workspace.yaml（hoisted 链接器）
  │   4c  loadProfileDirectory()：按 dsh.profile.bundles 顺序逐个解析 bundle——
  │       先从 dsh 安装锚解析包，再落到 profile 的 node_modules；
  │       读包内 dsh.bundle.patch → 有序 patch 文件列表；平台与版本不兼容
  │       或读不了的 bundle 跳过并记入 skippedBundles（启动时一次性打印）
  │   4d  解析 profile 自己的 cordis.patch.yml（用户层）
  │   4e  重写空根文件 cordis.yml 为 []（注释声明：整棵树由 patch 组合）
  ▼
[5] createRuntimeResolution()         安装依赖闭包 + 各 bundle 依赖闭包
  │   生成不可变运行时解析表，先于任何插件 import 安装进 Node 的
  │   ESM / CommonJS 解析器
  ▼
[6] readProfilePatches()              叠层定序：
  │   bundle 层（bundles 顺序）→ profile cordis.patch.yml →
  │   $DSH_HOME/cordis.patch.yml → --patch 覆盖层 → DSH_TELEMETRY_DISABLED 开关
  ▼
[7] boot()                            Loader 挂载空根 Include，按序 applyEntryPatches
  │   host 插件先 provide：profileContext、启动环境快照、
  │   PluginPackages（运行时解析）、cmdlineArgs / appExit / appReady
  │   各行按服务可用性激活；!!js 表达式在注入就绪后求值
  │   结算后按 required 名单审计（agent-loop、webserver、connection、...）：
  │   可选行失败仅告警，必需行失败 → 整树 disposal + exit 1
  ▼
[8] appReady.commit()                 FiberState.ACTIVE 且 ctx.get('loader') 就绪
  ▼
[9] web-app 胶水插件                   packages/bundle/web-app/src/index.ts
  │   注入 connection 服务后等待 Loader 结算 + 启动审计
  │   webserver 行监听 127.0.0.1:3080（--port/--host/--tls-* 可改）
  │   console.log('dsh web: ' + authenticatedUrl)  →  打开默认浏览器
  ▼
插件树就绪；浏览器持启动令牌换取签名 Cookie，进入会话页面
```

### 1.2 关键步骤逐条说明

**步骤 1 — CLI 解析。** `apps/cli/src/bin.ts` 的 `runCli()` 是唯一入口。参数语法在 `apps/cli/src/args.ts`：启动器解析完自己的旗标后就停止，第一个不认识的 token 开启"内层参数"，这些参数通过 `ctx.cmdlineArgs` 交给树里的应用插件自行解析（见 `packages/boot/cmdline/src/index.ts`）。所以 `dsh --profile web --help` 打印的是 web 应用自己的帮助，`dsh --profile headless "run the tests"` 里的任务文本属于 headless 应用。

**步骤 2 — 环境分层。** `loadLayeredEnv('dsh')` 按优先级合并三层：进程继承环境最高，其次是调用目录的 `.env`，再是 `$DSH_HOME/.env`。能决定进程如何启动的变量（`PATH`、`DSH_*`、`XDG_*` 等）一律拒绝从文件读入；四个代理变量只接受来自 Harness home 的文件（调用目录的 `.env` 常随克隆而来，不可信）。快照在步骤 7 通过 `DSH_LAUNCH_ENVIRONMENT_KEY` 提供给所有插件，保证整个进程看到同一份不可变环境。

**步骤 3 — 启动前置。** 代理必须在任何插件挂载前安装，因为 Node 的 fetch 自己不会读代理环境；快照而非 `process.env` 是为了让 `.env` 层声明的代理也生效。`installFailLoud('dsh')` 为未处理 rejection / 未捕获异常写一条带标签的诊断，等待释放钩子后以退出码 1 结束。

**步骤 4 — profile 装配。** 这是组合的核心，见 §2 / §4。注意空根文件 `cordis.yml` 每次启动都会被重写为 `[]`：整棵树是 patch 叠加的产物，如果 Loader 的树写回把组合后的行固化进根文件，下次启动就会重复插入所有 bundle 行。这个文件存在的唯一原因是 Loader 需要一个真实的 include 根来锚定 `baseUrl`。

**步骤 5 — 运行时解析。** `packages/boot/app-boot/src/profile.ts` 的 `createRuntimeResolution()` 从安装锚（`apps/cli` 的 `package.json`）做广度优先依赖遍历（含 peer / optional），再把每个选中 bundle 的依赖闭包叠加，产出一张不可变包表。它在任何 profile 行挂载之前装进 Node 内部解析器，于是 dsh 安装、bundle 携带的包与 profile 本地 `pnpm add` 的包共享同一个模块实例——尤其是 cordis 本体不允许出现两份。

**步骤 6 — 补丁定序。** `packages/boot/app-boot/src/profile-context.ts` 的 `readProfilePatches()` 是层序的唯一权威：

```ts
export function readProfilePatches(binName: string, context: ProfileContext, initialProfile?: Profile): PatchOptions[] {
  const patches = structuredClone([
    ...readBundlePatches(binName, context, initialProfile),
    ...(initialProfile?.patches ?? loadOptionalPatches(binName, context.patchPath) ?? []),
    ...(loadOptionalPatches(binName, join(context.home, PROFILE_PATCH_FILENAME)) ?? []),
    ...context.overlays,
  ])
  const telemetryPatch = resolveTelemetryPatch(context.telemetryDisabledEnv,
    composeEntries([patches]).some(row => row.id === TELEMETRY_ROW_ID))
  if (telemetryPatch !== undefined) patches.push(telemetryPatch)
  return patches
}
```

注意 home 层（`$DSH_HOME/cordis.patch.yml`）排在 profile 层之后，因此**机器级偏好覆盖 profile 级修改**。末尾的遥测开关只在组合结果里存在 `session-telemetry-otel` 行且 `DSH_TELEMETRY_DISABLED` 非空时生成一条 `disabled: true`；任何非空值（包括 `'0'`、`'false'`）都视为关闭——隐私开关宁可错关，不可错开。

**步骤 7 — boot 与 required 审计。** `boot()` 挂载空根 Include，把步骤 6 的补丁依序应用；host 插件在配置树挂载前提供 `profileContext`、启动环境、`PluginPackages` 与 `provideCmdline` 三件套。启动失败策略区分可选行与必需行：可选行失败告警后继续；必需行（`agent-loop`、`tool-executor-native`、`tool-executor-ptc`、`webserver`、`web-runtime`、`modules`、`connection`、`headless-runner`、`acp`、`sdk-jsonrpc-server`）无法激活则整树 disposal 后退出码 1。必需名单是全局的，覆盖共享 Agent 执行、应用端点与 Web 启动三块。

**步骤 8–9 — 就绪信号。** `runProfile()` 在 boot 结算、Fiber 处于 `ACTIVE` 且 loader 存在时提交 `appReady`。web-app 的胶水插件（`packages/bundle/web-app/src/index.ts`）注入 `connection` 服务后，先等 `loader.await()` 再跑一次启动审计，确认 `webServer` 与 `connection` 都在，才打印带进程令牌的 `dsh web: <url>` 并尝试打开浏览器。就绪信号不会在半启动的树上提前发出——可选插件失败不抑制就绪，必需失败或中途 disposal 则完全不发布。

### 1.3 启动失败策略速查

`packages/boot/app-boot/README.md` 的失败矩阵是权威；以下是常查的几行。"停止启动"指 disposal 已挂载插件并以非零码退出、不报告就绪；"继续"指保留成功的插件。

| 失败模式 | 可选行（启动时） | 必需行（启动时） | 后续配置 HMR |
|---|---|---|---|
| 根配置缺失 / 损坏 / 含非法条目 | 停止启动 | 停止启动 | 拒绝非法 patch，不改运行中的配置 |
| 模块 import 失败或求值抛错 | 告警；继续 | 停止启动 | 报错；保留成功兄弟行，修正后可激活 |
| 插件 config schema 校验失败 | 告警；继续 | 停止启动 | 新条目保持未激活；旧条目保留原实例 |
| `config` / `disabled` 的 `!!js` 求值抛错 | 告警；继续 | 停止启动 | 报求值错误而非视为禁用 |
| 同步 `apply()` 抛错 | 告警；继续 | 停止启动 | 报错；修正后的配置可激活 |
| 注入的服务不可用 | 告警；等待依赖 | 停止启动 | 继续等待；提供者出现后激活 |
| HTTP 端口绑定失败 | 告警；无此端点继续 | 停止启动 | 进程继续，端口修正后恢复 |
| `apply()` 返回 Promise 之外的游离异步工作产生 unhandled rejection | 致命：disposal + 非零退出 | 同左 | 同左（无论行 id） |
| 进程生命周期内同步回调抛未捕获异常（stream `data` 监听器、timer） | 致命 | 致命 | 致命 |
| 条目缺席或显式禁用 | 忽略 | 忽略 | 不激活；无必需启动审计 |

CLI 在必需失败时打印一次汇总消息，并保存完整启动诊断（`apps/cli/reference/README.md` 的 startup diagnostics）后以退出码 1 退出。等待服务的条目报为警告而非失败——提供者稍后出现即可激活；后续 HMR 不重跑必需启动审计、也不整体回滚更新。

---

## 2. Profile 与 Bundle：两个概念

| | Profile | Bundle |
|---|---|---|
| 本质 | `$DSH_HOME/profiles/<name>` 下的一个目录 | 一个 npm 包 |
| 声明位置 | 目录里 `package.json` 的 `dsh.profile.bundles` | 包 `package.json` 的 `dsh.bundle.patch` |
| 内容 | 有序 bundle 列表 + 自己的 `cordis.patch.yml` 用户层 + pnpm 管理的外来依赖 | 一份或多份有序 patch 文件（YAML 数组） |
| 谁拥有它 | 用户/部署可自由修改、patch、增删 bundle | 包作者维护，用户通过上层 patch 覆盖 |
| 是插件行吗 | 不是 | 不是——bundle 只贡献 patch 层，不参与行准入 |

一句话区分：**profile 回答"装了哪些层"，bundle 回答"这一层放了哪些行"**。`dsh --profile web` 启动时读 profile 的 bundle 列表，逐个取出 bundle 的 patch 文件，按顺序叠在一个空根上。

profile 目录里的 `package.json` 关键字段（摘自 `packages/bundle/base/README.md` 的最小自定义 profile 示例）：

```json
{
  "name": "my-profile",
  "private": true,
  "dsh": {
    "profile": {
      "bundles": ["@deepseek-ai/dsh-base"]
    }
  }
}
```

bundle 包的关键字段（`dsh.bundle.patch` 可以是单文件，也可以是有序文件列表；`dsh.bundle.platforms` 可选，声明支持的平台，如 `["darwin"]`，不匹配的宿主整层跳过并保留选择）：

```json
{
  "dsh": {
    "bundle": {
      "patch": ["./cordis.patch.yml", "./presets/standard.patch.yml"]
    }
  }
}
```

两种字段的校验分别在 `packages/boot/app-boot/src/profile.ts`：`bundlePatchFiles()` 验证 `patch` 必须是字符串或字符串列表，`bundlePatchPaths()` 把它解析为包内绝对路径；`loadProfile()` 按 `dsh.profile.bundles` 顺序消费。bundle 名的解析锚有两个：**dsh 安装锚优先**（`apps/cli` 的包目录），其次才是 profile 自己的 `node_modules`——这保证了 `@deepseek-ai/dsh-base` 永远来自与运行中 dsh 相同的安装，而不是 profile 里的本地副本。

**bundle 层装载失败怎么办。** bundle 不是插件行，行准入不检查它的 dsh 版本要求，所以 `loadProfileDirectory()` 在装载每一层时独立做三件事：平台声明检查（`assertBundlePlatform`）、版本兼容预检（`engines.dsh` 或 DSH peer 范围对照运行时版本，精确豁免记录在 profile 自己的 `compatibility.json`）、patch 解析。任何一步失败都只是**跳过该层**并记入 `skippedBundles`，不改 bundle 选择；launcher 用 `reportSkippedBundles()` 在每次启动时打印一次，其余层保持原顺序。注意：跳过某个 bundle 并不保证剩余组合仍能满足必需服务——`web` profile 若跳过 base，`agent-loop` 等必需行就无从提供，启动照样失败。profile manifest 与用户 patch 的错误则照常抛出、失败启动。

首次使用时 profile 自动初始化。模板来自 `packages/util/official-bundles/src/bundles.ts`：

```ts
export const PROFILE_TEMPLATES: Record<string, ProfileTemplate> = {
  acp: {
    bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-acp-app'],
  },
  web: {
    bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web-app'],
  },
  headless: {
    bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-headless'],
  },
  sdk: {
    bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-sdk-app'],
  },
  'sdk-minimal': {
    bundles: ['@deepseek-ai/dsh-sdk-minimal'],
  },
}
```

`initProfile()` 同时写入空 patch 模板（`packages/boot/app-boot/src/profile.ts` 的 `PROFILE_PATCH_TEMPLATE`），用户从这里开始改造：

```yaml
# Your patch layer for this dsh profile, applied after every bundle layer:
# a top-level YAML array of loader patch entries (id-targeted config
# overrides, disables, and insert lists; `!!js` expressions allowed).
[]
```

---

## 3. 五种内置 profile 对比

| Profile | 叠加的 bundle | 典型场景 | HMR（配置热重载） |
|---|---|---|---|
| `web` | `dsh-base` + `dsh-web-app` | 浏览器 GUI：交互聊天、模型/设置管理、会话历史、插件管理页 | 开（YAML 组合启用；模块监视仍需显式 `root` 配置） |
| `headless` | `dsh-base` + `dsh-headless` | 脚本 / CI 一次性任务：打印最终答案即退出，不监听端口 | 关（patch 仅启动时应用） |
| `sdk` | `dsh-base` + `dsh-sdk-app` | TypeScript / Python SDK 的 JSON-RPC stdio 长驻运行时 | 关（保存的修改报 `restart-required`） |
| `acp` | `dsh-base` + `dsh-acp-app` | 自动化客户端的 ACP stdio 接入（编码代理 persona） | 关 |
| `sdk-minimal` | 仅 `dsh-sdk-minimal` | 极简内嵌编码代理（Python SDK minimal 示例默认） | 关 |

三个要点：

1. **前四个 profile 都是 base-backed**：`dsh-base` 排第一，模式 bundle 排第二。模式 bundle 负责"这个 surface 特有的行"，并在覆盖共享行时**整行重述**自己的完整配置（见 §4 的 patch 语义）。
2. **`sdk-minimal` 是特例**：它不叠 `dsh-base`，单个 bundle 的一个 `insert` 就是完整应用树——SDK stdio 启动与 JSON-RPC 服务、一个环境变量配置的 DeepSeek 适配器、显式 agent 核心、MCP 资源、本地文件系统与图片附件、自动上下文压缩、平台选择的持久 shell PTY、`$DSH_HOME/sessions` 下的未压缩 JSONL 持久化。它刻意排除 Web、settings、托管凭证、遥测、文本文件工具、工作区指令、skills、jobs 和 subagents；`danger-full-access` 策略允许 shell 改写进程可达的任何路径，因此只应在隔离工作区使用。为什么这样做：为内嵌方提供一个**小而显式**的运行时——不继承共享核心就意味着每一行都是 profile 的显式声明，没有隐藏依赖，也没有 dsh-base 的默认权限面。启动时模型选择只来自 SDK 初始化请求，凭证走 `DEEPSEEK_API_KEY` 环境变量。
3. **`desktop` 是第六个保留名**：归 Electron 桌面应用独占，CLI 对它拒绝 boot 与 config-dump（`apps/cli/src/args.ts` 的 `rejectElectronProfile`）；Desktop 用同一套 `@deepseek-ai/dsh/profile-boot` 导出加载自己初始化好的 profile 目录。

### 3.1 模式 bundle 各自加了什么

模式 bundle 的 patch 站在 base 层之上，做三类事：重述与 base 取值不同的共享行、插入 surface 特有的行、挂应用自己的旗标解析提供者。

- **`dsh-web-app`**（`packages/bundle/web-app`）：重述 `system-prompt` 的 persona 前缀与 `session-query-sqlite`（Web 用内存索引、`openAt: never`），插入 `webserver` / `web-runtime` / `connection` 传输层、浏览器端插件花名册（`dsh.client` 行）、遥测行；再通过 `presets/<id>.patch.yml` 挂四个 agent preset。胶水插件 `src/index.ts` 负责 dist 服务、广告 URL、`DSH_WEB_URL` 变量与就绪广播。
- **`dsh-headless`**（`packages/bundle/headless`）：挂一次性 runner（必需行 `headless-runner`）。任务来自位置参数或 stdin；`--json` 以换行分隔事件流投影本次运行；`--session-id` 恢复既有会话。shipped 配置禁用托管凭证库及其 authorization / DeepSeek 账户消费方——API key 路线依次读启动环境、`<调用目录>/.env`、`$DSH_HOME/.env`，GUI 里保存的登录在不动文件的情况下被忽略。
- **`dsh-sdk-app`**（`packages/bundle/sdk-app`）：挂应用自有的零选项命令提供者，只有提供者接受调用后 `dsh-sdk-jsonrpc-server`（必需行 `sdk-jsonrpc-server`）才启动，因此 `dsh --profile sdk --help` 写完帮助即退出、不占用 stdin/stdout。stdout 保留给换行分隔的 JSON-RPC 帧；stdin EOF 绑定到 launcher 的受控成功关停。禁用模型生成会话标题（SDK 无标题面）。
- **`dsh-acp-app`**（`packages/bundle/acp-app`）：同样"命令提供者先接受调用、`dsh-acp` 才启动"；设置编码代理 persona 与默认模型路由（`deepseek-official` / `deepseek-v4-flash`），每会话工作目录由 ACP 客户端供给。协议上零私有方法、零私有 capability。

### 3.2 自定义 profile

两条路（`apps/cli/README.md`）：

```sh
dsh rescue --from-default-profile web     # 用 shipped web 模板重建名为 rescue 的 profile，然后启动
dsh plugin --profile tui add <package>    # 初始化一个 base-backed 的 tui profile 并安装插件
```

`--from-default-profile` 的替换是两段原子改名：先在新 profile 组装于 `.<name>.staging-*` 暂存目录，移入后才删除旧目录；移动失败会恢复旧目录（恢复也失败时旧目录留在暂存处并报 `AggregateError`）。shipped 名字大小写不敏感地保留，`desktop` 亦然。`dsh plugin init` 对没有 shipped 模板的名字使用 `DEFAULT_PROFILE_BUNDLES`（`packages/util/official-bundles/src/bundles.ts` 中仅 `['@deepseek-ai/dsh-base']`）。

profile 目录里的 `pnpm-workspace.yaml` 由 `initProfile()` 写入 hoisted 链接器设置——外来插件得到扁平的 `node_modules`，缺失的 peer（cordis 等）落到运行时解析，全树共享安装里唯一的 cordis 实例。

---

## 4. dsh-base：共享核心与"可被上层 patch"的纪律

`packages/bundle/base/cordis.patch.yml` 是一个纯静态 patch 文档：对空根的一次 `insert`，不挂服务、不发事件、不持状态。行顺序不携带加载语义（激活由服务可用性驱动），分组只为方便阅读。按职能分组，它包含：

- **模型连接**：`llm`（`@deepseek-ai/dsh-llm`）、`deepseek-llm-api-extensions`、`llm-retry`、`agent-default-model`（默认 `deepseek-official` / `deepseek-flash`）；
- **Agent 与工具**：`agent`、`read`/`write`/`edit` 文本工具、平台门控的 shell 栈（见下）、subagent 与 subagent_fork、任务/目标追踪、MCP 资源；
- **持久化**：`session`（durable sessions）、`session-log-deepseek`、会话标题生成、typert 三件套（registry / loader / gateway）与 `remote-access`；
- **沙箱与审批**：沙箱化文件系统提供者作为唯一文件写入路径、平台 shell 沙箱行、默认权限策略（工作区内可写、危险动作先询问）、`user-questions`；
- **设置与凭证**：`settings`、`config-editor`、`authorization` 与 DeepSeek 账户行（凭证来源依次是托管 `$DSH_HOME/.credentials.yaml` → 继承环境 → 项目 `.env` → 用户 `.env`）；
- **基础设施**：`timer`、`jobs`（`@deepseek-ai/dsh-jobs-local`）、`hmr`（默认 `root: []`）、`plugin-manager`（行与工具行默认禁用，见 §9）、遥测行（OTel 会话上传默认 `FEEDBACK_ONLY`）。

patch 文件开头即声明了它的组织纪律（摘录 `packages/bundle/base/cordis.patch.yml` 前 10 行）：

```yaml
# The dsh-base bundle patch: the shared core of each base-backed profile, applied as
# ONE insert over the empty profile root. Later bundle patches and the user's
# profile cordis.patch.yml address these rows by id, with the last write
# winning per row.
#
# A patch replaces the targeted row's whole `config` rather than merging into
# it, so a row whose value differs by mode does NOT live here: it belongs to
# each mode bundle, keeping any single row down to one bundle layer plus the
# user's.
```

这段注释给出两条设计约束，正是 bundle"可被上层 patch"的实现方式：

1. **按 id 寻址、整行替换**。上层 bundle 或用户 patch 用 `id` 找到目标行，替换它的整个 `config`——没有深合并。所以上层覆盖时必须重述想保留的所有字段。
2. **模式差异不进 base**。同一行在不同 surface 上取值不同时，base 只放共享插件身份与中性默认值（甚至不放），完整配置由每个模式 bundle 重述。这样任何一行至多只经过"一个 bundle 层 + 用户层"两次写入，不存在"默认值藏在哪一层"的追猎问题。

平台门控也发生在 base 自己的行上：`bash-sandbox` 与 `tool-bash` 带 `disabled: !!js process.platform === 'win32'`，`pwsh-sandbox` 与 `tool-pwsh` 带反相表达式，因此每台机器恰好有一个 shell 栈，且权限面在所有平台一致。想在 base-backed profile 里追加工具（摘自 `packages/bundle/base/README.md` 的 `str_replace_editor` 示例）就是一条 insert：

```yaml
- insert:
    - id: tool-str-replace-editor
      name: '@deepseek-ai/dsh-tool-str-replace-editor'
      config:
        maxOutputChars: 16000
```

反方向也有纪律：在沙箱化文件系统提供者之上再挂普通文件系统提供者会让 profile 拒绝加载——两者注册同一个服务，重复注册即失败退出（misconfiguration fails loud）。

---

## 5. patch 机制

### 5.1 层序与语义

补丁层序见 §1 步骤 6。底层的组合算法来自 vendored 的 include 插件（`vendor/include/README.md`）：`applyEntryPatches(data, patches, warn)` 与挂载时使用同一套算法——**patch 可以插入新条目，也可以覆盖 `id` 匹配条目上的字段**。整条链上没有任何深合并：id 命中的行整体替换 `config`，想保留的字段必须重述。

三个易错点：

- patch 引用不存在的 id 会打印一条 stderr 警告，不失败；
- 空文件或纯注释文件**导致 boot 失败**——想暂时关掉这一层，写 `[]`；
- insert 行的 name 可以是绝对路径、file URL 或包说明符；insert 行内的 `./`、`../` 相对路径会被转成 file URL，但既有行的 name 断言与替换的 `config` 值保持字面量。

一个真实的 id 定位替换（摘自 `packages/bundle/web-app/cordis.patch.yml`，注意没有 `name`、只有 `config`——整行替换语义下，重述的就是该行关注的全部字段）：

```yaml
- id: system-prompt
  config:
    personaPrefix: >-
      You are a coding agent powered by the {{model}} model.
```

### 5.2 preset patch

bundle 的 `dsh.bundle.patch` 声明为有序文件列表时，各文件按顺序拼接。`dsh-web-app` 正是这样组织 preset 的：主文件 `cordis.patch.yml` 放 host 行与 preset 注册表，`presets/<id>.patch.yml`（`standard`、`ptc`、`minimal`、`cordis`）各插入一个 `@deepseek-ai/dsh-agent-preset` 声明，按 `dsh.bundle.patch` 列出的顺序应用。Web GUI 里每个浏览器会话选择一个 preset，Agent presets 设置页保存的修改则落到 `$DSH_HOME/profiles/web/cordis.patch.yml`（用户层）。

### 5.3 `!!js` 的位置限制

YAML 里的 `!!js` 只允许出现在两处：插件条目的 `config` 内部，以及条目的 `disabled` 字段。Loader 在条目声明的注入（`inject`）激活后、针对该插件上下文对 `config` 表达式求值；`disabled` 在每次挂载决策时求值。**其他条目元数据一律保持字面量**——想按环境选择插件请用 overlay。另外注意必须写双叹号 `!!js`，`!js` 从不被允许。一个典型的注入求值（摘自 `packages/bundle/web-app/cordis.patch.yml`）：

```yaml
- id: webserver
  name: '@deepseek-ai/dsh-host-webserver'
  inject: [webStartup]
  config:
    host: !!js ctx.webStartup.host ?? '127.0.0.1'
    port: !!js ctx.webStartup.port ?? 3080
```

web-app 的启动流程也在此可见：`web-startup` 插件解析应用旗标并发布 `webStartup` 服务，`webserver` / `web-runtime` / `connection` 三行注入它；`dsh --profile web --help` 不提供 `webStartup`，所以打印帮助后直接退出，不会绑定端口。

---

## 6. HMR：配置热重载由 YAML 控制

HMR 是 base bundle 里的一行（摘自 `packages/bundle/base/cordis.patch.yml`）：

```yaml
# Profile configuration reloads by default; module roots are opt-in.
- id: hmr
  name: '@deepseek-ai/dsh-hmr'
  disabled: !!js "!ctx.get('profileContext')"
  config:
    root: []
```

两条开关由 YAML 组合决定：

- **有没有 profileContext**：`disabled` 表达式保证 HMR 只在 `dsh` 启动器拉起的 profile 里激活；手工构建的树（没有 launcher 提供 `profileContext`）该行自动禁用。
- **profile 模板是否启用**：五种内置模板中只有 `web` 启用（patch 保存即生效），`headless`、`sdk`、`acp`、`sdk-minimal` 都在各自的 YAML 里禁用该行，修改等到下次启动才应用。想恢复热重载，在 profile patch 里覆写：

```yaml
- id: hmr
  disabled: false
  config:
    root: ["."]
```

`root: []` 只保留显式配置监视（profile manifest + 两个用户 patch 文件）；`root: ["."]` 额外打开源码模块监视。禁用或缺失 HMR 行时，一切修改都需要重启。

重载失败策略与启动审计不同：**错误的 patch 被拒绝、运行中的配置不动**（格式损坏或非法的实时 patch 直接不应用，合法的编辑正常生效）；模块替换失败则报错并保留成功的兄弟行，修正 import 后可重新激活。HMR 不负责回滚整个更新，也不重跑必需启动审计。每次成功的 profile 重组会发出 `app-boot/config-reload` 事件（无 diff，监听方自行重读 Loader 条目）；重组共享 `reconcileProfile()` 一条路径——HMR 的文件监视、Plugin Manager 的配置写入都走它，替换失败的插件此前运行的代码不再执行，之后每次重组都会再次 import 它，文件修好即恢复。

运行面上，`packages/boot/hmr` 用一个队列（`runExclusive`）串行化模块替换、Include 刷新与 profile 配置重载；Plugin Manager 的配置写入也走同一队列，包安装/卸载则在队列外执行。HMR 的 profile 监视重组合时会一并替换"已加载文件在磁盘上变了"的插件（覆盖 profile 目录下的 `node_modules` 与链接包目标），因此链接包的构建产物变更在下一次重组时即可生效。监视文件变更的等待窗口默认开了 Chokidar 的 `awaitWriteFinish`（2 秒写稳定窗），连续快速编辑不易丢事件。

---

## 7. `--dump-config`：所见皆可 patch

不启动应用即可检查组合结果：

```sh
dsh --profile web --dump-config              # 完整组合树
dsh --profile web --dump-default-config      # 仅 bundle 层（不含用户层与 --patch）
dsh --profile web --dump-config-schema       # 条目与 patch 的 JSON Schema
```

实现入口在 `apps/cli/src/dump-config.ts`：`runDumpConfig()` 走与 boot 完全相同的 `prepareProfile()` 装配，但不挂载、不求值 `!!js`，把各层交给 `renderConfigDump()` 渲染。输出是一份**可加载的 YAML 文档**：组合后的条目列表按来源文件与修改它的 patch 层分组注释，`!!js` 表达式原样保留。三个旗标互斥；dump 不接受应用参数（它不运行应用旗标提供者，打印一棵与实际 boot 不同的树会误导）；`--dump-default-config` 会跳过用户层的解析，因此也是 `cordis.patch.yml` 坏掉时的恢复诊断。`--dump-config-schema` 需要实际 import 组合树里的模块来读 schema，只应对你已信任其插件的 profile 使用。

"所见皆可 patch"的含义：dump 里的每个条目都带 id，而 patch 算法按 id 寻址——所以**输出里看见的每一行，都可以被更高的一层（profile patch、home patch 或 `--patch`）整行替换或禁用**。这也解释了 dump 对准入失败的表现：被策略拒绝的插件行仍出现在 dump 里（它属于配置），而被拒 bundle 不贡献任何行（它还没来得及进入组合）。

---

## 8. 应用启动纪律：为什么只有 `dsh` profile 能启动应用

`docs/architecture.md` 的 Application launch 一节写明：受支持的 Node 应用通过命名的 `dsh` profile 启动；SDK 与 ACP 是 profile，不是独立公共 bin；vendored CLI、构建/测试专用可执行文件、进程内直接挂载插件都不算 Harness 应用启动器。这条纪律由 `scripts/verify-application-entrypoints.ts` 强制执行，它检查三类违规：

1. **包 bin 白名单**：扫描 `apps/*/package.json` 与 `packages/*/*/package.json`，出现 `bin` 字段而白名单（`apps/cli` 的 `dsh` 与 build-only 的 `dsh-pack-vfs-image`）没有的包，直接报"package bin bypasses the dsh launcher; applications use apps/cli profiles"。
2. **可执行源白名单**：所有以 `#!` 开头的 `.ts/.js/.mjs/.cjs` 源文件必须在显式分类表里（dsh 启动器、test-only driver、build-only wrapper 等），否则报"executable source has no application/build/test classification"。
3. **根脚本 launcher 策略**：`start:web`、`dev:web`、每个 `demo:*` 脚本必须显式引用 `apps/cli/src/bin.ts` 且不得直接引用任何包内 `src|lib` 入口——例如 `start:web` 是 `dsh-direct`，`dev:web` 走 `scripts/dev-web.ts` 包装器，包装器同样受检。

为什么这样设计：

- **组合是数据，不是代码**。新增一种应用形态 = 新增一个 profile（bundle 列表 + patch 层），而不是新增一个带私有 argv 解析的 bin。插件管理、HMR、配置热重载、`--patch` 覆盖、凭证与遥测开关只在 launcher 路径上成立；逃逸的 bin 会绕过这一切。
- **一次实现，多端复用**。Python SDK 的 wheel 打包的就是同一个 `dsh` CLI（客户端默认选 `sdk` profile）；Electron Desktop 调用同一 `runCli()` 入口与 `profile-boot` 导出。若各端有自己的 bin，打包、签名、升级路径都要复制一份。
- **包 bin/SDK argv 逃逸被禁止**，是因为包里的 `bin` 一旦存在，用户就会用它启动"半个应用"——没有 profile 的权限策略、没有凭证层、没有必需行审计。规则让"想做新应用"的唯一答案是写 profile/bundle，产品功能留在 bundle 而不是新 bin（`packages/boot/app-boot/README.md` 明确：产品功能属于 profile bundle，直接挂载插件仅限"给已运行的应用加插件"的场景）。

---

## 9. Plugin Manager 与插件市场

### 9.1 Plugin Manager（`packages/boot/plugin-manager`）

`ctx.pluginManager` 管理当前 profile 的插件与 bundle，不要求手改配置：

- **行级开关**：`listPlugins()` / `setPluginEnabled(id, enabled)`。持久化方式很克制——只改 profile `cordis.patch.yml` 里**最后一个匹配该 id（与模块名断言）的覆盖行的 `disabled`**，没有匹配则追加一条；若 bundle 层与其余 patch 已经给出目标状态，则删除冗余覆盖。patch 文件只记录"偏离 bundle 的部分"。Agent preset 的行保持只读。agent 侧的 `plugin_manager` 工具用同一条路径，例如启用工具行就是在 profile patch 里写：

  ```yaml
  - id: tool-plugin-manager
    disabled: false
  ```

- **bundle 级开关**：`listBundles()` / `setBundleEnabled(name, enabled)`，修改的是 `package.json` 的 `dsh.profile.bundles` 有序列表；禁用保留依赖，启用追加到列表末尾（可能改变配置优先级）。
- **安装 / 卸载 / 更新**：`installBundle(spec)` 复用 `dsh plugin` 的 pnpm 实现，多 registry 依序回退（私有 registry 不落公共池）；GitHub 仓库先做限时 `git ls-remote` 连通性检查；pnpm 11 的构建脚本审批（`approvedBuilds` / `pendingBuilds`）持久化在 profile 里。失败的安装会恢复 `package.json` 与 `pnpm-lock.yaml`。`updateBundle` 在 HMR 下先移除旧版（插件停止）再装新版，失败则回装旧版。
- **版本兼容与豁免**：安装与启动都会按 `engines.dsh`（缺省则看 DSH 相关 `peerDependencies`）对照运行时版本；豁免是 profile 自己的 `compatibility.json` 里精确的 `package-name@version → [runtime-version]` 映射，授予需要 `acceptRisk` 确认，且插件或 DSH 升级都不继承许可。

Web 的侧边栏 Plugins 页与 agent 的 `plugin_manager` 工具都走这套 Remote 方法；SDK profile 下无 HMR 时保存的修改返回 `restart-required`。

### 9.2 Plugin Market（`packages/boot/plugin-market`）

`ctx.pluginMarket` 是用户配置的插件索引的 Host 读取器：`search(url, query)` 分页检索、`entry(url, name)` 单条、`readEntryDetail(...)` 读长描述（npm 文件 CDN jsdelivr/unpkg/npmmirror 依序，回退 Git 部分克隆）。索引文档由 `packages/util/plugin-market-index` 校验格式；每条结果都被标注上当前 profile 的安装状态与 DSH 兼容性，以及可直接交给 `installBundle` 的精确安装参数。Web 的 Discover 页与 agent 的 `plugin_market` 工具是它的两个消费方——**市场只负责"看"，安装一律通过 `pluginManager.installBundle`**。shipped profile 默认不挂该服务（可选 bundle `@deepseek-ai/dsh-experimental-plugin-market-bundle`），也未配置任何索引源。

---

## 小结

| 机制 | 一句话 |
|---|---|
| Profile | `$DSH_HOME/profiles/<name>` 目录：有序 bundle 列表 + 用户 patch 层，同一个 `dsh` 二进制的多面来源 |
| Bundle | 声明 `dsh.bundle.patch` 的 npm 包，一个可安装的 patch 层 |
| Patch | 按 id 整行替换 / insert；层序 = bundle 顺序 → profile → home → `--patch` → 遥测开关 |
| `!!js` | 仅限条目 `config` 与 `disabled` 两处；注入激活后求值 |
| HMR | 由 YAML 组合决定；内置模板仅 `web` 开；`root: []` 只监视配置，`root: ["."]` 才监视源码 |
| `--dump-config` | 不 boot 打印组合树；dump 里看到的每一行都能被更高层 patch |
| 启动纪律 | 只有 dsh profile 是受支持的应用启动器；`verify-application-entrypoints` 把所有 bin/脚本钉进显式分类 |

延伸阅读：`docs/subsystems/boot.md`（服务方法与结果记录全集）、`apps/cli/reference/README.md`（层序与旗标的权威参考）、`apps/cli/composition.md`（每个 shipped profile 的精确组合图）、`.agents/notes/implemented/architecture/2026-08-05-profile-plugin-bundles.md`（profile/bundle 组合的设计依据）。
