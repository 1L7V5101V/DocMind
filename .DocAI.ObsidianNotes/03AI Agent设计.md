3.1 Agent架构设计

- ChatReactAgent ：对话式Agent，支持工具调用

- PlanExecuteAgent ：任务规划与执行Agent

- KnowledgeIndexAgent ：知识库索引管理Agent

3.2 工具调用设计

- MCP（Model Context Protocol）集成

- 技能注册与执行框架

- 工具参数校验与错误处理

3.3 会话管理设计

- 对话上下文维护

- 长期记忆存储

- 多轮对话状态追踪

## 3.1 Agent 架构设计

### 解决了什么问题

Agent 架构在项目中主要解决的是“用户自然语言任务如何被系统理解、拆解、调用工具并形成最终结果”的问题。

**核心矛盾：** 用户需要一句自然语言完成复杂操作（“根据知识库回答问题”“生成 PPT”“删除文件并确认”等），而不是手动选择每个后端接口。

**解决方案：** 统一 Agent 链路处理
- AgentExecutionService 完成任务规划、工具调用、RAG 检索、执行反思、结果汇总、对话记忆保存
- AgentController 提供[[三种入口]]：
  - /chat → 对话式（类似 ChatReactAgent）
  - /execute、/plan/execute → 复杂任务规划执行（类似 PlanExecuteAgent）
  - [[KnowledgeIndexAgent]] → 独立的知识库索引管理（异步任务、失败重试、索引删除）

**效果：** 用自然语言即可完成复杂跨服务操作

### 为什么这么设计

| 设计选择 | 为什么这么定 |
|---------|------------|
| **统一 Agent 执行器 + 多入口适配** | Agent 同时覆盖 RAG 问答、文本处理、文件管理、PPT 生成、知识库索引等多种业务，避免 ChatReactAgent / PlanExecuteAgent 等各自实现导致规划、工具注册、参数校验、权限控制、审计、重试重复建设 |
| **AgentPlanStep 表示执行步骤** | 同一套链路通过 toolName / params / dependsOn / successCriteria / retryPolicy 支持单轮问答和多步任务 |
| **AgentToolResult 记录每次调用结果** | 支持 ReAct（思考→行动→观察）和 Plan-and-Execute（规划→执行→反思）两种模式共用同一执行器 |

### 具体怎么设计的

**ChatReactAgent（/chat 系列接口）**
- 用户消息 → 构造 AgentExecutionRequest（含 conversationId）→ AgentExecutionService 执行
- 合并 ContextManager 上下文 → 模型生成计划 → 执行工具 → 结果写回上下文 → 模型综合生成回答 → ConversationManager 保存

**PlanExecuteAgent（/execute、/plan/execute）**
- 构造 planner prompt，要求模型输出 JSON 数组计划（id / description / toolName / params / reasoning）
- 规范化处理：工具不存在→降级 direct-answer；危险操作→标记审批；缺参→从上下文补齐
- 执行阶段：检查依赖→去重重复调用→控制迭代/工具次数→失败重试→反思重规划→单步重试
- 任务快照存 AgentTaskRegistry，前端可查计划、结果、时间线、执行状态

**KnowledgeIndexAgent（独立后台管道）**
- 索引管道：解析 → 内容 hash → 元数据增强 → 向量化 → 写库 → 质量评估
- 支持：同步/异步索引、任务状态查询、失败重试、任务列表、索引删除
- Redis 前缀 agent:knowledge:job:，保存 PARSING / EMBEDDING / INDEXED / FAILED 状态
- 偏后台任务管理，重点保证 RAG 知识库持续接收文档、更新索引、追踪失败原因

### 为什么不采用别的方式

