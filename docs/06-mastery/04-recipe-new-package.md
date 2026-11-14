# 05 · 扩展实战：添加一个工作区包


> 目标：按主仓库 `docs/cookbook/adding-a-package.md` 的清单，新建一个 `@deepseek-ai/dsh-<name>` 包并通过全部质量门。这份清单以 bash 与适配器包为模板验证过——如果清单与模板漂移，以修清单为准。

## 1. 创建包

```text
packages/<group>/<pkg>/
  package.json     # 从 packages/core/tools 拷贝，改 name/description/deps
  tsconfig.json    # extends ../../../tsconfig.base.json, rootDir src,
                   # outDir lib/types, references: vendor/cosmokit、vendor/cordis
                   # (+ vendor/schemastery 若用 Config，+ 每个 dsh 依赖)
  src/index.ts     # 服务默认导出 或 插件 (name/inject/apply/Config)
  locale/en.json   # 可选：meta.title / meta.description / meta.detail
  locale/zh.json   # 同字段翻译
  README.md        # 服务 API、事件、扩展点、设计笔记
                   # + 门控的 Model Experience 块
                   # + 门控的 Known Limitations and Deferred Work 小节
```

**选组**：优先加入匹配角色的现有组（`core`、`llm`、`shell`、`compaction`、`subagent`、`todo`、`session`、`client`/`host`、`util`、`test-support`）。新组允许，但只是纯容器：无 package.json、无源文件，包仍恰好在其下一层。

**package.json 不变量**（`pnpm run constraints` 强制）：

| 字段 | 要求 |
|------|------|
| `private` / `version` / `type` | `true` / 与根 package.json 一致 / `module` |
| `main` / `types` | `lib/index.js` / `lib/types/index.d.ts` |
| `exports["."]` | types → `./lib/types/index.d.ts`，default → `./lib/index.js` |
| `@deepseek-ai/cordis` | **同时**在 peerDependencies 与 devDependencies（同范围） |
| dsh peer 依赖 | 每个都在 devDependencies 镜像 |
| `@deepseek-ai/schemastery` | 放 `dependencies`（运行时校验器） |
| `files` | 恰好 `lib/index.js`、`lib/types/**/*.d.ts` + 门禁认可的运行时产物；有 bin 的 CLI 包把 `lib/bin.js` 紧跟 `lib/index.js` |

**包内相对导入用显式 `.ts`**（如 `export * from './types.ts'`）——编译器在 JS 产物里改写为 `.js`，声明里保留 `.ts` 供 NodeNext 消费者解析到同级 `.d.ts`。

## 2. 注册进根配置

| 文件 | 改动 |
|------|------|
| `tsconfig.base.json` | 已有组无需改；新组给 `@deepseek-ai/dsh-*` 通配加 `./packages/<group>/*/src` 候选 |
| `tsconfig.host.json` 或 `tsconfig.client.json` | 把包路径加进对应聚合的 `references`——普通包恰好属于一个聚合，绝不两者都属 |

自动覆盖无需编辑：根 workspaces、publint、tsdown、oxlint、constraints 检查。

`packages/client/*` 的包另有契约：extends `tsconfig.base.client.json`、声明 `dsh.client`、导出 `./client`、用共享 tsdown preset。

## 3. 决定包拓扑与命名

**可替换能力**：当角色独立演化时，把 Service Definition / Service Provider / Consumer 拆成包（shell 三件套是模板）。单一用途插件保持一个包。

**给现存角色命名**，不给未来扩张命名。数量语义要一致：单数 `ctx` 键给一个引擎/运行时/策略/控制器/解析器/仓库；复数给注册表或多成员服务。

角色后缀词汇表（节选，完整表见 cookbook）：

| 词 | 用在 | 不用在 |
|----|------|--------|
| `Controller` | 接受命令/用户意图，改变一个既有状态 | 执行任意工作、拥有 provider 舰队 |
| `Store` | 拥有一套数据，主供 CRUD/快照/订阅 | 校验状态机、仲裁权威 |
| `Registry` | 拥有动态命名注册集（查找/重复规则/生命周期/释放） | 主契约是派发或编排 |
| `Runtime` | 跑活跃工作，拥有派发/取消/跨调用生命周期 | 只存记录或返回目录 |
| `Presenter` | 领域值→渲染意图的纯转换 | 做 I/O、订阅、改状态 |
| `Policy` | 决定允许/选择/限制/观察什么 | 执行决策允许的机制 |
| `Executor` | 在一个能力里运行一个显式请求/已解析规格 | 拥有宽应用生命周期 |
| `Provider` | 提供能力定义的一个实现（多实现时加机制/厂商限定词） | 它是能力定义本身或注册表 |
| `Backend` | 在定义接口后实现可替换的持久化/传输/执行 | 用户可见服务 |
| `Handle` | 指向一个活跃资源并控制/观察它 | 创建管理整个资源池 |

`SDK` 只用于 JSON-RPC 客户端/服务端协议；产品名拼写正典是 `Typert`。

## 4. 写包 README（门禁强制结构）

`kind` 前言四选一：group / reference / library / bundle。结尾必须有规范序列：

```markdown
## Model Experience
### Request context and condition
#### What the model sees      ← 数据依赖字段或锚定的生成目录链接
#### Token effect             ← 固定/条件/保留/替换/封顶/零直接影响
#### KV Cache effect          ← append-only / 前缀稳定 / 替换 / 独立请求，及失效条件
## Known Limitations and Deferred Work
- **Consumer-visible gap** — 精确缺失操作、后果、维护者约束
```

- Model Experience 只总结数据依赖或 provider 拥有的文本；工具 schema 条目链接生成工具目录的锚点小节，只写目录里没有的增量。
- 无上下文影响的包用审计过的 `None, as ` / `Indirectly, through ` 句式；模型无关的通用包可加入 `NO_MODEL_EXPERIENCE_SECTION` 白名单。
- 限制小节只记录**本包拥有的**持久消费者缺口与非显然维护者约束；普通清理留在源码 TODO 或 Agent Note。

## 5. 可选的插件显示元数据

`locale/en.json` + 各语言同字段：

```json
{ "meta": { "title": "Workspace Tools", "description": "Tools for your workspace." } }
```

- 回退链：locale `meta.title` → 包根插件的 `package.json.name` → 完整 Cordis 插件名；描述同理。
- 英文标题/描述用句首大写（sentence case）。
- 一方包需提供 ja/es/fr/de/ru/pt 全字段翻译（`pnpm run check:i18n` 验证）。
- 图标：manifest 顶层 `"icon": "./icon.svg"` 或导出 `./icon`；相对 manifest 目录、≤256 KiB、SVG/PNG/JPEG/WebP。
- `meta.detail` 可指向包内 Markdown 详情文档（不能在包根）。

读取这些元数据**不激活插件**。

## 6. 验证

```sh
pnpm install        # 注册工作区
pnpm run doc-sync
pnpm run constraints && pnpm run typecheck && pnpm run lint
pnpm run build && pnpm run hygiene
```

行为专属的测试与覆盖要求按仓库测试政策执行（见本教程《测试体系》一章）。

## 新包决策速查

| 你的情况 | 做法 |
|----------|------|
| 给模型加一个新工具 | 单包插件，注册进 `ctx.tools`（见《添加一个工具》） |
| 可替换执行机制的 capability | 三包缝：定义 + `xxx-local` 实现 + consumer |
| 只是通用小工具 | `packages/util/`，零运行时依赖 |
| 新产品面 UI | `packages/client/` + client 契约 |
| 实验性能力 | `packages/experimental/`，状态标注与交付解耦 |
