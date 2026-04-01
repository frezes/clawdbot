# OpenClaw 部署指南

## 部署前准备

### 环境要求

- 已部署完成的 KubeSphere 集群
- 可用的 HTTPS 域名和证书配置能力
- 可访问的模型服务 API Key

## 在 KubeSphere 中部署

### 1. 安装扩展组件

在 KubeSphere 扩展商店中安装 OpenClaw，并在安装时配置基础环境变量：

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

- `OPENCLAW_GATEWAY_TOKEN`：OpenClaw Gateway 的访问令牌，用于 Web UI 连接认证和设备接入认证
- `CORESHUB_API_KEY`：模型服务的 API Key，OpenClaw 通过它访问配置中的大模型提供方

### 2. 配置 HTTPS 访问

当需要从浏览器、移动端或外部网络访问 OpenClaw 时，建议通过集群入口以 HTTPS 方式暴露服务，例如使用 **Ingress** 或 **Gateway API**。

建议按以下顺序完成配置并访问：

- 先为 OpenClaw 配置可访问的 HTTPS 域名和 TLS 证书
- 如果通过非本地回环地址访问，需要在 `openclaw.json` 的 `gateway.controlUi.allowedOrigins` 中加入对应的 HTTPS 域名，避免控制台跨域被拒绝
- 完成域名配置后，再通过 HTTPS 地址访问控制界面

可通过 `config` 覆盖 OpenClaw 配置，例如：

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
          openclaw.json: |
            {
              "gateway": {
                "port": 18789,
                "mode": "local",
                "controlUi": {
                  "enabled": true,
                  "allowedOrigins": ["https://control.example.com"]
                }
              }
            }
```

### 3. 访问 UI 并批准设备配对

使用以下命令获取 Gateway Token：

```bash
kubectl exec -n extension-openclaw deployment/openclaw -c main -- printenv OPENCLAW_GATEWAY_TOKEN
```

然后通过你的 HTTPS 地址访问 OpenClaw UI，并使用 Gateway Token 完成连接认证。

如需接入移动端或其他设备，可在 UI 可正常访问后执行以下命令批准配对请求：

```bash
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices list
kubectl exec -n extension-openclaw deployment/openclaw -c main -- node dist/index.js devices approve <REQUEST_ID>
```
