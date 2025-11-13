## Phase 6: 搜索 Agent 与系统优化

### 系统架构

```
前端 ←→ 后端
         ↓
    MCP 协调器
         ↓
    ┌────┴────┬────────┬────────┬────────┐
意图识别   简历修改   评分      优化      搜索Agent
    ↓        ↓          ↓          ↓          ↓
         LLM API    RAG检索              联网搜索API
```

### 完整目录结构 (基于 Phase 5)

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
│   │   │       ├── ScorePanel.js     # [保持不变]
│   │   │       ├── RadarChart.js     # [保持不变]
│   │   │       ├── SuggestionList.js # [保持不变]
│   │   │       └── SearchResult.js   # [新增] 搜索结果展示
│   │   ├── layout/
│   │   │   └── ChatLayout.js         # [保持不变]
│   │   ├── store/
│   │   │   ├── chat.js               # [保持不变]
│   │   │   └── evaluation.js         # [保持不变]
│   │   └── services/
│   │       ├── websocket.js          # [保持不变]
│   │       └── api.js                # [保持不变]
│   └── package.json                  # [保持不变]
│
└── backend/
    ├── app/
    │   ├── main.py                   # [保持不变]
    │   ├── config.py                 # [修改] 添加搜索 API 配置
    │   │
    │   ├── api/
    │   │   ├── websocket.py          # [保持不变]
    │   │   ├── resume.py             # [保持不变]
    │   │   └── health.py             # [保持不变]
    │   │
    │   ├── agents/
    │   │   ├── __init__.py
    │   │   ├── base_agent.py         # [保持不变]
    │   │   ├── coordinator.py        # [废弃]
    │   │   ├── intent_agent.py       # [保持不变]
    │   │   ├── modifier_agent.py     # [保持不变]
    │   │   ├── evaluator_agent.py    # [保持不变]
    │   │   ├── optimizer_agent.py    # [保持不变]
    │   │   └── search_agent.py       # [新增] 联网搜索 Agent
    │   │
    │   ├── rag/                      # [保持不变]
    │   │   └── ...
    │   │
    │   ├── mcp/                      # [保持不变]
    │   │   └── ...
    │   │
    │   ├── models/
    │   │   ├── __init__.py
    │   │   ├── resume.py             # [保持不变]
    │   │   ├── message.py            # [保持不变]
    │   │   ├── intent.py             # [修改] 添加 SEARCH 意图
    │   │   ├── knowledge.py          # [保持不变]
    │   │   ├── evaluation.py         # [保持不变]
    │   │   ├── context.py            # [保持不变]
    │   │   ├── agent_task.py         # [保持不变]
    │   │   └── search_result.py      # [新增] 搜索结果模型
    │   │
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # [保持不变]
    │   │   ├── resume_generator.py   # [保持不变]
    │   │   ├── llm_service.py        # [保持不变]
    │   │   ├── knowledge_service.py  # [保持不变]
    │   │   ├── scoring_service.py    # [保持不变]
    │   │   ├── version_service.py    # [保持不变]
    │   │   └── search_service.py     # [新增] 搜索服务实现
    │   │
    │   └── utils/
    │       ├── __init__.py
    │       ├── markdown_utils.py     # [保持不变]
    │       ├── text_utils.py         # [保持不变]
    │       ├── logger.py             # [保持不变]
    │       ├── prompts.py            # [修改] 添加搜索 Prompt
    │       └── cache_utils.py        # [新增] 缓存工具
    │   │
    ├── data/
    │   ├── knowledge_base/           # [保持不变]
    │   ├── vector_db/                # [保持不变]
    │   ├── scoring_rules/            # [保持不变]
    │   ├── sessions/                 # [保持不变]
    │   ├── preferences/              # [保持不变]
    │   └── cache/                    # [新增] 搜索缓存
    │       └── search_results/
    │
    ├── scripts/
    │   ├── build_knowledge_base.py   # [保持不变]
    │   ├── update_embeddings.py      # [保持不变]
    │   ├── validate_data.py          # [保持不变]
    │   └── performance_test.py       # [新增] 性能测试脚本
    │
    ├── tests/
    │   ├── __init__.py
    │   ├── test_api.py              # [保持不变]
    │   ├── test_agents.py           # [修改] 添加搜索 Agent 测试
    │   ├── test_intent.py           # [保持不变]
    │   ├── test_rag.py              # [保持不变]
    │   ├── test_retriever.py        # [保持不变]
    │   ├── test_scoring.py          # [保持不变]
    │   ├── test_mcp.py              # [保持不变]
    │   ├── test_context.py          # [保持不变]
    │   ├── test_coordinator.py      # [保持不变]
    │   ├── test_search.py           # [新增] 搜索功能测试
    │   └── test_performance.py      # [新增] 性能测试
    │
    ├── requirements.txt              # [修改] 添加搜索相关依赖
    ├── .env.example                  # [修改] 添加搜索 API Key
    ├── Dockerfile                    # [新增] Docker 配置
    ├── docker-compose.yml            # [新增] Docker Compose
    └── README.md                     # [修改] 完整文档
