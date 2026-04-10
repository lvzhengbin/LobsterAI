# LobsterAI 登录授权流程分析 & Lark 授权改造方案

> 文档创建时间：2026-04-09 / 最后更新：2026-04-09  
> 基于代码版本：当前 main 分支

---

## 一、现有登录授权流程深度解析

### 1.1 整体架构

现有认证系统采用 **"Portal 统一登录 → authCode 换 JWT 双 Token"** 的方案，分为以下几个层次：

| 层次 | 文件 | 职责 |
|------|------|------|
| 渲染层 UI | `src/renderer/components/LoginButton.tsx` | 登录按钮、用户信息展示 |
| 渲染层服务 | `src/renderer/services/auth.ts` | AuthService 封装，IPC 调用 |
| 渲染层状态 | `src/renderer/store/slices/authSlice.ts` | Redux 认证状态管理 |
| IPC 桥接 | `src/main/preload.ts` (L459–L477) | contextBridge 暴露 `window.electron.auth.*` |
| 主进程逻辑 | `src/main/main.ts` (L1912–L2174) | 所有 auth IPC handler、token 存取、HTTP 请求 |
| 持久化 | `src/main/sqliteStore.ts` | kv 表存储 `auth_tokens` |

### 1.2 完整时序（文字描述）

```
用户点击登录
  → AuthService.login()
    → fetchLoginUrl()：请求 overmind 服务获取 Portal 登录地址
    → window.electron.auth.login(loginUrl)
      → Main: shell.openExternal(loginUrl?source=electron)
        → 系统浏览器打开 Portal 登录页
          → 用户完成登录
            → Portal 重定向 lobsterai://auth/callback?code=<authCode>
              → Electron 触发 open-url (macOS) / second-instance (Win/Linux)
                → Main: handleDeepLink(url) 解析 code
                  → ipcRenderer.send('auth:callback', { code })
                    → AuthService.handleCallback(code)
                      → window.electron.auth.exchange(code)
                        → Main: POST /api/auth/exchange { authCode: code }
                          → Server 返回 accessToken(2h) + refreshToken(30d) + user + quota
                            → Main: saveAuthTokens → SQLite kv 'auth_tokens'
                              → dispatch(setLoggedIn) → Redux 更新 UI
                                → authService.loadServerModels() → 拉取可用模型
```

### 1.3 Token 生命周期管理

**启动恢复：**
- `authService.init()` → `window.electron.auth.getUser()` → Main 读 SQLite kv `auth_tokens` → fetchWithAuth `/api/user/profile` 验证
- 若有效：dispatch `setLoggedIn`；若失败：dispatch `setLoggedOut`

**被动刷新（代码位置：main.ts L1932–L1961）：**
```typescript
const fetchWithAuth = async (url, options) => {
  const tokens = getAuthTokens();           // 从 SQLite kv 读取
  let resp = await doFetch(tokens.accessToken);

  if (resp.status === 401 && tokens.refreshToken) {
    // 调用 /api/auth/refresh 换新 token
    const refreshResp = await net.fetch(`${serverBaseUrl}/api/auth/refresh`, {
      method: 'POST',
      body: JSON.stringify({ refreshToken: tokens.refreshToken })
    });
    if (refreshResp.ok) {
      saveAuthTokens(newAccessToken, newRefreshToken); // 持久化
      resp = await doFetch(newAccessToken);            // 重试原请求
    }
  }
  return resp;
};
```

**主动刷新：** 由 `openclawTokenProxy.ts` 管理，通过 `auth:refreshToken` channel 暴露给代理层。

**滚动续期：** 每次 `/api/auth/refresh` 都返回新 refreshToken（30 天有效期），连续使用不超过 30 天则不掉线。

**退出条件：** refreshToken 过期（30 天未使用）→ clearAuthTokens → 用户需重新登录。

### 1.4 关键 IPC Channels

| Channel | 方向 | 功能 |
|---------|------|------|
| `auth:login` | Renderer → Main | 打开系统浏览器到 Portal 登录页 |
| `auth:exchange` | Renderer → Main | 用 authCode 换取系统 JWT Token |
| `auth:getUser` | Renderer → Main | 获取用户 profile + quota（含 401 自动刷新） |
| `auth:getQuota` | Renderer → Main | 刷新配额信息 |
| `auth:getProfileSummary` | Renderer → Main | 获取积分明细列表 |
| `auth:logout` | Renderer → Main | 登出，清除本地 token |
| `auth:refreshToken` | Renderer → Main | 主动刷新 accessToken |
| `auth:getAccessToken` | Renderer → Main | 获取当前 accessToken（供 OpenClaw 代理注入） |
| `auth:getModels` | Renderer → Main | 获取服务端可用模型列表 |
| `auth:callback` | Main → Renderer | deep link 回调，携带 authCode |
| `auth:quotaChanged` | Main → Renderer | 服务端推送配额变更 |
| `auth:getPendingCallback` | Renderer → Main | 获取应用启动前缓存的 authCode |

