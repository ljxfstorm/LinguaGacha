# 内置网络代理设置

> 本文是本次开发的 PR 说明与设计文档。PR 打开后内容转为 GitHub 描述，本文件随本地脚手架一起删除，不回合上游。

## 一句话

应用内新增「网络」设置，提供直连、跟随系统、手动配置三种代理模式，让翻译、Agent 与更新检查使用同一份代理设置。

## 背景

应用的网络请求分两套栈：

- **Chromium 栈**：更新检查、更新包下载。默认跟随操作系统代理。
- **undici 栈**：翻译、OAuth、模型目录、Web Search、Agent 工作区。每个请求通过 Electron 的 `resolveProxy` 读取系统代理规则。

两套栈目前都只认系统代理。用户无法在应用内强制直连，也无法指定应用内代理。既有统一传输（#765）保证 undici 栈只有一个出站入口，本次改动不改变这一点。

## 目标

- 提供三种代理模式：直连、跟随系统（默认）、手动。
- 手动模式支持 HTTP、HTTPS、SOCKS5，可选用户名与密码。
- 两套栈使用同一份设置，行为可预期。
- 保持既有语义：回环地址直连；代理失败即报错，不静默直连；CLI 与 GUI 共用设置。

## 非目标

- 例外域名列表（`NO_PROXY`）。
- 环境变量代理（`HTTP_PROXY` 等）。
- 凭据加密存储。与现有 API Key 一致，明文写入 `config.json`。

## 模式

| 模式 | 含义 |
|---|---|
| 直连 | 所有出站请求不经过代理。 |
| 跟随系统 | 使用操作系统代理设置。默认值，行为与当前版本一致。 |
| 手动 | 使用应用内配置的代理服务器。 |

手动配置字段：协议（HTTP / HTTPS / SOCKS5）、主机、端口、可选用户名与密码。端口限制 1–65535，主机不能为空。

## 两套栈的行为

| 模式 | Chromium 栈 | undici 栈 |
|---|---|---|
| 跟随系统（默认） | 不变 | 不变，每请求读取系统规则 |
| 直连 | `setProxy({ mode: "direct" })` | 直连连接池 |
| 手动 · HTTP/HTTPS 无认证 | `proxyRules` | `ProxyAgent` |
| 手动 · HTTP/HTTPS 有认证 | `proxyRules` + `login` 事件回答凭据 | `ProxyAgent` token |
| 手动 · SOCKS5 无认证 | SOCKS 规则 | `Socks5ProxyAgent` |
| 手动 · SOCKS5 有认证 | 不下发规则，更新流量维持系统行为 | URI userinfo |

回环地址在两套栈都保持直连。代理不可用时请求直接失败，不回退直连。

## 设置如何生效

设置仍由 `AppSettingService` 唯一读写，main 进程不直接读 `config.json`。新增宿主操作 `apply_proxy`：后端把模式与手动配置推给 main，main 调用 `session.setProxy` 后回执。

- **启动**：`apply_proxy` 插在 `BackendResources.start()` 之后、`gateway.start()` 之前。这样启动期唯一的自动请求（模型目录检查）发出时，路由已经生效。
- **运行期**：设置保存要等 main 回执后才返回成功，保证界面显示成功时两套栈已使用新配置。
- **跟随系统**：不下发 `setProxy`，与当前行为完全一致。

### 为什么不各栈自行判断

- 设置只有一个读写入口，main 直接读文件会形成第二个读取方，长期容易漂移。
- 路由事实只有一份（Chromium session），undici 继续通过 `resolveProxy` 复用，不新增第二套判断逻辑。
- 代价是启动时多一次跨线程往返。用回执固定顺序，避免“先请求后生效”。

## 凭据

- undici 栈：HTTP/HTTPS 用 `ProxyAgent` 的 token，SOCKS5 用 URI userinfo。
- Chromium 栈：HTTP 代理认证由 `app.on('login')` 回答。
- SOCKS5 + 认证：Chromium 网络栈不支持该组合（上游限制）。此时不下发规则，更新流量维持系统行为，翻译链路正常。设置页常驻提示，日志同时记录。

## 实施计划

按提交顺序：

1. `src/domain/setting.ts`：新增模式与手动字段，补默认值与归一化。
2. `src/backend/network/system-proxy-http-client.ts`：手动模式注入凭据，路由解析逻辑不变。
3. 宿主协议与入口：新增 `apply_proxy`，GUI 与 CLI 应用 `setProxy`，启动与变更时序。
4. `src/backend/agent/workspace/runtime/`：子进程按 `resolve_proxy` 应答注入凭据，wire 协议不变。
5. `src/frontend/pages/basic-settings-page/`：新增「网络」分组、手动字段与「测试连接」，补五语文案。
6. 日志与文档：模式变更、测试结果、SOCKS5 限制写日志；更新 `docs/BACKEND.md` 与 `docs/FRONTEND.md`。

## 测试连接

设置页提供「测试连接」：后端经同一传输请求 `https://www.gstatic.com/generate_204`，并回报实际使用的路由（直连或代理地址）。结果写入日志窗口。

## 验证

- 静态检查：`npm run typecheck`、`npm run lint`、`npm run check`。
- 行为测试：设置归一化、凭据 dispatcher、`apply_proxy` 启动与变更时序、工作区注入、设置页交互、i18n 键完整性。
- 集成：`BackendResources` 与 `GuiBackendBootstrap` 集成测试。
- 真机：`npm run dev` 验证手动代理（HTTP、SOCKS5 各一）、直连模式、测试连接与更新检查。
- 实际结果在交付时补入本节。

## 已知限制

- SOCKS5 用户名/密码认证在 Chromium 栈不可用（上游 Chromium issue 40323993）。该组合下更新检查与下载维持系统代理行为。
- 不支持例外域名、环境变量代理与凭据加密。
