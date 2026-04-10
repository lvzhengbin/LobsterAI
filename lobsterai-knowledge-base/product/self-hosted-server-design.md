# LobsterAI 自建后端服务（lobsterai-server.vb.com）方案设计

> 文档日期：2026-04-10  
> 目标：基于 LobsterAI 客户端代码，设计一套可以完整替换 `lobsterai-server.youdao.com` 的自有后端服务  
> 内部 LLM 网关：`https://9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app`（OpenAI 兼容格式，需 API Key）  
> 自建服务域名：`https://lobsterai-server.vb.com`

---

## 一、方案概述

### 1.1 系统角色定位

```
┌────────────────────────────────────────┐
│      LobsterAI 客户端（Electron）        │
│  注册 lobsterai:// 协议，处理深链接       │
└────────────────┬───────────────────────┘
                 │ HTTPS（所有 API）
                 ▼
┌────────────────────────────────────────┐
│    自建后端：lobsterai-server.vb.com    │
│                                        │
│  认证服务：authCode → JWT Token         │
│  用户服务：profile / quota / summary    │
│  模型服务：可用模型列表                  │
│  代理服务：/api/proxy/v1 → 内部 LLM     │
│  登录页面：/login（Web 界面）           │
└────────────────┬───────────────────────┘
                 │ HTTPS + API Key
                 ▼
┌────────────────────────────────────────┐
│  内部 LLM 网关                          │
│  https://9cc138f8dc04cbf16240daa92d8d5 │
│  0e2.nebulab.app                        │
│  Format: OpenAI 兼容                   │
└────────────────────────────────────────┘
```

### 1.2 需要实现的接口全集（来自客户端代码逆向）

| 接口 | Method | 用途 | 认证方式 |
|------|--------|------|----------|
| `GET  /login` | GET | 登录页面（Web UI） | 无 |
| `POST /api/auth/exchange` | POST | authCode → accessToken + refreshToken | 无 |
| `POST /api/auth/refresh` | POST | refreshToken → 新 accessToken | 无 |
| `POST /api/auth/logout` | POST | 吊销 Token | Bearer Token |
| `GET  /api/user/profile` | GET | 获取用户信息 | Bearer Token |
| `GET  /api/user/quota` | GET | 获取用户配额 | Bearer Token |
| `GET  /api/user/profile-summary` | GET | 获取配额详情（含 credits 分解） | Bearer Token |
| `GET  /api/models/available` | GET | 获取用户可用模型列表 | Bearer Token |
| `POST /api/proxy/v1/*` | POST | 转发 LLM 请求到内部网关（支持流式） | Bearer Token |

---

## 二、客户端修改：切换服务器地址

### 2.1 只需修改一个文件

**`src/main/libs/endpoints.ts`**

```typescript
// 将原有的 youdao.com 地址替换为自建服务地址
export const getServerApiBaseUrl = (): string => {
  return isTestMode()
    ? 'https://lobsterai-server-test.vb.com'   // 测试环境（可选）
    : 'https://lobsterai-server.vb.com';        // 生产环境
};
```

### 2.2 渲染进程登录 URL 来源

客户端先尝试从 overmind API 获取登录页 URL（`getLoginOvermindUrl()`），失败后回退到 `${serverBaseUrl}/login`。

**方案 A（推荐）：** 直接依赖回退逻辑，overmind 调用失败后使用 `https://lobsterai-server.vb.com/login`，无需修改。

**方案 B：** 同步修改 `src/renderer/services/endpoints.ts` 中的 `getLoginOvermindUrl()`，返回自建服务的 URL。

---

## 三、协议详细设计

### 3.1 统一响应格式

**所有接口** 遵循同一 JSON Envelope 格式：

```json
{
  "code": 0,        // 0 = 成功，非零 = 失败
  "message": "ok",  // 描述信息
  "data": { ... }   // 具体数据（成功时）
}
```

### 3.2 认证接口

#### POST /api/auth/exchange

将 authCode 换取 accessToken + refreshToken。

**客户端请求体：**
```json
{ "authCode": "<one-time-code>" }
```

