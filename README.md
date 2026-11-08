# Multi-Agent Financial Analysis and Compliance Review API

A deployable project built with `LangGraph`, with command-line and `FastAPI` interfaces. It supports PDF financial reports, live market data, Neo4j, PostgreSQL, and Redis queues.

- Directed-graph orchestration with specialized agent roles
- Three memory layers: session state, knowledge graph memory, and execution audit logs
- Missing-data detection, replanning, and degraded output
- Restricted expression evaluator for numerical calculations outside the LLM
- Completeness checks, compliance review, and final report synthesis
- DeepSeek integration with a local mock fallback when no API key is available
- API deployment, health checks, configuration endpoints, and saved artifacts
- SQLite job persistence, background jobs, optional API key authentication, and request logging
- PDF upload and parsing
- Live market data integration
- Neo4j graph storage support
- PostgreSQL job persistence support
- Redis queue and standalone worker support

## Agent Roles

- `Supervisor`: Identifies analysis targets and prepares the task brief within a predefined workflow
- `Retrieval Agent`: Retrieves sample financial statements, earnings-call excerpts, and appendix data, and ingests entities
- `Quantitative Analyst`: Calculates financial metrics
- `Psychologist Agent`: Analyzes management sentiment
- `Critic & Compliance Agent`: Checks result completeness and compliance-related information
- `Synthesizer`: Produces the final research report
- `Replanner`: Triggers supplementary retrieval or degraded output when data is missing

## Project Structure

```text
Multi-Agent-project/
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

## Installation

```bash
pip install -r requirements.txt
```

Alternatively:

```bash
pip install -e .
```

The application also imports `python-dotenv`, which is currently missing from the dependency declarations. Install it separately:

```bash
pip install python-dotenv
```

## Usage

Run the default demo:

```bash
python run_demo.py
```

Print the complete state:

```bash
python run_demo.py --json
```

Specify an output directory and thread ID:

```bash
python run_demo.py --output-dir outputs --thread-id run-001
```

Analyze a PDF (replace the example path with your own file):

```bash
python run_demo.py --query "Analyze the key risks in the uploaded financial report" --pdf ./docs/apple_10k.pdf
```

Start the API server:

```bash
python start_api.py
```

Default address: `http://127.0.0.1:8000`

Check server health:

```bash
curl http://127.0.0.1:8000/health
```

Call the analysis endpoint. The following multiline curl examples use Windows Command Prompt (`cmd.exe`) syntax:

```bat
curl -X POST http://127.0.0.1:8000/api/v1/analyze ^
  -H "Content-Type: application/json" ^
  -d "{\"query\":\"Compare supply chain risks and R&D investment in Apple and Microsoft 2025 financial reports, and generate a compliance review report.\"}"
```

Upload and analyze a PDF:

```bat
curl -X POST http://127.0.0.1:8000/api/v1/analyze-upload ^
  -F "query=Analyze the key risks and market performance in the uploaded financial report." ^
  -F "thread_id=upload-001" ^
  -F "files=@./docs/apple_10k.pdf"
```

Run the tests:

```bash
python -m unittest discover -s tests -v
```

## DeepSeek Configuration

Copy `.env.example` to `.env` and configure:

```dotenv
DEEPSEEK_API_KEY=your_key
DEEPSEEK_MODEL=deepseek-chat
DEEPSEEK_BASE_URL=https://api.deepseek.com
```

- With `DEEPSEEK_API_KEY` configured, the system uses DeepSeek for generated content such as task briefs, executive summaries, and compliance summaries.
- Without a key, or when the model request fails, the system uses a local mock with preset responses for debugging and offline demonstrations. This fallback does not run a local language model or provide task-specific analysis.

## Service Features

- Persistent SQLite job table
- Background analysis jobs
- Job status queries
- Optional `X-API-Key` authentication
- Request logging and `X-Request-Id`

Additional environment variables:

```dotenv
MAS_DB_PATH=data/real_finan.db
MAS_API_KEY=your_api_key
MAS_DATABASE_URL=postgresql+psycopg://mas_user:mas_password@localhost:5432/real_finan
MAS_REDIS_URL=redis://localhost:6379/0
MAS_NEO4J_URI=bolt://localhost:7687
MAS_NEO4J_USERNAME=neo4j
MAS_NEO4J_PASSWORD=neo4j_password
MAS_MARKET_DATA_PROVIDER=yahoo
```

## Design Notes

### 1. Orchestration

The main workflow is:

`Supervisor -> Retrieval -> Quant -> Psychologist -> Critic -> Synthesizer`

When retrieval or quantitative analysis detects missing data, execution enters `Replanner`:

- First attempt: switch to supplementary appendix retrieval.
- If data remains unavailable on the second attempt: enter degraded mode and proceed to report generation.

The graph defines the execution order and routing rules in code. The critic records findings but currently does not block report generation or request human approval.

### 2. Three Memory Layers

- `SessionMemory`: Stores in-memory state snapshots after key steps.
- `KnowledgeGraphMemory`: Organizes companies, metrics, risks, and appendix fields in a graph.
- `ReasoningMemory`: Records execution audit events for agent steps.

LangGraph checkpoints are also held in memory. Persistent job records and exported state files do not currently provide automatic workflow recovery after a process restart.

### 3. Restricted Calculation

The project uses an AST allowlist to evaluate basic mathematical expressions instead of executing arbitrary Python code.

### 4. Output Artifacts

When artifact export is enabled, each run saves three files to the output directory:

- `*_report.md`: Final research report
- `*_audit.json`: Agent execution audit trail
- `*_state.json`: Complete final workflow state

### 5. API Endpoints

- `GET /health`
- `GET /api/v1/config`
- `POST /api/v1/analyze`
- `POST /api/v1/analyze-upload`
- `POST /api/v1/jobs`
- `POST /api/v1/jobs/upload`
- `GET /api/v1/jobs`
- `GET /api/v1/jobs/{job_id}`

Request body for `POST /api/v1/analyze`:

```json
{
  "query": "Compare supply chain risks and R&D investment in Apple and Microsoft 2025 financial reports, and generate a compliance review report.",
  "thread_id": "run-001",
  "export_artifacts": true
}
```

Request body for `POST /api/v1/jobs`:

```json
{
  "query": "Analyze supply chain appendix risks in Apple 2025 financial reports and generate a report.",
  "thread_id": "job-run-001",
  "export_artifacts": true
}
```

`POST /api/v1/analyze-upload` accepts `multipart/form-data` fields:

- `query`
- `thread_id`
- `export_artifacts`
- `files`

Authentication:

- When `MAS_API_KEY` is configured, requests to `/api/v1/*` require the `X-API-Key` header.
- `GET /health` remains accessible without authentication for health checks.

## Docker Deployment

Build the image:

```bash
docker build -t real-finan-api .
```

Run a container using an environment file containing your configuration:

```bash
docker run -p 8000:8000 --env-file .env.example real-finan-api
```

Start the stack with Docker Compose:

```bash
docker compose up --build
```

This starts:

- API server
- Redis worker
- PostgreSQL
- Redis
- Neo4j
