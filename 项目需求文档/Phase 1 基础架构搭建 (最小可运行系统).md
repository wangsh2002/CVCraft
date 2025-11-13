## Phase 1: 基础架构搭建 (最小可运行系统)

### 系统架构

```
前端 (React + WebSocket) ←→ 后端 (FastAPI + WebSocket)
                               ↓
                         简历解析/生成服务
```

### 完整目录结构

```
intelligent-resume-optimizer/
├── frontend/                          # 前端项目 (基于 MaaResource-main)
│   ├── public/
│   │   ├── index.html                # [保持不变]
│   │   ├── index.css                 # [保持不变]
│   │   └── manifest.json             # [保持不变]
│   ├── src/
│   │   ├── App.js                    # [修改] 集成聊天布局
│   │   ├── index.js                  # [保持不变]
│   │   ├── components/
│   │   │   ├── Basic/                # [保持不变] 原有组件
│   │   │   ├── Button/               # [保持不变] 原有组件
│   │   │   ├── Dialog/               # [保持不变] 原有组件
│   │   │   ├── Navbar/               # [保持不变] 原有组件
│   │   │   └── Chat/                 # [新增] 聊天组件
│   │   │       ├── ChatBox.js        # 聊天容器组件
│   │   │       ├── MessageList.js    # 消息列表组件
│   │   │       ├── InputArea.js      # 输入框组件
│   │   │       └── index.js          # 组件导出
│   │   ├── layout/
│   │   │   ├── Dialog.js             # [保持不变]
│   │   │   ├── Hint.js               # [保持不变]
│   │   │   ├── Navbar.js             # [保持不变]
│   │   │   ├── NormalResume.js       # [保持不变]
│   │   │   ├── Resume.js             # [保持不变]
│   │   │   └── ChatLayout.js         # [新增] 聊天+简历双栏布局
│   │   ├── store/
│   │   │   ├── dialog.js             # [保持不变]
│   │   │   ├── hint.js               # [保持不变]
│   │   │   ├── navbar.js             # [保持不变]
│   │   │   ├── resume.js             # [保持不变]
│   │   │   └── chat.js               # [新增] 聊天状态管理
│   │   ├── services/
│   │   │   ├── websocket.js          # [新增] WebSocket 连接管理
│   │   │   └── api.js                # [新增] REST API 调用
│   │   └── utils/
│   │       ├── constant.js           # [保持不变]
│   │       ├── helper.js             # [保持不变]
│   │       ├── theme0.js             # [保持不变]
│   │       ├── theme1.js             # [保持不变]
│   │       ├── theme2.js             # [保持不变]
│   │       ├── theme3.js             # [保持不变]
│   │       └── theme4.js             # [保持不变]
│   ├── package.json                  # [修改] 添加依赖: socket.io-client
│   └── README.md                     # [修改] 更新说明
│
└── backend/                           # [新增] Python 后端
    ├── app/
    │   ├── __init__.py
    │   ├── main.py                   # FastAPI 应用入口
    │   ├── config.py                 # 配置管理
    │   │
    │   ├── api/                      # API 路由层
    │   │   ├── __init__.py
    │   │   ├── websocket.py          # WebSocket 端点
    │   │   ├── resume.py             # 简历 CRUD API
    │   │   └── health.py             # 健康检查 API
    │   │
    │   ├── agents/                   # Agent 系统 (Phase 1: 空框架)
    │   │   ├── __init__.py
    │   │   └── base_agent.py         # Agent 基类定义
    │   │
    │   ├── rag/                      # RAG 系统 (Phase 1: 空框架)
    │   │   ├── __init__.py
    │   │   └── base_retriever.py     # 检索器基类
    │   │
    │   ├── mcp/                      # MCP 系统 (Phase 1: 空框架)
    │   │   ├── __init__.py
    │   │   └── protocol.py           # 协议定义
    │   │
    │   ├── models/                   # 数据模型
    │   │   ├── __init__.py
    │   │   ├── resume.py             # 简历数据模型
    │   │   ├── message.py            # 消息模型
    │   │   └── evaluation.py         # 评估模型 (Phase 1: 空定义)
    │   │
    │   ├── services/                 # 业务逻辑层
    │   │   ├── __init__.py
    │   │   ├── resume_parser.py      # Markdown 简历解析
    │   │   ├── resume_generator.py   # Markdown 简历生成
    │   │   ├── scoring_service.py    # 评分服务 (Phase 1: 空实现)
    │   │   └── search_service.py     # 搜索服务 (Phase 1: 空实现)
    │   │
    │   └── utils/                    # 工具函数
    │       ├── __init__.py
    │       ├── markdown_utils.py     # Markdown 处理工具
    │       ├── text_utils.py         # 文本处理工具
    │       └── logger.py             # 日志配置
    │
    ├── data/                         # 数据目录 (Phase 1: 空目录)
    │   ├── knowledge_base/           # 知识库数据
    │   │   ├── industry_keywords/
    │   │   ├── optimization_cases/
    │   │   └── excellent_resumes/
    │   └── vector_db/               # 向量数据库
    │
    ├── tests/                        # 测试目录
    │   ├── __init__.py
    │   ├── test_api.py              # API 测试
    │   ├── test_agents.py           # Agent 测试 (Phase 1: 空)
    │   ├── test_rag.py              # RAG 测试 (Phase 1: 空)
    │   └── test_mcp.py              # MCP 测试 (Phase 1: 空)
    │
    ├── requirements.txt              # Python 依赖
    ├── .env.example                  # 环境变量示例
    ├── Dockerfile                    # Docker 配置 (Phase 1: 空)
    └── README.md                     # 后端说明文档
```

