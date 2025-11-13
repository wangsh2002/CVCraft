## Phase 3: RAG 知识库系统

### 系统架构

```
前端 ←→ 后端
         ↓
    Agent 协调器
         ↓
    ┌────┴────┐
意图识别    简历修改 ← RAG 检索器
              ↓           ↓
        LLM API    ChromaDB 向量库
                          ↓
                    知识库 (JSON/Markdown)
```

### 完整目录结构 (基于 Phase 2)

```
intelligent-resume-optimizer/
├── frontend/                          # [Phase 3: 不变]
│   └── ...
│
└── backend/
    ├── app/
    │   ├── main.py                   # [修改] 启动时初始化向量库
    │   ├── config.py                 # [修改] 添加 ChromaDB 配置
    │   │
    │   ├── api/
    │   │   ├── websocket.py          # [保持不变]
    │   │   ├── resume.py             # [保持不变]
    │   │   └── health.py             # [保持不变]
    │   │
    │   ├── agents/
    │   │   ├── __init__.py
    │   │   ├── base_agent.py         # [保持不变]
    │   │   ├── coordinator.py        # [保持不变]
    │   │   ├── intent_agent.py       # [保持不变]
    │   │   ├── modifier_agent.py     # [修改] 集成 RAG 检索
    │   │   └── evaluator_agent.py    # [Phase 3: 空实现]
    │   │
    │   ├── rag/                      # RAG 系统
    │   │   ├── __init__.py
    │   │   ├── base_retriever.py     # [保持不变]
    │   │   ├── vector_store.py       # [新增] 向量数据库管理
    │   │   ├── embeddings.py         # [新增] 文本嵌入
    │   │   ├── retriever.py          # [新增] 检索器实现
    │   │   └── knowledge_base/       # 知识库加载
    │   │       ├── __init__.py
    │   │       ├── industry_keywords.py    # 行业关键词加载
    │   │       ├── optimization_cases.py   # 优化案例加载
    │   │       └── excellent_resumes.py    # 优秀简历加载
    │   │
    │   ├── mcp/                      # [Phase 3: 保持空框架]
    │   │   ├── __init__.py
    │   │   ├── protocol.py
    │   │   └── context_manager.py
    │   │
    │   ├── models/
    │   │   ├── __init__.py
    │   │   ├── resume.py             # [保持不变]
    │   │   ├── message.py            # [保持不变]
    │   │   ├── intent.py             # [保持不变]
    │   │   ├── knowledge.py          # [新增] 知识库数据模型
    │   │   └── evaluation.py         # [Phase 3: 保持空定义]
    │   │
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # [保持不变]
    │   │   ├── resume_generator.py   # [保持不变]
    │   │   ├── llm_service.py        # [保持不变]
    │   │   ├── knowledge_service.py  # [新增] 知识库服务
    │   │   ├── scoring_service.py    # [Phase 3: 保持空实现]
    │   │   └── search_service.py     # [Phase 3: 保持空实现]
    │   │
    │   └── utils/
    │       ├── __init__.py
    │       ├── markdown_utils.py     # [保持不变]
    │       ├── text_utils.py         # [修改] 添加向量相似度计算
    │       ├── logger.py             # [保持不变]
    │       └── prompts.py            # [修改] 添加 RAG 增强 Prompt
    │
    ├── data/                         # 数据目录
    │   ├── knowledge_base/           # 知识库源数据
    │   │   ├── industry_keywords/
    │   │   │   ├── tech.json         # [新增] 技术行业关键词
    │   │   │   ├── finance.json      # [新增] 金融行业关键词
    │   │   │   ├── marketing.json    # [新增] 市场行业关键词
    │   │   │   └── general.json      # [新增] 通用关键词
    │   │   ├── optimization_cases/   # 优化案例
    │   │   │   ├── before_after_1.json  # [新增] 案例1
    │   │   │   ├── before_after_2.json  # [新增] 案例2
    │   │   │   ├── before_after_3.json  # [新增] 案例3
    │   │   │   └── README.md         # [新增] 案例说明
    │   │   └── excellent_resumes/    # 优秀简历
    │   │       ├── senior_engineer.md    # [新增] 高级工程师
    │   │       ├── product_manager.md    # [新增] 产品经理
    │   │       ├── data_analyst.md       # [新增] 数据分析师
    │   │       └── README.md         # [新增] 简历说明
    │   └── vector_db/                # 向量数据库持久化
    │       └── chroma/               # ChromaDB 存储
    │
    ├── scripts/                      # 数据处理脚本
    │   ├── __init__.py
    │   ├── build_knowledge_base.py   # [新增] 构建知识库
    │   ├── update_embeddings.py      # [新增] 更新向量
    │   └── validate_data.py          # [新增] 验证数据完整性
    │
    ├── tests/
    │   ├── __init__.py
    │   ├── test_api.py              # [保持不变]
    │   ├── test_agents.py           # [保持不变]
    │   ├── test_intent.py           # [保持不变]
    │   ├── test_rag.py              # [新增] RAG 测试
    │   ├── test_retriever.py        # [新增] 检索器测试
    │   └── test_mcp.py              # [Phase 3: 保持空]
    │
    ├── requirements.txt              # [修改] 添加 chromadb, sentence-transformers
    ├── .env.example                  # [修改] 添加向量库配置
    ├── Dockerfile                    # [Phase 3: 保持空]
    └── README.md                     # [修改] 更新 RAG 使用说明
```