### 1.5 关键文件位置速查

| 功能 | 文件 | 行范围 |
|------|------|--------|
| protocol 注册 & deep link 处理 | `src/main/main.ts` | L1667–L1727 |
| auth IPC handlers | `src/main/main.ts` | L1912–L2174 |
| fetchWithAuth (被动刷新) | `src/main/main.ts` | L1932–L1961 |
| saveAuthTokens / getAuthTokens | `src/main/main.ts` | L1917–L1927 |
| preload auth API | `src/main/preload.ts` | L459–L477 |
| AuthService (渲染层) | `src/renderer/services/auth.ts` | 全文 186 行 |
| authSlice (Redux) | `src/renderer/store/slices/authSlice.ts` | 全文 81 行 |

---

## 二、改造为仅 Lark 授权的完整方案

### 2.1 Lark OAuth 流程设计

Lark（飞书）支持标准 OAuth 2.0 授权码模式，桌面应用使用 custom scheme 接收回调：

```
1. 打开浏览器 → 飞书授权页
   URL: https://open.feishu.cn/open-apis/authen/v1/authorize
         ?app_id=<LARK_APP_ID>
         &redirect_uri=lobsterai://auth/lark/callback   ← URL encoded
         &scope=contact:user.base:readonly
         &state=<随机UUID，用于CSRF防护>

2. 用户在飞书完成授权（扫码 / 账号密码）

3. 飞书回调 → lobsterai://auth/lark/callback?code=<code>&state=<state>
   → Electron open-url / second-instance 触发
   → Main handleDeepLink 解析 /lark/callback 路径

4. 主进程调用后端 POST /api/auth/lark/exchange { code, state }
   → 后端用 code 调飞书 API 换 user_access_token
   → 获取用户信息(open_id, name, avatar_url)
   → 创建/匹配本地用户 → 签发系统 JWT

5. 后端返回 accessToken + refreshToken + user + quota
   → Main 存入 SQLite kv 'auth_tokens'
   → Renderer dispatch setLoggedIn → UI 更新
```

### 2.2 改造范围

**需要新增/改造的文件：**

| 文件 | 变更类型 | 说明 |
|------|---------|------|
| `src/main/main.ts` | 改造 | 扩展 `handleDeepLink` + 新增 2 个 IPC handler |
| `src/main/preload.ts` | 扩展 | 新增 `larkLogin`, `larkExchange`, `onLarkCallback` |
| `src/renderer/services/auth.ts` | 改造 | `login()` 改走 Lark，新增 `handleLarkCallback` |
| `src/renderer/components/LoginButton.tsx` | UI 改造 | 替换为飞书登录按钮 |
| `src/renderer/services/i18n.ts` | 新增 key | Lark 登录相关文案 zh/en |
| `src/main/authConstants.ts` (新建) | 新增 | AuthIpcChannel 常量 |

**可以完全复用的部分：**
- `fetchWithAuth` 被动刷新逻辑（不变）
- `saveAuthTokens / getAuthTokens / clearAuthTokens`（不变）
- `authSlice` Redux 状态（可选扩展，不破坏现有结构）
- `lobsterai://` protocol 注册（`setAsDefaultProtocolClient` 不变）
- `pendingAuthCode` 启动前缓冲机制（扩展复用）
- `/api/auth/refresh` 和 `/api/auth/logout` 后端接口（不变）

### 2.3 后端 API 设计（需新增）

```
POST /api/auth/lark/exchange
  Body: { code: string, state: string }
  处理：
    1. 调用飞书 /authen/v1/oidc/access_token 换取 user_access_token
    2. 调用飞书 /authen/v1/user_info 获取用户信息
    3. 用 open_id 或 union_id 查找/创建系统用户
    4. 签发系统 JWT: accessToken(2h) + refreshToken(30d)
  Response: {
    code: 0,
    data: { accessToken, refreshToken, user: { userId, nickname, avatarUrl, ... }, quota }
  }

POST /api/auth/refresh       ← 复用现有，无需变动
POST /api/auth/logout        ← 复用现有，无需变动
```