| 方案                                               | 不采用的原因                                                   |
| ------------------------------------------------ | -------------------------------------------------------- |
| **ChatReactAgent / PlanExecuteAgent 完全拆成两套独立实现** | 底层都需同一套工具注册、参数校验、上下文维护、权限判断、结果汇总，拆分导致重复代码和入口能力不一致        |
| **所有任务做成简单聊天调用**                                 | 文件删除、版本切换、RAG 检索等需要明确工具调用和结构化参数，不能只依赖模型自由生成文本            |
| **知识索引塞进统一执行循环**                                 | 索引任务耗时较长，需异步执行、状态追踪、失败重试、质量评估，独立 KnowledgeIndexAgent 更适合 |

## 3.2 工具调用设计

### 解决了什么问题

工具调用设计解决的是“Agent 不能只会说话，还要能安全、可控地调用业务能力”的问题。

**核心矛盾：** 能力分散在不同服务中（RAG 问答、文本处理、文件管理、PPT 生成等），模型不知道有哪些能力可用、每个工具需要什么参数、哪些操作有风险、失败后如何处理。

**解决方案：**
- AgentToolRegistry 建立统一工具目录
- AgentExecutionService 负责参数校验、权限判断、执行分发、错误处理

**效果：** 模型只通过工具目录选择能力，服务端统一管控执行安全

### 为什么这么设计

| 设计选择                         | 为什么这么定                                                                                                    |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| **注册表模式（AgentToolRegistry）** | 工具必须对模型和系统同时可见：规划时需要工具名/描述/参数 schema/风险等级，执行时需要根据 toolName 找到业务逻辑并校验参数                                    |
| **planner prompt 动态拼接工具列表**  | 限定模型只能从注册工具中选择，减少幻觉式调用不存在工具；服务端仍会二次校验                                                                     |
| **现阶段内部工具注册 + 审批，预留 MCP 方向** | 项目已有 Spring Service/Redis/MinIO/RAG 等成熟组件，直接引入 MCP 增加复杂度；先用内部机制落地，后续将工具注册表适配为 MCP Tool List/Tool Call 更平滑 |

### 具体怎么设计的

**工具注册（AgentToolRegistry）**
- 构造函数中注册所有工具，每个声明：名称、描述、参数说明、必填参数、是否危险、分类、是否需要用户动作、风险等级
- 示例：rag-answer（检索+生成答案）、rag-search（只返回片段）、file-delete/file-restore/file-version-switch（标记危险）、ask-user（参数缺失时澄清）
- 参数 schema 由 AgentToolParameterDefinition 表示：根据参数名推断类型（int/bool），设置限制（topK 最大值、strategy 可选值等）

**工具执行（executeStep / dispatchTool）**
1. 解析模板变量 ${context.xxx}、${steps.xxx} → 引用前序工具结果
2. 参数校验 → 缺失/非法 → 返回 actionRequired，提示用户补齐
3. 危险操作审批 → AgentApprovalService 生成 agentApprovalToken，用户确认后执行
4. 分发 → dispatchTool 根据工具名调用 RAG / AI 文本处理 / 文件服务 / 回收站服务 / 版本服务 / SkillExecutorService

**技能注册与执行框架（SkillRegistry / SkillExecutorService / Skill）**
- 当前注册：文件上传、文件下载、HTML PPT 生成
- Agent 工具 html-ppt-generate → SkillExecutorService.executeSkill("HTML PPT Skill", ...)
- 设计分离：Agent 只关心工具能力，技能框架负责业务封装，新增技能先注册 Skill 再暴露为 Agent Tool

### 为什么不采用别的方式

| 方案                    | 不采用的原因                                                                  |
| --------------------- | ----------------------------------------------------------------------- |
| **模型直接调用任意后端接口**      | 无法控制权限、参数格式和危险操作，文件删除/版本切换等高风险能力必须服务端审批                                 |
| **所有工具逻辑硬编码在 prompt** | 工具列表不断扩展，AgentToolRegistry 可统一维护描述/参数 schema/风险等级                       |
| **完全依赖 MCP**          | 项目已有成熟的 Spring Service/Redis/MinIO/RAG 和技能框架，直接引入 MCP 增加复杂度；先内部落地再适配更平滑 |
| **忽略参数校验和错误处理**       | [[模型生成的 JSON 计划可能缺参/类型错误/工具名错误]]，服务端兜底避免错误扩散到业务层                        |

