# DocMind — AGENTS.md

## What is this

Spring Cloud microservices platform for enterprise AI document processing. 6 Maven modules deployed via Docker Swarm.

## Build & run

```bash
# Full build from root
mvn clean package -DskipTests

# Local dev (disables Nacos, targets localhost)
# Use the `local` Spring profile:
spring.profiles.active=local
# Each service has application-local.yml for local dev settings

# Build single service
cd <service>/ && mvn clean package -DskipTests
```

## Architecture

| Module | ArtifactId | Port | spring.application.name | Base package |
|--------|-----------|------|------------------------|-------------|
| common | common | — | — | com.javaee.common |
| user-service | user-service | 8081 | `user` | com.javaee.user |
| file-service | file-service | 8082 | `file` | com.javaee.fileservice |
| ai-service | ai-service | 8083 | `ai-service` | com.javaee.aiservice |
| document-service | document-service | 8084 | `document` | com.javaee.documentservice |
| gateway-service | gateway-service | 8080 | `gateway` | com.javaee.gateway |

**Important:** `spring.application.name` differs from artifact IDs. Docker Compose and gateway routing use the application name (short names like `user`, `file`, `document`, `ai`) as Docker service names and in `gateway.route.*-uri`.

### Common module
Shared JAR (not a boot app — `spring-boot-maven-plugin` has `<skip>true</skip>`). Contains:
- `JwtUtils`, `RedisUtils`, `ValidateUtils`, `FileUtils`, `DateUtils`
- `BaseSecurityConfig` + `JwtAuthenticationFilter` (JWT auth)
- `GlobalExceptionHandler`, `BusinessException`, `TokenException`
- `SwaggerConfig`, `WebMvcConfig`, `RedisConfig`, `FeignGlobalConfig`

Every service imports common and scans it via `@ComponentScan(basePackages = {"com.javaee.<service>", "com.javaee.common"})`.

### Gateway
**Not** `lb://` routing — uses `gateway.route.*-uri` with Docker network service names:
```yaml
gateway:
  route:
    user-uri: http://user:8081
    file-uri: http://file:8082
    ai-uri: http://ai:8083
    document-uri: http://document:8084
```
JWT auth whitelist (endpoints bypassing auth filter): `POST /api/users/login`, `/api/users/register`, `/api/users/refresh`. Add new public endpoints to `AuthGlobalFilter.java`.

## Key conventions

- **Java 17** via `swr.cn-north-4.myhuaweicloud.com/qxk/jdk-17:latest` Docker image
- **MyBatis-Plus**: mapper XML at `classpath:mapper/**/*.xml`, type aliases per-module in `application.yml`
- **Redis**: database per service — user=0, ai=1, file=1, document=2
- **Swagger (SpringDoc)**: `/api-docs` path, `/swagger-ui.html` UI
- **Lombok**: used throughout, `@Data` on entities
- **JWT**: JJWT 0.11.5, stored in `Authorization: Bearer <token>` header
- **Logging**: `com.javaee.<service>: debug` in dev, uses stdout MyBatis SQL logging
- **Nacos**: `spring.cloud.nacos.discovery.server-addr: nacos:8848` in production, disabled in `local` profile
- **AI models**: DashScope (qwen3.6-plus, text-embedding-v1, qwen3-rerank) + OpenAI-compatible endpoint at `https://coding.dashscope.aliyuncs.com/v1`; configurable multi-model support

## Infrastructure

| Service | Image/Version | Ports |
|---------|--------------|-------|
| MySQL | 8.0 | 3306 |
| Redis | 7.0 (AOF + password) | 6379 |
| RabbitMQ | 3.8-management | 5672, 15672 |
| MinIO | latest | 9000, 9001 |
| Nacos | 2.2.0 (standalone) | 8848, 9848, 9849 |
| Prometheus | 2.45.0 | 9090 |
| Grafana | 9.5.0 | 3000 |

Redis started with `--appendonly yes` and `--requirepass ${REDIS_PASSWORD}`. Nacos needs gRPC ports (9848, 9849).

## Deployment

```bash
# Deploy everything
./deploy.sh  # mvn clean package -> docker build -> swarm init -> volumes -> docker stack deploy

# Or manually:
docker stack deploy -c docker-compose-infra.yml docai-infra
docker stack deploy -c docker-compose-services.yml docai-services
```

Deploy order: MySQL → Redis → RabbitMQ → MinIO → Nacos → user/file/ai/gateway/document.

Network: `docker network create --driver overlay --attachable docai-network`

### Environment variables
Copy `.env.example` → `.env` and fill. Key vars: `DOC_AI_MANAGER_HOST`, `DOC_AI_NODE[1-3]_HOST`, `DASHSCOPE_API_KEY`, `OPENAI_API_KEY`, MYSQL_*, REDIS_*, MINIO_*, RABBITMQ_*, `DOC_AI_ADMIN_PASSWORD`, `DOC_AI_USER_PASSWORD`.

## Gotchas

- **`mysql-schema.sql`** is the **Nacos config tables** schema (from Alibaba), NOT the application schema. App tables (`user`, etc.) are created inline in `deploy.sh` via `docker exec mysql`.
- **Gateway 503**: If `lb://` fails, gateway uses `gateway.route.*-uri` with Docker service names. Ensure `spring-cloud-starter-loadbalancer` is in gateway's `pom.xml`.
- **New service**: Add `<module>` to root `pom.xml`, create service with `@EnableDiscoveryClient`, add Dockerfile, add to `deploy.sh` and `docker-compose-services.yml`.
- **New public API endpoint**: Must be added to JWT whitelist in `AuthGlobalFilter.java`.
- **File storage**: Defaults to MinIO (`file.storage.type: minio`). Configurable to local.
- **RabbitMQ**: Each service has its own exchange/queue bindings (see `RABBITMQ_IMPLEMENTATION.md`).
- **MinIO bucket permissions**: Controlled via `bucket-permissions` config — JWT role matched to bucket access groups.
- **Spring Cloud compatibility check**: Disabled (`spring.cloud.compatibility-verifier.enabled: false`) in all services.
