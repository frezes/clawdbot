## 核心功能

### 全渠道消息 (Multi-Channel Inbox)

- 支持 **WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、BlueBubbles、IRC、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、Nextcloud Talk、Nostr、Synology Chat、Tlon、Twitch、Zalo、Zalo Personal、WeChat、WebChat**
- 支持 macOS、iOS/Android 客户端
- 支持消息路由到不同的 Agent（隔离的工作区和会话）

### Gateway 控制平面

- 单个控制平面管理会话、存在状态、配置、Cron、Webhook
- 提供 Web UI 和 Canvas 托管
- CLI 命令行工具：gateway、agent、send、onboarding、doctor

### 强大的工具执行力

- **Browser** — 无头浏览器自动化
- **Canvas** — 实时可视化工作区
- **Nodes** — 设备节点管理
- **Cron** — 定时任务
- **Sessions** — 会话管理
- **Skills** — 可扩展技能系统

### 语音与对话

- **Voice Wake** — macOS/iOS 语音唤醒词
- **Talk Mode** — Android 连续语音（ElevenLabs + 系统 TTS 回退）
- **Live Canvas** — Agent 驱动的可视化工作区

### 本地优先与隐私

- 所有配置、长短期记忆和敏感数据存储在本地设备
- 支持 OpenAI、Claude 等云端模型
- 也可对接 **Ollama** 或 **LM Studio** 实现完全离线 AI

---

## 借助 KubeSphere 扩展部署

### 配置说明

扩展内部使用 [app-template](https://github.com/bjw-s-labs/helm-charts) 构建，允许你通过 values 生成任意工作负载及配置。

### 1. 快速部署访问

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

```bash
# 访问 Web UI
kubectl port-forward -n extension-openclaw svc/openclaw 18789:18789
# 打开 http://localhost:18789，输入 Gateway Token，点击 Connect
```
### 2. 安全配置访问

### 3. 更多自定义配置

### 4. 其他

#### 4.1 连接设备

```bash
# 批准配对请求
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices list
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices approve <REQUEST_ID>
```



