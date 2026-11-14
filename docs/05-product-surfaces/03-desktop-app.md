# 5.4 桌面应用架构

> Electron 壳与共享 Web 应用


> Desktop 不是"又一个产品"——它是**同一套 dsh Web 应用的原生外壳**。理解这一点，就理解了它所有的技术决策。核心文档：`apps/desktop/README.md`。

## 总体形态

```text
┌────────────────────────────────────────────────────┐
│ Electron 窗口                                       │
│  ┌──────────────────────────────────────────────┐ │
│  │ 加载页 → 等待 Host boot 注入 → 启动客户端     │ │
│  │ （同一个文档，不跳转）                        │ │
│  └────────────────▲─────────────────────────────┘ │
│         RunAsNode │ 子进程          │ HTTP/WS 转发  │
│  ┌────────────────┴──────────────┐ │              │
│  │ dsh 共享 profile runner       │◄┘ 带凭证       │
│  │ （完整 Web 应用 + Host）      │                │
│  └───────────────────────────────┘                │
│  Node IPC：boot 注入 / 就绪 / 关停                │
└────────────────────────────────────────────────────┘
```

关键事实：

- Electron RunAsNode 子进程启动**共享 profile runner**（与 CLI/npm 完全相同的启动路径），Electron 立即加载打包的 Web 入口 `dsh-app://app/`。
- 共享加载页等待 Host boot 注入，然后在**同一个文档**里启动客户端——没有第二个跳转文档。
- **Node IPC 携带 boot 注入、就绪与关停**；Web 拥有 RPC 与流，桌面载体只负责把本地页面连到已认证的 Host。

## 网络与安全细节（值得抄的作业）

| 决策 | 理由 |
|------|------|
| 默认监听 **OS 分配端口**，绑定 `127.0.0.1` | 不与 Web 的 3080 或保留端口冲突；`webserver.config.port` patch 可覆盖 |
| Electron 转发应用 HTTP 请求，**丢弃连接级响应头**（`transfer-encoding`/`connection`/`keep-alive`） | 这些头描述的是 Node fetch 而不是资源本身 |
| 插件 bundle 响应标 `no-store` | bundle 按启动修订变化，存 Chromium 磁盘缓存只会积累垃圾 |
| WebSocket 凭证只对**本应用窗口**附加 | 凭证要求 authority 与 scheme 完全匹配 |
| **没有任何 Electron session 存储 Host cookie**；每次 Host 启动清除旧版本留下的 `dsh-auth-*` | cookie 忽略端口——否则会跟着发往 `127.0.0.1` 的每个请求 |

## 安装与运行时所有权

- Desktop 与 npm CLI **共享产品数据**，但保持**包、激活与锁文件分离**。
- 桌面打包把精确的 dsh 生产运行时装进签名资源，拥有保留的 `$DSH_HOME/profiles/desktop`。
- 共享助手初始化 profile 文件、调和 bundle、解析安装与 bundle 依赖——**不替换 pnpm 拥有的包**。
- Shell 拥有的 UI 通过**捆绑的 pnpm** 跑插件事务（正常的用户与 profile 配置）；桌面的捆绑 CLI 管理它初始化的插件。
- 启动恢复对话框用的 UI 语言缓存在 `userData/desktop/locale-cache`——Host 设置可用之前也能用正确语言；缓存失败不阻塞启动。
- 更新安装等待本地分析数据摄入（1 秒截止）后才锁定 API 准入并停 Host——摄入失败不阻塞安装。

## 与其他产品面的同构性

```text
npm CLI:   dsh ──► profile runner ──► Web 应用 ──► 浏览器打开 127.0.0.1:3080
Desktop:   Electron ──► 同一个 profile runner ──► 同一个 Web 应用 ──► 装进原生窗口
Python SDK: deepseek-harness ──► 启动 dsh --profile sdk ──► JSON-RPC over stdio
```

三个载体、一个运行时。这就是「只有 dsh profile 能启动受支持的 Node 应用」纪律的红利：桌面不需要发明第二条启动路径。

## 已知边界

- 打包/Windows 签名/发布属于独立指南（`apps/desktop/windows-signing/README.md`）。
- 低层开发 override（`DSH_*` 环境变量类）支持本地开发替换打包运行时。
- 恢复与诊断文档在 `apps/desktop/docs/diagnostics/`。

## 设计启发

1. **壳与运行时分离**：Electron 只做窗口、转发、凭证过滤三件事；所有产品逻辑在共享 runner 里——桌面跟随 Web 演化，零分叉维护。
2. **cookie 与端口的陷阱**：`127.0.0.1` 上 cookie 忽略端口是真实攻击面——「每次启动清除 + 不持久化 + authority 匹配」三层防御值得记住。
3. **缓存乐观、失败降级**：语言缓存、分析摄入都遵循"尽力做、失败不阻塞主路径"。