**服务端响应体：**
```json
{
  "code": 0,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "...",
    "user": {
      "userId": "u_123456",
      "username": "张三",
      "email": "zhangsan@vb.com",
      "avatar": "https://..."
    },
    "quota": {
      "planName": "Standard",
      "subscriptionStatus": "active",
      "monthlyCreditsLimit": 1000,
      "monthlyCreditsUsed": 120
    }
  }
}
```

**Token 规格（JWT）：**

```json
// accessToken Header
{ "alg": "HS256", "typ": "JWT" }

// accessToken Payload
{
  "sub": "u_123456",
  "exp": 1712850000,      // 当前时间 + 2小时（必须！客户端靠 exp 触发主动刷新）
  "iat": 1712842800,
  "jti": "unique-token-id"
}
```

> ⚠️ **关键**：客户端会解析 JWT `exp` 字段（`main.ts L4787`），距过期 < 5 分钟时主动刷新。accessToken **必须是合法 JWT**，且 `exp` 字段为 Unix 秒时间戳。

**authCode 生成说明：**  
authCode 由登录页面（`/login`）在用户完成身份验证后生成，通过重定向回客户端深链接携带：

```
lobsterai://auth/callback?code=<authCode>
```

authCode 特性：
- **一次性**（消费后立即失效）
- 有效期建议 **60 秒**
- 服务端维护 authCode 黑名单（Redis 推荐）

---

#### POST /api/auth/refresh

**客户端请求体：**
```json
{ "refreshToken": "..." }
```

**服务端响应体：**
```json
{
  "code": 0,
  "data": {
    "accessToken": "eyJ...",   // 新 accessToken（2h 有效期）
    "refreshToken": "..."      // 新 refreshToken（Rolling，30d 有效期）
  }
}
```

> ⚠️ **Rolling Refresh Token**：每次 refresh 必须颁发新 refreshToken，旧 refreshToken 立即失效（防重放攻击）。客户端代码 `main.ts L4762` 会保存新 refreshToken。

**失败场景处理：**

| HTTP 状态 | body.code | 含义 |
|-----------|-----------|------|
| 200 | 0 | 成功 |
| 200 | 非零 | 业务失败（refreshToken 失效/过期） |
| 401 | - | 客户端会停止重试，要求重新登录 |

---

#### POST /api/auth/logout

**请求 Header：** `Authorization: Bearer <accessToken>`  
**响应：** `{ "code": 0 }` 即可

> 这是 best-effort 调用，客户端不检查结果（`main.ts L2132`），服务端失败无影响。  
> 服务端应将 accessToken 加入吊销黑名单（建议用 Redis 实现），refreshToken 一并失效。

---

### 3.3 用户接口

所有用户接口均需验证 `Authorization: Bearer <accessToken>`，accessToken 无效返回 **HTTP 401**（客户端会自动触发 Token 刷新后重试）。

#### GET /api/user/profile

**响应格式：**
```json
{
  "code": 0,
  "data": {
    "userId": "u_123456",
    "username": "张三",
    "email": "zhangsan@vb.com",
    "avatar": "https://..."
  }
}
```

---

#### GET /api/user/quota

客户端的 `normalizeQuota()` 函数支持三种格式（`main.ts L1986`），推荐使用标准格式：

**推荐格式（付费用户）：**
```json
{
  "code": 0,
  "data": {
    "planName": "Standard",
    "subscriptionStatus": "active",
    "monthlyCreditsLimit": 1000,
    "monthlyCreditsUsed": 120
  }
}
```

**免费用户格式：**
```json
{
  "code": 0,
  "data": {
    "planName": "Free",
    "subscriptionStatus": "free",
    "freeCreditsLimit": 100,
    "freeCreditsUsed": 30
  }
}
```

客户端 normalizeQuota 转换结果（统一为）：
```json
{
  "planName": "Standard",
  "subscriptionStatus": "active",
  "creditsLimit": 1000,
  "creditsUsed": 120,
  "creditsRemaining": 880
}
```

---

#### GET /api/user/profile-summary

用于设置页面展示详细的 credits 分解信息，格式自由，客户端直接传递给 UI 展示：

```json
{
  "code": 0,
  "data": {
    "totalCredits": 1000,
    "usedCredits": 120,
    "remainingCredits": 880,
    "breakdown": [
      { "model": "gpt-4o", "credits": 80 },
      { "model": "claude-3-5-sonnet", "credits": 40 }
    ]
  }
}
```

