# LobsterAI 登录授权 & LobsterAI Server 自有代理 完整流程分析

> 分析日期：2026-04-10  
> 基于源码：`src/main/main.ts`、`src/renderer/services/auth.ts`、`src/main/libs/claudeSettings.ts`、`src/main/libs/openclawTokenProxy.ts`、`src/main/libs/openclawConfigSync.ts`、`src/main/libs/endpoints.ts`、`src/main/sqliteStore.ts`

---

## 一、登录授权完整流程

### 1.1 架构概述

LobsterAI 使用基于 **深链接（custom protocol）+ authCode 一次性令牌** 的 OAuth 2.0 风格登录方案，全程涉及：

| 参与方 | 职责 |
|--------|------|
| Electron 主进程 | 注册 `lobsterai://` 协议、处理深链接回调、与服务器换取 Token |
| Electron 渲染进程 | 触发登录、监听回调事件、更新 Redux 状态 |
| 系统浏览器 | 打开 Portal 页面完成实际登录 |
| LobsterAI Server | 提供登录页 URL、消费 authCode、签发 accessToken/refreshToken |
| SQLite (`lobsterai.sqlite` kv 表) | 持久化存储双 Token |

### 1.2 服务端环境选择

**关键文件：** `src/main/libs/endpoints.ts`

```typescript
export const getServerApiBaseUrl = (): string => {
  return isTestMode()        // 读 SQLite app_config.app.testMode
    ? 'https://lobsterai-server.inner.youdao.com'   // 测试/开发
    : 'https://lobsterai-server.youdao.com';        // 生产
};
```

`isTestMode()` 优先读 SQLite 中的 `app_config.app.testMode`，未初始化前回退为 `!app.isPackaged`（开发模式默认为 true）。

### 1.3 登录触发阶段

**关键文件：** `src/renderer/services/auth.ts`  
**入口：** `authService.login()`

```
用户点击"登录"按钮
  └── authService.login()
        └── fetchLoginUrl()
              └── GET https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/*/login-url
                    ├── 成功：取 data.value 作为登录页 URL
                    └── 失败：回退到 `${serverBaseUrl}/login`
        └── window.electron.auth.login(loginUrl)
              └── IPC: auth:login [main.ts ~L2019]
                    └── URL 附加 ?source=electron
                    └── shell.openExternal(finalUrl)  —— 打开系统浏览器
```

浏览器打开的是 **Portal 页面**（`lobsterai-server.youdao.com/login`），用户在此输入账号密码完成有道/网易账号体系的实际身份认证。

### 1.4 回调接收阶段

登录成功后，Portal 页面重定向到：

```
lobsterai://auth/callback?code=<authCode>
```

**macOS** 触发 `app.on('open-url')` 事件  
**Windows/Linux** 通过 `second-instance` 命令行参数捕获

**关键代码：** `src/main/main.ts L1689`

```typescript
const handleDeepLink = (url: string) => {
  const parsed = new URL(url);
  if (parsed.hostname === 'auth' && parsed.pathname === '/callback') {
    const code = parsed.searchParams.get('code');
    if (mainWindow && !mainWindow.isDestroyed()) {
      mainWindow.webContents.send('auth:callback', { code });   // 通知渲染进程
    } else {
      pendingAuthCode = code;   // 窗口未就绪则缓存，等渲染初始化后主动取
    }
  }
};
```

**窗口未就绪的兜底：** 渲染进程 `AuthService.init()` 调用 `auth:getPendingCallback` IPC 主动拉取缓存的 authCode。

### 1.5 Token 换取阶段

**关键代码：** `src/main/main.ts L2036` + `src/renderer/services/auth.ts L99`

```
渲染进程收到 auth:callback 事件
  └── authService.handleCallback(code)
        └── window.electron.auth.exchange(code)
              └── IPC: auth:exchange [main.ts L2036]
                    └── POST https://lobsterai-server.youdao.com/api/auth/exchange
                          Body: { authCode: code }
                    └── 响应: { code: 0, data: { accessToken, refreshToken, user, quota } }
                    └── saveAuthTokens(accessToken, refreshToken)  —— 写入 SQLite kv['auth_tokens']
        └── store.dispatch(setLoggedIn({ user, quota }))
        └── authService.loadServerModels()
              └── GET /api/models/available  —— 获取该用户可用的模型列表
              └── store.dispatch(setServerModels(models))
              └── updateServerModelMetadata(models)  —— 写入主进程内存缓存
```