### 核心实现内容

#### 1. 知识库数据准备 (data/knowledge_base/)

**行业关键词 (industry_keywords/tech.json)**:

```json
{
  "category": "技术",
  "skills": {
    "编程语言": ["Python", "Java", "JavaScript", "Go", "C++"],
    "框架": ["React", "Vue", "Django", "Spring Boot", "FastAPI"],
    "数据库": ["MySQL", "Redis", "MongoDB", "PostgreSQL"],
    "DevOps": ["Docker", "Kubernetes", "Jenkins", "Git"]
  },
  "positions": ["软件工程师", "全栈工程师", "后端工程师", "架构师"],
  "hot_keywords": ["微服务", "云原生", "AI", "大数据"]
}
```

**优化案例 (optimization_cases/before_after_1.json)**:

```json
{
  "id": "case_001",
  "type": "工作描述优化",
  "industry": "技术",
  "before": "负责公司产品开发",
  "after": "负责电商平台核心交易系统开发，使用 Spring Boot + MySQL 架构，日均处理订单 10 万+，系统可用性达 99.9%",
  "improvements": ["量化成果", "技术栈明确", "突出规模"],
  "tags": ["量化", "技术栈", "成果导向"]
}
```

**优秀简历 (excellent_resumes/senior_engineer.md)**:
完整的优秀简历示例，包含最佳实践。

#### 2. 向量数据库管理 (app/rag/vector_store.py)

- ChromaDB 初始化和配置
- Collection 管理（关键词、案例、简历）
- 文档添加/更新/删除
- 批量导入
- 持久化配置

#### 3. 文本嵌入 (app/rag/embeddings.py)

- 通义千问 Embedding API 调用
- 批量嵌入优化
- 缓存机制
- 降维处理

#### 4. 检索器实现 (app/rag/retriever.py)

- 相似度检索:
  - 余弦相似度
  - Top-K 结果
  - 阈值过滤
- 上下文感知检索:
  - 根据简历当前状态
  - 根据用户意图
  - 根据行业类别
- 混合检索策略:
  - 关键词检索
  - 案例检索
  - 简历模板检索
- 结果排序和去重

#### 5. 知识库加载器 (app/rag/knowledge_base/)

- industry_keywords.py: 加载行业关键词
- optimization_cases.py: 加载优化案例
- excellent_resumes.py: 加载优秀简历
- 数据验证
- 增量更新

#### 6. 知识库服务 (app/services/knowledge_service.py)

- 统一知识库访问接口
- 缓存热门查询
- 统计分析

#### 7. 简历修改 Agent 集成 RAG (app/agents/modifier_agent.py)

- 修改前检索相关案例
- 参考优秀实践
- 补充行业关键词
- 生成改进建议时引用案例

#### 8. 数据构建脚本 (scripts/build_knowledge_base.py)

- 读取 JSON/Markdown 数据
- 生成向量嵌入
- 导入 ChromaDB
- 验证数据完整性

### Phase 3 技术栈 (新增)

```
后端:
- chromadb==0.4.18
- sentence-transformers==2.2.2  (备选本地嵌入)
- pandas==2.0.3  (数据处理)
```

### 验收标准

✅ 知识库数据完整:

  - 3 个行业关键词库
  - 至少 10 个优化案例
  - 至少 3 个优秀简历模板
    ✅ ChromaDB 成功构建并持久化
    ✅ 向量嵌入生成正确
    ✅ 检索功能正常:
  - 相似度检索准确率 > 85%
  - Top-3 结果相关性高
    ✅ RAG 增强修改建议质量明显提升
    ✅ 响应时间 < 3 秒（含检索）
    ✅ 能引用具体案例说明优化理由

### 测试用例

1. **关键词检索**:
   - 简历提到 "Python"，检索到相关技术栈
   - 简历缺少量化成果，检索到量化案例

2. **案例检索**:
   - 工作描述模糊 → 检索到"工作描述优化"案例
   - 项目经验缺技术栈 → 检索到技术栈补充案例

3. **RAG 增强修改**:
   - "优化我的工作经历" → 参考优秀案例进行优化
   - "添加项目经验" → 参考项目模板补充

4. **性能测试**:
   - 单次检索时间 < 500ms
   - 批量嵌入 100 条文本 < 5s