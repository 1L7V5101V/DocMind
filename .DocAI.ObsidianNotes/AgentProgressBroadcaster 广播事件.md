`AgentProgressBroadcaster` 是基于 **[[WebSocket]] (STOMP)** 的实时事件推送器，让前端工作台能实时看到 Agent 执行进度。它通过 `SimpMessagingTemplate` 向以下三个通道广播 `AgentProgressEvent`：

| 通道 | 谁订阅 | 示例 |
|------|--------|------|
| `/topic/agent/progress` | 全部用户 | 全局监控 |
| `/topic/agent/users/{userId}` | 指定用户 | 用户个人工作台 |
| `/topic/agent/tasks/{traceId}` | 指定任务 | 任务详情页 |
| `/topic/agent/knowledge/{jobId}` | 知识索引任务 | 索引进度页 |

### 事件类型 `eventType`

来自 `AgentExecutionService.publishTaskEvent()` 的调用点：

| eventType             | 触发时机          | progress（前端表示任务执行的**整体进度百分比**） |
| --------------------- | ------------- | ------------------------------ |
| `task_started`        | Agent 开始执行    | 0                              |
| `task_continued`      | 续接历史任务        | 当前进度                           |
| `step_planned`        | 每一步规划完成       | 按计划步骤数计算                       |
| `step_finished`       | 每一步执行完成（含结果）  | 按已调用工具数计算                      |
| `reflection_finished` | 反思完成          | —                              |
| `task_waiting_user`   | 等待用户审批/补充参数   | 95                             |
| `task_finished`       | 任务正常结束        | 100                            |
| `task_cancelled`      | 任务取消（用户取消/超时） | 100                            |

另外 `KnowledgeIndexAgent` 也会广播知识索引任务的事件（`parsing`/`embedding`/`indexed`/`failed`），走 `/topic/agent/knowledge/{jobId}` 通道。

### 事件数据结构 `AgentProgressEvent`

```json
{
  "eventType": "step_finished",
  "traceId": "uuid",
  "userId": "user123",
  "status": "success",
  "message": "步骤已执行: rag-answer",
  "progress": 60,
  "timestamp": 1718000000000,
  "payload": { ... }
}
```

广播失败不会影响主任务链路（`catch (Exception ignored)`）。