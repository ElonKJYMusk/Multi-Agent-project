# 企业级多智能体金融分析与合规审计 API

这是一个基于 `LangGraph` 搭建的可部署项目，提供命令行与 `FastAPI` 两种使用方式，并已经扩展到可接入真实 PDF 财报、实时市场数据、Neo4j、PostgreSQL 和 Redis 队列。

- 有向图编排与角色化代理协作
- 三层记忆架构（会话状态、知识图谱记忆、推理审计日志）
- 失败检测、重规划与降级输出
- 受限表达式执行器，避免让 LLM 直接做数值计算
- 合规校验与最终报告合成
- `DeepSeek` 优先的大模型接入，缺少 Key 时自动回退本地 mock
- API 服务化部署、健康检查、配置接口与产物落盘
- SQLite 作业持久化、后台任务接口、可选 API Key 鉴权和请求日志
- 真实 PDF 上传解析
- 真实市场数据接口接入
- Neo4j 图谱存储支持
- PostgreSQL 任务持久化支持
- Redis 队列和独立 worker 支持

## 系统角色

- `Supervisor`：任务拆解与路由
- `Retrieval Agent`：样例财报/电话会/附录数据检索与实体摄取
- `Quantitative Analyst`：计算财务指标
- `Psychologist Agent`：分析管理层语气
- `Critic & Compliance Agent`：核查结果完整性与合规字段
- `Synthesizer`：输出最终研报
- `Joiner/Replanner`：在缺失数据时触发补检索与降级

## 项目结构

```text
project3/
├─ src/real_finan/
│  ├─ api/
│  ├─ agents.py
│  ├─ config.py
│  ├─ database.py
│  ├─ documents.py
│  ├─ graph.py
│  ├─ knowledge_store.py
│  ├─ llm.py
│  ├─ logging_utils.py
│  ├─ market_data.py
│  ├─ memory.py
│  ├─ queueing.py
│  ├─ reporting.py
│  ├─ service.py
│  ├─ state.py
│  ├─ tools.py
│  └─ data/sample_financial_data.py
├─ .env.example
├─ Dockerfile
├─ docker-compose.yml
├─ requirements.txt
├─ run_demo.py
├─ start_api.py
└─ start_worker.py
```

## 安装

```bash
pip install -r requirements.txt
```

或者：

```bash
pip install -e .
```

## 运行

默认运行：

```bash
python run_demo.py
```

输出完整状态：

```bash
python run_demo.py --json
```

指定输出目录与线程 ID：

```bash
python run_demo.py --output-dir outputs --thread-id run-001
```

携带真实 PDF：

```bash
python run_demo.py --query "分析上传财报的核心风险" --pdf ./docs/apple_10k.pdf
```

启动 API 服务：

```bash
python start_api.py
```

默认地址：

```bash
http://127.0.0.1:8000
```

健康检查：

```bash
curl http://127.0.0.1:8000/health
```

调用分析接口：

```bash
curl -X POST http://127.0.0.1:8000/api/v1/analyze ^
  -H "Content-Type: application/json" ^
  -d "{\"query\":\"对比分析 Apple 与 Microsoft 2025 年财报的供应链风险与研发投入，并生成合规审计报告。\"}"
```

上传 PDF 并分析：

```bash
curl -X POST http://127.0.0.1:8000/api/v1/analyze-upload ^
  -F "query=请分析上传财报的核心风险与市场表现" ^
  -F "thread_id=upload-001" ^
  -F "files=@./docs/apple_10k.pdf"
```

运行测试：

```bash
python -m unittest discover -s tests -v
```

## DeepSeek 配置

复制 `.env.example` 后设置：

```bash
DEEPSEEK_API_KEY=your_key
DEEPSEEK_MODEL=deepseek-chat
DEEPSEEK_BASE_URL=https://api.deepseek.com
```

说明：

- 配置了 `DEEPSEEK_API_KEY` 后，执行摘要、任务概览和合规摘要会调用 `DeepSeek`
- 未配置 Key 时，系统自动使用本地 fallback，方便本地调试和离线演示

## 服务化增强

新增能力：

- SQLite 持久化作业表
- 后台异步分析任务
- 作业状态查询
- 可选 `X-API-Key` 鉴权
- 请求级日志和 `X-Request-Id`

新增环境变量：

```bash
MAS_DB_PATH=data/real_finan.db
MAS_API_KEY=your_api_key
MAS_DATABASE_URL=postgresql+psycopg://mas_user:mas_password@localhost:5432/real_finan
MAS_REDIS_URL=redis://localhost:6379/0
MAS_NEO4J_URI=bolt://localhost:7687
MAS_NEO4J_USERNAME=neo4j
MAS_NEO4J_PASSWORD=neo4j_password
MAS_MARKET_DATA_PROVIDER=yahoo
```

## 设计说明

### 1. 编排方式

主流程为：

`Supervisor -> Retrieval -> Quant -> Psychologist -> Critic -> Synthesizer`

当检索或计算阶段发现缺失附录数据时，控制流会进入 `Replanner`：

- 第一次失败：切换到附录补检索策略
- 第二次仍失败：触发降级模式，继续生成带缺口说明的报告

### 2. 三层记忆

- `SessionMemory`：保存每个关键步骤后的状态快照
- `KnowledgeGraphMemory`：把公司、指标、风险、附录字段注入图结构
- `ReasoningMemory`：记录每个代理步骤的审计事件

### 3. 安全性

项目没有直接执行任意 Python 代码，而是通过 AST 白名单实现了一个受限表达式执行器，只允许基础数学运算。

### 4. 产物输出

每次运行都会自动在输出目录中生成三类产物：

- `*_report.md`：最终研报
- `*_audit.json`：代理审计轨迹
- `*_state.json`：完整工作流最终状态

### 5. API 接口

- `GET /health`
- `GET /api/v1/config`
- `POST /api/v1/analyze`
- `POST /api/v1/analyze-upload`
- `POST /api/v1/jobs`
- `POST /api/v1/jobs/upload`
- `GET /api/v1/jobs`
- `GET /api/v1/jobs/{job_id}`

`POST /api/v1/analyze` 请求体：

```json
{
  "query": "对比分析 Apple 与 Microsoft 2025 年财报的供应链风险与研发投入，并生成合规审计报告。",
  "thread_id": "run-001",
  "export_artifacts": true
}
```

`POST /api/v1/jobs` 请求体：

```json
{
  "query": "分析 Apple 2025 财报供应链附录风险，并输出报告。",
  "thread_id": "job-run-001",
  "export_artifacts": true
}
```

`POST /api/v1/analyze-upload` 使用 `multipart/form-data`：

- `query`
- `thread_id`
- `export_artifacts`
- `files`

鉴权：

- 若配置了 `MAS_API_KEY`，调用 `/api/v1/*` 需要请求头 `X-API-Key`
- `GET /health` 保持匿名可访问，方便探活

## Docker 部署

构建镜像：

```bash
docker build -t real-finan-api .
```

运行容器：

```bash
docker run -p 8000:8000 --env-file .env.example real-finan-api
```

使用 `docker-compose`：

```bash
docker compose up --build
```

这会同时启动：

- API 服务
- Redis worker
- PostgreSQL
- Redis
- Neo4j


