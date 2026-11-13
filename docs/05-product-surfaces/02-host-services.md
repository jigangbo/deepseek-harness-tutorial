# Host 服务层 — Web 服务器与宿主能力

> Host 是 Web GUI 的进程内宿主：一个 `node:http` 服务器（`packages/host/webserver`）加上目录选择、静态前端、插件清单、遥测等宿主能力。Host 不是 agent 循环的一部分，也不是能力缝——它只认识一个 harness 概念：index 渲染所代表的 Peer。

## Web 服务器（ctx.webServer）

`dsh-host-webserver` 是浏览器的 HTTP 载体：

- 一个 `node:http` 服务器（配置 `tls` 时用 `node:https`）；
- **命名路由注册表**：`exact`（逐字匹配）与 `prefix`（匹配 p 与 p/任意）两种；
- 可选 gzip 响应压缩、index.html 变换回调、一个**后备席位**（fallback）。

**匹配顺序固定**：exact 表 → 最长匹配前缀 → 注册的 fallback。注册顺序没有请求侧语义——命名路由被组合为互不相交，fallback 回答无人认领的一切；**只有一个 owner，第二个注册抛错**。

```ts
interface WebRoute {
  kind: 'exact' | 'prefix'
  path: string                    // 绝对路径，无尾斜杠
  handler: (req, res) => void | Promise<void>   // 拥有完整响应生命周期（可保持开放，如 SSE）
}

// 也可注册资源型路由：
type WebResource = (request: Request, peer?: PeerScope) => Promise<Response>
```

资源型路由（Fetch `Request` → `Response`）：HTTP GET/HEAD 无 Peer 到达；`readResource(request, peer)` 以调用者准入的 Peer 进程内读取，遵循同一匹配顺序但**只到达资源**——返回 owner 私有数据或执行动作的路由即使回答 GET 也必须是 handler。

## 静态前端（host-frontend-static）

交付组合用 `dsh-host-frontend-static` 占据 fallback 席位——SPA dist 服务器，语义锁定：

- Connection 先认证 dist 根与配置的 index，再读 HTML；
- **非 index 资产公开**；非 GET/HEAD 返回 405；
- dist 根之外的路径穿越返回 403；存在文件直接服务；缺席或非文件目标返回空 404；未知扩展按 octet-stream。

## 监听配置的安全语义

```ts
interface Config {
  host: string     // 具体 IPv4/IPv6 字面量
  port: number     // 0 = OS 分配
  tls?: TlsConfig
  compression?: 'none' | 'gzip'
}
```

- **环回字面量**（127/8、`::1` 及映射形）保持本机；其他字面量服务该地址所属网络（未设 tls 时是明文 HTTP）。
- **未指定地址（0.0.0.0 / :: 等）在加载时拒绝**——它会把端口暴露在所有接口上。
- TLS 材料只读一次：文件不可读/空/PEM 非法/密钥不匹配在初始化时拒绝，**不回退 HTTP**；替换证书需要 reload。

## 其他 Host 能力

| 包 | ctx 键 | 职责 |
|----|--------|------|
| `host/directory-picker` | `ctx.directoryPicker` | 工作区目录选择的判别式交互缝：native 后端开 OS 选择器，browse 后端为应用内浏览器提供列表/创建原语 |
| `host/frontend-static` | — | 上述 SPA dist 服务 |
| `host/plugin-inventory` | — | 插件清单面 |
| `host/product-telemetry-otel` | `ctx.productTelemetry` | OTLP/HTTP 导出**显式提交**的分析事件；**挂载本身不收集任何东西** |
| `host/open-in-app` | — | 应用内打开外链 |
| `bundle/web-app` | `ctx.webApp` | 宣告一个 URL 的 Web GUI：配置的 publicUrl，否则监听器 origin（`DSH_WEB_URL`、启动 URL 与 harness-dev-context 读同一个值） |

路由的消费者：`client-connection`（认证传输）、`client-modules`（插件 bundle 服务）、`client-hmr`（客户端 HMR 事件流）、`api-gateway`（/api 桥）——**包括 /api 在内的每个功能路由都由另一个插件注册**，webserver 自身零业务。

## 与 Desktop 的关系

webserver **只服务浏览器**：Electron 走 `file://` 加载构建产物、经 IPC 桥发送 fetch，不经过这个服务器（见桌面应用章）。

## 设计启发

1. **HTTP 载体零业务**：服务器只提供路由/压缩/变换/fallback 四个原语；每个功能路由（包括 /api 桥）都是别的插件注册的——机制与功能彻底分离。
2. **fallback 席位的唯一性**：一个席位一个 owner，抢占式组合变成显式声明。
3. **配置安全在加载时判定**：0.0.0.0 拒绝、TLS 材料预检且不回退——错误配置活不到运行时。
