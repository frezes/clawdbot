# 在 KubeSphere 上运行 Clawdbot：自托管 AI 助手的云原生实践

> 说明：Clawdbot 项目已更名为 **Moltbot**，当前官方仓库为 https://github.com/moltbot/moltbot。本文以最新仓库内容和 README 为准。


## 一、什么是 Clawdbot？

**Clawdbot** 是一个开源的、自托管的个人 AI 助手框架。它以本地服务或守护进程的形式运行，通过接入不同的消息通道（如 Telegram、Discord、Slack 等）与用户交互，并根据配置执行自动化任务。

从官方 README 可以明确以下几点：

- Clawdbot 并非 SaaS，而是**完全自托管**
- 核心运行方式为 **CLI + Gateway 服务**
- 通过配置文件定义 Bot 的行为和接入通道
- 适合作为长期运行的后台服务

该项目更偏向“**可编排的 AI 助手代理**”，而不是传统意义上的云服务或平台组件。


## 二、为什么在 KubeSphere 上运行 Clawdbot？

虽然 Clawdbot 可以直接在本地或单台服务器上运行，但在团队或长期运行场景中，使用 Kubernetes 平台更具优势，而 KubeSphere 提供了完整的可视化运维能力。

将 Clawdbot 部署在 KubeSphere 上，主要解决以下问题：

- **部署标准化**：避免手动维护本地守护进程
- **配置集中管理**：通过 ConfigMap / Secret 管理 Bot 配置和密钥
- **运行状态可观测**：统一查看日志和 Pod 状态
- **可重复部署**：同一套定义可在不同环境复用

对于希望将 Bot 类服务纳入云原生运维体系的团队，这是一种更稳妥的方式。


## 三、部署前准备

### 1. KubeSphere 环境

- 已部署完成的 KubeSphere 集群（要求 v4.x 及以上）
- 已安装 KubeSphere Gateway 及 certmanager 扩展
- 可访问镜像仓库（公有或私有）



## 四、在 KubeSphere 中部署 Clawdbot

### 1. 将 Moltbot(Clawdbot) 扩展组件推送到 KubeSphere 扩展商店，并进行安装

![](http://pek3b.qingstor.com/kubesphere-community/images/clawdbot1.png)

您需要将相关秘钥通过扩展组件配置加载到 Clawdbot。

```yaml
clawdbot:
  secrets:
    create: true
    data:
      # Set via --set or environment variables
      anthropicApiKey: ""
      openaiApiKey: ""
      discordBotToken: ""
      telegramBotToken: ""
      gatewayToken: ""  # Auto-generated if empty
```

> 注:  当前配置中禁用了控制界面中的设备识别和配对功能。

### 2. 配置 Ingress，将 Clawdbot 服务暴露

您可以通过 KubeSphere Gateway 扩展 配合 cert-manager 扩展，使用 HTTPS 协议 将 Clawdbot 服务以 Ingress 的方式对外暴露。

首先，在集群中创建并启用 集群网关，作为统一的 Ingress 入口。随后，在该网关之上为 Clawdbot 服务创建对应的应用路由，并通过 cert-manager 自动签发和管理 TLS 证书，从而实现安全的 HTTPS 访问。

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: clawdbot
  namespace: extension-clawdbot
  annotations:
    cert-manager.io/cluster-issuer: default-issuer       # 借助 cert-manager, 为 ingress 自动生成和创建证书
    kubesphere.io/creator: admin
spec:
  ingressClassName: kubesphere-router-cluster
  tls:
    - hosts:
        - 172.31.19.4.nip.io
      secretName: clawdbot-cert
  rules:
    - host: 172.31.19.4.nip.io
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: clawdbot
                port:
                  number: 18789

```

### 3. 获取 token，登录 Clawdbot 控制页面

使用如下命令获取 gateway token:

```
kubectl get secret clawdbot -n extension-clawdbot \
  -o jsonpath='{.data.gatewayToken}' | base64 --decode
```

然后我们浏览器中输入 ingress 地址+ token 访问 Clawdbot 控制页面:

```
https://172.31.19.4.nip.io:32045/?token=${token}
```

![](http://pek3b.qingstor.com/kubesphere-community/images/clawdbot2.png)




## 五、运维建议

在生产或长期运行场景中，建议：

- 为 Moltbot 设置合理的 CPU / 内存限制

- 通过修改 ConfigMap 调整 Bot 行为，而不是重新构建镜像

- 定期更新镜像，跟进上游版本变更

- 对 Secret 中的 Token 进行定期轮换

需要注意的是，Moltbot 本身并不负责外部平台的权限管理，相关 OAuth 或 Bot Token 仍需在对应平台侧正确配置。


## 六、总结

Moltbot (Clawdbot) 是一个典型的可自托管 AI 助手框架，适合以后台服务的方式长期运行。通过 KubeSphere 部署，可以将其纳入标准化的 Kubernetes 运维体系，获得更好的可维护性和可观测性。

这种方式并不会改变 Moltbot 的功能边界，但可以显著降低部署和运行成本，适合团队级或多环境使用场景。