### 1.6 Token 持久化与恢复

**关键文件：** `src/main/main.ts`（`saveAuthTokens` / `getAuthTokens`）  
**存储介质：** SQLite `kv` 表，key = `auth_tokens`

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "..."
}
```

- `accessToken` 有效期 **2 小时**，JWT 结构，可本地解析 `exp`
- `refreshToken` 有效期 **30 天**，每次刷新后签发新值（Rolling Refresh Token）

**应用启动恢复：**

```
App 启动 → AuthService.init()
  └── window.electron.auth.getUser()
        └── IPC: auth:getUser [main.ts L2069]
              └── getAuthTokens()  —— 读 SQLite
              └── fetchWithAuth(GET /api/user/profile)  —— 验证 Token 有效性
              └── 成功 → setLoggedIn()；失败 → setLoggedOut()
```

### 1.7 主动 Token 刷新（Proactive Refresh）

**关键代码：** `src/main/main.ts L4782`

每次 `authTokensGetter` 被调用时，主进程会解析 JWT 的 `exp` 字段，若距过期 **< 5 分钟**则异步触发后台刷新（fire-and-forget）：

```typescript
setAuthTokensGetter(() => {
  const tokens = getAuthTokens();
  const payload = JSON.parse(Buffer.from(tokens.accessToken.split('.')[1], 'base64').toString());
  const expiresAt = payload.exp * 1000;
  if (expiresAt - Date.now() < 5 * 60 * 1000) {
    void refreshOnce('proactive');   // 后台异步刷新，不阻塞调用方
  }
  return tokens;
});
```

`refreshOnce()` 有并发去重保护（`pendingTokenRefresh` Promise 复用），防止 Rolling refreshToken 被重复消费。

### 1.8 被动 Token 刷新（401/403 触发）

当 API 请求返回 401/403，代理层自动换新 Token 并重试。注册位置：`src/main/main.ts L4811`

```typescript
registerProxyTokenRefresher('lobsterai-server', async () => {
  const resp = await net.fetch(`${serverBaseUrl}/api/auth/refresh`, {
    method: 'POST',
    body: JSON.stringify({ refreshToken: tokens.refreshToken }),
  });
  saveAuthTokens(body.data.accessToken, body.data.refreshToken || tokens.refreshToken);
  return body.data.accessToken;
});
```

### 1.9 退出登录

```
authService.logout()
  └── POST /api/auth/logout  (best-effort，失败不影响本地清除)
  └── clearAuthTokens()      —— 删除 SQLite kv['auth_tokens']
  └── clearServerModelMetadata()
  └── store.dispatch(setLoggedOut())
  └── store.dispatch(clearServerModels())
```

### 1.10 登录完整时序

```
用户        渲染进程         主进程          浏览器        LobsterAI Server
 |              |               |               |                |
 | 点击登录      |               |               |                |
 |────────────► |               |               |                |
 |              | auth.login()  |               |                |
 |              |──────────────►|               |                |
 |              |               | GET /login-url|                |
 |              |               |────────────────────────────────►|
 |              |               |◄────────────────────────────────|
 |              |               | openExternal(loginUrl)          |
 |              |               |──────────────►|                |
 |              |               |               | 显示登录页      |
 | 输入账号密码   |               |               |                |
 |───────────────────────────────────────────── ►|               |
 |              |               |               | 验证成功        |
 |              |               |               | 302 → lobsterai://auth/callback?code=xxx
 |              |               | open-url 事件  |                |
 |              |               |◄──────────────|                |
 |              | auth:callback |               |                |
 |              |◄──────────────|               |                |
 |              | exchange(code)|               |                |
 |              |──────────────►|               |                |
 |              |               | POST /api/auth/exchange        |
 |              |               |────────────────────────────────►|
 |              |               |◄─────────────── {accessToken, refreshToken}
 |              |               | saveAuthTokens(SQLite)          |
 |              | setLoggedIn() |               |                |
 |              |◄──────────────|               |                |
 | 登录成功      |               |               |                |