> **注意：** Lark 的授权码 `code` 有效期为 **5 分钟且只能使用一次**，后端需在此时间窗口内完成 exchange。

### 2.4 主进程改造（src/main/main.ts）

#### 2.4.1 扩展 handleDeepLink（支持 /lark/callback 路径）

```typescript
const handleDeepLink = (url: string) => {
  try {
    const parsed = new URL(url);
    if (parsed.hostname === 'auth') {
      // 现有 Portal 回调路径（如需保留可选）
      if (parsed.pathname === '/callback') {
        const code = parsed.searchParams.get('code');
        if (code) {
          if (mainWindow && !mainWindow.isDestroyed()) {
            mainWindow.webContents.send('auth:callback', { code });
          } else {
            pendingAuthCode = code;
          }
        }
      }
      // 新增：Lark OAuth 回调路径
      if (parsed.pathname === '/lark/callback') {
        const code = parsed.searchParams.get('code');
        const state = parsed.searchParams.get('state');
        if (code) {
          if (mainWindow && !mainWindow.isDestroyed()) {
            mainWindow.webContents.send('auth:lark:callback', { code, state });
          } else {
            pendingAuthCode = code; // 复用缓冲机制，可按需拆分
          }
        }
      }
    }
  } catch (e) {
    console.error('[Main] Failed to parse deep link:', e);
  }
};
```

#### 2.4.2 新增 auth:larkLogin IPC Handler

```typescript
// 放在 "── Auth IPC handlers ──" 区块内
// LARK_APP_ID 从企业配置或环境变量读取（不硬编码）
const getLarkAppId = (): string => {
  return (getStore().get<string>('lark_app_id') || process.env.LARK_APP_ID || '');
};

ipcMain.handle('auth:larkLogin', async () => {
  try {
    const appId = getLarkAppId();
    if (!appId) {
      return { success: false, error: 'Lark App ID not configured' };
    }
    const state = crypto.randomUUID(); // CSRF 防护
    getStore().set('lark_oauth_state', state);
    const redirectUri = encodeURIComponent('lobsterai://auth/lark/callback');
    const url = `https://open.feishu.cn/open-apis/authen/v1/authorize?app_id=${appId}&redirect_uri=${redirectUri}&scope=contact:user.base:readonly&state=${state}`;
    await shell.openExternal(url);
    return { success: true };
  } catch (error) {
    console.error('[Auth] Lark login failed:', error);
    return { success: false, error: error instanceof Error ? error.message : 'Failed to open Lark login' };
  }
});
```

#### 2.4.3 新增 auth:larkExchange IPC Handler

```typescript
ipcMain.handle('auth:larkExchange', async (_event, { code, state }: { code: string; state: string }) => {
  try {
    // CSRF state 校验
    const savedState = getStore().get<string>('lark_oauth_state');
    getStore().delete('lark_oauth_state');
    if (state && savedState && state !== savedState) {
      console.warn('[Auth] Lark OAuth state mismatch, potential CSRF');
      return { success: false, error: 'State mismatch' };
    }

    const serverBaseUrl = getServerApiBaseUrl();
    const resp = await net.fetch(`${serverBaseUrl}/api/auth/lark/exchange`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code, state }),
    });
    if (!resp.ok) {
      return { success: false, error: `Exchange failed: ${resp.status}` };
    }
    const body = await resp.json() as {
      code: number;
      message?: string;
      data: { accessToken: string; refreshToken: string; user: Record<string, unknown>; quota: Record<string, unknown> };
    };
    if (body.code !== 0 || !body.data) {
      return { success: false, error: body.message || 'Exchange failed' };
    }
    saveAuthTokens(body.data.accessToken, body.data.refreshToken);
    return { success: true, user: body.data.user, quota: normalizeQuota(body.data.quota) };
  } catch (error) {
    console.error('[Auth] Lark exchange failed:', error);
    return { success: false, error: error instanceof Error ? error.message : 'Exchange failed' };
  }
});
```

### 2.5 Preload 脚本改造（src/main/preload.ts）

在 `auth` 命名空间内扩展（不删除现有方法，渐进式改造）：

```typescript
auth: {
  login: (loginUrl?: string) => ipcRenderer.invoke('auth:login', { loginUrl }),
  exchange: (code: string) => ipcRenderer.invoke('auth:exchange', { code }),
  // ... 其他现有方法 ...

  // 新增 Lark 专用方法
  larkLogin: () => ipcRenderer.invoke('auth:larkLogin'),
  larkExchange: (code: string, state: string) =>
    ipcRenderer.invoke('auth:larkExchange', { code, state }),
  onLarkCallback: (handler: (payload: { code: string; state: string | null }) => void) => {
    const wrapped = (_event: any, payload: any) => handler(payload);
    ipcRenderer.on('auth:lark:callback', wrapped);
    return () => ipcRenderer.removeListener('auth:lark:callback', wrapped);
  },
},
```

### 2.6 渲染层 AuthService 改造（src/renderer/services/auth.ts）

```typescript
class AuthService {
  async init() {
    this.destroy();
    store.dispatch(setAuthLoading(true));
    try {
      const result = await window.electron.auth.getUser();
      if (result.success && result.user) {
        store.dispatch(setLoggedIn({ user: result.user, quota: result.quota }));
        await this.loadServerModels();
      } else {
        store.dispatch(setLoggedOut());
      }
    } catch {
      store.dispatch(setLoggedOut());
    }

    // 改为监听 Lark 回调（替代原 onCallback）
    this.unsubCallback = window.electron.auth.onLarkCallback(async ({ code, state }) => {
      await this.handleLarkCallback(code, state ?? '');
    });

    // quota 变更和窗口焦点监听保持不变
    this.unsubQuotaChanged = window.electron.auth.onQuotaChanged(() => {
      this.refreshQuota();
      this.loadServerModels();
    });
    this.unsubWindowState = window.electron.window.onStateChanged((state) => {
      if (state.isFocused && store.getState().auth.isLoggedIn) {
        const now = Date.now();
        if (now - this.lastRefreshTime > 30_000) {
          this.lastRefreshTime = now;
          this.refreshQuota();
          this.loadServerModels();
        }
      }
    });
  }

