# 第 1 章 什么是 AI Agent 与 Harness


欢迎来到 DeepSeek Harness(以下简称 `dsh`)教程的第一章。本章不写一行需要你运行的代码,目标是帮你建立三个基本概念:

1. **LLM Agent** 是什么——为什么"能对话的模型"和"能做事的 agent"之间隔着一个循环;
2. **Agent Harness**(智能体框架)是什么——模型之外的所有工程,以及 DeepSeek Harness 这个项目具体是什么;
3. **一切皆插件**(everything-is-a-plugin)——`dsh` 的核心架构思想,也是后面所有章节的基础。

读完后,你将能看懂 `dsh` 的分层架构图、一次典型交互的完整时序,并掌握一组贯穿全书的核心术语。

---

## 1. LLM Agent:模型 + 循环 = 能做事的智能体

### 1.1 模型只会生成文本

先把一件事说透:**LLM(大语言模型)本质上只会做一件事——读入一段文本,生成一段文本**。

你可以给它一段代码让它解释,它可以输出 JSON,甚至可以输出一段"看起来像在执行命令"的文字。但模型自己:

- 不能读写你的文件;
- 不能执行命令;
- 不能上网搜索;
- 甚至记不住上一次对话——除非你把历史再喂给它。

所以当你对一个模型说"帮我修复这个测试"时,它最多只能回复"你可以试试把 `foo` 改成 `bar`"。**动手**的那一步,模型永远做不到。

### 1.2 Agent 循环:让"生成文本"变成"做事"

Agent 的思路非常直接:模型虽然只能生成文本,但我们可以**约定一种文本格式表示"我要调用某个工具"**(即 tool call,工具调用)。模型输出这种格式时,由外部程序替它真正执行,再把结果作为文本塞回对话,让模型继续。如此往复,直到模型认为任务完成,输出普通回复。

这个"请求 → 工具调用 → 结果 → 再请求"的过程,就是 **Agent 循环(agent loop)**:

```text
       ┌─────────────────────────────────────────────┐
       │                 模型 (LLM)                  │
       │       唯一能力:读入文本,生成文本          │
       └───────────▲─────────────────────┬───────────┘
                   │                     │
          ① 发送请求                    ② 生成回复:
            (系统提示词、对话历史、        要么是给用户的话,
             工具清单)                    要么是工具调用
                   │                     │
       ┌───────────┴─────────────────────▼───────────┐
       │                Agent 循环                   │
       │                                             │
       │   回复是话? ──▶ 输出给用户,本轮结束        │
       │                                             │
       │   回复是工具调用?                          │
       │        │                                    │
       │        ▼ ③ 交给工具执行器                  │
       │   ┌─────────────────┐                       │
       │   │ 文件 / 终端 /    │──▶ ④ 拿到执行结果    │
       │   │ 浏览器 / 搜索... │                       │
       │   └─────────────────┘                       │
       │        │                                    │
       │        ▼ ⑤ 结果追加进对话历史               │
       │        └──▶ 回到 ①,带着新上下文再问模型    │
       └─────────────────────────────────────────────┘
```

几个关键观察:

- **模型始终只负责"决定",不负责"执行"**。它说"我要运行 `pnpm test`",真正敲下回车的是循环外的工具执行器。
- **循环是普通程序代码**,不是模型的一部分。它由条件判断和 `while` 组成:回复里有 tool call 就执行再继续,没有就结束。
- **对话历史在持续增长**。每一轮的工具调用和结果都会追加进去,模型下一轮请求时能看到全部上下文——这就是 agent 能"记住"自己刚做了什么的原因。

> 在 DeepSeek Harness 中,这个循环的实现位于 `packages/core/agent-loop/`,第 4 节我们会逐行走读它。

### 1.3 循环有多简单

把这个循环的骨架写出来,其实就是十几行伪代码(示意用途,非仓库代码):

