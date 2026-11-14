# 总览篇 — 全局架构

> 三章建立全局心智模型：库长什么样、一次对话怎么跑、程序怎么启动。

| 章节 | 内容 |
|------|------|
| [全局架构与包地图](01-global-architecture.md) | 六层 ASCII 架构图、53 个包组地图、三类事件域、能力缝三角色、"新行为放哪里"速查表 |
| [一次对话的完整生命周期](02-turn-lifecycle.md) | turn/step 逐步走读：15 步编号时序图 + 每阶段的源码映射表——**全库最核心的执行路径** |
| [启动流程与 Profile/Bundle 组合](03-boot-and-profiles.md) | 从 `dsh web` 到插件树就绪的 9 步启动序列、五种内置 profile、patch 层叠、HMR |

**下一篇**：[核心引擎篇 — Session 模块](../02-core-engine/session/)
