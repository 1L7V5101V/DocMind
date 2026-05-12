
**KnowledgeIndexAgent 和其他两种模式的本质区别：**

|          | ChatReactAgent          | PlanExecuteAgent                 | KnowledgeIndexAgent         |
| -------- | ----------------------- | -------------------------------- | --------------------------- |
| **入口**   | `/chat`（对话）             | `/execute`、`/plan/execute`（规划执行） | 内部触发（文档上传/修改时调用）            |
| **执行方式** | 同步，一轮对话                 | 同步，多步规划→执行→反思                    | **异步**，丢给线程池排队              |
| **核心能力** | 聊天、RAG 问答、简单工具          | 复杂任务拆解、依赖编排、重试                   | 文档→分段→向量化→写库的索引管道           |
| **用户感知** | 实时看到回答                  | 看到计划→逐步执行→最终结果                   | 提交后立即返回，后台慢慢跑，可通过 jobId 查进度 |
| **失败处理** | 直接返回错误                  | 反射重规划 / 单步重试                     | 状态标记 FAILED，支持 retryJob 重试  |
| **数据存储** | ConversationManager（对话） | AgentTaskRegistry（任务快照）          | Redis 任务记录（保留 7 天）          |

简单说：前两者是**面向用户的实时交互流**，KnowledgeIndexAgent 是**面向系统的后台管道**——用户不必等索引跑完，提交文档后就能干别的事。

### 为什么这个也是Agent 链路里面的
看代码结构就清楚了——`KnowledgeIndexAgent` 放在 `com.javaee.aiservice.agent` 包下，和 `AgentExecutionService` 同级，共享同一套基础设施：

- **AgentProgressBroadcaster** — 索引进度通过 WebSocket 广播到 `/topic/agent/progress`，和 Agent 执行事件共用一个通道
- **Redis 任务追踪** — 同样用 `agent:knowledge:job:` 前缀存状态（PENDING→PARSING→EMBEDDING→INDEXED/FAILED），和 AgentTaskRegistry 的模式一致
- **retryJob** — 重试接口和 Agent 的单步重试设计思路相同

它叫 Agent 的原因是**遵循了相同的抽象**：

```
输入 → 任务（Job/Task）→ 管道执行 → 状态追踪 → 完成/失败 → 可查询/可重试
```

只是它不是面向用户的对话交互，而是**面向系统自身的 Agent**——文档变化时自动触发，独立跑完索引管道，不需要用户等待。所以架构图上把它和 ChatReactAgent、PlanExecuteAgent 并列，因为三者共用同一种"任务驱动"的设计范式。