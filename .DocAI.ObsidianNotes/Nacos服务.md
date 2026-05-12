**Nacos** 是阿里巴巴开源的**服务发现 + 配置管理**平台，这个项目里它干两件事：

1. **服务发现（电话簿）** — 每个微服务启动时向 Nacos 注册自己的 IP:端口，调用方（比如网关要转发到 AI 服务）只需知道服务名 `ai-service`，Nacos 自动解析到当前可用的实例地址。实例挂了会自动摘除。

2. **配置中心** — 项目的配置（数据库地址、模型 API Key 等）放在 Nacos 上管理，修改配置热生效，无需重启服务。

这个项目在 Nacos 上注册了 5 个服务：`user-service`、`file-service`、`ai-service`、`document-service`、`gateway-service`。Nacos 本身也是 Docker 部署的，配置文件里指定地址为 `nacos:8848`。