在项目设计中，**`ctx`** 是 **context（上下文）** 的缩写。它作为 Redis 的 key 前缀（`ctx:{conversationId}`），由 `ContextManager` 管理，用来存储 **Agent 执行过程中的结构化状态数据**，例如 `userId`、`traceId`、`lastTool`、`lastObservation`、`answer`、`source`、`fileUrl`、`stepResults` 等。

与之区分的是：
- **`conv:`** — 对话消息历史（用户/助手聊天记录）
- **`agent:task:`** — Agent 任务执行快照（计划、时间线、状态）