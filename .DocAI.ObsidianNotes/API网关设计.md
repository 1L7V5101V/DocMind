这个项目的 API 网关主要用了三个设计模式：

1. **过滤器链模式（Filter Chain / 责任链）** — `AuthGlobalFilter` 实现 `GlobalFilter` + `Ordered`，请求依次经过白名单检查 → JWT 验证 → 添加用户请求头 → 转发下游。Spring Cloud Gateway 本身就是这个模式的内核。

2. **网关代理模式（Gateway Proxy）** — `GatewayConfig` 中定义路由规则：按路径前缀 `/api/users/**` → `user` 服务、`/api/ai/**` → `ai` 服务等，统一入口屏蔽后端服务细节。

3. **异步消息驱动（通过 RabbitMQ）** — 鉴通过程中异步发送网关日志和告警消息到 MQ，不阻塞主请求链路。