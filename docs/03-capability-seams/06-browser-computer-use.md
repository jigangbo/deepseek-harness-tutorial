# 浏览器与计算机操作 — 互斥命名注册的两个缝

> 让 agent 操作浏览器（browser-use）与整台计算机（computer-use）。这两个缝与 fs/shell 的形状刻意不同：**每个服务实例每个名字只允许一个 provider**，且服务本身没有通用操作 API——因为不同后端的能力差异太大，强行统一接口反而失真。

## browser-use（ctx.browserUse）

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-browser-use`（`ctx.browserUse`） |
| Provider（实验） | `browser-use-playwright-mcp`、`browser-use-chrome-devtools-mcp`、`browser-use-stagehand-native` |

设计要点（来自缝定义）：

- **每个服务实例一个 provider 拥有的名字**——注册即占名，重复注册冲突；
- **Provider 自带工具与浏览器资源**（按活会话）：工具名如 `stagehand_*` 出现在模型请求里，是 provider 的贡献，不是缝的；
- **共享服务没有通用浏览器操作 API**——Playwright MCP、Chrome DevTools MCP、Stagehand native 各有不同的动作词汇，缝不假装它们相同。

```text
ctx.browserUse（只管：占名 + 生命周期）
   ├─ 注册：provider 用自己的名字占位（每实例每名一个）
   ├─ 工具：provider 注册自己的动作工具（stagehand_* 等）
   └─ 资源：provider 声明自己的浏览器资源（按活会话）
缝里没有：通用操作 API / 运行时选择 / 会话工作流锁
```

这就是「互斥命名注册」模式：缝的角色退化为**占位与生命周期**，能力表达完全归 provider。UI 侧的 browser 面板（ui-sidebar-browser）消费 provider 声明的资源。

## computer-use（ctx.computerUse）

| 角色 | 包 |
|------|-----|
| Service Definition | `dsh-computer-use`（`ctx.computerUse`） |
| Provider（实验） | `computer-use-cua-driver-mcp`、`computer-use-cua-driver-native` |

同样的互斥命名模式，且**每个 provider 也拥有自己的模型工具**；服务没有通用动作 API、运行时选择或会话工作流锁。

### 权限控制：ctx.cuaAccess

`experimental/cua`（`ctx.cuaAccess`）拥有 Computer Use 的权限控制：

- **原生 provider 拥有应用同意与取消**；
- 可选的 Client 页面通过其 Remote 查询与变更这些决定；
- CUA（Computer Use Agent）在 macOS 上还有专门的签名注意事项（`dsh-cua-macos-signing` 技能）——屏幕控制需要辅助功能权限，固定签名密钥避免反复授权。

## 实验状态说明

browser-use 与 computer-use 的全部 provider 都在 `packages/experimental/` 下：[实验状态](../../packages/experimental/README.md#status)独立于交付与否——包可以已发布但仍标记实验性，表示"评估中的能力，无产品支持承诺"。

## 与其他缝的对照

| | fs/shell 缝 | browser/computer 缝 |
|---|---|---|
| 接口 | 统一操作词汇（读/写/执行） | 无通用操作 API |
| 注册 | 多 provider 并存 | 每名字一个 provider，互斥 |
| 工具 | Consumer 工具拥有稳定名 | Provider 自带工具 |
| 适合场景 | 能力语义收敛的领域 | 后端能力差异巨大的领域 |

**缝不是教条**：当统一接口需要靠"lowest common denominator"压平差异时，dsh 选择让 provider 全权表达、缝只管注册与生命周期。判断标准是"消费者是否真的可以用同一词汇完成任务"。

## 设计启发

1. **接口统一是收益不是义务**：两种缝形态并存，教读者按领域语义选型。
2. **工具所有权跟 provider**：谁定义动作词汇，谁注册工具——避免"缝里长满 provider 特有开关"。
3. **权限内建在缝旁**：CUA 的同意/取消是一等公民（cuaAccess），不是后实现的补丁。