---

### 3.4 模型列表接口

#### GET /api/models/available

**响应格式（关键！客户端对字段有严格解析，见 `main.ts L2185`）：**

```json
{
  "code": 0,
  "data": [
    {
      "modelId": "gpt-4o",
      "modelName": "GPT-4o",
      "provider": "openai",
      "apiFormat": "openai",
      "supportsImage": true
    },
    {
      "modelId": "gpt-4o-mini",
      "modelName": "GPT-4o Mini",
      "provider": "openai",
      "apiFormat": "openai",
      "supportsImage": true
    },
    {
      "modelId": "deepseek-chat",
      "modelName": "DeepSeek Chat",
      "provider": "deepseek",
      "apiFormat": "openai",
      "supportsImage": false
    }
  ]
}
```

**字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `modelId` | string | 发送给 LLM 网关的 model 字段值（**至关重要**） |
| `modelName` | string | UI 展示名称 |
| `provider` | string | provider 标识，仅用于 UI 分组展示 |
| `apiFormat` | string | 固定 `"openai"`（内部网关兼容 OpenAI 格式） |
| `supportsImage` | boolean | 是否支持图片输入，影响 UI 是否显示图片按钮 |

> ⚠️ **关键**：`modelId` 的值会直接映射到 `app_config.model.defaultModel`，并在代理转发时作为请求体的 `model` 字段发送给内部网关。必须是内部网关实际支持的 model 名称。

---

### 3.5 LLM 代理接口（核心）

#### POST /api/proxy/v1/*

这是最关键的接口，负责将 LobsterAI 客户端的 LLM 请求转发到内部网关。

**路径匹配：**

客户端（通过 Token Proxy）会请求：
```
POST /api/proxy/v1/chat/completions    # 聊天补全
POST /api/proxy/v1/messages            # Anthropic 格式（若有）
```

**请求 Header：**
```
Authorization: Bearer <accessToken>
Content-Type: application/json
Accept: text/event-stream              # 流式请求时
```

**请求 Body（OpenAI 格式，直接透传来自 OpenClaw）：**
```json
{
  "model": "gpt-4o",
  "messages": [...],
  "stream": true,
  "max_tokens": 4096,
  "tools": [...],
  "tool_choice": "auto"
}
```

**服务端处理逻辑：**

```
收到请求
  ├── 1. 验证 Authorization Bearer Token
  │       invalid → 返回 401
  ├── 2. 检查用户配额（creditsRemaining > 0）
  │       超额 → 返回 429 + 自定义错误信息
  ├── 3. 提取 model 字段，验证在可用模型列表中
  │       不存在 → 返回 400
  ├── 4. 构建转发请求：
  │       URL:  https://9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app/v1/chat/completions
  │       Header: Authorization: Bearer <内部_API_KEY>
  │       Body: 原始请求体（直接透传）
  ├── 5. 流式响应：直接 pipe SSE 数据到客户端
  │   非流式响应：等待完整响应后返回
  └── 6. 记录 token 消耗，更新用户 creditsUsed
```

**转发请求 Header 构建：**
```
Authorization: Bearer <INTERNAL_LLM_API_KEY>   # 内部网关 API Key
Content-Type: application/json
Accept: text/event-stream                       # 若客户端请求流式
```

> ⚠️ **注意**：转发时**不透传**客户端的 Authorization header，而是替换为内部 API Key。

**流式响应代理（SSE 透传关键）：**

```
客户端请求（stream: true）
  → 服务端连接内部网关（stream: true）
  → 内部网关返回 SSE 流
  → 服务端逐 chunk 透传给客户端
     Response Header: Content-Type: text/event-stream
                      Cache-Control: no-cache
                      Connection: keep-alive
     Body: data: {"choices":[...]}\n\n
           data: [DONE]\n\n
```

---

### 3.6 登录页面（/login）

这是在系统浏览器中打开的 Web 页面，用户在此完成身份认证。

**客户端打开 URL：**
```
https://lobsterai-server.vb.com/login?source=electron
```

**认证成功后重定向：**
```
lobsterai://auth/callback?code=<one-time-authCode>
```

