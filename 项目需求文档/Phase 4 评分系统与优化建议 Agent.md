## Phase 4: 评分系统与优化建议 Agent

### 系统架构

```
前端 (新增评分面板) ←→ 后端
                        ↓
                   Agent 协调器
                        ↓
        ┌───────────────┼───────────────┐
    意图识别      简历修改 ← RAG    评分 Agent
                                       ↓
                              评分服务 + LLM
                                       ↓
                              优化建议 Agent ← RAG
```

### 完整目录结构 (基于 Phase 3)

```
intelligent-resume-optimizer/
├── frontend/
│   ├── src/
│   │   ├── App.js                    # [保持不变]
│   │   ├── components/
│   │   │   ├── Basic/                # [保持不变]
│   │   │   ├── Button/               # [保持不变]
│   │   │   ├── Dialog/               # [保持不变]
│   │   │   ├── Navbar/               # [保持不变]
│   │   │   └── Chat/
│   │   │       ├── ChatBox.js        # [保持不变]
│   │   │       ├── MessageList.js    # [保持不变]
│   │   │       ├── InputArea.js      # [保持不变]
│   │   │       ├── ScorePanel.js     # [新增] 评分面板
│   │   │       ├── RadarChart.js     # [新增] 雷达图组件
│   │   │       └── SuggestionList.js # [新增] 建议列表
│   │   ├── layout/
│   │   │   └── ChatLayout.js         # [修改] 添加评分面板
│   │   ├── store/
│   │   │   ├── chat.js               # [保持不变]
│   │   │   └── evaluation.js         # [新增] 评估状态管理
│   │   └── services/
│   │       └── api.js                # [修改] 添加评估 API
│   └── package.json                  # [修改] 添加 recharts (图表库)
│
└── backend/
    ├── app/
    │   ├── main.py                   # [保持不变]
    │   ├── config.py                 # [修改] 添加评分配置
    │   │
    │   ├── api/
    │   │   ├── websocket.py          # [修改] 集成评分和建议
    │   │   ├── resume.py             # [修改] 添加评估端点
    │   │   └── health.py             # [保持不变]
    │   │
    │   ├── agents/
    │   │   ├── __init__.py
    │   │   ├── base_agent.py         # [保持不变]
    │   │   ├── coordinator.py        # [修改] 添加评分和建议路由
    │   │   ├── intent_agent.py       # [保持不变]
    │   │   ├── modifier_agent.py     # [保持不变]
    │   │   ├── evaluator_agent.py    # [新增] 评分 Agent
    │   │   └── optimizer_agent.py    # [新增] 优化建议 Agent
    │   │
    │   ├── rag/                      # [保持不变]
    │   │   └── ...
    │   │
    │   ├── mcp/                      # [Phase 4: 保持空框架]
    │   │   └── ...
    │   │
    │   ├── models/
    │   │   ├── __init__.py
    │   │   ├── resume.py             # [保持不变]
    │   │   ├── message.py            # [保持不变]
    │   │   ├── intent.py             # [保持不变]
    │   │   ├── knowledge.py          # [保持不变]
    │   │   └── evaluation.py         # [新增] 完整评估模型
    │   │
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # [保持不变]
    │   │   ├── resume_generator.py   # [保持不变]
    │   │   ├── llm_service.py        # [保持不变]
    │   │   ├── knowledge_service.py  # [保持不变]
    │   │   ├── scoring_service.py    # [新增] 评分服务实现
    │   │   └── search_service.py     # [Phase 4: 保持空实现]
    │   │
    │   └── utils/
    │       ├── __init__.py
    │       ├── markdown_utils.py     # [保持不变]
    │       ├── text_utils.py         # [修改] 添加评分算法
    │       ├── logger.py             # [保持不变]
    │       └── prompts.py            # [修改] 添加评分和建议 Prompt
    │
    ├── data/
    │   ├── knowledge_base/           # [保持不变]
    │   │   └── ...
    │   ├── vector_db/                # [保持不变]
    │   │   └── ...
    │   └── scoring_rules/            # [新增] 评分规则
    │       ├── completeness.json     # 完整性评分规则
    │       ├── relevance.json        # 匹配度评分规则
    │       └── attractiveness.json   # 吸引力评分规则
    │
    ├── scripts/                      # [保持不变]
    │   └── ...
    │
    ├── tests/
    │   ├── __init__.py
    │   ├── test_api.py              # [保持不变]
    │   ├── test_agents.py           # [修改] 添加评分/建议测试
    │   ├── test_intent.py           # [保持不变]
    │   ├── test_rag.py              # [保持不变]
    │   ├── test_retriever.py        # [保持不变]
    │   ├── test_scoring.py          # [新增] 评分系统测试
    │   └── test_mcp.py              # [Phase 4: 保持空]
    │
    ├── requirements.txt              # [保持不变]
    ├── .env.example                  # [保持不变]
    ├── Dockerfile                    # [Phase 4: 保持空]
    └── README.md                     # [修改] 更新评分功能说明
```

### 核心实现内容

#### 1. 评估数据模型 (app/models/evaluation.py)

