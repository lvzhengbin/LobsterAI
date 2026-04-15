# 有道域名服务使用梳理

> 整理时间：2026-04-14
> 涉及域名：`lobsterai-server.youdao.com` / `api-overmind.youdao.com`

---

## 一、lobsterai-server.youdao.com — LobsterAI 后端服务

**测试环境对应内网域名**：`lobsterai-server.inner.youdao.com`

### 接口清单

| 接口路径 | 方法 | 功能 | 文件 |
|---------|------|------|------|
| `/api/auth/exchange` | POST | authCode → accessToken + refreshToken（登录） | `src/main/main.ts` |
| `/api/auth/refresh` | POST | refreshToken → 新 accessToken（续期） | `src/main/main.ts` |
| `/api/auth/logout` | POST | 退出登录，清除 Token | `src/main/main.ts` |
| `/api/models/available` | GET | 获取用户可用的 LLM 模型列表 | `src/main/main.ts` |
| `/api/proxy/v1/chat/completions` | POST | LLM 调用代理（注入 Token 后转发） | `src/main/libs/claudeSettings.ts` |

**认证方式**：Bearer Token（Header: `Authorization: Bearer <accessToken>`）

### Token 生命周期

| Token | 有效期 | 存储位置 |
|-------|--------|---------|
| accessToken | 2 小时，JWT 格式可本地解析 `exp` | SQLite `kv['auth_tokens']` |
| refreshToken | 30 天，Rolling Refresh（每次刷新签发新值） | SQLite `kv['auth_tokens']` |

**主动刷新**：`authTokensGetter()` 检测到距 accessToken 过期 < 5 分钟，异步后台刷新。

**被动刷新**：API 返回 401/403 时，代理层自动刷新 Token 并重试一次。

### Token Proxy 机制

主进程启动一个本地 HTTP 服务（`127.0.0.1:<port>`），OpenClaw 发送的请求先到这个代理，代理自动注入 accessToken 后转发到 `/api/proxy/v1/chat/completions`。OpenClaw 无需直接持有用户 Token。

```
OpenClaw → http://127.0.0.1:<port>/v1/chat/completions
         ↓ (注入 Authorization 头)
         → https://lobsterai-server.youdao.com/api/proxy/v1/chat/completions
```

实现文件：`src/main/libs/openclawTokenProxy.ts`

### 请求体示例

```json
// POST /api/auth/exchange
{ "authCode": "one_time_code" }

// POST /api/auth/refresh
{ "refreshToken": "refresh_token_value" }
```

---

## 二、api-overmind.youdao.com — 有道产品中台（Luna 硬件平台）

所有路径前缀：`/openapi/get/luna/hardware/lobsterai/{test|prod}/`

**响应格式统一**：`{ data: { value: <实际数据> } }`

### 接口清单

| 功能 | 测试路径 | 生产路径 | 使用文件 |
|------|---------|---------|---------|
| 自动更新检查 | `.../test/update` | `.../prod/update` | `src/renderer/services/appUpdate.ts` |
| 手动检查更新 | `.../test/update-manual` | `.../prod/update-manual` | `src/renderer/services/appUpdate.ts` |
| Skill 商店数据 | `.../test/skill-store` | `.../prod/skill-store` | `src/renderer/services/skill.ts` |
| 登录页 URL | `.../test/login-url` | `.../prod/login-url` | `src/renderer/services/auth.ts` / `src/main/main.ts` |
| MCP 市场数据 | `.../test/mcp-marketplace` | `.../prod/mcp-marketplace` | `src/main/main.ts` |

### 各接口响应数据结构

**更新检查**（`/update`、`/update-manual`）：
```json
{
  "data": {
    "value": {
      "latestVersion": "2026.x.x",
      "downloadUrl": "https://...",
      "releaseNotes": "..."
    }
  }
}
```

