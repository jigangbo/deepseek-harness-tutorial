# 能力缝篇 — 可替换的能力

> 一切皆插件在能力层的落地：**Service Definition / Service Provider / Consumer** 三角色。读 [设计思想](00-seam-design.md) 掌握总纲，再按需进入具体缝。

| 章节 | 缝 | 核心问题 |
|------|-----|----------|
| [00 设计思想](00-seam-design.md) | 总纲 | 三角色模型、为什么换 provider = 换产品、设计清单 |
| [01 文件系统](01-filesystem.md) | `ctx.fs` | local/sandbox/ssh 三世界的统一接口 + 交付物与工作区 |
| [02 命令执行与沙箱](02-shell-subprocess-sandbox.md) | `ctx.shell`/`ctx.subprocess`/`ctx.sandbox` | 三层缝联动、策略单一家 |
| [03 终端与 PTC](03-terminal-and-ptc.md) | `ctx.terminals`/`ctx.ptcRuntime` | 持久 PTY、程序化工具调用（沙箱化 Node） |
| [05 网络访问](05-web.md) | `ctx.web` | 搜索与抓取 provider 可替换 |
| [06 浏览器与计算机](06-browser-computer-use.md) | `ctx.browserUse`/`ctx.computerUse` | 互斥命名注册——另一种缝形态 |
| [07 LSP 与技能](07-lsp-skills.md) | `ctx.lsp`/`ctx.skills` | 归一化四操作、目录/正文两级加载 |
| [08 子代理与团队](08-subagent-team.md) | `ctx.subagents`/`ctx.agentTeams` | 六种后端、统一激活管理 |
| [09 任务与工作流](09-jobs-workflow.md) | `ctx.jobs`/`ctx.workflowEngine` | 后台作业记录、PTC 程序化编排 |
| [10 MCP 与自扩展](10-mcp-extensions.md) | MCP/`ctx.cordisInspect` | 向外接与向内开两个开放边界 |

**上一篇**：[核心引擎篇](../02-core-engine/)　**下一篇**：[持久化篇](../04-persistence/)
