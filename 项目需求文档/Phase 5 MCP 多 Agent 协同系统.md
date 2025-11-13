# Phase 5: MCP 多 Agent 协同系统

## 系统架构

```
前端 ←→ 后端
         ↓
    MCP 协调器 (多 Agent 调度中心)
         ↓
    ┌────┴────┬────────┬────────┐
意图识别   简历修改   评分Agent  优化Agent
    ↓        ↓          ↓          ↓
Context Manager (统一上下文管理)
    ↓
    ├─ 对话历史
    ├─ 简历版本
    ├─ 用户偏好
    └─ Agent 状态
```

## 完整目录结构 (基于 Phase 4)

```
intelligent-resume-optimizer/
├── frontend/                          # [Phase 5: 不变]
│   └── ...
│
└── backend/
    ├── app/
    │   ├── main.py                   # [修改] 初始化 MCP
    │   ├── config.py                 # [修改] MCP 配置
    │   │
    │   ├── api/
    │   │   ├── websocket.py          # [修改] 使用 MCP 协调器
    │   │   ├── resume.py             # [修改] 版本管理集成
    │   │   └── health.py             # [保持不变]
    │   │
    │   ├── agents/
    │   │   ├── __init__.py
    │   │   ├── base_agent.py         # [修改] 集成 MCP 协议
    │   │   ├── coordinator.py        # [废弃] 迁移到 MCP
    │   │   ├── intent_agent.py       # [修改] MCP 适配
    │   │   ├── modifier_agent.py     # [修改] MCP 适配
    │   │   ├── evaluator_agent.py    # [修改] MCP 适配
    │   │   └── optimizer_agent.py    # [修改] MCP 适配
    │   │
    │   ├── rag/                      # [保持不变]
    │   │   └── ...
    │   │
    │   ├── mcp/                      # MCP 系统 (核心实现)
    │   │   ├── __init__.py
    │   │   ├── protocol.py           # [完善] MCP 协议定义
    │   │   ├── context_manager.py    # [新增] 上下文管理器
    │   │   ├── coordinator.py        # [新增] MCP 协调器
    │   │   ├── message_bus.py        # [新增] Agent 间消息总线
    │   │   ├── task_scheduler.py     # [新增] 任务调度器
    │   │   └── conflict_resolver.py  # [新增] 冲突解决器
    │   │
    │   ├── models/
    │   │   ├── __init__.py
    │   │   ├── resume.py             # [保持不变]
    │   │   ├── message.py            # [保持不变]
    │   │   ├── intent.py             # [保持不变]
    │   │   ├── knowledge.py          # [保持不变]
    │   │   ├── evaluation.py         # [保持不变]
    │   │   ├── context.py            # [新增] 上下文模型
    │   │   └── agent_task.py         # [新增] Agent 任务模型
    │   │
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # [保持不变]
    │   │   ├── resume_generator.py   # [保持不变]
    │   │   ├── llm_service.py        # [保持不变]
    │   │   ├── knowledge_service.py  # [保持不变]
    │   │   ├── scoring_service.py    # [保持不变]
    │   │   ├── version_service.py    # [新增] 版本管理服务
    │   │   └── search_service.py     # [Phase 5: 保持空实现]
    │   │
    │   └── utils/
    │       ├── __init__.py
    │       ├── markdown_utils.py     # [保持不变]
    │       ├── text_utils.py         # [保持不变]
    │       ├── logger.py             # [修改] MCP 日志增强
    │       └── prompts.py            # [保持不变]
    │
    ├── data/
    │   ├── knowledge_base/           # [保持不变]
    │   ├── vector_db/                # [保持不变]
    │   ├── scoring_rules/            # [保持不变]
    │   ├── sessions/                 # [新增] 会话数据
    │   │   └── {session_id}/
    │   │       ├── context.json      # 上下文快照
    │   │       ├── history.json      # 对话历史
    │   │       └── versions/         # 简历版本
    │   └── preferences/              # [新增] 用户偏好
    │       └── default.json
    │
    ├── scripts/                      # [保持不变]
    │   └── ...
    │
    ├── tests/
    │   ├── __init__.py
    │   ├── test_api.py              # [保持不变]
    │   ├── test_agents.py           # [保持不变]
    │   ├── test_intent.py           # [保持不变]
    │   ├── test_rag.py              # [保持不变]
    │   ├── test_retriever.py        # [保持不变]
    │   ├── test_scoring.py          # [保持不变]
    │   ├── test_mcp.py              # [新增] MCP 系统测试
    │   ├── test_context.py          # [新增] 上下文管理测试
    │   └── test_coordinator.py      # [新增] 协调器测试
    │
    ├── requirements.txt              # [修改] 添加 redis (会话缓存)
    ├── .env.example                  # [修改] Redis 配置
    ├── Dockerfile                    # [Phase 5: 保持空]
    └── README.md                     # [修改] MCP 架构说明
```