  // 入口：直接发起 Lark OAuth
  async login() {
    await window.electron.auth.larkLogin();
  }

  // 处理 Lark 授权回调
  async handleLarkCallback(code: string, state: string) {
    try {
      const result = await window.electron.auth.larkExchange(code, state);
      if (result.success) {
        store.dispatch(setLoggedIn({ user: result.user, quota: result.quota }));
        await this.loadServerModels();
      } else {
        console.error('[Auth] Lark exchange failed:', result.error);
        // 可 dispatch 一个错误 action 给 UI 显示提示
      }
    } catch (e) {
      console.error('[Auth] Lark callback failed:', e);
    }
  }

  // logout / refreshQuota / fetchProfileSummary / getAccessToken / destroy 无需改动
}
```

### 2.7 UI 改造（src/renderer/components/LoginButton.tsx）

主要变更点：
1. 登录按钮文案改为"飞书登录" / "Sign in with Lark"（通过 i18n）
2. 可添加飞书 Logo 图标（使用 SVG 或 `@iconify/react`）
3. 调用路径不变：`await authService.login()` → 内部已改为 `larkLogin()`
4. 用户信息展示区域不变（`state.auth.user` 字段兼容）

### 2.8 i18n 新增 Key

**src/renderer/services/i18n.ts：**

```typescript
// zh 区块新增
authLarkLogin: '飞书登录',
authLarkLoading: '正在跳转飞书授权...',
authLarkFailed: '飞书授权失败，请重试',

// en 区块新增
authLarkLogin: 'Sign in with Lark',
authLarkLoading: 'Redirecting to Lark authorization...',
authLarkFailed: 'Lark authorization failed, please try again',
```

### 2.9 数据库 / 持久化变更

| 项目 | 现状 | 改造后 |
|------|------|--------|
| `kv.auth_tokens` | `{ accessToken, refreshToken }` | **不变** — 仍存储系统 JWT |
| `kv.lark_oauth_state` | 不存在 | **新增** — 临时存储 CSRF state（exchange 后删除） |
| `kv.lark_app_id` | 不存在 | **可选** — 若需运行时配置 Lark App ID |

> **重要：** `auth_tokens` 存储的是**系统 JWT**，不是 Lark token。Lark user_access_token 由后端持有，客户端无需存储。

### 2.10 常量文件（新建 src/main/authConstants.ts）

遵循项目 String Literal Constants 规范：

```typescript
export const AuthIpcChannel = {
  Login: 'auth:login',
  LarkLogin: 'auth:larkLogin',
  LarkExchange: 'auth:larkExchange',
  Exchange: 'auth:exchange',
  GetUser: 'auth:getUser',
  GetQuota: 'auth:getQuota',
  GetProfileSummary: 'auth:getProfileSummary',
  Logout: 'auth:logout',
  RefreshToken: 'auth:refreshToken',
  GetAccessToken: 'auth:getAccessToken',
  GetModels: 'auth:getModels',
  Callback: 'auth:callback',
  LarkCallback: 'auth:lark:callback',
  QuotaChanged: 'auth:quotaChanged',
  GetPendingCallback: 'auth:getPendingCallback',
} as const;
export type AuthIpcChannel = typeof AuthIpcChannel[keyof typeof AuthIpcChannel];
```

---

## 三、实施步骤与风险评估

### 3.1 实施步骤（推荐顺序）

```
Phase 1 — 后端（前置条件，阻塞后续）
  Step 1: 在飞书开放平台创建自建应用，获取 App ID & App Secret
  Step 2: 配置 redirect_uri 白名单（精确值：lobsterai://auth/lark/callback）
          注意：飞书后台同时填写原始值和 URL encode 后的值，避免匹配失败
  Step 3: 实现 POST /api/auth/lark/exchange 接口
          （用 lark_open_id 查找/创建用户，签发系统 JWT）