```ts
async function runAgent(task: string) {
  const history = [{ role: 'user', content: task }]
  while (true) {
    // ① 请求模型:带上系统提示词、对话历史、可用工具清单
    const reply = await llm.stream({ messages: history, tools: registry.schemas() })
    history.push(reply)

    // ② 回复里没有工具调用 → 任务完成,把话交给用户
    const calls = reply.content.filter(b => b.type === 'tool-call')
    if (calls.length === 0) return reply.text

    // ③④⑤ 执行每个工具调用,把结果追加进历史,回到 ①
    for (const call of calls) {
      const result = await registry.execute(call)
      history.push(result)
    }
  }
}
```

真正的工程难度不在循环本身,而在循环的**周边**:每一步如何落盘、工具失败如何恢复、用户中途取消如何传播、上下文超出窗口如何压缩。这些正是接下来两节的内容。

---

## 2. Agent Harness:模型之外的全部工程

### 2.1 只有循环还不够

上面那个循环图看似简单,但要让它变成一个**可用、可控、可恢复**的产品,中间隔着大量的工程:

- **会话日志**:循环跑了 200 步后进程崩了,怎么办?所有输入输出必须先落盘,才能恢复现场、回放、 fork。
- **工具执行**:工具定义、参数校验、并发控制、超时处理、输出截断——每一项都有大量细节。
- **权限审批**:模型要执行 `rm -rf` 之前,总得有人按一下"允许"。
- **能力供给**:文件系统、命令行、持久终端、浏览器、语言服务器……这些能力本身也要有干净的接口。
- **用户界面**:Web UI、桌面端、CLI、SDK,都要实时呈现流式输出和工具卡片。

这些"模型之外的全部工程",就是 **Agent Harness**(智能体框架,有时也叫 agent scaffold)。Harness 像马的挽具:模型是那匹马,挽具决定它的力气往哪里使、什么时候勒住。

### 2.2 模型与 Harness 的分工

| 职责 | 归属 | 例子 |
|---|---|---|
| 决定"下一步做什么" | 模型 | 选择调用 `bash` 工具并给出命令 |
| 决定"允许做什么" | Harness + 用户 | 沙箱策略、审批弹窗 |
| 真正执行动作 | Harness | 起子进程、写文件、发请求 |
| 记录发生过什么 | Harness | 追加式会话事件日志 |
| 组装模型可见的上下文 | Harness(基于日志) | 系统提示词、工具 schema、历史消息 |
| 把文本变成界面 | Harness | Web UI 的流式渲染、工具卡片 |

一个重要原则(后面章节会反复出现):**模型可见的东西必须等于日志里记录过的东西**。凡是会进入模型请求的内容(系统提示、历史消息、工具清单),都必须能从会话日志完整重建。这条规则保证了恢复、fork、导出、快照回放全都有一份唯一事实来源。

### 2.3 DeepSeek Harness(dsh)是什么

**DeepSeek Harness(`dsh`)是 DeepSeek AI 开源的一个 agent harness**,构建于"一切皆插件"架构之上,底层由 Cordis 插件框架驱动(见 `README.md` 与下一节)。

- 它**不是**一个模型,而是一个承载任意模型的运行时;
- 它自带 Web UI(`npx @deepseek-ai/dsh web` 即可启动)、桌面端、无界面 headless 模式、JSON-RPC SDK 与 Python SDK;
- 它把文件读写、bash/PowerShell 执行、持久终端、网页搜索与抓取、子 agent 委派、后台任务等做成一组可组合的能力;
- 它处于 developer preview 阶段,API 仍会破坏性变更(运行前请阅读第 6 节的安全须知)。

### 2.4 dsh 自带哪些能力

"模型之外的全部工程"听上去抽象,落到实处就是一组模型可见的工具。以下是 `docs/tool-catalog.md` 里的一部分真实条目,按用途归类:

