# 10 · MCP 外部工具与运行时自扩展

> 两个开放边界


> dsh 的开放性有两个方向：**向外接**（MCP：外部工具服务器作为原生工具）与**向内开**（extensions：agent 运行时自检、自修改）。

## 一、MCP（mcp 组）

Model Context Protocol 让外部进程的工具进入模型工具集。

| 包 | ctx 键 | 职责 |
|----|--------|------|
| `mcp-manager` | `ctx.mcpManager` | 按其 volatile servers 配置**调和 mcp-client 实例**；聚合连接状态；向管理面提供 restart 与 probe |
| `mcp-client` | `ctx.mcpResources`（connection-owned providers） | 连接外部 MCP 服务器 |
| `mcp-resources` | — | 作用域化的 MCP 资源访问：**在调用 agent 的作用域内服务共享资源工具** |

```text
settings: servers: [ { name, command/url, ... } ]
        │ volatile 配置（改了热重载）
        ▼
ctx.mcpManager ──调和──► mcp-client 实例（每服务器一个）
        │                    │ 连接状态聚合
        ▼                    ▼
外部 MCP 工具注册进 ctx.tools（模型 schema 自动进组装）
外部 MCP 资源经 ctx.mcpResources 按调用 agent 作用域暴露
```

要点：

- **配置驱动舰队**：servers 列表是 volatile 设置——增删服务器走设置链路（配置章），不走代码；
- **作用域资源**：MCP 资源工具在"调用 agent 的作用域"内服务——资源访问继承 agent 的权限世界，不是全局共享；
- 工具注册走 `ctx.tools`——MCP 工具与原生工具在模型眼里无差别（审批、守卫、PTC 全部适用）。

## 二、运行时自扩展（extensions 组）

agent 可以**检查并修改自己的运行时**：

| 包 | ctx 键 | 职责 |
|----|--------|------|
| `extensions/cordis-host-runner` | `ctx.dynamicCordisRunner`、`ctx.cordisInspect` | 内存中的插件定义注册表；Host 半边的 **vm 沙箱**；请求-运行往返；浏览器页面经 wire 通过其 remote 命名空间到达同一服务 |
| `extensions/tool-cordis` | （consumer） | 模型工具：注册 host inspect providers、镜像 client provider 清单、经动态 Cordis 传输路由 client 查询 |

能力：

- **活体检查**：`ctx.cordisInspect` 注册检查 provider，模型/UI 可以查询当前插件树、服务与注册状态（`--dump-config` 的活体版）；
- **动态挂载/卸载**：agent 写出的插件定义进入内存注册表，经 vm 沙箱在 Host 侧行跑——"agent 给自己装插件"；
- 浏览器半边与 Host 半边共享同一服务（经 wire 的 remote 命名空间），client 查询被路由进动态 Cordis 传输。

## 生态桥（experimental）

- `hooks-claude-code` / `hooks-codex`：把 Claude Code / Codex 的 hook 协议桥接到 dsh 扩展点（消费 `ctx.shell` 与 `ctx.sessionPersistence`）；
- `claude-code-mods`（`ctx.claudeCodeMods`）：加载 defineMod 插件，从 harness 扩展点引发 hook 链，经其 Remote 在提示词上方绘制横幅带；
- `inspector`（`ctx.inspector`）：Worker 托管的 CDP 目标 + 传输无关的 Host/Client 观察与 Cordis 树查询 API——调试器形态的自检。

## 两个边界的对照

| | MCP（向外接） | extensions（向内开） |
|---|---|---|
| 扩展来源 | 外部进程服务器 | agent 自己 + 用户配置 |
| 注册面 | `ctx.tools` / `ctx.mcpResources` | 内存定义注册表 + vm 沙箱 |
| 生命周期 | volatile 配置调和 | 请求-运行往返，动态挂载 |
| 风险面 | 外部代码在进程外 | agent 写的代码在沙箱内 |

## 设计启发

1. **外部工具无差别化**：MCP 工具进 `ctx.tools` 后与原生工具同权同责——审批/守卫/快照体系自动覆盖。
2. **自扩展要沙箱**：agent 写的插件代码跑在 vm 沙箱里（Host 半边），浏览器经 wire 只读同一服务——"自我修改"的能力与隔离同步设计。
3. **配置即舰队**：MCP 服务器集合是 volatile 设置——用户不改代码就能扩展模型能力。