> ⚠️ **重要**：必须使用 `lobsterai://` 自定义协议，客户端已注册此协议处理器（`main.ts L1680`）。

**简易实现（企业内部场景）：**

```html
<!-- /login 页面逻辑 -->
<!DOCTYPE html>
<html>
<head><title>LobsterAI 登录</title></head>
<body>
  <form id="loginForm">
    <input type="text" id="username" placeholder="用户名/工号" required />
    <input type="password" id="password" placeholder="密码" required />
    <button type="submit">登录</button>
  </form>
  <script>
    document.getElementById('loginForm').onsubmit = async (e) => {
      e.preventDefault();
      const resp = await fetch('/api/auth/internal-login', {
        method: 'POST',
        body: JSON.stringify({
          username: document.getElementById('username').value,
          password: document.getElementById('password').value,
        }),
        headers: { 'Content-Type': 'application/json' },
      });
      const data = await resp.json();
      if (data.code === 0) {
        // 生成 authCode 并重定向到客户端
        window.location.href = `lobsterai://auth/callback?code=${data.data.authCode}`;
      } else {
        alert('登录失败：' + data.message);
      }
    };
  </script>
</body>
</html>
```

**与 `?source=electron` 参数：** 服务端可据此区分来源，登录成功后生成 authCode 重定向。

---

## 四、完整交互流程与时序图

### 4.1 首次登录时序

```
LobsterAI客户端           lobsterai-server.vb.com        内部LLM网关
      |                            |                          |
      | 1. 点击登录                 |                          |
      | shell.openExternal(        |                          |
      |   https://lobsterai-       |                          |
      |   server.vb.com/login      |                          |
      |   ?source=electron)        |                          |
      |────────────────────────────► 返回登录页 HTML           |
      |                            |                          |
      | 2. 用户填写账号密码（浏览器）|                          |
      |                            |                          |
      | 3. POST /api/auth/internal-login                       |
      |───────────────────────────►|                          |
      |                            | 验证身份，生成 authCode  |
      |                            | (一次性，60s 有效)        |
      |◄───────────────────────────| { code:0, data:{authCode}}
      |                            |                          |
      | 4. 浏览器重定向              |                          |
      |    lobsterai://auth/callback?code=<authCode>           |
      |                            |                          |
      | 5. Electron 捕获深链接       |                          |
      |    app.on('open-url')       |                          |
      |    解析 code 参数            |                          |
      |                            |                          |
      | 6. POST /api/auth/exchange  |                          |
      |    { authCode: "..." }      |                          |
      |───────────────────────────►|                          |
      |                            | 验证 authCode，生成 JWT  |
      |                            | accessToken(2h)          |
      |                            | refreshToken(30d)        |
      |◄───────────────────────────| {accessToken,refreshToken,
      |                            |  user, quota}            |
      |                            |                          |
      | 7. 保存 Token 到 SQLite      |                          |
      |    store kv['auth_tokens']  |                          |
      |                            |                          |
      | 8. GET /api/models/available                           |
      |    Header: Bearer accessToken                          |
      |───────────────────────────►|                          |
      |◄───────────────────────────| [{modelId, modelName,...}]
      |                            |                          |
      | 9. 用户选择模型，开始对话    |                          |
```

### 4.2 LLM 对话请求时序

```
LobsterAI客户端     Token Proxy(本机)    lobsterai-server.vb.com    内部LLM网关
      |                   |                       |                      |
      | 用户发消息          |                       |                      |
      | OpenClaw引擎触发    |                       |                      |
      |                   |                       |                      |
      | POST http://127.0.0.1:<port>/v1/chat/completions                  |
      |──────────────────►|                       |                      |
      |                   | 读 SQLite accessToken  |                      |
      |                   | 检查 JWT exp < 5min?   |                      |
      |                   | → 否：直接使用          |                      |
      |                   |                       |                      |
      |                   | POST /api/proxy/v1/chat/completions           |
      |                   | Header: Bearer accessToken                    |
      |                   |──────────────────────►|                      |
      |                   |                       | 1. 验证 Token         |
      |                   |                       | 2. 检查配额           |
      |                   |                       | 3. 构建转发请求       |
      |                   |                       |                      |
      |                   |                       | POST /v1/chat/completions
      |                   |                       | Header: Bearer <内部APIKey>
      |                   |                       |─────────────────────►|
      |                   |                       |                      | 处理请求
      |                   |                       |◄─── SSE 流式数据 ─────|
      |                   |◄─── SSE 流式数据 ──────|                      |
      |◄─── SSE 流式数据 ──|                       |                      |
      |                   |                       |                      |
      | 渲染进程收到消息流   |                       |                      |