```

---

## 二、未配置 Provider 时走 lobsterai-server 代理完整流程

### 2.1 触发条件

当用户**没有在设置中启用任何第三方 Provider**（所有 provider 的 `enabled: false` 或无 apiKey），AI 功能通过以下路径降级到 **LobsterAI Server 自有代理**。

> **前提：** 用户已登录（有有效的 accessToken + 服务端返回了可用模型列表）。

### 2.2 Provider 解析优先级（claudeSettings.ts: resolveMatchedProvider）

**关键文件：** `src/main/libs/claudeSettings.ts L158`

```
resolveCurrentApiConfig()
  └── resolveMatchedProvider(appConfig)
        ├── 1. 读 defaultModelProvider（SQLite app_config.model.defaultModelProvider）
        │     如果是 'lobsterai-server' → 直接 tryLobsteraiServerFallback(modelId)
        ├── 2. 如果指定其他 provider 且 enabled=true、有 apiKey → 直接使用该 provider
        ├── 3. 找不到匹配 provider → resolveFallbackModel()
        │     └── 遍历所有 providers，取第一个 enabled=true 且有 models 的
        │           ├── 找到 → 使用该 provider
        │           └── 找不到 → tryLobsteraiServerFallback(modelId)  ←【无配置时走这里】
        └── 4. 全部失败 → 返回 error: "No available model configured"
```

### 2.3 tryLobsteraiServerFallback 详解

**关键代码：** `src/main/libs/claudeSettings.ts L139`

```typescript
function tryLobsteraiServerFallback(modelId?: string): MatchedProvider | null {
  const tokens = authTokensGetter?.();           // 获取当前 Token（含主动刷新触发）
  const serverBaseUrl = serverBaseUrlGetter?.();
  if (!tokens?.accessToken || !serverBaseUrl) return null;  // 未登录 → null（不降级匿名）
  if (!modelId?.trim()) return null;             // 无 modelId → null

  const baseURL = `${serverBaseUrl}/api/proxy/v1`;

  return {
    providerName: 'lobsterai-server',
    providerConfig: {
      enabled: true,
      apiKey: tokens.accessToken,                // accessToken 作为 apiKey
      baseUrl: baseURL,
      apiFormat: 'openai',
    },
    modelId: modelId.trim(),
    apiFormat: 'openai',
    baseURL,
  };
}
```

**安全关键点：** 未登录时返回 `null`，系统报错无法使用，不存在匿名访问。

### 2.4 可用模型来源

登录成功后（`authService.loadServerModels()`），系统从服务端获取用户配额内可用的模型列表：

```
GET https://lobsterai-server.youdao.com/api/models/available
  └── 响应：[{ modelId, modelName, provider, apiFormat, supportsImage }]
  └── updateServerModelMetadata(models)  —— 写入主进程内存 Map
  └── store.dispatch(setServerModels(models))  —— 渲染进程展示（providerKey: 'lobsterai-server'）
```

用户在 UI 中选择服务器模型后，modelId 存入 `app_config.model.defaultModel`，`tryLobsteraiServerFallback` 据此构建请求。

### 2.5 OpenClaw Token Proxy（核心转发机制）

#### 设计动机

OpenClaw 网关通过静态配置文件（`openclaw.json`）读取 provider 的 baseUrl 和 apiKey，无法动态感知 Token 刷新。引入轻量级本地 HTTP 代理，让 Token 在代理层动态注入，对 OpenClaw 透明。

#### Token Proxy 生命周期

**启动（`src/main/main.ts L4849`）：**

```typescript
await startOpenClawTokenProxy({
  getAuthTokens,            // 实时读 SQLite Token
  refreshToken: refreshOnce, // 单例 Token 刷新
  getServerBaseUrl: getServerApiBaseUrl,
});
// 绑定 127.0.0.1:<随机端口>，不对外暴露
```

**OpenClaw 配置中的描述（`src/main/libs/openclawConfigSync.ts L422`）：**

```typescript
[ProviderName.LobsteraiServer]: {
  normalizeBaseUrl: (url) => {
    const proxyPort = getOpenClawTokenProxyPort();
    return proxyPort
      ? `http://127.0.0.1:${proxyPort}/v1`   // 指向 Token Proxy
      : stripChatCompletionsSuffix(url);
  },
  resolveApiKey: () => proxyPort ? 'proxy-managed' : `\${LOBSTER_APIKEY_SERVER}`,
},
```

写入 `openclaw.json` 的实际值：
- `baseUrl = "http://127.0.0.1:<port>/v1"`
- `apiKey = "proxy-managed"`

#### Token Proxy 请求处理

**关键文件：** `src/main/libs/openclawTokenProxy.ts L79`

```
OpenClaw 网关 → POST http://127.0.0.1:<port>/v1/chat/completions
  └── handleRequest()
        ├── tokens = getAuthTokens()  （含主动刷新检查）
        ├── upstreamPath = '/api/proxy' + req.url
        │   → '/api/proxy/v1/chat/completions'
        ├── upstreamUrl = serverBaseUrl + upstreamPath
        │   → 'https://lobsterai-server.youdao.com/api/proxy/v1/chat/completions'
        ├── forwardRequest(upstreamUrl, 'POST', accessToken, body)
        │     headers: { Authorization: `Bearer ${accessToken}` }
        │     支持 SSE 流式透传
        ├── 301 成功 → pipe 响应（流式或 Buffer）
        └── 401/403 → tokenRefresher('openclaw-proxy') → 用新 Token 重试一次