## 核心实现内容

### 1. MCP 协议定义 (app/mcp/protocol.py)

```python
class MCPMessage:
    - message_id: str
    - from_agent: str
    - to_agent: str (可选, 广播时为 None)
    - message_type: MessageType (REQUEST/RESPONSE/EVENT)
    - payload: Dict
    - timestamp: datetime
    - context_id: str

class MessageType(Enum):
    - REQUEST: Agent 请求
    - RESPONSE: Agent 响应
    - EVENT: 事件通知
    - BROADCAST: 广播消息

class AgentCapability:
    - agent_name: str
    - supported_intents: List[IntentType]
    - priority: int
    - dependencies: List[str]
```

### 2. 上下文管理器 (app/mcp/context_manager.py)

**功能**:

- **对话历史管理**:
  - 存储用户和 AI 的完整对话
  - 支持分页和搜索
  - 自动摘要长对话

- **简历版本管理**:
  - 每次修改生成新版本
  - 版本对比 (diff)
  - 版本回退
  - 版本分支 (实验性修改)

- **用户偏好记录**:
  - 语言风格偏好
  - 行业偏好
  - 常用模板
  - 拒绝的建议

- **会话状态维护**:
  - 当前活跃的 Agent
  - Agent 执行状态
  - 待处理任务队列
  - 错误和重试记录

**接口**:

```python
class ContextManager:
    def get_conversation_history(session_id, limit)
    def add_message(session_id, message)
    def get_current_resume(session_id)
    def save_resume_version(session_id, resume, metadata)
    def rollback_version(session_id, version_id)
    def compare_versions(version_1, version_2)
    def get_preferences(user_id)
    def update_preferences(user_id, preferences)
    def get_agent_state(session_id, agent_name)
    def update_agent_state(session_id, agent_name, state)
```

### 3. MCP 协调器 (app/mcp/coordinator.py)

**功能**:

- **Agent 注册和发现**:
  - Agent 启动时注册能力
  - 维护 Agent 能力表

- **任务分发**:
  - 根据意图选择合适的 Agent
  - 支持 Agent 链 (Pipeline)
  - 支持并行执行

- **结果聚合**:
  - 收集多个 Agent 的结果
  - 合并和去重
  - 优先级排序

- **优先级管理**:
  - 紧急任务优先
  - Agent 负载均衡

- **冲突解决**:
  - 检测 Agent 间的冲突操作
  - 调用冲突解决器
  - 事务性保证

**核心流程**:

```python
1. 接收用户消息
2. 调用意图识别 Agent
3. 根据意图查找合适的 Agent
4. 检查 Agent 依赖和顺序
5. 创建执行计划
6. 依次或并行执行 Agent
7. 聚合结果
8. 检测冲突并解决
9. 保存上下文
10. 返回结果
```

### 4. 消息总线 (app/mcp/message_bus.py)

- 发布/订阅模式
- Agent 间异步通信
- 消息路由
- 消息持久化 (可选)
- 消息重放

### 5. 任务调度器 (app/mcp/task_scheduler.py)

- 任务队列管理
- 优先级队列
- 定时任务
- 重试机制
- 超时控制

### 6. 冲突解决器 (app/mcp/conflict_resolver.py)

**冲突类型**:

- **并发修改冲突**:
  - 两个 Agent 同时修改同一字段
  - 解决策略: 后执行的 Agent 优先 / 用户确认

- **逻辑冲突**:
  - 优化建议相互矛盾
  - 解决策略: LLM 判断 / 优先级规则

- **依赖冲突**:
  - Agent A 依赖 Agent B 的结果
  - 解决策略: 调整执行顺序

**解决策略**:

```python
class ConflictResolver:
    def detect_conflict(agent_results)
    def resolve_by_priority(conflicted_results)
    def resolve_by_llm(conflicted_results)
    def ask_user_confirmation(conflicts)
```

### 7. 上下文模型 (app/models/context.py)

