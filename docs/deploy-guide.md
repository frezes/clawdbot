# 在 KubeSphere 上运行 Clawdbot：自托管 AI 助手的云原生实践

> **项目更名说明**：Clawdbot 项目已更名为 **Moltbot**，当前官方仓库为 [https://github.com/moltbot/moltbot](https://github.com/moltbot/moltbot)。本文档基于最新仓库内容编写。


## 一、什么是 Clawdbot？

**Clawdbot** 是一个开源的、自托管的个人 AI 助手框架。它以本地服务或守护进程的形式运行，通过接入不同的消息通道（如 Telegram、Discord、Slack 等）与用户交互，并根据配置执行自动化任务。

核心特性：

- **完全自托管**：非 SaaS，数据和运行完全由用户控制
- **CLI + Gateway 服务**：核心运行方式，支持多种部署模式
- **配置驱动**：通过配置文件定义 Bot 的行为和接入通道
- **长期运行**：适合作为后台服务持续运行

Clawdbot 更偏向"**可编排的 AI 助手代理**"，而非传统意义上的云服务或平台组件。

---

## 二、为什么在 KubeSphere 上运行？

虽然 Clawdbot 可以直接在本地或单台服务器上运行，但在团队协作或长期运行场景中，使用 Kubernetes 平台更具优势。KubeSphere 提供了完整的可视化运维能力，将 Clawdbot 部署在 KubeSphere 上可解决以下问题：

| 优势 | 说明 |
|------|------|
| **部署标准化** | 避免手动维护本地守护进程，实现一键部署 |
| **配置集中管理** | 通过 ConfigMap / Secret 管理 Bot 配置和密钥 |
| **运行状态可观测** | 统一查看日志和 Pod 状态，实时监控运行状况 |
| **可重复部署** | 同一套定义可在不同环境复用，保证一致性 |

对于希望将 Bot 类服务纳入云原生运维体系的团队，这是一种更稳妥的方式。

---

## 三、部署前准备

### 环境要求

- **KubeSphere 集群**：已部署完成，版本要求 v4.x 及以上
- **已安装扩展**：KubeSphere Gateway 及 cert-manager 扩展
- **镜像仓库**：可访问公有或私有镜像仓库

---

## 四、在 KubeSphere 中部署

### 步骤一：安装扩展组件

将  Moltbot(Clawdbot) 扩展组件推送到 KubeSphere 扩展商店，并进行安装。在安装过程中，通过扩展组件配置加载相关密钥：

![安装界面](http://pek3b.qingstor.com/kubesphere-community/images/clawdbot1.png)

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

> **注意**：当前配置中禁用了控制界面中的设备识别和配对功能。

### 步骤二：配置 Ingress

通过 KubeSphere Gateway 扩展配合 cert-manager 扩展，使用 HTTPS 协议将 Clawdbot 服务以 Ingress 的方式对外暴露。

首先，在集群中创建并启用集群网关，作为统一的 Ingress 入口。随后，在该网关之上为 Clawdbot 服务创建对应的应用路由，并通过 cert-manager 自动签发和管理 TLS 证书，从而实现安全的 HTTPS 访问。

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: clawdbot
  namespace: extension-clawdbot
  annotations:
    cert-manager.io/cluster-issuer: default-issuer  # 借助 cert-manager 自动生成和管理 TLS 证书
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

### 步骤三：访问控制页面

使用以下命令获取 Gateway Token：

```bash
kubectl get secret clawdbot -n extension-clawdbot \
  -o jsonpath='{.data.gatewayToken}' | base64 --decode
```

然后在浏览器中输入以下地址访问 Clawdbot 控制页面：

```
https://{ingress 暴露地址}/?token=${token}
```

![控制页面](http://pek3b.qingstor.com/kubesphere-community/images/clawdbot2.png)

---

## 五、运维建议

在生产或长期运行场景中，建议遵循以下最佳实践：

- **资源限制**：为 Clawdbot 设置合理的 CPU / 内存限制，避免资源争用
- **配置管理**：通过修改 ConfigMap 调整 Bot 行为，而非重新构建镜像
- **版本更新**：定期更新镜像，跟进上游版本变更，获取新功能和修复
- **密钥轮换**：对 Secret 中的 Token 进行定期轮换，增强安全性

> **注意**：Clawdbot（Moltbot）本身并不负责外部平台的权限管理，相关 OAuth 或 Bot Token 仍需在对应平台侧正确配置。

---

## 六、总结

Clawdbot 是一个典型的可自托管 AI 助手框架，适合以后台服务的方式长期运行。通过 KubeSphere 部署，可以将其纳入标准化的 Kubernetes 运维体系，获得更好的可维护性和可观测性。

这种方式并不会改变 Clawdbot 的功能边界，但可以显著降低部署和运行成本，适合团队级或多环境使用场景。