## 3.3 会话管理设计

### 解决了什么问题

会话管理设计解决的是“Agent 如何支持多轮对话、上下文延续和任务状态追踪”的问题。

**核心矛盾：** 每次用户输入被当作孤立请求时，Agent 无法理解“继续刚才那个文件”“按照上一步结果生成 PPT”等多轮任务。

**解决方案：** 三层分离管理
- ConversationManager → 保存对话消息历史（聊天历史）
- ContextManager → 保存结构化上下文（可复用状态）
- AgentTaskRegistry → 保存 Agent 执行快照（任务执行过程）

**效果：** 覆盖多轮对话、跨步骤引用、执行过程追踪三类需求

### 为什么这么设计

| 设计选择 | 为什么这么定 |
|---------|------------|
| **消息历史/上下文字段/任务快照拆开管理** | 目标不同：消息历史用于用户查看（User/Assistant 自然语言），结构化上下文用于 Agent 执行引用（userId/role/lastTool/stepResults），任务快照用于前端工作台展示计划/时间线/反思/审批 |
| **对话绑定 userId，权限隔离** | 访问历史/删除/继续对话时 assertOwner 校验归属；Agent 执行时从 RequestUserContext 获取用户写入上下文；任务查询/取消/重试判断所有者，管理员才能查看全部 |
| **不将所有信息塞进聊天记录** | 避免把任务过程和长期上下文混在一起，结构更清晰 |

### 具体怎么设计的

**ConversationManager（对话消息）**
- Redis 存储，key 前缀 conv:
- 创建会话：生成 UUID，保存 userId / createdAt / updatedAt / messages，过期 24h
- 每次 Agent 执行完：addMessageForUser 保存用户任务和最终回答
- 消息上限 100 条，超限保留最近，防止无限增长

**ContextManager（结构化上下文）**
- Redis 存储，[[key 前缀 ctx]]（Redis 的 key 前缀（`ctx:{conversationId}`））:
- 执行开始：读取当前会话上下文，与请求中传入的 context 合并
- 执行过程：mergeToolResultIntoContext 写入 lastTool / lastObservation / answer / source / fileUrl / stepResults
- 后续步骤可通过 ${steps.stepId.xxx} 引用前序结果，支持多步链式执行
- 执行结束：更新后的 context 写回 Redis，供下一轮对话使用

**长期记忆与任务快照**
- 并非永久记忆库，采用 [[Redis TTL 短期持久化]]：对话 24h，[[Agent 任务快照]] 7 天，知识索引任务 7 天
- AgentTaskRegistry：key 前缀 agent:task:，zset 支持按时间倒序查询

**WebSocket 实时进度**
- [[AgentProgressBroadcaster 广播事件]]：任务开始/步骤完成/反思完成/等待用户确认/任务结束
- 通道：/topic/agent/progress、/topic/agent/users/{userId}、/topic/agent/tasks/{traceId}
- 前端可实时展示 Agent 工作流

### 为什么不采用别的方式

| 方案                  | 不采用的原因                                                     |
| ------------------- | ---------------------------------------------------------- |
| **只用自然语言聊天历史作为上下文** | Agent 需要结构化状态（工具结果/文件名/知识库 ID/审批 token/步骤结果），自然语言难以稳定解析和复用 |
| **所有状态永久保存到数据库**    | 会话和任务更多服务于近期交互，Redis TTL 降低存储成本，避免历史无限累积影响隐私和性能            |
| **上下文完全交给大模型记忆**    | 模型没有可靠的跨请求状态，必须由服务端保存和注入                                   |
| **任务执行状态只放内存**      | 服务重启或前端刷新丢失快照，Redis 做持久化兜底                                 |
| **引入复杂工作流引擎**       | 当前 Agent 任务规模较轻，AgentTaskRegistry 已满足查询/取消/重试/展示需求         |