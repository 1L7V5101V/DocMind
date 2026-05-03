# DocMind：文档智能处理平台（微服务）

**核心技术栈**：Java 17、Spring Boot 3.2.0、Spring Cloud 2023.0.3、Spring Cloud Alibaba、MyBatis-Plus、MySQL、Redis、RabbitMQ、Spring AI、Docker / Docker Swarm、AI Skills

**项目描述**：面向文档全流程管理与智能分析的微服务平台，覆盖用户、文件、文档与AI服务等能力。AI服务基于Spring AI构建，整合AI Skills组件，提供文档摘要、纠错、关键词提取与分析能力，并扩展 AI Agent 架构，支持 RAG 知识库问答、多范式对话代理等场景，通过微服务协同与容器化部署保障平台高可用。

核心职责与贡献：

1. **设计并实现统一 AI Agent 执行链路**：基于 AgentExecutionService 支持任务规划、工具调用、结果观察、反思重规划和最终回答，兼容对话式 Agent 与 Plan-Execute Agent 场景，并通过 Redis 和 WebSocket 实现任务状态持久化、执行时间线和实时进度推送。构建 Agent 工具注册与执行框架，基于 AgentToolRegistry 统一管理 RAG、文本处理、文件管理、PPT 生成等工具能力，支持参数 Schema 校验、缺参追问、模板变量解析、失败重试和危险操作审批，提升 Agent 调用安全性和可控性。多轮对话任务完成率达 89%。通过spring ai统一调度底层大模型，保障 Agent 决策链路的高效与稳定。
2. **设计并实现企业级 RAG 知识库问答系统**：设计并实现 RAG 知识库问答系统，支持章节分段、固定长度分段、语义分段和自动分段策略，集成 DashScope Embedding 生成向量，采用 Redis 持久化向量与元数据，并基于内存 HNSW 索引实现高效语义检索。在 1000 条业务问答样本上实现 Top-5 召回率 93%、检索支撑回答准确率 82%，优化向量检索性能后 P95 响应时间降至 420ms，服务可用性达 99.95%。
3. **实现混合检索、知识索引管理Agent与重排序能力**：结合向量相似度、BM25 关键词召回和 DashScope Rerank 精排优化检索结果，并通过 userId 、 knowledgeBaseId 元数据过滤实现多用户知识库隔离。设计 KnowledgeIndexAgent 知识索引管理 Agent，支持异步索引、内容 Hash 去重、旧索引清理、失败任务重试、索引任务状态查询和质量评估，保障知识库更新链路稳定可靠。
4. **构建统一 AI 服务入口与 Skills 能力封装**：基于 Java 17、Spring Boot 3.2.0 与 Spring AI，封装文档摘要、纠错、关键词提取等 AI Skills 核心能力，依托AIOps实现异常管控与性能监控，对接外部大模型实现多模型兼容；支撑的文档智能任务人工验收通过率达 91%，单任务平均响应耗时控制在 1150ms 以内。
5. **实现核心业务数据持久化与缓存优化**：运用 MyBatis-Plus、MySQL设计用户、文件、文档等数据表结构，保障一致性与查询性能；结合 Redis 实现热点数据缓存，提升系统响应速度，降低数据库压力。
6. **异步解耦与文档存储**：基于 RabbitMQ 处理文档上传、AI 任务调度等异步场景，优化任务执行效率；整合 MinIO 实现文档文件的安全存储与高效访问，支撑全流程文档处理。