### 核心实现内容

#### 1. 后端 FastAPI 框架 (app/main.py)

- FastAPI 应用初始化
- CORS 配置
- WebSocket 路由挂载
- REST API 路由挂载
- 基础中间件（日志、异常处理）
- 启动/关闭事件处理

#### 2. WebSocket 通信 (app/api/websocket.py)

- WebSocket 连接管理
- 消息接收和解析
- 简单的回声功能
- 连接状态维护
- 错误处理

#### 3. 简历 API (app/api/resume.py)

- POST /api/resume/upload - 上传简历
- GET /api/resume/{id} - 获取简历
- PUT /api/resume/{id} - 更新简历
- DELETE /api/resume/{id} - 删除简历
- 简历存储（内存/文件）

#### 4. 数据模型 (app/models/)

- Resume: 简历结构（基本信息、工作经历、项目经验等）
- Message: WebSocket 消息格式
- Evaluation: 评估结果结构（空定义）

#### 5. 简历解析服务 (app/services/resume_parser.py)

- Markdown 转结构化数据
- 识别简历各部分（基本信息、工作经历等）
- 提取关键字段

#### 6. 简历生成服务 (app/services/resume_generator.py)

- 结构化数据转 Markdown
- 格式化输出
- 模板支持

#### 7. 前端聊天组件 (src/components/Chat/)

- ChatBox: 聊天容器
- MessageList: 消息列表展示
- InputArea: 输入框和发送按钮
- 消息气泡样式

#### 8. WebSocket 服务 (src/services/websocket.js)

- WebSocket 连接封装
- 自动重连机制
- 消息发送/接收
- 事件监听

#### 9. 聊天状态管理 (src/store/chat.js)

- 消息列表管理
- 连接状态管理
- 会话 ID 管理

#### 10. 双栏布局 (src/layout/ChatLayout.js)

- 右侧聊天窗口
- 左侧简历预览
- 响应式布局
- 把ai助手集成到原来前端项目的上方工具栏里，点击后不刷新页面，ai助手能直接从侧栏划出，还在原来的页面里。再点击ai助手划回右侧隐藏起来。

### Phase 1 技术栈

```
后端:
- fastapi==0.104.1
- uvicorn==0.24.0
- websockets==12.0
- pydantic==2.5.0
- python-dotenv==1.0.0
- pytest==7.4.3

前端:
- react: ^18.x
- socket.io-client: ^4.x
- axios: ^1.x
```

### 验收标准

✅ 后端服务启动成功 (http://localhost:8000)
✅ WebSocket 连接建立成功 (ws://localhost:8000/ws/chat)
✅ 前端能发送消息并收到回复
✅ 简历上传、获取、更新功能正常
✅ Markdown 解析和生成正确
✅ 前端双栏布局正常显示
✅ 所有 API 有基础的错误处理

### 测试用例

1. 上传一份 Markdown 简历
2. 在聊天框输入 "你好"，收到回复
3. 修改简历内容，右侧实时更新
4. 断开 WebSocket 连接后自动重连