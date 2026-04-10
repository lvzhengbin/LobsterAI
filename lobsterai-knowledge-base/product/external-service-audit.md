# LobsterAI 外部服务调用审计报告

> 审计日期：2026-04-09  
> 目标：梳理系统中所有调用外部服务的模块，评估将外部接口改为内部服务的可行性，确保数据通讯与存储不被外部系统收集。

---

## 一、LobsterAI 自有服务器（有道域名）

这类调用虽然是"内部"服务，但域名走的是 `youdao.com`，需要确认是否已迁移到内部网络：

| 用途 | URL | 发送数据 |
|------|-----|---------|
| 主业务 API | `https://lobsterai-server.youdao.com` | 用户 token、对话数据 |
| 测试环境 | `https://lobsterai-server.inner.youdao.com` | 同上 |
| 版本更新检查 | `https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/*/update` | 版本号、平台、架构 |
| Skill Store | `https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/*/skill-store` | 请求元数据 |
| 登录 URL 获取 | `https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/*/login-url` | — |
| Portal 页面 | `https://c.youdao.com/dict/hardware/*/lobsterai-portal.html#/pricing\|profile` | — |
| MCP 市场（生产） | `https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/prod/mcp-marketplace` | — |
| MCP 市场（测试） | `https://api-overmind.youdao.com/openapi/get/luna/hardware/lobsterai/test/mcp-marketplace` | — |

### MCP 市场数据来源详解

MCP 市场的数据来自**两个层次**：

#### 1. 远程 API（主数据源）

在 `src/main/main.ts` 的 `mcp:fetchMarketplace` IPC handler 中，根据 `app.isPackaged` 判断环境，分别请求生产或测试端点。响应结构为 `json.data.value`，其中包含 `{ servers, categories }` 两个字段，渲染进程通过 `convertMarketplaceToRegistry()` 将远程数据转换为本地 `McpRegistryEntry` 格式后展示。

**调用链路：**

```
UI (McpManager.tsx)
  └── mcpService.fetchMarketplace()            [src/renderer/services/mcp.ts]
        └── window.electron.mcp.fetchMarketplace()
              └── IPC: mcp:fetchMarketplace    [src/main/main.ts ~L2432]
                    └── HTTPS GET → api-overmind.youdao.com
                          └── 解析 json.data.value → 返回 { servers, categories }
```

#### 2. 本地内置注册表（静态兜底）

`src/renderer/data/mcpRegistry.ts` 中硬编码了约 13 个主流 MCP 服务（Tavily、GitHub、GitLab、Context7、Notion、Slack、Playwright、Firecrawl 等），作为本地静态备用集合，在远程接口不可用时可供参考。

**关键文件**: `src/main/libs/endpoints.ts`, `src/renderer/services/endpoints.ts`, `src/main/main.ts` (L2432), `src/renderer/data/mcpRegistry.ts`, `src/renderer/services/mcp.ts`

---

## 二、LLM 提供商 API（最高数据风险）

所有用户对话消息、代码、推理内容都会发送到这些服务。

### 国内提供商

| 提供商 | 基础 URL | 风险 |
|--------|----------|------|
| 有道智云 | `https://openapi.youdao.com/llmgateway/api/v1/chat/completions` | **高** - 有道外部 |
| DeepSeek | `https://api.deepseek.com` | 高 |
| Moonshot/Kimi | `https://api.moonshot.cn` | 高 |
| 通义千问 | `https://dashscope.aliyuncs.com` | 高 |
| 智谱 GLM | `https://open.bigmodel.cn/api` | 高 |
| MiniMax | `https://api.minimaxi.com` | 高 |
| 火山引擎豆包 | `https://ark.cn-beijing.volces.com/api` | 高 |
| StepFun | `https://api.stepfun.com/v1` | 高 |
| 小米 MiMo | `https://api.xiaomimimo.com` | 高 |
| Ollama | `http://localhost:11434` | 低（本地） |

### 海外提供商

| 提供商 | 基础 URL |
|--------|----------|
| Anthropic | `https://api.anthropic.com` |
| OpenAI | `https://api.openai.com/v1` |
| Google Gemini | `https://generativelanguage.googleapis.com/v1beta` |
| OpenRouter | `https://openrouter.ai/api` |
| GitHub Copilot | `https://api.individual.githubcopilot.com` |

**关键文件**: `src/shared/providers/constants.ts`

---

## 三、IM 网关（外部 HTTP 直接调用）

以下 IM 平台在 `src/main/im/imGatewayManager.ts` 中有直接 HTTP API 调用：