| 用途 | 工具名(模型可见) | 背后的能力缝 |
|---|---|---|
| 文件读写 | `read` / `edit` / `write` / `glob` / `grep` | `ctx.fs`、`ctx.subprocess` |
| 命令执行 | `bash` / `pwsh`(一次性)、`terminal_*`(持久终端) | `ctx.shell`、`ctx.terminals` |
| 网络 | `web_search` / `web_fetch` | `ctx.web` |
| 委派与协作 | `subagent`、`send_message` 等控制工具、`team_task_*` | `ctx.subagents`、`ctx.agentTeams` |
| 后台任务 | `job_list` / `job_output` / `job_kill` | `ctx.jobs` |
| 规划与目标 | `todo_write`、`create_goal` 等 | agent 会话状态、`ctx.goals` |
| 浏览器 | `stagehand_*` 系列(实验性) | `ctx.browserUse` |
| 语言服务 | `lsp`(跳转定义、诊断等查询) | `ctx.lsp` |

注意两点:

- **工具名和实现是解耦的**。比如 `web_search` 的 schema 保持稳定,而搜索后端藏在 `ctx.web` 缝后面,换 provider 不改 schema(工具目录里明确写了这一设计动机);
- **能力的启用是组合决定的**。比如持久终端工具是 opt-in 的——profile/bundle 决定挂不挂 `dsh-tool-terminal`,没挂上模型就永远看不到这些工具。

---

## 3. 一切皆插件(everything-is-a-plugin)

### 3.1 Cordis:底座框架