```

### 4.3 Token 自动刷新时序

```
LobsterAI客户端(主进程)    lobsterai-server.vb.com
        |                          |
        | [每次需要 Token 时]        |
        | 解析 JWT exp 字段          |
        | exp - now < 5分钟？       |
        |                          |
        | ── 是 ──► 异步触发 refreshOnce()
        |                          |
        | POST /api/auth/refresh    |
        | { refreshToken: "..." }   |
        |─────────────────────────►|
        |                          | 验证 refreshToken
        |                          | 生成新 accessToken (2h)
        |                          | 生成新 refreshToken (30d)
        |                          | 旧 refreshToken 立即失效
        |◄─────────────────────────| {accessToken, refreshToken}
        |                          |
        | 更新 SQLite kv['auth_tokens']
        |                          |
        [继续使用新 accessToken 发请求]
```

### 4.4 Token 401 自动重试时序

```
Token Proxy(本机)        lobsterai-server.vb.com
      |                          |
      | POST /api/proxy/v1/...   |
      | Bearer <过期 accessToken> |
      |─────────────────────────►|
      |◄─────────────────────────| 401 Unauthorized
      |                          |
      | 触发 tokenRefresher('openclaw-proxy')
      |                          |
      | POST /api/auth/refresh    |
      | { refreshToken: "..." }   |
      |─────────────────────────►|
      |◄─────────────────────────| {新 accessToken, 新 refreshToken}
      |                          |
      | 用新 Token 重试原始请求    |
      | POST /api/proxy/v1/...   |
      | Bearer <新 accessToken>   |
      |─────────────────────────►|
      |◄─────────────────────────| 200 + SSE 数据
```

---

## 五、服务端实现建议

### 5.1 技术栈推荐

| 层次 | 推荐 | 说明 |
|------|------|------|
| Web 框架 | Node.js + Fastify / Go + Gin / Python + FastAPI | 均可，需支持 SSE 流式响应 |
| Token 存储 | Redis | authCode 黑名单、refreshToken 状态管理、配额缓存 |
| 用户数据 | PostgreSQL / MySQL | 用户信息、持久化配额记录 |
| JWT 库 | jsonwebtoken（Node.js）/ golang-jwt（Go） | 签发和验证 |

### 5.2 数据模型

```sql
-- 用户表
CREATE TABLE users (
  id          VARCHAR(64) PRIMARY KEY,
  username    VARCHAR(128) UNIQUE NOT NULL,
  email       VARCHAR(256),
  password    VARCHAR(256),           -- bcrypt hash
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);

