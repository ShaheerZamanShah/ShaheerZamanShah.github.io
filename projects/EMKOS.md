# EMKOS — Enterprise Multi-Agent Knowledge Operating System

<div align="center">

**A production-grade multi-agent AI platform for enterprise knowledge management.**

[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react)](https://react.dev)
[![Groq](https://img.shields.io/badge/Groq-LLM-FF6B35?style=flat-square)](https://groq.com)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat-square&logo=docker)](https://www.docker.com)
[![Render](https://img.shields.io/badge/Deploy-Render-46E3B7?style=flat-square)](https://render.com)
[![Pinecone](https://img.shields.io/badge/Vectors-Pinecone-000000?style=flat-square)](https://www.pinecone.io)

[Live Demo](#-deploy-on-render) · [Architecture](#-architecture) · [RAG Pipeline](#-rag-pipeline) · [API](#-api-reference) · [Local Dev](#-local-development)

**Repository:** [github.com/ShaheerZamanShah/EMKOS](https://github.com/ShaheerZamanShah/EMKOS)

</div>

---

## What is EMKOS?

EMKOS lets teams upload policy documents (PDF, DOCX, TXT, Markdown) and ask natural-language questions. A **LangGraph multi-agent pipeline** classifies intent, retrieves relevant knowledge, generates grounded answers, and streams them to a modern React UI.

| Capability | Status | Description |
|---|---|---|
| Document Q&A (RAG) | ✅ | Hybrid BM25 + dense retrieval over uploaded docs |
| Knowledge Base UI | ✅ | Upload, delete, status tracking, chunk counts |
| Multi-agent chat | ✅ | Intent → RAG → Critic → streamed response |
| Source citations | ✅ | Clean answers + expandable Sources panel |
| JWT + RBAC | ✅ | 4 roles (Employee, Manager, Admin, Compliance) |
| Groq / Ollama LLM | ✅ | Cloud or local inference |
| Docker + Render | ✅ | Single-container production deploy (Render Free) |
| Pinecone vectors | ✅ | Cloud embeddings + vector search (fits 512 MB RAM) |
| SQL Agent | 🔜 | Natural language database queries |
| Jira / Slack tools | 🔜 | Action integrations |
| Human-in-the-loop | 🔜 | Approval workflows |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         React 19 + Vite + TypeScript                    │
│   Chat · Knowledge Base · Audit · Zustand · TanStack Query · SSE       │
└───────────────────────────────┬─────────────────────────────────────┘
                                  │ HTTP / SSE
┌───────────────────────────────▼─────────────────────────────────────┐
│              FastAPI (async) + Uvicorn — slim production image         │
│   /api/* · /health · (production) serves React static from /static     │
└───────────────┬───────────────────────┬─────────────────────────────┘
                │                       │
    ┌───────────▼──────────┐   ┌────────▼────────┐   ┌─────────────────┐
    │  PostgreSQL / SQLite │   │  LangGraph      │   │  Groq / Ollama  │
    │  docs, chunks, chat  │   │  Agent Graph    │   │  LLM Provider   │
    └──────────────────────┘   └────────┬────────┘   └─────────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
        Intent Classifier          RAG Agent               Policy Guard
              │                         │                         │
              ▼                         ▼                         ▼
        General Agent          Hybrid Retriever            Action Agent
              │              (BM25 + Pinecone dense)             │
              └────────────────────────┬────────────────────────┘
                                       ▼
                              Critic → Response Formatter

Production only (EMBEDDING_PROVIDER=pinecone):
┌─────────────────────────────────────────────────────────────────────────┐
│  Pinecone — Inference API (embeddings) + Serverless index (vectors)     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Agent graph flow

```
User message
    │
    ▼
Intent Classifier (Groq)
    │
    ├── rag_query ──────► RAG Agent ──► Critic ──► Formatter ──► SSE stream
    ├── general_chat ───► General Agent ──► Critic ──► Formatter
    ├── compliance ─────► Compliance Agent ──► Critic
    ├── sql_query ──────► SQL Agent (stub)
    └── action ─────────► Policy Guard ──► Action / Approval
```

---

## RAG pipeline

Document ingestion and retrieval are optimized for accuracy and speed.

### Ingestion (upload → ready)

**Local dev** (`EMBEDDING_PROVIDER=local`):

```
File upload (PDF/DOCX/TXT/MD)
    → Text extraction (pypdf / python-docx)
    → Text cleaning (whitespace, PDF artifacts)
    → Semantic chunking (heading-aware, 512-token target)
    → Local embedding (all-MiniLM-L6-v2, batch of 64)
    → Store Document + DocumentChunks (text + vectors) in DB
```

**Production** (`EMBEDDING_PROVIDER=pinecone`):

```
File upload (PDF/DOCX/TXT/MD)
    → Text extraction + cleaning + chunking
    → Pinecone Inference API embed (multilingual-e5-large)
    → Store chunk text in PostgreSQL
    → Upsert vectors to Pinecone index (metadata: chunk_id, document_id)
```

- Runs as a **background async task** — API returns immediately with `status=pending`
- **No local ML models** in production Docker image (fits Render Free 512 MB RAM)
- Local dev preloads the embedding model at startup; production skips preload

### Retrieval (question → context)

```
User query
    → Compound query decomposition ("X and Y" → sub-queries)
    → BM25 keyword search (top 30, from Postgres chunk text)
    → Dense vector search (top 30 — Pinecone in prod, local cosine in dev)
    → Reciprocal Rank Fusion (RRF)
    → Filter TOC / low-quality chunks
    → Cosine re-score + document title term boost
    → Neighbor chunk expansion (split sections)
    → Diversify across documents (max 3–4 per doc)
    → Top 8 chunks → LLM context
```

### Answer generation

- LLM receives labeled `[Source N]` context but is instructed **not** to inline citations
- Sources appear in a **Sources (N)** toggle in the UI with relevance scores
- Citations are stored in message metadata and sent via SSE `done` event

---

## Project structure

```
EMKOS/
├── Dockerfile                 # Multi-stage: Node build + slim Python runtime
├── render.yaml                # Render Blueprint (web + Postgres + Pinecone env)
├── .dockerignore
├── .env.example               # Template — copy to backend/.env
├── LICENSE
├── README.md
│
├── backend/
│   ├── requirements.txt       # Full deps (local dev — includes sentence-transformers)
│   ├── requirements-prod.txt  # Slim prod deps (Pinecone, no PyTorch)
│   └── app/
│       ├── main.py            # FastAPI app, lifespan, static SPA serving
│       ├── config.py          # Pydantic settings (env + .env)
│       ├── database.py        # Async SQLAlchemy engine
│       ├── seed.py            # Default dev user
│       │
│       ├── api/               # REST endpoints
│       │   ├── auth.py        # Register, login, JWT refresh
│       │   ├── chat.py        # Chat + SSE streaming
│       │   ├── documents.py   # Upload, list, delete (+ Pinecone cleanup)
│       │   └── audit.py       # Audit log viewer
│       │
│       ├── agents/            # LangGraph multi-agent system
│       │   ├── graph.py       # StateGraph assembly + run_agent_graph()
│       │   ├── intent_classifier.py
│       │   ├── rag_agent.py   # Retrieval + grounded generation
│       │   ├── general_agent.py
│       │   ├── critic.py      # Quality review
│       │   ├── policy_guard.py
│       │   ├── specialist_agents.py
│       │   └── state.py       # AgentState TypedDict
│       │
│       ├── rag/               # RAG pipeline
│       │   ├── ingestion.py   # PDF/DOCX/text extraction
│       │   ├── chunker.py     # Semantic chunking
│       │   ├── embeddings.py  # Local or Pinecone Inference provider
│       │   ├── pinecone_store.py  # Vector upsert/query/delete (production)
│       │   ├── retriever.py   # Hybrid BM25 + dense + RRF
│       │   └── rag_service.py # Ingest orchestration
│       │
│       ├── auth/              # JWT + RBAC
│       ├── models/            # SQLAlchemy ORM (User, Document, Message…)
│       ├── schemas/           # Pydantic request/response models
│       ├── services/
│       │   ├── chat_service.py    # Chat + agent streaming
│       │   └── llm_provider.py    # Groq / Ollama abstraction
│       └── tests/
│
├── frontend/
│   ├── src/
│   │   ├── pages/             # ChatPage, KnowledgePage, AuditPage, LoginPage
│   │   ├── components/
│   │   │   ├── chat/          # ChatWindow, MessageBubble, ChatInput
│   │   │   └── layout/        # AppLayout, Sidebar
│   │   ├── store/             # Zustand (authStore, chatStore)
│   │   └── lib/               # api.ts (Axios), utils.ts
│   ├── vite.config.ts         # Dev proxy → backend
│   └── package.json
│
└── infra/
    └── docker-compose.dev.yml # Local PostgreSQL + Redis
```

---

## Tech stack

| Layer | Technologies |
|---|---|
| **Backend** | Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Uvicorn |
| **Agents** | LangGraph, LangChain, LangChain-Groq |
| **RAG** | rank-bm25, pypdf, python-docx, numpy; **local:** sentence-transformers; **prod:** Pinecone Inference |
| **Frontend** | React 19, TypeScript, Vite 8, Tailwind CSS v4, Zustand, TanStack Query |
| **LLM** | Groq Cloud (primary), Ollama (local fallback) |
| **Database** | PostgreSQL (Render prod), SQLite (local quick test) |
| **Vectors** | Pinecone serverless index (prod), JSON embeddings in DB (local dev) |
| **Deploy** | Docker multi-stage build, [Render](https://render.com) Blueprint (Free tier) |

---

## Deploy on Render

The repo includes everything needed for a one-click Render deploy via [Blueprint](https://render.com/docs/blueprint-spec).

### Prerequisites

1. [Render](https://render.com) account
2. [Groq API key](https://console.groq.com) (free tier)
3. [Pinecone](https://www.pinecone.io) account (Starter — free, no credit card)
4. GitHub repo connected: [ShaheerZamanShah/EMKOS](https://github.com/ShaheerZamanShah/EMKOS)

### Pinecone setup (one-time)

1. Create a project at [app.pinecone.io](https://app.pinecone.io)
2. Create a **serverless** index:

| Setting | Value |
|---|---|
| Name | `emkos` |
| Dimensions | `1024` |
| Metric | `cosine` |
| Cloud / Region | AWS `us-east-1` (required on free tier) |

3. Copy your **API key** from the Pinecone console

### Steps

1. Push this repo to GitHub
2. In Render Dashboard → **New** → **Blueprint**
3. Connect the `EMKOS` repository
4. Render reads `render.yaml` and creates:
   - **Web Service** `emkos` (Docker, **Free** plan — $0/month)
   - **PostgreSQL** `emkos-db` (free tier)
5. Set secret env vars in the Render dashboard:
   - **`GROQ_API_KEY`**
   - **`PINECONE_API_KEY`**
6. Deploy — first build takes ~3–5 min (slim image, no ML model download)
7. Open your Render URL (e.g. `https://emkos.onrender.com`)

> **Free tier notes:** Render sleeps after 15 min inactivity (first request ~30–60s). Embeddings run on Pinecone's API — the container stays under 512 MB RAM. Pinecone Starter includes 5M embedding tokens/month and 2 GB vector storage.

### Production environment variables

| Variable | Required | Value / Description |
|---|---|---|
| `GROQ_API_KEY` | ✅ | Groq API key |
| `PINECONE_API_KEY` | ✅ | Pinecone API key |
| `PINECONE_INDEX_NAME` | ✅ | `emkos` |
| `EMBEDDING_PROVIDER` | ✅ | `pinecone` |
| `EMBEDDING_MODEL` | ✅ | `multilingual-e5-large` (Pinecone Inference model) |
| `DATABASE_URL` | Auto | Injected from Render Postgres |
| `JWT_SECRET_KEY` | Auto | Generated by Render |
| `SERVE_STATIC` | `true` | Serve React from same container |
| `AUTH_DISABLED` | `true` | Skip login for demo |
| `DEBUG` | `false` | Disable SQL echo |
| `RERANKER_ENABLED` | `false` | Keep off (rerankers need heavy local models) |

> **Note:** `EMBEDDING_MODEL` is one variable — its value depends on provider. Use `multilingual-e5-large` on Render (Pinecone), and `sentence-transformers/all-MiniLM-L6-v2` locally.

### Cost (completely free demo stack)

| Service | Plan | Cost |
|---|---|---|
| Render web service | Free | $0 |
| Render PostgreSQL | Free | $0 |
| Pinecone | Starter | $0 |
| Groq | Free tier | $0 |
| **Total** | | **$0/month** |

### Docker build locally (production image)

```bash
docker build -t emkos .
docker run -p 8000:8000 \
  -e GROQ_API_KEY=gsk_your_key \
  -e PINECONE_API_KEY=pcsk_your_key \
  -e PINECONE_INDEX_NAME=emkos \
  -e EMBEDDING_PROVIDER=pinecone \
  -e EMBEDDING_MODEL=multilingual-e5-large \
  -e DATABASE_URL=sqlite+aiosqlite:///./emkos.db \
  -e SERVE_STATIC=true \
  -e AUTH_DISABLED=true \
  emkos
```

Open http://localhost:8000

---

## Local development

### Prerequisites

| Tool | Version |
|---|---|
| Python | 3.12+ |
| Node.js | 20+ |
| Docker Desktop | Optional (for PostgreSQL) |

### 1. Clone

```bash
git clone https://github.com/ShaheerZamanShah/EMKOS.git
cd EMKOS
```

### 2. Environment

```bash
copy .env.example backend\.env
# Edit backend\.env — set GROQ_API_KEY=gsk_...
# Local dev uses EMBEDDING_PROVIDER=local (default) — no Pinecone key needed
```

### 3. Database (optional — or use SQLite)

```bash
docker compose -f infra/docker-compose.dev.yml up -d
```

Or use SQLite (no Docker):

```env
DATABASE_URL=sqlite+aiosqlite:///./emkos.db
```

### 4. Backend

```bash
cd backend
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS/Linux

pip install -r requirements.txt
uvicorn app.main:app --reload --port 8002
```

### 5. Frontend

```bash
cd frontend
npm install
npm run dev
```

### 6. Open

| Service | URL |
|---|---|
| Frontend | http://localhost:5174 |
| API docs | http://localhost:8002/api/docs |
| Health | http://localhost:8002/health |

With `AUTH_DISABLED=true`, the app opens directly to the workspace — no login required.

---

## API reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | — | Create account |
| `POST` | `/api/auth/login` | — | Login → JWT tokens |
| `POST` | `/api/auth/refresh` | — | Refresh access token |
| `GET` | `/api/auth/me` | Bearer | Current user |
| `POST` | `/api/chat/stream` | Bearer* | SSE streaming chat |
| `GET` | `/api/chat/conversations` | Bearer* | List conversations |
| `GET` | `/api/chat/conversations/{id}` | Bearer* | Get messages |
| `DELETE` | `/api/chat/conversations/{id}` | Bearer* | Archive conversation |
| `POST` | `/api/documents/upload` | Bearer* | Upload document (async ingest) |
| `GET` | `/api/documents/` | Bearer* | List documents |
| `DELETE` | `/api/documents/{id}` | Bearer* | Delete document + chunks |
| `GET` | `/api/audit/` | Admin | Audit logs |
| `GET` | `/health` | — | Health check |

\* Skipped when `AUTH_DISABLED=true`

### SSE stream format

```json
{"type": "metadata", "conversation_id": "..."}
{"type": "token", "content": "Hello"}
{"type": "done", "conversation_id": "...", "citations": [...]}
```

---

## Authentication & RBAC

| Permission | Employee | Manager | Admin | Compliance |
|---|:---:|:---:|:---:|:---:|
| Chat & documents | ✅ | ✅ | ✅ | ✅ |
| Create tickets / messages | ❌ | ✅ | ✅ | ❌ |
| SQL queries | ❌ | ✅ | ✅ | ✅ |
| Approve actions | ❌ | ✅ | ✅ | ❌ |
| Audit logs | ❌ | ❌ | ✅ | ✅ |
| Manage users | ❌ | ❌ | ✅ | ❌ |

---

## LLM configuration

### Groq (recommended)

```env
LLM_PROVIDER=groq
GROQ_API_KEY=gsk_your_key_here
GROQ_DEFAULT_MODEL=meta-llama/llama-4-scout-17b-16e-instruct
```

Get a free key at [console.groq.com](https://console.groq.com).

### Embeddings (local vs production)

**Local development** (`backend/.env`):

```env
EMBEDDING_PROVIDER=local
EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
RERANKER_ENABLED=false
```

**Production on Render:**

```env
EMBEDDING_PROVIDER=pinecone
EMBEDDING_MODEL=multilingual-e5-large
PINECONE_API_KEY=pcsk_your_key
PINECONE_INDEX_NAME=emkos
RERANKER_ENABLED=false
```

### Ollama (local, no API key)

```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_DEFAULT_MODEL=qwen3:32b
```

---

## Development phases

| Phase | Status | Scope |
|---|---|---|
| 1 | ✅ | Auth, Groq, streaming chat UI |
| 2 | ✅ | RAG: chunking, embeddings, hybrid retrieval, knowledge base |
| 3 | ✅ | LangGraph multi-agent orchestration |
| 4 | 🔜 | SQL Agent + safety guards |
| 5 | 🔜 | Jira, Slack, Teams, Salesforce integrations |
| 6 | 🔜 | Human-in-the-loop approvals |
| 7 | 🔜 | Langfuse, Prometheus, Grafana observability |
| 8 | 🔜 | Evaluation framework |
| 9 | ✅ | Docker + Render deployment |

---

## Key design decisions

**Why Groq?** Sub-100ms first-token latency; critical when the agent graph makes multiple LLM calls per request.

**Why LangGraph?** Stateful graph with conditional routing, retries, and approval nodes — linear chains cannot model enterprise workflows.

**Why Pinecone for production?** Render Free has 512 MB RAM — PyTorch + sentence-transformers exceed that. Pinecone Inference handles embeddings via API; vectors live in a managed index. Container RAM stays ~150–200 MB.

**Why hybrid RAG?** BM25 catches exact keywords (policy names, SKUs); dense vectors capture semantic paraphrases. RRF combines both without score normalization issues.

**Why SSE over WebSockets?** Simpler (plain HTTP), works through proxies, auto-reconnects — ideal for one-way LLM token streaming.

**Why single Docker container on Render?** Frontend + API on one origin eliminates CORS complexity and reduces cost vs. two services.

---

## License

MIT — see [LICENSE](LICENSE).

---

<div align="center">
Built by <a href="https://github.com/ShaheerZamanShah">Shaheer Zaman Shah</a>
</div>