```

## 核心实现内容

### 1. 搜索服务 (app/services/search_service.py)

**搜索引擎集成**:

- 必应搜索 API
- 谷歌自定义搜索 API (备选)
- DuckDuckGo API (备选)

**功能**:

- **行业最新动态搜索**:
  - 查询关键技术趋势
  - 行业薪资水平
  - 热门职位要求

- **技能热度分析**:
  - 搜索技能提及频率
  - 技能组合趋势
  - 新兴技能识别

- **竞品简历分析**:
  - 搜索同职位优秀简历
  - 提取关键词和描述模式
  - 对比分析

- **公司信息查询**:
  - 公司背景
  - 公司技术栈
  - 公司文化

**接口**:

```python
class SearchService:
    def search_industry_trends(industry: str, limit: int)
    def search_skill_popularity(skills: List[str])
    def search_job_requirements(position: str, industry: str)
    def search_company_info(company_name: str)
    def search_salary_range(position: str, city: str)
```

### 2. 搜索 Agent (app/agents/search_agent.py)

- 继承 base_agent.py
- 注册 SEARCH 意图能力
- 解析搜索查询:
  - 提取搜索关键词
  - 确定搜索类型
  - 设置搜索范围

- 调用搜索服务:
  - 多源搜索
  - 结果去重和排序
  - 可信度评估

- 结果处理:
  - 摘要生成
  - 关键信息提取
  - 格式化输出

- 缓存策略:
  - 热门查询缓存
  - 过期时间设置
  - 缓存更新

### 3. 搜索结果模型 (app/models/search_result.py)

```python
class SearchResult:
    - query: str
    - source: str (bing/google/duckduckgo)
    - results: List[SearchItem]
    - total_count: int
    - search_time: float
    - cached: bool
    - timestamp: datetime

class SearchItem:
    - title: str
    - url: str
    - snippet: str
    - relevance_score: float
    - source_credibility: float
```

### 4. 意图模型扩展 (app/models/intent.py)

```python
class IntentType(Enum):
    - MODIFY: 修改
    - ENHANCE: 增强
    - POLISH: 润色
    - EVALUATE: 评估
    - SUGGEST: 建议
    - SEARCH: 搜索  # 新增
```

### 5. 缓存工具 (app/utils/cache_utils.py)

- Redis 缓存封装
- 缓存键生成
- 过期策略
- 缓存预热
- 缓存统计

### 6. 搜索 Prompt (app/utils/prompts.py)

- 搜索查询生成 Prompt
- 搜索结果摘要 Prompt
- 可信度评估 Prompt

### 7. 前端搜索结果组件 (src/components/Chat/SearchResult.js)

- 搜索结果卡片
- 来源标识
- 可展开详情
- 引用到简历

### 8. 性能优化

**缓存策略**:

- Redis 缓存热门搜索
- 搜索结果缓存 1 小时
- RAG 检索结果缓存
- LLM 响应缓存 (相同 Prompt)

**并发优化**:

- Agent 并行执行
- 数据库连接池
- 异步 I/O

**资源优化**:

- 限流和降级
- 超时控制
- 重试机制

### 9. Dockerfile

```dockerfile
FROM python:3.9-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 10. docker-compose.yml

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    volumes:
      - ./backend:/app
      - ./data:/app/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
    volumes:
      - ./frontend:/app
      - /app/node_modules

volumes:
  redis_data:
```

### 11. 性能测试脚本 (scripts/performance_test.py)

- 并发用户测试
- 响应时间统计
- 吞吐量测试
- 资源占用监控

### 12. 完整文档 (README.md)

- 项目介绍
- 架构说明
- 安装指南
- 使用教程
- API 文档
- 部署指南
- 常见问题

## Phase 6 技术栈 (新增)

```
后端:
- httpx==0.25.2  (异步 HTTP 客户端)
- beautifulsoup4==4.12.2  (网页解析)
- aioredis==2.0.1  (异步 Redis 客户端)

Docker:
- Docker 20.10+
- Docker Compose 2.0+
```

## 验收标准

✅ 搜索功能正常:

  - 能搜索行业动态
  - 能分析技能热度
  - 搜索结果相关性高
    ✅ 搜索结果准确且有用:
  - 前 3 条结果相关性 > 80%
  - 摘要清晰准确
    ✅ 缓存策略有效:
  - 热门查询命中率 > 70%
  - 缓存提速 > 5 倍
    ✅ 性能达标:
  - API 响应时间 P95 < 2 秒
  - 支持 100 并发用户
  - CPU 占用 < 60%
  - 内存占用 < 2GB
    ✅ Docker 部署成功:
  - 一键启动所有服务
  - 服务间通信正常
    ✅ 文档完整:
  - 安装步骤清晰
  - API 文档齐全
  - 示例代码可运行

## 测试用例

1. **搜索功能测试**:
   - "Python 工程师的薪资水平如何" → 搜索并返回结果
   - "AI 行业最新动态" → 返回相关新闻
   - "React 技能热度" → 分析技能趋势

2. **缓存测试**:
   - 同一查询第二次 → 验证从缓存返回
   - 缓存过期 → 重新搜索

3. **性能测试**:
   - 100 并发用户同时对话
   - 测量响应时间和成功率
   - 监控资源占用

4. **Docker 部署测试**:
   - `docker-compose up` 启动
   - 验证所有服务正常
   - 前后端联通

5. **端到端测试**:
   - 上传简历 → 评分 → 搜索优化建议 → 修改 → 重新评分
   - 验证完整流程