```python
class ScoreDimension:
    - dimension: str (完整性/匹配度/吸引力)
    - score: float (0-100)
    - details: Dict (具体评分项)
    - suggestions: List[str]

class EvaluationResult:
    - completeness: ScoreDimension
    - relevance: ScoreDimension
    - attractiveness: ScoreDimension
    - overall_score: float
    - strengths: List[str]
    - weaknesses: List[str]
    - priority_suggestions: List[Suggestion]

class Suggestion:
    - priority: int (1-5)
    - category: str
    - description: str
    - example: str
    - impact: str (高/中/低)
```

#### 2. 评分服务 (app/services/scoring_service.py)

**完整性评分 (0-100)**:

- 基本信息完整度 (30%):
  - 姓名、联系方式、求职意向
  - 评分逻辑: 必填项 * 权重
- 工作经历完整度 (35%):
  - 时间、公司、职位、职责、成果
  - 每段经历的详细度
  - 量化成果数量
- 项目经验完整度 (25%):
  - 项目名称、角色、技术栈、成果
  - 项目描述的详细度
- 教育背景完整度 (10%):
  - 学校、专业、学历、时间

**匹配度评分 (0-100)**:

- 关键词匹配度 (40%):
  - 从 RAG 检索行业关键词
  - 计算覆盖率
  - TF-IDF 权重
- 行业相关性 (30%):
  - 工作经历与目标行业的相关性
  - 项目经验的行业匹配度
- 技能匹配度 (30%):
  - 技能列表与行业要求的匹配
  - 技能熟练度表述

**吸引力评分 (0-100)**:

- 语言表达质量 (40%):
  - 动词使用（主动 vs 被动）
  - 专业术语准确性
  - 句式多样性
  - LLM 辅助评分
- 亮点突出度 (35%):
  - 量化成果数量
  - 成果的显著性
  - 关键成就的突出展示
- 格式规范性 (25%):
  - Markdown 格式正确性
  - 排版美观度
  - 长度适中

#### 3. 评分 Agent (app/agents/evaluator_agent.py)

- 继承 base_agent.py
- 调用 scoring_service 获取三维评分
- 生成详细评分说明:
  - 每个维度的得分理由
  - 具体扣分项
  - 亮点识别
- 识别优势和劣势
- 输出结构化评估结果

#### 4. 优化建议 Agent (app/agents/optimizer_agent.py)

- 继承 base_agent.py
- 基于评分结果生成建议:
  - 根据得分最低的维度优先建议
  - 从 RAG 检索相关优化案例
  - 生成具体可操作的建议
- 建议分类:
  - 关键问题（必须修复）
  - 重要优化（显著提升）
  - 细节改进（锦上添花）
- 每条建议包含:
  - 问题描述
  - 优化方法
  - 参考案例
  - 预期提升效果

#### 5. API 端点 (app/api/resume.py)

```python
POST /api/resume/evaluate
- 接收简历 ID
- 调用评分 Agent
- 返回评估结果

GET /api/suggestions/{resume_id}
- 获取优化建议
- 支持按优先级过滤
```

#### 6. 前端评分面板 (src/components/Chat/ScorePanel.js)

- 三维评分展示
- 每个维度的详细说明
- 优势和劣势列表

#### 7. 前端雷达图 (src/components/Chat/RadarChart.js)

- 使用 recharts 库
- 三个维度的可视化
- 交互式展示

#### 8. 前端建议列表 (src/components/Chat/SuggestionList.js)

- 按优先级分组
- 可展开详情
- 点击应用建议

#### 9. 评分规则配置 (data/scoring_rules/)

- completeness.json: 完整性评分权重和规则
- relevance.json: 匹配度关键词库和权重
- attractiveness.json: 吸引力评判标准

#### 10. WebSocket 集成 (app/api/websocket.py)

- 识别 EVALUATE 意图
- 调用评分 Agent
- 流式返回评分过程
- 推送评估结果
- 自动触发优化建议

### Phase 4 技术栈 (新增)

```
前端:
- recharts: ^2.x  (图表库)

后端:
- scikit-learn: ^1.3.0  (TF-IDF、余弦相似度)
```

### 验收标准

✅ 三维评分稳定且合理:

  - 同一简历多次评分误差 < 5 分
  - 高质量简历总分 > 80
  - 低质量简历能识别出问题
    ✅ 评分说明具体可操作
    ✅ 优化建议针对性强:
  - 至少 5 条建议
  - 每条建议都有具体例子
    ✅ 前端雷达图正确展示
    ✅ 评分时间 < 5 秒
    ✅ 建议能引用 RAG 检索的案例

### 测试用例

1. **完整性评分**:
   - 缺少联系方式 → 完整性扣分，建议补充
   - 工作经历无量化成果 → 完整性扣分

2. **匹配度评分**:
   - 技术简历缺少关键技术栈 → 匹配度低
   - 补充热门技术 → 匹配度提升

3. **吸引力评分**:
   - 被动语态 → 吸引力扣分，建议改主动
   - 缺少量化 → 建议添加数据

4. **综合测试**:
   - 评估一份完整简历
   - 应用建议后重新评分
   - 验证分数提升