Phase 2 — 主进程
  Step 4: 新建 src/main/authConstants.ts
  Step 5: 扩展 handleDeepLink 支持 /lark/callback
  Step 6: 新增 auth:larkLogin IPC handler
  Step 7: 新增 auth:larkExchange IPC handler

Phase 3 — 渲染层
  Step 8:  扩展 preload.ts 暴露 3 个新 Lark 方法
  Step 9:  改造 AuthService.login() 和 init() 中的回调监听
  Step 10: 改造 LoginButton.tsx UI（替换为飞书登录按钮）
  Step 11: 新增 i18n 文案

Phase 4 — 清理旧流程（验证稳定后执行）
  Step 12: 下线 auth:login / auth:exchange / fetchLoginUrl 等旧 Portal 登录逻辑
  Step 13: 清理 onCallback 监听和 pendingAuthCode 旧缓冲逻辑（改为 Lark 专用版本）
```

### 3.2 风险矩阵

| 风险 | 等级 | 说明 | 缓解措施 |
|------|------|------|---------|
| 飞书 redirect_uri 白名单严格匹配 | 🟡 中 | custom scheme 需在飞书开放平台后台精确配置 | 开发阶段提前申请，原始值和 URL encode 值均填写 |
| Windows 下 custom scheme 注册 | 🟡 中 | `setAsDefaultProtocolClient` 依赖安装时写注册表 | Electron 已处理，需验证打包后的行为 |
| Lark code 5 分钟有效期 | 🟡 中 | 用户操作慢或网络问题可能超时 | 后端设置超时告警，前端给出友好错误提示 |
| 企业 Lark 应用权限审核周期 | 🟡 中 | 飞书企业自建应用上线需审核 | 提前 1-2 周申请，预留缓冲 |
| 飞书 API 调用频率限制 | 🟢 低 | 登录场景调用频率较低 | 关注飞书文档配额限制，生产环境保持合理并发 |
| CSRF 攻击 | 🟢 低 | 已通过 state 参数 + SQLite 校验防护 | 实现后审查 state 的生成和校验逻辑 |

### 3.3 切换策略

1. **并行阶段**：Phase 1-3 完成后，新旧登录入口并存（通过配置 feature flag 控制），用于功能验证
2. **完全切换**：验证稳定后执行 Phase 4，下线旧 Portal 登录流程，入口统一为飞书登录

---

## 四、快速定位代码参考

### 改造时需要关注的代码位置

```
src/main/main.ts
  L1667–1727   → protocol 注册 & handleDeepLink（需扩展）
  L1912–1927   → saveAuthTokens / getAuthTokens / clearAuthTokens（复用）
  L1932–1961   → fetchWithAuth（复用，不改）
  L2004–2174   → 现有 auth IPC handlers（在此区域新增 larkLogin / larkExchange）

src/main/preload.ts
  L459–477     → auth 命名空间（在此扩展新方法）

src/renderer/services/auth.ts
  L15–54       → init()（改监听 onLarkCallback）
  L59–62       → login()（改调 larkLogin）
  L92–102      → handleCallback()（新增 handleLarkCallback 并行）

src/renderer/components/LoginButton.tsx
  L225–252     → 未登录状态下的登录按钮（改为飞书登录 UI）
```

---

*本文档由代码静态分析自动生成，实施前请与后端团队对齐飞书 API 接口规范和用户体系设计。*
