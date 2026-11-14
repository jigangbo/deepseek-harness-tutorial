# 09 · 后台任务与工作流

> 长时间运行的两个层级


> 工具不想阻塞 turn 怎么办？后台作业。多步骤编排怎么写？工作流。前者是"登记与观察"，后者是"程序化编排"——共享 `ctx.subagents` 与 `ctx.jobs` 两个底座。

## 一、后台作业（jobs 组）

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-jobs`（`ctx.jobs`） |
| Provider | `jobs-local`（进程内注册表） |
| Consumer | `tool-jobs`（模型控制工具）、`api-job-controller`（UI 花名册）、bash/pwsh/PTY 工具（生产者） |

机制（来自缝定义）：

- **生产者**（后台 bash/pwsh、PTY 发送）登记运行中的工作；
- **声明记录的作业**额外流式输出原始输出，供非消费的观察者读取；
- `tool-jobs` 是模型侧控制器：读、列、杀（`job_*` 工具）；
- API 侧：`ctx.jobController` 经生成的 Remote 命名空间流式传输单个作业的观察记录；花名册留在会话控制流上。

与工具契约的衔接（见精通篇工具章）：`ctx.jobs.start({ kind, label, owner, run })` 发布 id 后，取消改用 **task 自有的信号**——外层调用取消只是不再等待，不杀已发布的工作；`job_kill`、owner 释放、服务 teardown 拥有那个生命周期。

## 二、工作流（workflow 组）

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-workflow`（`ctx.workflowEngine`，每上下文一个引擎，如 bash） |
| Provider | `workflow-ptc`（PTC 程序化过程引擎） |
| Consumer | `tool-workflow`、`experimental/tool-ralph`（固定 Ralph 流程） |

设计：

- **一个引擎，没有命名 provider 注册表**——与 bash 一样，工作流引擎是上下文级的单例语义；
- `workflow-ptc` 复用 PTC 运行时执行编排程序：`agent()` 调用经 `ctx.subagents` 扇出（子代理章的激活结果在这里被 await）；
- `tool-workflow` 让模型启动与观察运行；运行状态是模型可见的（进日志）。

```text
模型 / 用户 → tool-workflow（启动）
                │
        ctx.workflowEngine (workflow-ptc)
                │ 程序执行（PTC）
        ├─ agent() 调用 ──► ctx.subagents 扇出
        ├─ 中间结果 ──► 类型化返回契约
        └─ 运行记录 ──► 日志 / job 观察 / UI 卡片
```

## 三、调度与延伸（相关组）

| 包 | ctx 键 | 职责 |
|----|--------|------|
| `schedule` | `ctx.schedule` | Host 拥有的**定时后续消息**：任务独立于会话激活存储；到期消息排队进目标会话 |
| `goal` | `ctx.goals` | 同会话目标：从日志折叠修订版目标状态，活的延续激活进程本地 |
| `plan`（plan-mode） | `ctx.planMode` | 计划协作状态：折叠 plan/mode 事件、turn 边界冲刷用户选择、`/plan` 命令、退出 schema 稳定 |
| `preset` | `ctx.agentPresets` | 按会话 agent 组合：急切挂载 YAML 声明的 preset 修订，绑定 Agents 与冷读者到作用域贡献，退役修订保留到最后一个用户释放 |
| `guard` | — | 循环卫生：建议性重复调用提醒 + `tools/execute` 的 deadline 强制器 |

## 两层长运行的边界

```text
后台作业：   单个工作单元的登记/输出/取消          （工具级）
工作流：     多步骤编排，含子代理扇出与控制流      （程序级）
调度：       未来某时刻把消息排队进会话            （时间级）
目标：       跨 turn 的持续意图                    （意图级）
```

四者都遵守"模型可见 ⟺ 已记录"：作业观察记录、工作流运行状态、调度任务、目标状态全部可从日志或持久存储重建。

## 设计启发

1. **作业是记录，不是线程**：进程内注册表 + 输出环 + 观察记录——"运行中的工作"成为可序列化的事实，UI 与模型都能安全旁观。
2. **工作流复用 PTC**：不发明第二种脚本语言——编排程序就是 PTC 程序，类型化工具返回直接可用。
3. **时间与意图各有其缝**：schedule/goal 看似"功能"，实则各自是带持久事件的领域域——任何跨 turn 状态都走同一条日志纪律。