```

### 2.6 完整数据流图

```
用户发消息（未配置任何 Provider，已登录）
          │
          ▼
  resolveMatchedProvider()
  ├── 所有 provider disabled/无配置
  └── resolveFallbackModel() → 无结果
        └── tryLobsteraiServerFallback(modelId)
              ├── !accessToken → null（报错，用户需登录）
              └── 构造 MatchedProvider {
                    providerName: 'lobsterai-server',
                    baseURL: 'https://lobsterai-server.youdao.com/api/proxy/v1',
                    apiKey: accessToken,
                    apiFormat: 'openai'
                  }
          │
          ▼
  openclawConfigSync.sync()
  └── 写入 openclaw.json:
      providers['lobsterai-server'].baseUrl = "http://127.0.0.1:<proxyPort>/v1"
      providers['lobsterai-server'].apiKey  = "proxy-managed"
          │
          ▼
  OpenClaw 网关 执行 Agent
  └── POST http://127.0.0.1:<proxyPort>/v1/chat/completions
          │
          ▼
  OpenClaw Token Proxy (openclawTokenProxy.ts)
  ├── 读最新 accessToken（主动刷新检查）
  └── POST https://lobsterai-server.youdao.com/api/proxy/v1/chat/completions
        Header: Authorization: Bearer <accessToken>
        Body: { model: "xxx", messages: [...], stream: true }
          │
          ▼
  LobsterAI Server（有道域名）
  ├── 验证 accessToken + 用户配额
  └── 按配额转发到内部 LLM Provider
  └── SSE 流式响应
          │
          ▼
  Token Proxy 透传 SSE 流
  → OpenClaw 网关接收
  → IPC 推送渲染进程
  → 用户看到流式消息
```

---

## 三、关键安全注意事项

| 项目 | 实现 |
|------|------|
| Token Proxy 网络访问 | 仅 `127.0.0.1`，不对局域网/外网暴露 |
| Token 存储方式 | SQLite 明文存储，无加密（潜在风险点） |
| accessToken 有效期 | 2 小时，JWT 可本地解析 `exp` |
| refreshToken 有效期 | 30 天，每次刷新签发新值（Rolling） |
| 未登录保护 | `tryLobsteraiServerFallback` 检测无 Token 时返回 `null` |
| 401 自动重试 | 代理层捕获 401/403，自动换新 Token 重试一次 |
| refreshToken 并发保护 | `pendingTokenRefresh` 去重，防止 Rolling Token 被消费两次 |

---

## 四、两种调用路径对比

| 对比项 | 配置了第三方 Provider | 未配置（lobsterai-server 代理） |
|--------|----------------------|---------------------------------|
| 请求目标 | 第三方 LLM API 直连 | `lobsterai-server.youdao.com/api/proxy/v1` |
| 认证方式 | 第三方 API Key | LobsterAI accessToken |
| 配额控制 | 第三方自行管理 | LobsterAI Server 统一管理 |
| Token 刷新 | 无需（静态 API Key） | 自动刷新（主动 + 被动双重机制） |
| 数据路径 | 数据直达第三方 LLM | 经 LobsterAI Server 中转 |
| 离线可用 | 只需第三方网络可达 | 依赖 LobsterAI Server 可用 |
| 用户门槛 | 需要自备各平台 API Key | 仅需 LobsterAI 账号 + 配额 |

---

## 五、关键文件索引

| 文件 | 职责 |
|------|------|
| `src/main/libs/endpoints.ts` | 服务器 base URL 决策（prod/test/inner） |
| `src/main/main.ts` L1680 | `lobsterai://` 协议注册与深链接处理 |
| `src/main/main.ts` L2019 | `auth:login` IPC（打开系统浏览器） |
| `src/main/main.ts` L2036 | `auth:exchange` IPC（authCode → Token） |
| `src/main/main.ts` L2069 | `auth:getUser` IPC（启动时恢复登录） |
| `src/main/main.ts` L4744 | `refreshOnce()` 单例 Token 刷新（去重保护） |
| `src/main/main.ts` L4782 | `setAuthTokensGetter`（含主动刷新 5min 触发） |
| `src/main/main.ts` L4811 | `lobsterai-server` 代理 Token 刷新器注册 |
| `src/main/main.ts` L4849 | Token Proxy 启动 |
| `src/main/libs/claudeSettings.ts` L139 | `tryLobsteraiServerFallback()` |
| `src/main/libs/claudeSettings.ts` L158 | `resolveMatchedProvider()` Provider 优先级解析 |
| `src/main/libs/openclawTokenProxy.ts` | Token 注入代理（本机 HTTP 服务） |
| `src/main/libs/openclawConfigSync.ts` L422 | lobsterai-server 在 OpenClaw 配置中的描述 |
| `src/renderer/services/auth.ts` | 渲染进程 AuthService |
| `src/main/sqliteStore.ts` | Token 持久化（kv 表，key=`auth_tokens`） |