`dsh` 建立在 [Cordis](https://github.com/cordiverse/cordis) 之上。用 `docs/architecture.md` 的原话说:

> plugins contribute services, typed events, and reversible effects to a shared context. Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so each is replaceable from configuration.

翻译成工程师的语言,Cordis 提供四样东西(详见 `docs/cordis-primer.md`):

1. **Context(上下文)**:一个服务容器。服务通过 `ctx.<key>` 彼此发现,而不是互相 import 具体实现;
2. **Plugin(插件)**:实现 Cordis `Service` 的对象,声明自己依赖哪些服务,由框架在依赖就绪后挂载;
3. **Typed Events(类型化事件)**:服务通过 TypeScript 声明合并定义事件名,按 `emit` / `waterfall` / `parallel` / `serial` / `bail` 五种模式分发;
4. **Reversible effects(可逆注册)**:所有注册(工具、提示词段、适配器、监听器)都通过 `ctx.effect()` / `ctx.on()` 完成,插件卸载时自动撤销。

关键推论:**`dsh` 没有需要打补丁的特权内核**。模型适配器、工具注册表、会话日志、agent 循环本身——全都是普通插件,都可以通过配置替换。

### 3.2 用一个能力缝看三种角色

`docs/glossary.md` 定义了 **capability seam(能力缝)**:一个可替换能力由三种角色组成——

- **Service Definition(服务定义)**:声明接口的抽象类;
- **Service Provider(服务提供者)**:实现接口的具体插件;
- **Consumer(消费者)**:注入并使用该服务的插件,通常是一个模型可见的工具。

以命令执行为例,仓库里正好是一套三件套。先看服务定义 `packages/shell/shell/src/index.ts`:

```ts
// 通过声明合并,把 `shell` 这个 key 挂到 Cordis 的 Context 类型上
declare module '@deepseek-ai/cordis' {
  interface Context {
    shell: ShellExecutor
  }
}

// 抽象类即接口:子类实现两个抽象方法,加载后即注册为 ctx.shell
export abstract class ShellExecutor extends Service {
  constructor(ctx: Context) {
    super(ctx, 'shell')
  }

  // ...此处省略 sandboxMode 属性与 JSDoc...

  // 把调用方的原始请求补全成完整规格(默认值、上限钳制)
  abstract resolve(request: ShellExecRequest): ShellExecSpec

  // 按规格准备并启动命令,返回进程句柄
  abstract execute(spec: ShellExecSpec): Promise<ShellExecution>
}
```

服务提供者是 `packages/shell/bash-local/src/index.ts` 里的本地 bash 执行器:

```ts
// 子类实现抽象方法,作为插件加载后占据 ctx.shell
export class LocalBashExecutor extends ShellExecutor {
```

消费者是模型可见的 `bash` 工具,在 `packages/shell/tool-bash/src/index.ts` 中通过 `inject` 声明依赖:

```ts
export const name = 'tool-bash'
export const inject = ['tools', 'shell', 'systemPrompt', 'shellEnv', 'workingDirectory']
```

工具执行时只面向 `ctx.shell` 接口编程,完全不关心背后是谁:

```ts
const spec = ctx.shell.resolve({ ...request, signal: exec.signal })
const foreground = await ctx.shell.execute(spec)
```

于是:**换一个 provider**(比如 `dsh-bash-sandbox` 沙箱执行器),`bash` 工具、权限策略、UI 卡片原封不动,整个产品的执行语义却变了。这正是"seam 是完整能力,不是单个接口"的含义——`dsh-shell`(定义)、`dsh-bash-local` / `dsh-bash-sandbox`(提供者)、`dsh-tool-bash`(消费者)三个包合起来才是这条缝。

### 3.3 分层架构总图

把视角拉高,一个运行中的 `dsh` 大致是下面这样的分层结构(每一层都由插件组成,层间通过 `ctx.<key>` 服务与类型化事件通信):

```text
┌─────────────────────────────────────────────────────────────┐
│                       用户界面 (UI)                          │
│     Web UI(浏览器)│ Desktop(Electron)│ headless│ SDK  │
├─────────────────────────────────────────────────────────────┤
│              Profile + Bundle 组合层(启动时装配)           │
│   profile: web / headless / sdk / sdk-minimal / acp         │
│   bundle:  dsh-base(公共底座)+ 各应用专属 bundle           │
│   patch:   profile patch → 用户 home patch → --patch 覆盖   │
├─────────────────────────────────────────────────────────────┤
│                    Agent 循环(ctx.agentLoop)               │
│   turn → step → 模型请求 → 工具调用 → 结果 → 循环           │
├─────────────────────────────────────────────────────────────┤
│                  工具与能力缝(capability seams)            │
│   ctx.tools   模型可见工具注册表(bash/read/edit/web/...)  │
│   ctx.shell   命令执行        ctx.fs        文件系统        │
│   ctx.subprocess 子进程      ctx.terminals 持久终端         │
│   ctx.browserUse 浏览器      ctx.web 搜索/抓取              │
├─────────────────────────────────────────────────────────────┤
│                 会话持久化(ctx.sessions)                   │
│   SessionEvent 追加式日志 · 投影(projection)· 版本迁移      │
├─────────────────────────────────────────────────────────────┤
│                   LLM 适配器(ctx.llm)                      │
│        DeepSeek API / OpenAI 兼容端点 / 自定义 provider      │
└─────────────────────────────────────────────────────────────┘
```

从上到下读这张图:

- **UI 层**只观察事件、调用服务,不包含业务逻辑;
- **组合层**在启动时决定"到底装哪些插件":profile 是一份具名组合清单,bundle 是配置行及其代码的分发格式,patch 可以按行覆盖任何配置——包括换掉上面任何一层里的任何插件;
- **agent 循环**自己也是个插件:`packages/core/agent-loop/src/index.ts` 导出的就是一个 Cordis 服务:

  ```ts
  export class AgentLoop extends Service implements AgentFactory {
    static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']
  ```

  它通过 `inject` 声明依赖会话注册表、LLM 适配器、工具注册表等服务,加载后占据 `ctx.agentLoop`;
- **能力缝层**是第 3.2 节那一套三件套的复数形式;
- **持久化层**保证"模型可见 ⟺ 已记录"(第 2.2 节);
- **LLM 适配器**把统一的请求词汇翻译成具体 provider 的线上协议,新增一个模型 provider 就是注册一个适配器,循环一行不改。

### 3.4 在代码里认出一个插件

读完上面三个例子,你已经能凭两个特征在 `dsh` 仓库里认出插件。`docs/cordis-primer.md` 说插件"can be a function with optional `inject` and `apply(ctx)` fields, or a `Service` subclass",对应仓库里的两种真实形态:

**形态一:函数式插件**(本教程出现的 `dsh-tool-bash`、`dsh-bash-local` 都属于这种)——模块级导出 `inject` 数组和 `apply` 函数:

```ts
// packages/shell/tool-bash/src/index.ts(简化)
export const name = 'tool-bash'
export const inject = ['tools', 'shell', 'systemPrompt', 'shellEnv', 'workingDirectory']

export function apply(ctx: Context, config: Config = {}): void {
  // 在这里注册工具、监听事件、声明提示词段……
}
```

**形态二:Service 子类插件**(能力缝的服务定义与重型服务常用)——比如 3.2 节的 `ShellExecutor`,以及整个 agent 循环本身:

```ts
// packages/core/agent-loop/src/index.ts(简化)
export class AgentLoop extends Service implements AgentFactory {
  static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']
  // 构造时占据 ctx.agentLoop;Service.init 中完成启动工作
}
```

两种形态的共同点:`inject` 声明依赖,框架在依赖就绪后才挂载它——**加载顺序由服务需求表达,而不是手工排队**;挂载期间的一切注册都是可逆效果,卸载时自动撤销。

---

## 4. 一次典型交互走读

现在把三张图拼起来:你在 Web UI 里输入"总结这个仓库的主要包",到看到最终回复之间,系统里发生了什么。

### 4.1 时序图

```text
  你            Web UI              会话日志 + Agent 循环           模型            工具
  │               │                        │                        │               │
  │ ① 输入任务    │                        │                        │               │
  │──────────────▶│                        │                        │               │
  │               │ ② 消息进入 inbox,      │                        │               │
  │               │    写入 user/message   │                        │               │
  │               │───────────────────────▶│                        │               │
  │               │                 ③ turn/start,认领输入,         │               │
  │               │                 组装系统提示 + 工具 schema       │               │
  │               │                        │──④ 流式请求(完整上下文)▶│               │
  │               │                        │◀─⑤ 流式 chunk(文字)...─│               │
  │               │◀─ 实时渲染文字 ────────│                        │               │
  │               │                 ⑥ 回复末尾出现 tool_call        │               │
  │               │                 (bash: "pnpm ls --depth 0")     │               │
  │               │                 ⑦ 需要审批?向用户请求批准       │               │
  │  允许         │◀── 审批卡片 ───────────│                        │               │
  │──────────────▶│───────────────────────▶│                        │               │
  │               │                 ⑧ tools/execute:               │               │
  │               │                        │──── 执行命令 ──────────────────────────▶│
  │               │                        │◀─── stdout/stderr ─────────────────────│
  │               │                 ⑨ tool/result 追加进会话日志     │               │
  │               │                        │──⑩ 带着工具结果再次请求─▶│               │
  │               │                        │   ...(循环,直到不再有工具调用)...       │
  │               │                 ⑪ turn/end(最终回复已落盘)      │               │
  │  看到总结     │◀──── 最终回复 ────────│                        │               │
  │◀──────────────│                        │                        │               │
```

### 4.2 每一步对应什么

事件名都能在 `docs/architecture.md` 的 "Turn flow" 一节找到,与实现一一对应:

- **①②**:用户输入进入 agent 的 inbox(收件箱)。注意"认领输入"是循环的显式动作——没有被认领的消息只是排队,不触发模型。
- **③ turn/start**:开启一个 **turn**。循环此时会快照模型选择、组装提示词与工具 schema,并让 `agent/pre-step` 瀑布事件的监听器有机会改写或拒绝本步输入。
- **④⑤**:真正的模型调用是流式的。看 `packages/core/agent-loop/src/agent.ts` 里 `step()` 的核心几行:

  ```ts
  const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
  live.start()
  started = true
  for await (const chunk of stream) {
    signal.throwIfAborted()
    live.push(chunk)
  }
  ```

  每个 chunk 既推给 UI 实时渲染(`agent/assistant-stream` 事件),也累积起来等待"结算"后写入会话日志。
- **⑥⑦⑧**:流结束后,循环检查回复里的工具调用块:

  ```ts
  const toolCalls = message.content.filter(block => block.type === 'tool-call')
  if (toolCalls.length === 0) return { kind: 'completed' }
  const maxParallelToolCalls = this.loopCtx.agentLoop.config.maxParallelToolCalls.get()
  try {
    const result = await this.loopCtx.tools.execute({
      calls: toolCalls, signal, agent: this, log: { turn, step },
      maxParallelToolCalls, toolSet: decision.toolSet,
    })
  ```

  没有工具调用,step 以 `completed` 结束;有,则交给工具执行管线(`ctx.tools.execute`)。**审批发生在执行之前**——比如 bash 工具会先经过沙箱策略与 `ctx.approval` 审批缝,用户拒绝时工具根本不会运行。
- **⑨⑩**:工具结果以 `tool/result` 事件追加进日志,然后循环回到 ④——**同一步(step)里模型会看到工具结果并再次被请求**;如此往复,直到某次回复不再包含工具调用。
- **⑪**:turn 以 `turn/end` 收束,记录结束原因(完成、达到 token 上限、被取消、出错)。这些边界事件在日志里同样可查:

  ```ts
  this.session.append('turn/start', { turn })
  // ...
  this.session.append('step/start', { turn, step })
  // ...
  this.session.append('step/end', { turn, step })
  // ...
  this.session.append('turn/end', { turn, reason: turnEnds! })
  ```

回顾第 1 节的循环图:**"Agent 循环"那六个箭头,在这里全部有了名字、事件和落盘记录**。而且注意一个设计取向:凡是上面列出的关键事件(`turn/*`、`step/*`、`user/message`、`assistant/message`、`tool/*`)都是**持久会话事件**;`agent/pre-step`、`llm/stream` 等则是只在进程内活跃的扩展点。哪些要持久、哪些只在现场,`docs/architecture.md` 有明确划分。

### 4.3 顺带认识:三类事件域

走读中出现的各种事件名,按 `docs/architecture.md` 的划分属于三个域,后面章节会反复遇到:

| 域 | 例子 | 特点 |
|---|---|---|
| **会话事件**(session events) | `turn/start`、`assistant/message`、`tool/result` | **持久**:追加进日志并广播,重启后仍在;是恢复与回放的唯一事实来源。 |
| **Agent 事件**(`agent/*`) | `agent/pre-step`、`agent/assistant-stream`、`agent/status` | **活跃**:携带运行中的 Agent 对象,用于观察或拦截进行中的工作;进程消失即消失。 |
| **能力事件**(capability events) | `fs/*`、`tools/*`、`telemetry/*` | **挂接点**:给某条能力缝附加策略或适配器,不需要 import 循环。 |

判断新事件该落在哪个域,规则只有一句:这个事实需要在进程重启之后仍然存在吗?需要,就是会话事件;不需要,就在后两个域里挑合适的扩展点。

---

## 5. 核心术语速查表

下面这些术语贯穿全教程。定义用我自己的话,原始出处见 `docs/glossary.md`、`docs/cordis-primer.md` 与 `docs/architecture.md`。

| 术语 | 一句话解释 |
|---|---|
| **Session** | 一次完整工作的持久化容器,本体是一条**追加式事件日志**;恢复、fork、导出、快照回放都从它派生。 |
| **Turn** | 一次被接纳输入的完整处理:从 `turn/start` 到 `turn/end`,"模型和它的工具都停下"或某个终止策略介入为止。 |
| **Step** | 一次模型请求**加上**这次回复引发的所有工具执行;一个 turn 包含零个或多个 step。第 4 节的 ③–⑩ 大致就是一个 step。 |
| **Round** | 更外层的策略迭代,一个 round 包含一个 turn(如 goal 轮次、Ralph 轮次);round 计数属于具体策略,不统计会话里的每个 turn。 |
| **Exchange** | 相邻的一组 turn:从某个"接纳人类输入"的 turn 开始,到下一个这样的 turn 之前结束;由通知唤醒的 turn(子 agent 完成、goal 轮次、定时任务)算进前一个 exchange。 |
| **Plugin** | 实现 Cordis `Service` 的对象(一个带 `inject` 与 `apply` 的模块,或一个 Service 子类),向共享 Context 贡献服务、事件与可逆效果。 |
| **Service** | 通过声明合并占据 `ctx.<key>` 的能力单例(如 `ctx.shell`、`ctx.llm`);其他插件按 key 查找,而不是 import 具体实现。 |
| **Event** | 类型化事件的通信机制,借 TypeScript 声明合并定义事件名;五种分发模式 `emit` / `waterfall` / `parallel` / `serial` / `bail` 对应"旁观 / 包裹 / 并行 / 顺序 / 首个拦截"。 |
| **Waterfall** | 瀑布事件的包裹中间件语义:监听器拿到 `(...args, next)`,**必须调用 `next()`** 才能把(可能改写过的)结果传给下游;不调用即短路。这是 `dsh` 里实现拦截与改写的主要手段。 |
| **Profile** | 一份具名插件组合清单,存放在 Harness home;列出叠加哪些 bundle,可携带 out-of-tree 插件与用户自己的 `cordis.patch.yml`。内置模板:`web` / `headless` / `sdk` / `sdk-minimal` / `acp`。 |
| **Bundle** | 插件配置行及其挂载代码的**分发格式**;它插入的每一行都保持可被上层 patch 覆盖。`dsh-base` 是大多数 profile 共享的第一层。 |
| **Seam** | 一条完整的能力缝 = Service Definition(定义)+ Service Provider(提供者)+ Consumer(消费者)。单个角色不叫 seam;换一个 provider 即可整体更换一种能力(见 3.2 节)。 |
| **Projection** | 从已提交的会话事件**折叠**出来的只读视图(如当前模型选择、inbox 状态、todo 列表);读取方通过 `snapshot()` 取一致的快照,而不必遍历原始事件。 |

小测验(答案都在本章里):turn 和 step 谁包含谁?换一个 bash 执行器需要改 `bash` 工具的代码吗?为什么工具调用前可以有审批弹窗?

---

## 6. 安全须知

DeepSeek Harness 是**实验性的开发者预览软件**。以下要点摘自仓库根目录的 [SAFETY.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/SAFETY.md),动手实验前请完整阅读原文:

- **未经安全审计**。项目不应被视为安全或生产就绪。
- **它能执行模型生成的代码与命令**,加载第三方插件,访问网络、进程、凭据与文件。模型输出错误、软件缺陷、配置失误、恶意输入或不可信插件,都可能损害宿主机、修改或删除文件、泄露数据。
- **沙箱与审批不等于隔离保证**。即使限制被正确执行,也保护不了那些本来就授权给它的资源。不要把它当作不可信工作负载的唯一安全防线。
- **负责任地使用**:以最小权限运行;优先使用一次性虚拟机、容器或专用环境;对它能碰到的文件保持备份;不要暴露敏感凭据;放行命令与插件前逐个审查。

本教程是面向源码阅读的学习材料,不改变上述任何结论。

---

## 7. 下一章

本章建立了三个心智模型:Agent = 模型 + 循环;Harness = 模型之外的全部工程;`dsh` = 建立在 Cordis 上的一切皆插件。

下一章 [《Cordis 框架入门》](./02-cordis-framework.md) 将正式动手:从一个最小插件开始,逐一实践 Context、Service、`inject` 依赖声明、类型化事件与可逆注册——也就是本章 3.1 节那四个概念的展开。理解了 Cordis,`dsh` 仓库里的每个包在你眼里都会变成"谁提供了什么、谁在消费"。

## 延伸阅读(仓库内文档)

- `README.md` / `README.zh.md` ——项目概览与运行方式
- `docs/architecture.md` ——官方架构文档,本章多处引用
- `docs/cordis-primer.md` ——Cordis 五个核心思想
- `docs/glossary.md` ——领域术语的权威定义
- `docs/tool-catalog.md` ——全部模型可见工具的清单
- `docs/user/guide/index.md` ——Web UI 上手指南
