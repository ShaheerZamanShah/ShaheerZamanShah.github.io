# AgentOps

Self-hostable **Eval, Observability & CI/CD Gate** platform for LangGraph/LangChain agents.

## Quick Start (Local — No Docker)

**Prerequisites:** PostgreSQL 16+, Node.js 20+, Python 3.11+

```powershell
cd agentops
copy .env.local.example .env
# Edit .env — set your Postgres password

# One-time setup
.\scripts\setup-local.ps1

# Create database in psql/pgAdmin:
#   CREATE DATABASE agentops;

# Start API + Dashboard (evals run inline without Redis)
.\scripts\start-local.ps1
```

Open **http://localhost:3000** · API key: `agentops-dev-key-change-me`

## Quick Start (Docker)

```bash
cd agentops
cp .env.example .env
docker compose up -d
```

Open **http://localhost:3000**

## Architecture

```
┌─────────────┐     spans      ┌──────────────┐     persist    ┌──────────┐
│  SDK        │ ──────────────▶│  FastAPI API │ ──────────────▶│ Postgres │
│ (LangChain) │                │  + Celery    │                └──────────┘
└─────────────┘                │  Workers     │     pub/sub    ┌──────────┐
                               └──────┬───────┘ ──────────────▶│  Redis   │
                                      │                          └──────────┘
                                      │ WebSocket
                               ┌──────▼───────┐
                               │  Dashboard   │
                               │  (Next.js)   │
                               └──────────────┘
```

## Services

| Service | Port | Description |
|---------|------|-------------|
| API | 8000 | FastAPI ingestion + query API |
| Dashboard | 3000 | Observability UI |
| Postgres | 5432 | Trace + eval storage |
| Redis | 6379 | Live tailing + Celery broker (optional locally) |
| Celery Worker | — | Eval runs, alerts, A/B tests |
| Celery Beat | — | Alert rule scheduler (60s) |

## Features

- **Observability**: Trace ingestion, nested waterfall view, cost/token tracking
- **Eval Engine**: Test suites, LLM-as-judge graders (factuality, hallucination, tool use)
- **Regression Detection**: Baseline comparison with configurable thresholds
- **A/B Testing**: Side-by-side variant comparison with statistical significance
- **CI/CD Gate**: GitHub Action to gate merges on eval thresholds
- **Alerting**: Metric-based rules with webhook delivery

## SDK Usage

```bash
pip install -e ./sdk
```

```python
from agentops import AgentOps

ops = AgentOps(
    api_key="agentops-dev-key-change-me",
    api_url="http://localhost:8000",
    agent_name="my-agent",
)
instrumented = ops.wrap(compiled_graph)
result = instrumented.invoke({"messages": [...]})
ops.shutdown()
```

## Development

```bash
# Backend tests (requires Postgres)
cd backend && pytest tests/ -v

# SDK tests
cd sdk && pytest tests/ -v

# Dashboard
cd dashboard && npm run build
```

## Deployment Checklist

1. Set strong `SEED_API_KEY` in production `.env`
2. Set `AUTO_CREATE_TABLES=false` — use Alembic migrations only
3. Run `alembic upgrade head` before starting the API
4. Set `OPENAI_API_KEY` or `OLLAMA_BASE_URL` for LLM graders
5. Build dashboard with `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_API_KEY`
6. Enable Redis + Celery for production eval throughput

## Documentation

- [Quickstart](docs/quickstart.md)
- [API Reference](docs/api-reference.md)
- [Architecture](docs/architecture.md)
- [Engineering Decisions](DECISIONS.md)

## License

MIT