```python
class SessionContext:
    - session_id: str
    - user_id: str
    - resume_id: str
    - current_version: int
    - conversation_history: List[Message]
    - preferences: UserPreferences
    - agent_states: Dict[str, AgentState]
    - created_at: datetime
    - updated_at: datetime

class UserPreferences:
    - language_style: str (formal/casual)
    - target_industry: str
    - preferred_templates: List[str]
    - rejected_suggestions: List[str]
    - auto_save: bool

class AgentState:
    - agent_name: str
    - status: str (idle/running/completed/failed)
    - last_execution: datetime
    - error_count: int
    - context_data: Dict
```

### 8. Agent 任务模型 (app/models/agent_task.py)

```python
class AgentTask:
    - task_id: str
    - agent_name: str
    - intent_type: IntentType
    - priority: int
    - dependencies: List[str]
    - input_data: Dict
    - output_data: Dict
    - status: TaskStatus
    - created_at: datetime
    - started_at: datetime
    - completed_at: datetime
    - error_message: str

class TaskStatus(Enum):
    - PENDING: 待执行
    - RUNNING: 执行中
    - COMPLETED: 已完成
    - FAILED: 失败
    - CANCELLED: 已取消
```

### 9. 版本管理服务 (app/services/version_service.py)

- 版本创建和存储
- 版本对比 (diff 算法)
- 版本回退
- 版本合并
- 版本元数据管理

### 10. Agent 基类适配 (app/agents/base_agent.py)

```python
class BaseAgent:
    def __init__(self, mcp_coordinator):
        self.coordinator = mcp_coordinator
        self.context_manager = coordinator.context_manager
        
    def register_capability(self):
        """向 MCP 注册 Agent 能力"""
        
    def send_message(self, to_agent, message_type, payload):
        """通过 MCP 发送消息"""
        
    def receive_message(self, message: MCPMessage):
        """接收 MCP 消息"""
        
    def execute(self, context: SessionContext, input_data):
        """执行 Agent 任务 (子类实现)"""
        
    def get_state(self, context_id):
        """获取 Agent 状态"""
        
    def update_state(self, context_id, state_data):
        """更新 Agent 状态"""
```

### 11. WebSocket 集成 (app/api/websocket.py)

```python
async def handle_user_message(websocket, message):
    # 1. 创建或获取会话上下文
    context = context_manager.get_or_create_context(session_id)
    
    # 2. 保存用户消息到历史
    context_manager.add_message(session_id, message)
    
    # 3. 调用 MCP 协调器
    result = await mcp_coordinator.process_message(context, message)
    
    # 4. 流式返回结果
    for chunk in result.stream():
        await websocket.send(chunk)
    
    # 5. 保存 AI 响应到历史
    context_manager.add_message(session_id, result.response)
    
    # 6. 如果简历被修改,保存新版本
    if result.resume_modified:
        context_manager.save_resume_version(
            session_id, 
            result.updated_resume, 
            metadata=result.change_summary
        )
```

### 12. 日志增强 (app/utils/logger.py)

- 结构化日志
- 链路追踪 (Trace ID)
- Agent 执行日志
- 性能监控日志
- 错误堆栈记录

## Phase 5 技术栈 (新增)

```
后端:
- redis==5.0.1  (会话缓存和消息队列)
- celery==5.3.4  (可选, 异步任务)
```

## 验收标准

✅ MCP 协调器正确调度多个 Agent
✅ 上下文管理功能完整:

  - 对话历史正确保存和检索
  - 简历版本管理正常
  - 版本回退功能正常
  - 用户偏好生效
    ✅ Agent 间通信正常:
  - 消息总线稳定
  - 消息不丢失
    ✅ 冲突检测和解决:
  - 能检测并发修改冲突
  - 冲突解决策略有效
    ✅ 任务调度正确:
  - 优先级队列工作正常
  - 依赖关系正确处理
    ✅ 性能优化:
  - 并行执行 Agent 提速 > 30%
  - 会话恢复时间 < 1 秒
    ✅ 日志完整且可追踪

## 测试用例

1. **上下文管理测试**:
   - 创建会话 → 发送多条消息 → 获取历史
   - 修改简历 → 创建新版本 → 回退版本
   - 设置偏好 → 验证生效

2. **多 Agent 协同测试**:
   - "优化并评分我的简历" → 同时调用修改和评分 Agent
   - 验证结果聚合正确

3. **冲突解决测试**:
   - 同时发起两个修改请求
   - 验证冲突检测和解决

4. **版本管理测试**:
   - 连续修改 5 次 → 查看版本历史
   - 对比任意两个版本
   - 回退到指定版本

5. **性能测试**:
   - 1000 条对话历史检索
   - 100 个并发会话