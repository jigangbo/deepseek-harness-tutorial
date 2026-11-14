# 6.1 测试体系

> 七层测试金字塔与质量纪律


> dsh 的测试体系本身就是一份架构教材：它把"绿灯才有意义"落实为一套可执行的纪律。读源码前先读这一章，你会明白每个包里 `tests/` 目录的形状为什么是那样。

## 总览

仓库用七层测试覆盖同一个产品，每层回答不同的问题：

| 层 | 命令 | 回答的问题 | 关键约束 |
|----|------|------------|----------|
| 单元 | `pnpm run test` | 每个函数/事件的局部行为正确吗 | vitest，测试跟着被测代码走 |
| 覆盖门 | `pnpm run test:coverage` | 有没有死代码 | **CI 覆盖门：`packages/*/*/src` 每文件 100%** |
| 真实 API e2e | `pnpm run test:e2e` | 对真模型整条链路通吗 | 无 key 自动跳过，keyless CI 保持绿 |
| 期望输出 | `pnpm run test:expected` | 组装后的 CLI/进程行为符合预期吗 | 无需录制会话的 keyless 断言 |
| 性能门 | `pnpm run test:bench` | 用户路径会退化吗 | 合成输入 + 时间/堆/扩展性预算 |
| 会话快照 | `pnpm run test:snapshot` | 一整个会话的持久结果符合录制吗 | 无 key：录制时注入模型回放 |
| Web 快照 | `pnpm run test:web` | 浏览器里 UI 渲染对吗 | Chromium 全量 + WebKit 抽测 |

```text
            ┌─────────────────────────┐
            │  会话快照 (整段会话回放) │  ← 无 key 复现真实交互
            ├─────────────────────────┤
            │  Web 快照 (浏览器渲染)   │
            ├─────────────────────────┤
            │  真实 API e2e (有 key)   │  ← 唯一能证明"对真模型有效"的层
            ├─────────────────────────┤
            │  期望输出 / 性能门       │
            ├─────────────────────────┤
            │  单元 + 覆盖门 (100%)    │  ← 数量最多，证明局部正确
            └─────────────────────────┘
```

## 覆盖门为什么是每文件 100%

`pnpm run test:coverage` 是 CI 的覆盖门，对 `packages/*/*/src` 要求**每个文件 100% 行覆盖**。它的哲学很有意思：

> 未覆盖的行往往是门帮你找出来的死代码——删掉它，而不是补测试硬凑。

行覆盖是必要而非充分条件：它证明"每一行都跑过"，不证明"功能按交付形态工作"。所以 100% 只是入场券，行为正确性由其他层保证。

唯一豁免：`packages/shell/pwsh-local/src` 的执行器套件在无 `pwsh` 的机器上自跳过，该文件被 `vitest.config.ts` 豁免；CI 运行器装了 pwsh，执行完整标准。

## 快照体系：无 key 复现真实会话

快照层是 dsh 最有特色的设计。每个场景录制一次真实会话（需要 key），之后**永远无 key 回放**：

```text
录制 (test:snapshot:record，需 DEEPSEEK_API_KEY)
  真实模型 ──► 会话事件流 ──► snapshots/<profile>/session[.vN].jsonl 落盘

回放 (test:snapshot，无 key)
  录制文件中的用户输入 + 模型响应 ──► MockAdapter 逐事件回放
     ──► 重新走完整 loop ──► 持久化结果与录制比对
```

要点：

- **父/子文件命名**：父为 `session[.vN].jsonl`，子角色为 `session.<ordinal>[.vN].jsonl`；v0 省略 `.v0`，正版本号必须小写 `.vN`，文件名必须与头部声明一致。
- **`snapshot.yml`** 声明 profile、组合/头部类、录制策略、回放元数据、工作区事实。
- **变更型场景**还会逐字节比对 `workspace.expected/` 整棵树（录制/刷新永不改写它）。
- **何时必须加/改快照**：任何非平凡的模型/协议/人类可见变更，同一 PR 内必须新增或更新 keyless 录制会话场景——包级、e2e、纯 mock 证据都不能替代组装后的完整转写。
- 双 SDK 投影：`SessionEventMap`/loop/生命周期变更要同时更新 TypeScript SDK 快照（`snapshots/sdk/`）与 Python SDK 快照（CI 专属脚本）。

这就是 AGENTS.md 里「模型可见 ⟺ 已记录」的测试面：快照直接断言**从日志能完整重建模型所见**。

