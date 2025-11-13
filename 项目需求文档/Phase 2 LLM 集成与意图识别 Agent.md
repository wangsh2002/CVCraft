## Phase 2: LLM 集成与意图识别 Agent

### 系统架构

```
前端 (React + WebSocket) ←→ 后端 (FastAPI)
                               ↓
                         Agent 协调器 (空实现)
                               ↓
                    ┌──────────┴──────────┐
              意图识别 Agent        简历修改 Agent
                    ↓                     ↓
              通义千问 API          简历解析/生成服务
```

### 完整目录结构 (基于 Phase 1)

```
intelligent-resume-optimizer/
├── frontend/                          # [Phase 1 不变]
│   └── ...
│
└── backend/
    ├── app/
    │   ├── main.py                   # [保持不变]
    │   ├── config.py                 # [修改] 添加 LLM API 配置
    │   │
    │   ├── api/
    │   │   ├── websocket.py          # [修改] 集成意图识别
    │   │   ├── resume.py             # [保持不变]
    │   │   └── health.py             # [保持不变]
    │   │
    │   ├── agents/                   # Agent 系统
    │   │   ├── __init__.py
    │   │   ├── base_agent.py         # [修改] 完善 Agent 基类
    │   │   ├── coordinator.py        # [新增] Agent 协调器 (简单实现)
    │   │   ├── intent_agent.py       # [新增] 意图识别 Agent
    │   │   └── modifier_agent.py     # [新增] 简历修改 Agent
    │   │
    │   ├── rag/                      # [Phase 2: 保持空框架]
    │   │   ├── __init__.py
    │   │   └── base_retriever.py
    │   │
    │   ├── mcp/                      # [Phase 2: 保持空框架]
    │   │   ├── __init__.py
    │   │   ├── protocol.py
    │   │   └── context_manager.py    # [新增] 空实现
    │   │
    │   ├── models/
    │   │   ├── __init__.py
    │   │   ├── resume.py             # [保持不变]
    │   │   ├── message.py            # [保持不变]
    │   │   ├── intent.py             # [新增] 意图模型
    │   │   └── evaluation.py         # [Phase 2: 保持空定义]
    │   │
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # [修改] 增强结构化解析
    │   │   ├── resume_generator.py   # [保持不变]
    │   │   ├── llm_service.py        # [新增] LLM API 调用封装
    │   │   ├── scoring_service.py    # [Phase 2: 保持空实现]
    │   │   └── search_service.py     # [Phase 2: 保持空实现]
    │   │
    │   └── utils/
    │       ├── __init__.py
    │       ├── markdown_utils.py     # [保持不变]
    │       ├── text_utils.py         # [修改] 添加文本相似度计算
    │       ├── logger.py             # [保持不变]
    │       └── prompts.py            # [新增] Prompt 模板管理
    │
    ├── data/                         # [Phase 2: 保持空目录]
    │   ├── knowledge_base/
    │   └── vector_db/
    │
    ├── tests/
    │   ├── __init__.py
    │   ├── test_api.py              # [保持不变]
    │   ├── test_agents.py           # [新增] Agent 单元测试
    │   ├── test_intent.py           # [新增] 意图识别测试
    │   ├── test_rag.py              # [Phase 2: 保持空]
    │   └── test_mcp.py              # [Phase 2: 保持空]
    │
    ├── requirements.txt              # [修改] 添加 langchain, openai
    ├── .env.example                  # [修改] 添加 API Key 配置
    ├── Dockerfile                    # [Phase 2: 保持空]
    └── README.md                     # [修改] 更新使用说明
```

### 核心实现内容

#### 1. LLM 服务封装 (app/services/llm_service.py)

- 通义千问 API 调用
- 重试机制
- 速率限制
- 错误处理
- 流式输出支持

#### 2. 意图识别 Agent (app/agents/intent_agent.py)

- 继承 base_agent.py
- 5 种意图识别:
  - MODIFY: 修改（增删改）
  - ENHANCE: 增强（丰富内容）
  - POLISH: 润色（语言优化）
  - EVALUATE: 评估（打分分析）
  - SUGGEST: 建议（优化建议）
- Few-shot 学习示例
- 意图置信度输出
- 提取目标字段和操作类型

#### 3. 简历修改 Agent (app/agents/modifier_agent.py)

- 继承 base_agent.py
- 定位修改目标:
  - 基本信息 (姓名、联系方式、求职意向)
  - 工作经历 (公司、职位、时间、描述)
  - 项目经验 (项目名、角色、技术栈、成果)
  - 教育背景
  - 技能清单
- 执行修改操作:
  - 添加、删除、更新
  - 重排序
  - 合并
- 生成修改说明
- 生成差异对比

#### 4. Agent 协调器 (app/agents/coordinator.py)

- 简单的顺序调度:
  1. 意图识别 Agent
  2. 根据意图调用对应 Agent
- Agent 间数据传递
- 统一错误处理
- 日志记录

#### 5. 意图模型 (app/models/intent.py)

- IntentType 枚举
- IntentResult 结构:
  - intent_type
  - confidence
  - target_section
  - target_field
  - operation
  - new_value

#### 6. Prompt 模板管理 (app/utils/prompts.py)

- 意图识别 Prompt
- 简历修改 Prompt
- Few-shot 示例库
- Prompt 版本管理

#### 7. WebSocket 集成 (app/api/websocket.py)

- 调用 Agent 协调器
- 流式返回 LLM 输出
- 实时更新简历

#### 8. 简历解析增强 (app/services/resume_parser.py)

- 细粒度字段提取
- 支持多种 Markdown 格式
- 字段定位索引

### Phase 2 技术栈 (新增)

```
后端:
- langchain==0.1.0
- openai==1.3.0  (通义千问兼容接口)
- tiktoken==0.5.1
- numpy==1.24.0
```

### 验收标准

✅ 通义千问 API 调用成功
✅ 意图识别准确率 > 90% (测试集 100 条)
✅ 5 种意图都能正确识别
✅ 简历修改操作准确执行:

  - "把我的名字改成张三" ✓
  - "在工作经历中添加字节跳动的实习经验" ✓
  - "删除第二段项目经验" ✓
    ✅ 返回清晰的修改说明和差异对比
    ✅ 对话流畅自然
    ✅ Agent 调用日志完整

### 测试用例

1. **意图识别测试**:
   - "帮我润色一下自我评价" → POLISH
   - "给我的简历打个分" → EVALUATE
   - "添加一段实习经历" → MODIFY

2. **简历修改测试**:
   - 修改基本信息
   - 添加工作经历
   - 删除项目经验
   - 更新技能列表

3. **多轮对话测试**:
   - "把公司名改成阿里巴巴" → "再把职位改成高级工程师"