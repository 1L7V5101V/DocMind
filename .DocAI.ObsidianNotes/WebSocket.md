WebSocket 是一种 **全双工通信协议**，允许服务端主动向客户端推送数据，而不像 HTTP 那样必须由客户端先发起请求。

在项目中，WebSocket 的用途就是配合 `AgentProgressBroadcaster`，让后端**主动推**Agent 执行进度给前端工作台，而不需要前端轮询。简单说：

- HTTP 模式：前端每隔几秒问一次"执行完了吗？" → 浪费、延迟
- WebSocket 模式：后端执行完一步，**直接推**给前端"第 2 步完成了" → 实时、高效

项目中用的是 **STOMP over WebSocket**（Spring Messaging），前端订阅 `/topic/agent/progress`、`/topic/agent/users/{userId}` 等通道即可实时接收事件。