## 宁用真实现，不用 mock

> Mock 只留给昂贵或非确定性的边界（LLM 适配器、网络、时钟），下游一切保持真实。

典型范式见 `packages/acp/acp/tests/harness.ts` 的 `makeBridgeHarness()`：挂载真实的 loop、session store、工具注册表、JSONL 持久化，唯一 mock 是按脚本回放的 `MockAdapter`。手搓替身只能证明"桥能搬字节"，证明不了"交付的工具行为如断言"。

恢复类测试进一步按 step 区分 chunk 前/后失败，证明失败 chunk 不派生任何消息或工具副作用。

## 验证世界，而不是验证自述

e2e 断言的原则：**重新执行命令或从外部重新读文件**，而不是在 agent 自己的输出里找关键词——后者会被"作弊 agent"骗过。配套纪律：

- 断言未动过的文件字节级相同。
- e2e 自有资源自清理：测试里创建，`afterEach` 里释放（失败/重试/超时也要清）。
- 共享 fixture 放普通 `tests/harness.ts`，**绝不放 `*.e2e.ts`**（import 一个 spec 会重复注册 `describe` 并重复真实 API 调用）。

## 测真实入口路径

- 产品可见插件必须有**真实组合测试**：通过 Loader + app/process 启动真正的 `cordis.yml`，手搓 `ctx.plugin(...)` 不算数；只 mock 外部服务或非确定性输入。
- 守卫只有当回归真的会变红时才算守卫：对没有 `inject` 的组合插件，Loader 冒烟在"默认导出顶替命名导出"时仍会绿——要显式 `expect('default' in mod).toBe(false)` 并用 `unwrapExports` 往返断言，且回归演示：引入坏改动→看红→还原。
- "真实入口"指交付物：包 `bin` 用**纯 node 跑构建出的 `lib/bin.js`**（tsx 会掩盖 settle 竞态、模块解析、吞掉的加载失败），并断言缺配置时非零退出。

## 源码平面解析与子进程启动模式

两条容易踩坑的执行规则：

1. **测试只从源码平面解析**：所有 vitest 配置把工作区裸导入经 tsconfig paths 指到 `src`，绝不走包 `exports` 到构建产物 `lib/`——否则陈旧产物会加载模块单例的第二份拷贝（诡异 bug 之源）。显式消费构建产物的只有 lib 模式子进程与内建冒烟。
2. **子进程启动模式**：CI/有构建的测试车道从构建 `lib/` 用共享双模式启动器拉起 profile/Cordis 子进程（禁止手写 `--import tsx`）；不加载 Cordis 的协议/OS fixture 用纯 Node 直接跑可擦除 `.ts`；只有"被测对象就是源码路径解析"的测试才允许选 `src`，并在测试里声明该契约。

## 并发与可靠性纪律

fork worker 并发跑多个 spec、覆盖门按分区并发、自托管 runner 共享一台主机——**只有进程是隔离的**，端口、可预测路径、外部命名空间、继承的子进程都不是。因此：

- 获取的每份资源都要通过 teardown 持有并归还；
- 「单独跑就过、并发就挂」的 spec 是测试缺陷，不是 runner 不稳；
- 完整的分配/恢复/同步/超时预算/平台/teardown 规则由 `dsh-ci-test-reliability` 技能持有，并提供既有概率性失败的分类诊断流程；fork 意外退出会报告 PID、文件、code、signal（`scripts/coverage-fork-diagnostics.ts`）。

## 有 key 就别省：真实 API 策略

> 我们是 DeepSeek——不要吝啬真实 API 测试。

无 key 测试只证明管道通了；只有带 key 的运行证明 agent 对真模型有效。最高价值的是**冒烟测试**：启动一个交付的 `dsh` profile，发一条 prompt，检查世界——它们专抓「单测全绿、产品是坏的」这类 mock 抓不到的问题（见 `docs/postmortem/0001-acp-default-export-drops-inject.md`）。自跳过机制让无密钥的 CI 与贡献者不被阻塞，但它不是省钱信号。

## 与其他篇章的关系

- 快照层的录制/回放建立在持久化篇的会话格式之上（头部/版本/生成文件纪律完全一致）。
- 「测真实入口」与总览篇的应用启动纪律同源：只有 dsh profile 能启动应用，测试也走同一条门。
- 扩展实战三篇（工具/适配器/包）的每一步都会告诉你需要哪几层测试。
