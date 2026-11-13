# deepseek-harness-tutorial

DeepSeek Harness (dsh) 的系统性中文教程 —— **从入门到精通**，独立于主仓库的文档仓。

- 在线阅读：构建后由 GitHub Pages 提供（见下方"构建"）。
- 源码仓库：<https://github.com/deepseek-ai/deepseek-harness>
- 教程覆盖：Cordis 框架、核心引擎（session / agent-loop / tools / llm）逐文件精读、能力缝家族、持久化、产品形态、测试与扩展实战。

## 目录结构

```text
docs/
  index.md              首页：架构总图 + turn 流程图 + 代码映射表
  00-getting-started/   入门篇：背景知识、Cordis 框架、阅读指南
  01-overview/          总览篇：全局架构、turn 生命周期、启动与 Profile
  02-core-engine/       核心引擎篇：8 个核心包逐文件精读
  03-capability-seams/  能力缝篇：fs/shell/沙箱/终端/PTC/web/浏览器…
  04-persistence/       持久化篇：会话日志、格式迁移、投影、压缩、检索
  05-product-surfaces/  交互与产品形态篇：审批、Web、桌面、SDK/ACP
  06-mastery/           精通篇：测试、配置、扩展实战、架构模式总结
```

## 本地构建

```sh
pip install -r requirements.txt
mkdocs serve     # 本地预览 http://127.0.0.1:8000
mkdocs build     # 输出到 site/
```

## 进度

见 [TODO.md](TODO.md)。教程由 AI 辅助生成，基于主仓库 2026-11 前后的 master 分支；主仓库处于 developer preview，API 可能变化，请以源码为准。
