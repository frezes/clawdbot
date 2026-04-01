## 核心功能

### 本地优先 Gateway

- 以 Gateway 作为统一控制平面，集中管理会话、频道、工具、状态、Cron 和 Webhook
- 内置 Web UI、WebChat 和 Canvas Host，同时提供 `gateway`、`agent`、`message send`、`onboarding`、`doctor` 等 CLI 能力

### 多渠道消息接入

- 可直接接入 **WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、BlueBubbles / iMessage、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、WeChat、WebChat** 等多种消息渠道
- 支持 macOS 菜单栏应用，以及 iOS / Android 节点协同接入

### 多 Agent 路由

- 支持按渠道、账号或对话对象将消息路由到不同 Agent
- 每个 Agent 可拥有独立工作区和会话上下文，便于隔离不同任务与身份

### 语音与可视化交互

- 提供 **Voice Wake** 与 **Talk Mode**，支持唤醒词、连续语音对话和系统 TTS 回退
- 提供 **Live Canvas**，让 Agent 以可视化方式驱动和展示工作过程

### 工具与自动化

- 内置 **Browser、Canvas、Nodes、Sessions、Cron、Skills** 等工具能力
- 支持浏览器自动化、设备节点控制、定时任务、Webhook 和技能扩展

### 本地数据与模型灵活性

- 配置、记忆及敏感数据默认保存在本地设备，强调本地优先与隐私控制
- 既可接入 OpenAI、Claude 等云模型，也可对接 **Ollama**、**LM Studio** 等本地模型实现离线运行

---

## 部署配置

> 扩展内部使用 [app-template](https://github.com/bjw-s-labs/helm-charts) 构建，允许你通过 values 生成任意工作负载及配置。

### 1. 快速部署本地访问

适合在本地或内网环境中快速验证 OpenClaw 是否可用。这里通过设置基础环境变量启动服务，再使用 `kubectl port-forward` 将网关端口转发到本机进行访问。

```yaml
openclaw:
  app-template:
    controllers:
      main:
        containers:
          main:
            env:
              OPENCLAW_GATEWAY_TOKEN: "your-gateway-token"
              CORESHUB_API_KEY: "your-coreshub-api-key"
```

- `OPENCLAW_GATEWAY_TOKEN`：OpenClaw Gateway 的访问令牌，用于 Web UI 连接认证和设备接入认证，建议设置为自定义高强度随机字符串
- `CORESHUB_API_KEY`：模型服务的 API Key，OpenClaw 通过它访问配置中的大模型提供方

```bash
# 访问 Web UI
kubectl port-forward -n extension-openclaw svc/openclaw 18789:18789
# 打开 http://localhost:18789，输入 Gateway Token，点击 Connect
```
### 2. 安全配置访问

当需要从浏览器、移动端或外部网络访问 OpenClaw 时，建议通过集群入口以 HTTPS 方式暴露服务，例如使用 **Ingress** 或 **Gateway API**。

建议按以下顺序完成配置并访问：

- 先为 OpenClaw 配置可访问的 HTTPS 域名，并在 `openclaw.json` 的 `gateway.controlUi.allowedOrigins` 中加入对应域名，避免控制台跨域被拒绝
- 通过该 HTTPS 地址访问 OpenClaw UI，并使用 Gateway Token 完成连接认证
- 在 UI 可正常访问后执行设备配对审批

**扩展的完整配置示例如下：**

```yaml
openclaw:
  app-template:
    controllers:
      main:
        containers:
          main:
            env:
              OPENCLAW_GATEWAY_TOKEN: "your-gateway-token"
              CORESHUB_API_KEY: "your-coreshub-api-key"

    configMaps:
      config:
        enabled: true
        data:
          # Shell alias for interactive sessions (sourced by ~/.bashrc)
          # @schema type:string
          bash_aliases: |
            alias openclaw='node /app/dist/index.js'
          # OpenClaw configuration - JSON5 format
          # Sensitive values use ${ENV_VAR} substitution from environment secret
          # @schema type:string
          openclaw.json: |
            {
              // Gateway configuration
              "gateway": {
                "port": 18789,
                "mode": "local",
                "controlUi": {
                  "enabled": true,
                  "allowedOrigins": ["https://control.example.com"]
                  // required for non-loopback Control UI access
                  // dangerouslyAllowHostHeaderOriginFallback: false, // dangerous Host-header origin fallback mode
                  // allowInsecureAuth: false,
                  // dangerouslyDisableDeviceAuth: false,
                },
              },

              // Browser configuration (Chromium sidecar)
              "browser": {
                "enabled": true,
                "defaultProfile": "default",
                "profiles": {
                  "default": {
                    "cdpUrl": "http://localhost:9222",
                    "color": "#4285F4"
                  }
                }
              },

              // Agent configuration
              "agents": {
                "defaults": {
                  "workspace": "/home/node/.openclaw/workspace",
                  "model": {
                    // Uses CORESHUB_API_KEY from environment
                    "primary": "coreshub/MiniMax-M2.5"
                  },
                  "userTimezone": "UTC",
                  "timeoutSeconds": 600,
                  "maxConcurrent": 1
                },
                "list": [
                  {
                    "id": "main",
                    "default": true,
                    "identity": {
                      "name": "OpenClaw",
                      "emoji": "🦞"
                    }
                  }
                ]
              },

              "models": {
                "mode": "merge",
                "providers": {
                  "coreshub": {
                    "baseUrl": "https://openapi.coreshub.cn/v1",
                    "apiKey": "${CORESHUB_API_KEY}",
                    "api": "openai-completions",
                    "models": [
                      {
                        "id": "MiniMax-M2.5",
                        "name": "MiniMax-M2.5 (Custom Provider)",
                        "reasoning": false,
                        "input": [
                          "text"
                        ],
                        "cost": {
                          "input": 0,
                          "output": 0,
                          "cacheRead": 0,
                          "cacheWrite": 0
                        },
                        "contextWindow": 204800,
                        "maxTokens": 4096
                      }
                    ]
                  }
                }
              },

              // Session management
              "session": {
                "scope": "per-sender",
                "store": "/home/node/.openclaw/sessions",
                "reset": {
                  "mode": "idle",
                  "idleMinutes": 60
                }
              },

              // Logging
              "logging": {
                "level": "info",
                "consoleLevel": "info",
                "consoleStyle": "compact",
                "redactSensitive": "tools"
              },

              // Tools configuration
              "tools": {
                "profile": "full",
                "web": {
                  "search": {
                    "enabled": false
                  },
                  "fetch": {
                    "enabled": true
                  }
                }
              }
            }
```

**批准设备配对：**

```bash
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices list
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices approve <REQUEST_ID>
```

### 3. 更多自定义配置

扩展支持通过 Helm values 和 OpenClaw 配置文件进行灵活配置，满足不同部署环境和使用需求。

**Helm 部署配置**

- 该扩展基于 `app-template` 构建，你可以按需自定义服务暴露、持久化、资源限制、环境变量等任意 Helm 配置

**OpenClaw 应用配置**

- OpenClaw 本身的运行配置可通过 `config` 中的 `openclaw.json` 进行覆盖，例如调整 `agents`、`models`、`session`、`tools` 等配置项
