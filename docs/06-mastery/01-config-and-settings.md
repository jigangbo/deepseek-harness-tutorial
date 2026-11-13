# 配置体系与用户设置 — 从 cordis.yml 到凭证保险箱

> dsh 的配置不止"一个 yaml 文件"：它是一套分层组合 + 热更新 + 秘密隔离的体系。本章讲清四层配置如何叠加、用户设置表单如何投影、秘密值如何永远不进配置。

## 配置的四个层次

一次运行中的每个插件行（plugin row）的配置，由有序叠层决定：

```text
  ┌──────────────────────────────────────────────┐
  │ 1. Bundle 声明的行 (dsh-base 等)              │  ← 基线
  ├──────────────────────────────────────────────┤
  │ 2. Profile 的 cordis.patch.yml                │  ← 用户的 profile 级覆盖
  ├──────────────────────────────────────────────┤
  │ 3. Harness home 的 patch 文件                 │  ← 用户全局覆盖
  ├──────────────────────────────────────────────┤
  │ 4. --patch 命令行 overlay                     │  ← 一次性注入
  └──────────────────────────────────────────────┘
  规则：按 id 定位行，整行替换或插入新行；preset patch 在
  自己声明的 config.plugins 内套用同样的层优先级。
```

查看你机器上的最终组合：

```sh
dsh --profile web --dump-config
```

打印出的每一行都可以被你自己的 patch 替换——这就是「所见皆可 patch」。

## cordis.yml 语法要点

- 每行一个插件：`id`（可选，稳定标识用于 patch 定位与 HMR）、`name`（包名）、`config`（该插件声明的 Config 字段）。
- **`!!js` 的位置限制**：只有 plugin `config` 下与入口 `disabled` 字段允许 `!!js`（JS 表达式求值）；其他元数据一律字面量。因此**条件组合用 overlay 表达**，而不是在 YAML 里写脚本。
- Raw/Web `cordis.yml` 里以裸包名引用的插件必须出现在该组合 resolver manifest 的 `dependencies` 中（`verify-cordis-config` 门禁强制）。

## volatile 配置与用户设置表单

插件用 `.volatile()` 标记的字段进入另一条链路——**运行时表单**：

```text
插件声明 Config 字段 (.volatile())
        │
        ▼
dsh-settings 服务（settings.yaml 导入、修订版描述符）
        │  表单投影：只暴露 volatile 字段，剔除运行时引用
        ▼
UI 设置页 / SDK 调用 ──► 校验整个插件 Config ──► dsh-config-editor 持久化
        │                                        （应用文件锁 + HMR 队列）
        ▼
写回活动 profile 的 Cordis patch → HMR 热应用
```

关键行为（来自 `packages/settings/settings/README.md`）：

- 表单按 **profile entry id** 识别插件；只暴露"活动的、可唯一寻址的"条目的 volatile 字段；普通配置仍走 Cordis 配置文件。
- **拒绝过期写入**（stale writes）与**保留秘密值**：客户端没收到的字段在路径编辑时被保留。
- **Reset** 恢复被 profile 覆盖之下的值（含 schema 默认）；home patch 与命令行 overlay 优先级更高，会被其覆盖的表单写入直接拒绝。
- 旧版 `settings.yaml` 一次性迁移：按 section id 写入对应插件条目，文件先改名 `.imported` 再写盘；运行组合拒绝的 section 记日志、只留在改名文件里。
- 自带设置页的插件用 `configure({ auto: false }, ctx.fiber)` 注册策略（放在可选的 `ctx.inject(['settings'], ...)` 子上下文里）：Settings 服务晚加载或被替换也能接上策略，业务插件不依赖 Settings 也能跑。

## 秘密值永不进配置：凭证缝

`dsh-credentials` 的核心思想：**配置里只放"引用"，值存在凭证库里**。

```ts
import { credentialRef } from '@deepseek-ai/dsh-credentials'

const ref = credentialRef('DEEPSEEK_API_KEY')      // POSIX 标识符，品牌化类型
const hit = await ctx.credentials.resolve(ref)     // { value, source } | undefined
const info = await ctx.credentials.describe(ref)   // { configured, source?, writable }——绝不含值
await ctx.credentials.set(ref, 'sk-…')             // 存储值优先于环境变量回退
await ctx.credentials.unset(ref)                   // 空操作当缺席；环境回退重新生效
```

设计要点：

| 规则 | 效果 |
|------|------|
| 配置/设置文件只出现 `DEEPSEEK_API_KEY` 这样的引用名 | 可同步、可共享、可渲染进 UI 的配置永不泄密 |
| `describe()` 返回"是否已设/来源/可否修改"，不返回值 | UI 能报告状态而不可见值 |
| **轮换即时生效**：每次操作时 resolve | 换 key 无需重启、无需改配置 |
| 空的 key 值=缺席；空的 record=有意的已存凭证 | 两种语义不打架 |
| 记录（record）支持无对应环境变量的凭证 | OAuth 授权授予、provider 环境值等持久保存 |
| `.env` 只作回退 | env 优先于 `.env`，存储值最高 |

本地实现挂载：

```yaml
- name: '@deepseek-ai/dsh-credentials-local'
  config:
    path: /absolute/path/to/.credentials.yaml
```

完整字段清单以生成的 `docs/config-catalog.md` 为准——**配置目录也是生成物**（元习惯：文档由机器保持同步）。

## 配置全家福：谁负责什么

| 包 | ctx 键 | 职责 |
|----|--------|------|
| `packages/settings/settings` | `ctx.settings` | volatile 字段表单投影、校验编辑、重置 |
| `packages/boot/config-editor` | `ctx.configEditor` | 在应用文件锁 + HMR 队列下持久化 profile patch，并调和 Loader 条目 |
| `packages/credentials/credentials` | `ctx.credentials` | 凭证引用与记录的 resolve/describe/set/unset |
| `packages/credentials/credentials-local` | （实现） | 文件路径后端 |
| `packages/credentials/authorization` | `ctx.authorization` | 授权流注册表：谁能发起什么登录流，键即其写下的记录 |
| `packages/boot/hmr` | `ctx.hmr` | 模块与精确配置的串行重载；应用变更共享同一队列 |

## 实践建议

1. **选 volatile 还是普通配置？** 用户会频繁在 UI 里调的（模型选择、开关）用 volatile；部署结构性的（后端地址、路径）留在 cordis.yml。
2. **秘密一律走引用**。团队同步的 settings 文件、要截图的 issue、要渲染的 UI，都不该出现明文 key。
3. **patch 文件保持小**：只写你要改的行；整行替换意味着你要为该行的全部字段负责。
4. **条件组合用 overlay**，不用 `!!js`——overlay 在层优先级里有明确位置，`!!js` 只在允许的两个位置有效。
5. **响亮失败**：引用了不存在的插件 id、写错字段名，让它在加载时炸出来，不要包一层 try 让它静默降级。

## 与其他篇章的关系

- 层叠与 patch 机制在总览篇《启动流程与 Profile/Bundle 组合》有完整推导。
- 设置页在 UI 侧的渲染见交互与产品形态篇。
- `agent-default-model` 是 volatile 配置的典型消费者：读写默认模型选择走 `ctx.agentDefaultModel` → `ctx.configEditor`。