| 平台 | 外部端点 | 发送数据 |
|------|---------|---------|
| 钉钉 | `https://oapi.dingtalk.com/gettoken`<br>`https://api.dingtalk.com/v1.0/...` | AppKey、AppSecret、消息内容 |
| 飞书 | `https://open.feishu.cn`（via SDK） | AppId、AppSecret、消息内容 |
| Telegram | `https://api.telegram.org/bot{token}/...` | Bot Token |
| Discord | `https://discord.com/api/v10/...` | Bot Token |
| QQ Bot | `https://bots.qq.com/app/getAppAccessToken` | AppId、ClientSecret |

以下通过 OpenClaw 插件管理（间接外部调用）：

| 平台 | 插件 | npm 源 |
|------|------|--------|
| 微信 | `@tencent-weixin/openclaw-weixin` | 公共 npm |
| 企业微信 | `@wecom/wecom-openclaw-plugin` | 公共 npm |
| 网易 IM（云信） | `openclaw-nim` + `nim-web-sdk-ng@10.9.77-alpha.4` | 公共 npm |
| 网易 Bee | `openclaw-netease-bee` | 公共 npm |
| POPO | `moltbot-popo@1.0.66` | **`npm.nie.netease.com`（网易内网 npm）** |

---

## 四、OAuth 认证流（第三方身份）

| 服务 | 端点 | Client ID |
|------|------|-----------|
| GitHub Copilot | `https://github.com/login/device/code`<br>`https://github.com/login/oauth/access_token`<br>`https://api.github.com/copilot_internal/v2/token` | `Iv1.b507a08c87ecfe98`（VS Code 公开 ID） |
| 通义千问 | `https://chat.qwen.ai/api/v1/oauth2/...` | `f0304373b74a44d2b584a3fb70ca9e56` |

**关键文件**: `src/main/libs/githubCopilotAuth.ts`, `src/main/libs/qwenOAuth.ts`

---

## 五、SDK 依赖（隐式外部通信）

| SDK | 版本 | 外部服务 |
|-----|------|---------|
| `@anthropic-ai/claude-agent-sdk` | 0.2.12 | Anthropic API |
| `@larksuiteoapi/node-sdk` | 1.58.0 | 飞书 API |
| `@wecom/wecom-aibot-sdk` | 0.1.0 | 腾讯企业微信 API |
| `nim-web-sdk-ng` | 10.9.77-alpha.4 | 网易云信服务器 |
| `rss-parser` | 3.13.0 | RSS 源地址（动态） |

---

## 六、内部化改造评估

根据数据敏感度，按改造优先级排列：

### P0 - 必须内部化（用户对话数据直接外泄风险）

**1. LLM 提供商代理层**
- 现状：客户端直接调用各 LLM API，API Key 存本地，对话内容直接发第三方
- 方案：建内部 LLM Proxy 网关，客户端只对接内部地址，由网关转发并控制日志/审计
- 涉及文件：`src/shared/providers/constants.ts`, `src/renderer/services/api.ts`

**2. 有道智云 LLM Gateway**（`openapi.youdao.com`）
- 现状：有道外部 API，需确认是否为内部可控服务
- 方案：确认是否已对接内部模型服务，如非内部则迁移

### P1 - 高优先级内部化（凭证 + 消息内容）

**3. 有道域名业务服务器**
- `lobsterai-server.youdao.com` / `api-overmind.youdao.com`
- 需确认这些服务是否跑在内部机房还是公网，是否可迁移到内网域名

**4. IM 网关直连外部 API**
- 钉钉/飞书/Telegram/Discord/QQ 的凭证和消息内容在主进程中明文传输
- 方案：通过内部代理或网关中转，避免 API Secret 暴露在客户端

### P2 - 中优先级（凭证安全）

**5. GitHub Copilot / Qwen OAuth**
- 使用公开 Client ID，走外部 OAuth，Token 存本地
- 可接受，但需评估是否要统一走内部 SSO

**6. POPO 插件**（`npm.nie.netease.com`）
- 当前从网易内网 npm 拉取，需确认构建环境和分发包中已包含此依赖

### P3 - 可观察（本地/低风险）

- **Ollama**（`localhost:11434`）- 本地模型，无外泄风险
- **MCP Bridge**（`127.0.0.1`）- 本机内部通信，无风险
- **OpenClaw Gateway**（本地端口）- 本机服务，无风险

---

## 七、整体风险汇总

| 类别 | 外部调用数 | 最高风险点 |
|------|-----------|-----------|
| LLM API | 15 个提供商 | 用户对话全量外发 |
| 有道业务服务 | 5 个端点 | 用户 token、版本信息 |
| IM 网关 | 5 个平台直连 | 凭证 + 消息内容 |
| OAuth 流 | 2 套 | Token 存本地 |
| 隐式 SDK | 4 个 | 随 IM/AI 调用外发 |

**最紧迫的改造**是建立内部 LLM Proxy 网关，让所有模型调用从客户端经内部网关中转，彻底消除客户端直连第三方 LLM 导致的对话数据外泄风险。
