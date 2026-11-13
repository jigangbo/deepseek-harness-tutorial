# 架构设计模式总结 — 可以带走的工程智慧

> dsh 是一个把"架构纪律"执行到编译门与测试门的大型 TypeScript 单仓。本章把散落在 AGENTS.md、`docs/defensive-patterns/`、各包 README 里的设计规则提炼为可以迁移到你自己项目的模式清单。

## 五个元习惯

`docs/defensive-patterns/README.md` 开宗明义：目录里每条规则都是这五个习惯在某类工件上的应用。先记习惯，再记规则：

1. **把正确性写进结构**（Make correctness explicit）——用命名的调用点、显式生命周期、显式条件与类型表达正确性，而不是事后用时序补偿。
2. **单一事实来源**（One source of truth）——一份数据一条读路径，一个事实一个家，一份共享状态一个 owner。
3. **经由注册缝扩展**（Extend through the registered seam）——新能力走注册；既有载体不再长新逻辑，而不是在它内部长 if 分支。
4. **改动面 = 评审面**（Change face equals review face）——一个改动只携带它声明的行为变更所需的行；迁移搬代码而不是重写代码；性能敏感改动必须自带测量。
5. **要求证据**（Require evidence）——主张必须可证伪、可复算：前后测量，或由门禁检查，而不是靠人肉保持同步。

## 类型系统即架构

dsh 把大量架构约束下沉到类型层，编译器成了第一道架构评审：

| 模式 | 做法 | 防御的缺陷 |
|------|------|-----------|
| **声明合并注册服务** | 服务通过 `declare module` 合并进 `SessionEventMap` / ctx 类型；新增事件必须补类型与 `@mode`/`@param` JSDoc | 无类型的事件乱流；文档与实现漂移 |
| **判别标签收口** | 封闭联合以 `assertNever` 结尾；可合并联合落进有文档的 default | 新增事件类型后漏改消费者 |
| **品牌化 id** | 跨边界的不透明 id 用 `Branded<B>`（`dsh-brand`），禁止裸 `string` | 两种 id 互相错传（sessionId 当 workspaceId） |
| **禁止新的 `as unknown`** | 保留或缩减遗留基线；替代品必须用类型或校验 | 类型系统被 `unknown` 中转架空 |
| **typed 边界信任 TS** | 同进程静态接口只信任类型；运行时校验只放在解析器/配置/队列/模型工具 JSON/持久文件/worker/进程/wire 边界 | 到处防御式校验，代码膨胀且掩盖真正需要校验的边界 |

最后一条最反直觉也最有价值：**"信任 TypeScript"不是乐观，而是把校验预算集中到真正的边界上**。校验放错地方（比如同进程调用之间反复判空）不但浪费，还会掩盖"这里其实是个 wire 边界"的事实。

## 显式优于隐式

- **边界上的默认值必须显式**：默认化是 owning 实现里的显式 `resolve(request): Spec` 步骤，而不是 `run()` 里藏一个 `?? default`（`dsh-shell` 的 request/spec 拆分是模板）。
- **不许在插件里硬编码可调参数**：部署相关的选择是经校验的 `Config` 字段，可从 cordis.yml 改；`DEFAULT_*` 常量或测试钩子不算可配置。协议常量、外部规范、安全不变量保持固定。
- **配置错误要响亮失败**：自包含的在加载时失败，否则在最早可解析点失败；绝不静默跳过缺失的引用。
- **空 `catch` 必须命名错误与理由**，且 `try` 只包一条语句。
- **错误处理有明确所有权**：类型化结构化失败、本地化可操作报告；定义处理归属、遏制与失败保证；停止依赖工作并可见地报告不可信状态（`dsh-error-handling` 技能持有细则）。

## 事件与扩展点纪律

```ts
// Waterfall 短路是最常见的扩展事故：监听器必须委托
ctx.on('tools/pre-execute', (payload, next) => {
  if (deny(payload)) return { action: 'deny', reason } // 认领决策
  return next()                                        // 否则必须 next()
})
```