---

## 六、lobsterai-server 的系统归属

### 6.1 是自有系统还是第三方服务？

**`lobsterai-server` 是 LobsterAI 产品的自有后端服务，不是引入的第三方外部系统。**

判断依据：

| 维度 | 说明 |
|------|------|
| 域名归属 | `lobsterai-server.youdao.com` / `lobsterai-server.inner.youdao.com`，完全在有道/网易的域名体系下 |
| 接口功能 | 全部是 LobsterAI 产品专属接口（Token 换取/刷新、用户配额、LLM 代理等），无通用第三方 API 特征 |
| 数据链路 | 客户端 → LobsterAI Server → 内部 LLM 资源，整个链路在有道体系内闭环 |
| 测试环境 | 有独立内网域名（`.inner.youdao.com`），说明有完整的研发运维体系 |

### 6.2 代码是否在本仓库？

**不在本仓库（`LobsterAI/`）中。**

本仓库目录结构只包含：

| 目录/文件 | 内容 |
|-----------|------|
| `src/` | Electron 客户端（主进程 + 渲染进程） |
| `SKILLs/` | 自定义技能定义 |
| `openclaw-extensions/` | OpenClaw 插件 |
| `vendor/` | 本地 vendor 依赖 |
| `scripts/`、`build/`、`docs/` | 构建/文档等辅助目录 |

**完全没有任何服务端代码。**

`lobsterai-server` 只以"远程 HTTP 服务"的形式出现在客户端代码中：

- **硬编码 URL**：`src/main/libs/endpoints.ts` 中的 `lobsterai-server.youdao.com`
- **HTTP 调用**：客户端通过 `net.fetch` 调用其接口
- **无服务端实现**：仓库内无 server 源码、无数据库 Schema、无 Dockerfile 等

### 6.3 系统架构定位

```
┌─────────────────────────────────────────┐
│        本仓库（LobsterAI 客户端）         │
│                                         │
│   Electron 主进程 + React 渲染进程       │
│   OpenClaw Token Proxy（本机回环）        │
└─────────────────┬───────────────────────┘
                  │ HTTPS
                  ▼
┌─────────────────────────────────────────┐
│  lobsterai-server（有道自有后端·独立仓库） │
│                                         │
│   用户账号 & Token 管理                  │
│   用户配额控制                           │
│   LLM 请求代理转发                       │
└─────────────────┬───────────────────────┘
                  │ 内部调用
                  ▼
┌─────────────────────────────────────────┐
│       有道内部 LLM 资源 / 模型服务         │
└─────────────────────────────────────────┘
```

**结论：** `lobsterai-server` 的服务端代码在有道/网易内部的另一个独立仓库中维护，本项目仅是其客户端消费方。如需修改服务端接口行为，需联系负责 `lobsterai-server` 的后端团队。
