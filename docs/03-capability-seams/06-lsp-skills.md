# 3.7 LSP 与技能系统

> 语言智能与可复用指令


## 一、LSP（lsp 组）

语言服务器协议让 agent 拥有 IDE 级的代码导航（跳定义、找引用、悬停文档、诊断）。

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-lsp`（`ctx.lsp`） |
| Provider | `lsp-stdio`（通用 stdio 语言服务器宿主，经 `ctx.subprocess` 孵进程） |
| Consumer | `tool-lsp`（模型工具） |

关键设计（来自缝定义）：

- **Provider 注册与选择 + 归一化查询执行，恰好四种操作**；
- **缝不提供协议逃生门**——后端把 LSP 私有协议翻译进归一化请求/结果，而不是让接口长满 provider 特有的口子；
- 一个 stdio 后端服务任意符合 LSP 的语言服务器（tsserver 之外的 typescript-language-server、pyright、gopls…）。

四种归一化操作（导航域）配合 fs 缝的读取，让"找符号→读上下文→改代码"成为一轮 PTC 程序就能完成的编排。

## 二、技能系统（skill 组）

**技能（Skill）= 可复用的指令包**：一段按需加载的说明书 + 可选资源。与工具不同，技能是"教模型怎么做事"，不是"给模型新能力"。

| 角色 | 包 |
|------|-----|
| Service Definition + 注册表 | `dsh-skill`（`ctx.skills`） |
| Provider | `skill-filesystem`（文件系统技能）、`skill-office`（Office 技能）、`experimental/skill-badge` |
| Consumer | `tool-skill` |

机制：

- **合并 provider 技能目录**：多个 provider 的目录合并成会话级目录；
- **tool-skill 渲染会话前缀目录 + 加载完整技能体**——目录（名字+描述）常驻系统提示，正文按需加载，省 token；

```text
provider 目录（名字+描述）──合并──► 会话前缀目录（常驻系统提示）
                                        │ 模型调用 Skill 工具
                                        ▼
                              完整技能体按需加载（正文进当前请求）
```
- `Skill` 工具是模型显式调用技能的入口；用户也可以用 `/skill-name` 斜杠调用（ZCode 的技能机制与此同源）。
- 官方插件图标与预览走 `dsh-official-plugin-artwork` 规范；badge 技能（experimental）展示徽章类技能形态。

### 技能 vs 上下文注入 vs 工具

| | 技能 | 上下文注入（agent.inject） | 工具 |
|---|------|---------------------------|------|
| 进模型的方式 | 目录常驻 + 按需全文 | 下一次被准许的请求 | schema 常驻 + 调用时结果 |
| 持久性 | 技能文件在仓库/家目录 | 会话事件 | 注册期 |
| 典型用途 | 工作流程、领域规程 | 临时事实（文件路径、任务状态） | 执行动作 |

## 设计启发

1. **归一化面最小化**：LSP 四操作 ——协议丰富性留在后端翻译层，接口保持可测可替。
2. **目录/正文两级加载**：技能系统用"常驻目录 + 按需正文"实现了 prompt 经济学——这与 system-prompt 组装的"段 + step 准入 schema"一脉相承。
3. **指令与能力分家**：技能教方法，工具给手段；模型先读技能再调工具的编排让长流程可复用。