-- 配额表
CREATE TABLE user_quotas (
  user_id             VARCHAR(64) PRIMARY KEY,
  plan_name           VARCHAR(64) DEFAULT 'Free',
  subscription_status VARCHAR(32) DEFAULT 'free',
  credits_limit       INTEGER DEFAULT 100,
  credits_used        INTEGER DEFAULT 0,
  reset_at            TIMESTAMP,      -- 月度重置时间
  updated_at          TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Token 记录表（可用 Redis 替代）
CREATE TABLE refresh_tokens (
  token       VARCHAR(512) PRIMARY KEY,
  user_id     VARCHAR(64) NOT NULL,
  expires_at  TIMESTAMP NOT NULL,
  revoked     BOOLEAN DEFAULT FALSE,
  created_at  TIMESTAMP DEFAULT NOW()
);
```

**Redis Key 设计：**

```
authcode:<code>     → userId（TTL: 60s，一次性）
rtoken:<token>      → userId（TTL: 30d，Rolling 时删除旧 key）
revoked:<jti>       → 1（TTL: accessToken 剩余有效期，用于立即吊销）
```

### 5.3 JWT 签发规范

```typescript
// accessToken（必须包含 exp，格式为 Unix 秒）
const accessToken = jwt.sign(
  {
    sub: userId,
    jti: randomUUID(),
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 2 * 3600,  // 2小时
  },
  JWT_SECRET,
  { algorithm: 'HS256' }
);

// refreshToken（不透露过多信息，用随机字符串 + Redis 映射）
const refreshToken = crypto.randomBytes(64).toString('hex');
await redis.set(`rtoken:${refreshToken}`, userId, 'EX', 30 * 24 * 3600);
```

### 5.4 LLM 代理核心实现（Node.js 示例）

```typescript
// POST /api/proxy/v1/*
app.post('/api/proxy/v1/*', async (req, res) => {
  // 1. 验证 Token
  const userId = await verifyToken(req.headers.authorization);
  if (!userId) return res.status(401).json({ code: 1, message: 'Unauthorized' });

  // 2. 检查配额
  const quota = await getUserQuota(userId);
  if (quota.creditsRemaining <= 0) {
    return res.status(429).json({ code: 429, message: '配额已用尽' });
  }

  // 3. 构建转发 URL
  const upstreamPath = req.path.replace('/api/proxy', '');
  const upstreamUrl = `https://9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app${upstreamPath}`;

  // 4. 转发请求
  const isStream = req.body?.stream === true;
  const upstreamResp = await fetch(upstreamUrl, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.INTERNAL_LLM_API_KEY}`,
      'Content-Type': 'application/json',
      ...(isStream ? { 'Accept': 'text/event-stream' } : {}),
    },
    body: JSON.stringify(req.body),
  });

  // 5. 透传响应
  if (isStream) {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    upstreamResp.body?.pipeTo(new WritableStream({
      write(chunk) { res.write(chunk); },
      close() { res.end(); },
    }));
  } else {
    const data = await upstreamResp.json();
    // 6. 扣减配额（根据响应中的 usage.total_tokens）
    const tokens = data.usage?.total_tokens || 0;
    await deductCredits(userId, tokens);
    res.json(data);
  }
});
```

> 📌 **流式场景的配额扣减**：SSE 流式时，在 `data: [DONE]` 事件或流结束时，从 SSE 数据汇总的 usage 统计中扣减。可在客户端发完最后一个 chunk 后处理，或异步队列处理。

---

## 六、详细注意事项

### 6.1 JWT accessToken 必须可解析 exp 字段

客户端代码（`main.ts L4787`）：

```typescript
const payload = JSON.parse(Buffer.from(tokens.accessToken.split('.')[1], 'base64').toString());
const expiresAt = payload.exp * 1000;
if (expiresAt - Date.now() < 5 * 60 * 1000) {
  void refreshOnce('proactive');
}
```

**要求：**
- accessToken 必须是标准 3 段 JWT（`header.payload.signature`）
- payload 必须有 `exp` 字段（Unix 秒时间戳）
- 如果不是合法 JWT，会触发 `catch` 但不报错，Token 还是会被使用，只是无法触发主动刷新

### 6.2 refreshToken 必须支持 Rolling 续期

客户端每次收到新 refreshToken 时会覆盖保存（`main.ts L4762`）：

```typescript
saveAuthTokens(body.data.accessToken, body.data.refreshToken || tokens.refreshToken);
```

如果 `/api/auth/refresh` 响应不包含 `refreshToken`，客户端会继续使用旧 refreshToken（回退逻辑）。建议**始终颁发新 refreshToken**。

### 6.3 并发 Token 刷新去重

客户端用 `refreshOnce()` + `pendingTokenRefresh` 保证并发情况下 refreshToken 只被消费一次（多个并发请求都触发刷新时，只有一个实际发出）。服务端即便收到并发两个相同 refreshToken 请求，也应做幂等处理（第二个返回 refreshToken 已失效 / 200 成功都可，只要不产生两个有效 refreshToken）。

### 6.4 /api/auth/logout 必须是 best-effort

客户端用 `.catch(() => {})` 忽略失败（`main.ts L2132`），服务端此接口失败不影响客户端侧的登出流程。但服务端应确保：
- 收到 logout 时将 accessToken `jti` 加入吊销黑名单
- 将对应 refreshToken 标记为已撤销

### 6.5 /api/models/available 字段顺序影响默认选择

客户端渲染时会展示 lobsterai-server 提供的模型列表，用户选择后写入 `defaultModel`。数组第一个元素通常作为推荐模型展示，建议将最常用模型排在前面。

### 6.6 代理接口的 CORS 不需要考虑

所有请求都来自 Electron 主进程（`net.fetch`），不走浏览器，**无 CORS 限制**。但如果登录页面是 WebView 中的 Web 内容，可能需要配置。

### 6.7 内部 LLM 网关 API Key 安全

```
INTERNAL_LLM_API_KEY = <9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app 的 API Key>
```

此 key 只存在于服务端，**绝不**暴露给客户端。客户端持有的只是用户的 accessToken。

### 6.8 /api/proxy/v1 路径的 URL 拼接

来自 Token Proxy 的请求（`openclawTokenProxy.ts L94`）：

```typescript
const upstreamPath = `/api/proxy${req.url || '/'}`;
// 即：客户端请求 /v1/chat/completions → 转发到 /api/proxy/v1/chat/completions
```

服务端需将 `/api/proxy` 前缀剥离，剩余部分拼接到内部网关 URL：

```
/api/proxy/v1/chat/completions
         ↓ 剥离 /api/proxy
/v1/chat/completions
         ↓ 拼接内部网关
https://9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app/v1/chat/completions
```

### 6.9 响应头透传（SSE 流式）

流式 SSE 响应必须透传以下响应头：

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no     # Nginx 部署时必须，禁止 Nginx 缓冲 SSE
```

---

## 七、部署配置参考

### 7.1 环境变量

```bash
# JWT 签名密钥
JWT_SECRET=<长随机字符串，>=32字节>

# 内部 LLM 网关 API Key
INTERNAL_LLM_API_KEY=<nebulab API Key>

# 内部 LLM 网关地址
INTERNAL_LLM_BASE_URL=https://9cc138f8dc04cbf16240daa92d8d50e2.nebulab.app

# Redis 连接
REDIS_URL=redis://localhost:6379

# 数据库连接
DATABASE_URL=postgresql://user:pass@host/lobsterai
```

### 7.2 Nginx 配置（SSE 关键配置）

```nginx
server {
  listen 443 ssl;
  server_name lobsterai-server.vb.com;

  location /api/proxy/ {
    proxy_pass http://backend:3000;
    proxy_set_header Host $host;
    proxy_set_header Authorization $http_authorization;

    # SSE 流式关键配置
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    chunked_transfer_encoding on;

    # 禁用 gzip，避免 SSE 被压缩导致流式失效
    gzip off;
  }

  location / {
    proxy_pass http://backend:3000;
    proxy_set_header Host $host;
  }
}
```

### 7.3 客户端唯一修改

修改 `src/main/libs/endpoints.ts`，将 `getServerApiBaseUrl()` 返回值改为自建服务地址：

```typescript
export const getServerApiBaseUrl = (): string => {
  return isTestMode()
    ? 'https://lobsterai-server-test.vb.com'
    : 'https://lobsterai-server.vb.com';
};
```

**重新打包客户端**后即可完整对接自建服务。

---

## 八、接口快速参考卡

| 接口 | Method | 请求 | 响应关键字段 |
|------|--------|------|------------|
| `/login` | GET | - | HTML 登录页 |
| `/api/auth/exchange` | POST | `{ authCode }` | `{ data: { accessToken, refreshToken, user, quota } }` |
| `/api/auth/refresh` | POST | `{ refreshToken }` | `{ data: { accessToken, refreshToken? } }` |
| `/api/auth/logout` | POST | Bearer Token | `{ code: 0 }` |
| `/api/user/profile` | GET | Bearer Token | `{ data: { userId, username, email } }` |
| `/api/user/quota` | GET | Bearer Token | `{ data: { monthlyCreditsLimit, monthlyCreditsUsed, planName } }` |
| `/api/user/profile-summary` | GET | Bearer Token | `{ data: { ... credits breakdown } }` |
| `/api/models/available` | GET | Bearer Token | `{ data: [ { modelId, modelName, provider, apiFormat, supportsImage } ] }` |
| `/api/proxy/v1/chat/completions` | POST | Bearer Token + OpenAI body | SSE 流 或 OpenAI JSON |