- **注册即 effect**：一切贡献走 `ctx.effect()`/`ctx.on()`；registry 的 `register()` 返回 disposer。每个 registry 都要有 HMR 安全测试（dispose 掉贡献 fiber，断言清理干净）。
- **Waterfall 监听器必须 `next()`**：返回而不委托=短路整条链。这是 dsh 最容易踩的坑，所以 cordis 源码里 `isBailed` 的语义、文档、教程反复强调。
- **模型可见 ⟺ 已记录**：任何进模型请求的内容必须能从会话日志重建；新的模型可见输入必须新增会话事件。这条不变量把"提示词工程"变成了"日志工程"，让 fork/续跑/转写/遥测全部有据可依。
- **改 loop 必须改地图**：新行为挂在文档化的扩展点上；实在要改 `agent-loop`，同一 PR 更新 `docs/architecture.md` 的流程图。

## 能力缝三角色（详见能力缝篇）

Definition / Provider / Consumer 三角色齐备才构成缝；扩展包只依赖 Definition；一个 provider 换掉整个产品行为。配套两条：

- **文档即接口**：每个包 README 必须覆盖 purpose / configuration / extension points / Model Experience，并带「已知限制与延期工作」小节（脚本 `verify-package-readme-*` 强制）。
- **能力图谱自动生成**：`scripts/gen-doc-graphs.ts` 从 Cordis 声明发现服务、按角色分类并带完备性守卫，产出 `docs/capability-seams.md`——文档由机器保持与代码同步（元习惯 5 的范例）。

## 单仓工程化

- **源码平面 vs 产物平面永不混用**：静态门与测试经 tsconfig paths 把工作区导入解析到 `src`；消费 `lib/` 的门必须显式声明依赖。陈旧 `lib/` 会把模块单例加载出第二份——这类 bug 在大单仓里极难定位，规则从根上掐断。
- **双面编译显式化**：同时有 Host/Client 程序的包提供 face 级叶子配置 + solution 根；全仓程序从 face 配置播种，绝不从 solution 根播种。
- **ESM 一致**：`"type": "module"`；跨包用包名导入、本地相对导入带 `.ts`；配置子进程跑构建 `lib/`。
- **重复检测进门**：`pnpm run duplication` 做跨文件克隆检测；并行值优先对称，无解释的不对称往往意味着漏了提取。
- **文档与代码同 PR**：README/JSDoc 契约随代码更新；双语文档逐段对齐；`doc-sync`/`verify-doc-budgets` 门禁控制文档膨胀。
- **偏好维护中的依赖，而不是手搓**：当它真的能删掉你拥有的代码与测试时（`dependencies-over-hand-rolling` 笔记）。

## 变更管理

- **有目的地选择 PR 历史**：独立变更拆分；引入性 bug 在传播前于引入 PR 修掉。重写用 `--force-with-lease` 且远端有动就中止，绝不裸 `--force`。
- **每个外部可感知的破坏性变更立即写升级指南**；持久化类型变更要「确认」（typed acknowledgements）；SQLite 用单调 `SCHEMA_VERSION`；已提交的世代路径永不改名/替换/删除。
- **测试描述行为而不是正确性**：废弃行为连同它的测试一起改，并在 PR 里解释为什么。
- **Agent Notes 只写持久决策依据**：机制性/局部修改豁免；归档笔记冻结，永不当现行权威。

## 把这套纪律搬回你的项目

按收益/成本排序的落地顺序：

1. **先立"单一事实来源"清单**——找出你系统里每个被两处以上代码读取的状态，给它一个 owner。
2. **把校验集中到真边界**——列出你的 wire/file/process 边界，边界内信任类型。
3. **扩展点先行**——新需求先问"挂在哪个缝"，而不是"改哪个函数"。
4. **让文档机器可检查**——像 gen-doc-graphs 那样，能从代码生成的文档绝不手写。
5. **快照式回归**——把"一次真实交互的完整输出"录下来做 keyless 回放，这是对抗"单测全绿、产品是坏的"的最强武器。

## 延伸阅读（主仓库内）

- `docs/defensive-patterns/` 五个分册：package-surface / remote-and-api / client-components / conversation-and-session / general
- `docs/glossary.md` 术语正典
- `.agents/skills/dsh-error-handling/`、`.agents/skills/dsh-prose-standard/`、`.agents/skills/dsh-ci-test-reliability/`
- `docs/postmortem/` 复盘集：每篇都是一条规则的诞生现场