**Skill 商店**（`/skill-store`）：
```json
{
  "data": {
    "value": {
      "localSkill": [{ "name": "...", "description": "..." }],
      "marketplace": [{ "id": "...", "name": "...", "description": {} }],
      "marketTags": [{ "name": "...", "name_en": "..." }]
    }
  }
}
```

**MCP 市场**（`/mcp-marketplace`）：
```json
{
  "data": {
    "value": {
      "servers": [{ "name": "...", "url": "..." }],
      "categories": [{ "id": "...", "name": "..." }]
    }
  }
}
```

---

## 三、测试/生产环境切换

由 `testMode` 开关控制，集中在两个配置文件：

- `src/main/libs/endpoints.ts` — 主进程 API base URL 决策
- `src/renderer/services/endpoints.ts` — 渲染进程端点配置

```
testMode=false（生产）:
  lobsterai-server.youdao.com          ← 认证 + LLM 代理
  api-overmind.youdao.com/.../prod/*   ← 更新 / Skill / MCP / 登录URL

testMode=true（测试，开发模式自动启用）:
  lobsterai-server.inner.youdao.com    ← 认证 + LLM 代理（内网）
  api-overmind.youdao.com/.../test/*   ← 更新 / Skill / MCP / 登录URL
```

**testMode 来源优先级（从高到低）**：
1. SQLite `app_config.app.testMode` 字段（用户手动配置，持久化）
2. `!app.isPackaged`（开发模式自动启用）

> 注意：两个域名均**硬编码在源码中**，没有 `.env` 文件控制。

---

## 四、完整登录流程时序

```
用户点击登录
    ↓
渲染进程 → fetchLoginUrl()
    ↓
GET api-overmind.youdao.com/.../login-url
  备用: https://lobsterai-server.youdao.com/login
    ↓
主进程 openExternal() → 系统浏览器打开登录页
    ↓
用户输入账号密码 → Server 验证成功
    ↓
302 重定向 → lobsterai://auth/callback?code=<authCode>
    ↓
主进程捕获 deep link → 发送 auth:callback IPC → 渲染进程
    ↓
POST /api/auth/exchange → 换取 accessToken + refreshToken
    ↓
存入 SQLite kv['auth_tokens']
    ↓
登录成功 → loadServerModels()
```

---

## 五、功能归类总结

| 业务功能 | 使用域名 |
|---------|---------|
| 用户登录 / Token 管理 | `lobsterai-server` |
| LLM 模型调用（代理） | `lobsterai-server` |
| 可用模型列表 | `lobsterai-server` |
| 应用自动 / 手动更新 | `api-overmind` |
| Skill 商店 | `api-overmind` |
| MCP 市场 | `api-overmind` |
| 登录页 URL 下发 | `api-overmind` |

---

## 六、相关源码文件索引

| 文件路径 | 用途 |
|---------|------|
| `src/main/libs/endpoints.ts` | 主进程 API base URL 中枢（testMode 决策） |
| `src/renderer/services/endpoints.ts` | 渲染进程端点配置（业务 API） |
| `src/main/main.ts` | IPC 处理、deep link、Token 管理、MCP Marketplace |
| `src/main/libs/claudeSettings.ts` | Provider 解析，LobsterAI Server 降级逻辑 |
| `src/main/libs/openclawTokenProxy.ts` | Token 注入代理（本地 127.0.0.1） |
| `src/renderer/services/auth.ts` | 渲染进程登录流程 |
| `src/renderer/services/appUpdate.ts` | 应用更新检查 |
| `src/renderer/services/skill.ts` | Skill 商店数据获取 |

---

## 七、相关文档

- [auth-and-lobsterai-server-flow.md](./auth-and-lobsterai-server-flow.md) — 详细的登录和代理流程分析
- [external-service-audit.md](./external-service-audit.md) — 外部服务审计报告
- [self-hosted-server-design.md](./self-hosted-server-design.md) — 自建服务器迁移方案
