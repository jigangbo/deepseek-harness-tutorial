# 第 18 章 文件系统

> ctx.fs 与交付物、工作区


> 一切代码工作的地基。`ctx.fs` 让"文件访问"在本地、沙箱围栏、SSH 远程三种世界里保持同一接口——模型工具完全感知不到背后是谁。核心源码：`packages/fs/` 组。

## 组内地图

| 包 | 角色 | 说明 |
|----|------|------|
| `fs` | Service Definition | `ctx.fs` 契约：读/写/列/编辑/删除/移动/监视 |
| `fs-local` | Provider | 普通宿主文件 |
| `fs-sandbox` | Provider | 会话变更围栏在工作区与 temp 根内（与 `ctx.sandboxPolicy` 共享模式与根） |
| `fs-ssh` | Provider | SSH 远程（与 `ctx.ssh` 配对） |
| `fs-observation-policy` | 事件门消费者 | 经 `fs/*` 事件门贡献被观察态检查 |
| `fs-patch` | 库 | patch hunk 应用 |
| `tool-fs` | Consumer | 模型可见文件工具（read/write/edit…） |
| `tool-fs-search` | Consumer | grep/glob 发现工具 |
| `tool-str-replace-editor` | Consumer | 字面替换编辑工具 |

## 服务契约（模型工具的底座）

消费者可以：解析稳定身份、读文本或有界字节、列目录、监视目标、写文件、应用字面编辑或 patch hunk、删除文件、移动文本文件（可选 patching）。关键语义：

- **可选版本守卫拒绝过期变更**（乐观并发控制）。
- **移动分别报告**目的地发布与源移除——两阶段语义，崩溃时可区分。
- 没挂任何后端就没有 `ctx.fs`，工具在注册时失败——**缺失能力响亮失败**，不是运行到一半才炸。

```text
         ┌── fs-local ──► 宿主磁盘
tool-fs ─┼── fs-sandbox ─► Landlock/bwrap/Seatbelt 围栏内
         └── fs-ssh ─────► SSH 通道
（tool schema 与策略插件对三者零感知）
```

## 事件门：fs/observation-policy

`fs/*` 能力事件允许策略插件介入文件访问。`fs-observation-policy` 经这个门贡献"被观察态"（observed-state）检查——例如"编辑一个你没读过的文件"这类守护。注意事件门的消费者不 import 循环（capability events 的定义）。

## 交付物（deliverables 组）

turn 结束时"agent 改了什么"需要显式呈现：

- `deliverables/workspace-changes`（`ctx.workspaceChanges`）：服务**每 turn 变更文件摘要**——按 Session 与事件序号，直到 Session 释放；日志只携带 turn 事实，摘要由服务从工作区对比派生。
- 显式文件交付 + 记录的工作区变更两种来源。
- Web 端由 ui-deliverables 渲染变更卡片（changed-file cards 由 Coding Tools 偏好控制）。

## 工作区（workspace 组）

- `ctx.workspaceRegistry`：`WorkspaceId` 品牌化的记录，跑在 `ctx.storageDomain`（持久化篇）之上。
- 稳定 sessionId 账户驱动 Host RPC 与 GUI 投影；`api-workspace-controller` 把命令与"重连安全的状态投递"投影到 Remote 命名空间。
- 归档/置顶/排序语义的浏览器侧行为见产品形态篇 Web 前端章（Client 模型拥有浏览器行序与竞态消解，Host 拥有权威状态）。

## 与沙箱策略的联动

`ctx.sandboxPolicy` 是**部署默认模式 + 工作区根的唯一家**。`bash-sandbox`、`fs-sandbox`、`terminal-bash` 三个执行家族都读它——保证 bash 与 fs **不会围到不同的根**。工具层使用它导出的纯 `sandbox/mode` 折叠，只有沙箱化执行器与 provider 读服务本体。

## 模型工具全景（fs 侧）

| 工具 | 包 | 功能 |
|------|-----|------|
| read / write / edit / ls 等 | `tool-fs` | 经 `ctx.fs` 的读写编辑族；write/edit 的 UI 卡片是 `diff` 意图（oldText: null 表示新文件） |
| read 的完成态卡片 | tool-fs | 由持久 `result.meta` 重建文件窗口（path/offset/lines/totalLines/lang） |
| grep / glob | `tool-fs-search` | 发现工具；`search` 卡片带 truncated/total，UI 永不把截断结果当完整呈现 |
| str-replace | `tool-str-replace-editor` | 字面替换编辑 |

工具卡片纯函数与 `presentationMeta` 机制见精通篇《添加一个工具》。

## 设计启发

1. **环境差异全在 provider**：local/sandbox/ssh 的差异（权限、路径规范化、错误语义）被缝吸掉；策略插件与工具 schema 一行不改。
2. **没有后端 = 注册期失败**：能力缺失在组合时暴露，不是调用时。
3. **服务契约比直觉细**：版本守卫、移动的两阶段报告——这些"多余"的语义正是分布式/可恢复场景需要的诚实接口。
