# 🤖 AI Customer Support Agent

A **production-grade RAG (Retrieval-Augmented Generation) system** built with LangChain, ChromaDB, and OpenAI that functions as an enterprise customer support assistant for a fictional company, **TechCorp Global**.

> Designed to showcase advanced RAG architecture, scalable modular design, and production-quality code suitable for a professional portfolio.

---

## 📐 System Architecture

```
User Query
    │
    ▼
┌─────────────────────┐
│  Query Rewriting     │  LLM rewrites the question for better semantic search
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Query Classification│  Classifies into: Refund, Shipping, Warranty, etc.
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Query Routing       │  Applies metadata filter to target relevant docs
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Multi-Query Gen     │  Generates 3 variant search queries
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Hybrid Retrieval    │  Dense (embedding) + Keyword search with RRF fusion
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Reranking           │  LLM scores relevance (1-10), selects top chunks
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Context Compression │  Extracts only query-relevant sentences
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Answer Generation   │  Grounded LLM response with source citations
└─────────────────────┘
```

### Retrieval Validation & Fallback

If all retrieved documents score poorly during reranking, the system automatically **retries with a broader search** (no category filter) to prevent empty responses.

---

## 🗂 Project Structure

```
ai_support_agent/
│
├── app.py                          # Streamlit UI (chat interface)
│
├── config/
│   └── settings.py                 # Central configuration (API keys, params)
│
├── data/
│   └── documents/                  # Synthetic enterprise knowledge base (10 docs)
│
├── ingestion/
│   ├── document_loader.py          # Load .txt/.pdf files with metadata
│   ├── text_splitter.py            # Chunk documents (800 chars, 150 overlap)
│   ├── embedding_generator.py      # OpenAI embeddings singleton
│   └── vector_indexer.py           # ChromaDB build & load
│
├── retrieval/
│   ├── query_rewriter.py           # LLM-based query rewriting
│   ├── query_classifier.py         # Classify query into categories
│   ├── query_router.py             # Build metadata filter for routing
│   ├── hybrid_search.py            # Dense + keyword search with RRF
│   ├── reranker.py                 # LLM-based relevance reranking
│   ├── context_filter.py           # Context compression
│   └── retriever.py                # Full retrieval pipeline orchestrator
│
├── rag/
│   ├── rag_pipeline.py             # Top-level RAG orchestrator
│   ├── answer_generator.py         # Grounded answer generation
│   └── source_formatter.py         # Citation formatting
│
├── utils/
│   ├── logger.py                   # Centralized logging
│   └── helpers.py                  # Token counting, hashing, dedup
│
├── scripts/
│   ├── generate_fake_documents.py  # Generate 10 synthetic enterprise docs
│   └── build_index.py              # End-to-end index build script
│
├── vectorstore/                    # ChromaDB persistent storage
├── logs/                           # Log files
├── requirements.txt
└── README.md
```

---

## 🧠 Advanced RAG Features

| Feature                   | Description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| **Query Rewriting**       | LLM rewrites user questions into keyword-rich search queries |
| **Query Classification**  | Classifies into 8 categories for targeted retrieval          |
| **Query Routing**         | Applies metadata filters to search relevant document subsets |
| **Multi-Query Retrieval** | Generates 3 variant queries, retrieves for each              |
| **Hybrid Search**         | Combines dense embeddings + keyword matching via RRF         |
| **Reranking**             | LLM scores each chunk's relevance (1-10 scale)               |
| **Context Compression**   | Extracts only relevant sentences to minimize token usage     |
| **Retrieval Validation**  | Retries with broader search if initial retrieval fails       |
| **Source Grounding**      | Every answer includes document name, section, and category   |

---

## 📚 Knowledge Base

The system includes 10 synthetic enterprise documents (~2,000-4,000 words each):

| Document                          | Category          | Content                                           |
| --------------------------------- | ----------------- | ------------------------------------------------- |
| `refund_policy.txt`               | Refund Policy     | Eligibility, process, timelines, damaged products |
| `shipping_policy.txt`             | Shipping          | Domestic/international options, rates, tracking   |
| `warranty_policy.txt`             | Warranty          | Coverage, claims, TechCare plans, repairs         |
| `product_manual_smartphone.txt`   | Product Support   | Nova X1 specs, setup, camera, battery             |
| `product_manual_laptop.txt`       | Product Support   | ProBook Ultra 16 specs, keyboard, performance     |
| `troubleshooting_guide.txt`       | Technical Support | Display, audio, connectivity, recovery            |
| `account_management_guide.txt`    | Account Issues    | 2FA, payments, cloud, profiles                    |
| `faq_support_guide.txt`           | General FAQ       | Top questions across all categories               |
| `enterprise_support_policy.txt`   | Enterprise        | Support tiers, SLAs, on-site service              |
| `subscription_billing_policy.txt` | Billing           | Plans, pricing, cancellation, trials              |

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- OpenAI API key

### 1. Install dependencies

```bash
cd ai_support_agent
pip install -r requirements.txt
```

### 2. Set your API key

```bash
# Windows PowerShell
$env:OPENAI_API_KEY = "sk-your-key-here"

# Linux / macOS
export OPENAI_API_KEY="sk-your-key-here"
```

Or enter it in the Streamlit sidebar at runtime.

### 3. Generate documents & build the vector index

```bash
python -m ai_support_agent.scripts.build_index
```

This will:

1. Generate 10 synthetic documents in `data/documents/`
2. Load and chunk all documents (800 chars / 150 overlap)
3. Embed chunks with OpenAI `text-embedding-3-small`
4. Store in a persistent ChromaDB collection in `vectorstore/`

### 4. Launch the Streamlit app

```bash
streamlit run ai_support_agent/app.py
```

Open **http://localhost:8501** in your browser.

---

## 🔄 Rebuilding the Index

If you add, edit, or remove documents in `data/documents/`, rebuild:

```bash
# Optionally delete the old index first
rm -r ai_support_agent/vectorstore/*

# Rebuild
python -m ai_support_agent.scripts.build_index
```

---

## 🧪 Example Queries to Test

| Query                                          | Expected Category        | Expected Source                 |
| ---------------------------------------------- | ------------------------ | ------------------------------- |
| "What happens if I return a damaged product?"  | Refund Policy            | refund_policy.txt               |
| "How long does standard shipping take?"        | Shipping                 | shipping_policy.txt             |
| "How do I file a warranty claim?"              | Warranty                 | warranty_policy.txt             |
| "My laptop won't turn on"                      | Technical Support        | troubleshooting_guide.txt       |
| "What is the battery life of the Nova X1?"     | Product Support          | product_manual_smartphone.txt   |
| "How do I enable two-factor authentication?"   | Account Issues           | account_management_guide.txt    |
| "How do I cancel my subscription?"             | Subscription & Billing   | subscription_billing_policy.txt |
| "Does TechCorp offer enterprise support SLAs?" | General FAQ / Enterprise | enterprise_support_policy.txt   |

---

## ⚙️ Configuration

All tuneable parameters are in [`config/settings.py`](config/settings.py):

| Parameter           | Default                  | Description                           |
| ------------------- | ------------------------ | ------------------------------------- |
| `EMBEDDING_MODEL`   | `text-embedding-3-small` | OpenAI embedding model                |
| `LLM_MODEL`         | `gpt-4o-mini`            | Chat model for generation & rewriting |
| `CHUNK_SIZE`        | 800                      | Characters per chunk                  |
| `CHUNK_OVERLAP`     | 150                      | Overlap between adjacent chunks       |
| `TOP_K_RETRIEVAL`   | 10                       | Broad retrieval count                 |
| `TOP_K_RERANK`      | 5                        | Documents kept after reranking        |
| `TOP_K_FINAL`       | 3                        | Documents sent to the LLM             |
| `MULTI_QUERY_COUNT` | 3                        | Variant queries generated             |

---

## 🏗 Technology Stack

- **LangChain** — LLM orchestration, prompt templates, document loaders
- **ChromaDB** — Persistent vector database with metadata filtering
- **OpenAI** — Embeddings (`text-embedding-3-small`) and chat (`gpt-4o-mini`)
- **Streamlit** — Web UI with chat interface
- **tiktoken** — Accurate token counting
- **pypdf** — PDF document support (optional)

---

## 📝 Design Decisions

### Why `chunk_size=800` / `chunk_overlap=150`?

- **800 chars** fits roughly one policy section or FAQ answer — enough context for the LLM to generate a complete response without retrieving the entire document.
- **150 chars overlap** ensures that sentences at the boundary of two chunks appear in both, so no information is lost during splitting.

### Why Hybrid Search (Dense + Keyword)?

Pure embedding search excels at semantic similarity but can miss documents when the user uses exact terminology (policy codes, product names). Keyword search catches these. RRF fusion combines both rankings fairly.

### Why LLM-based Reranking?

Cross-encoder models are heavier and require extra dependencies. Using the same chat model with a simple scoring prompt is lightweight, portable, and surprisingly effective for ≤20 candidate documents.

### Why Context Compression?

Without compression, the LLM prompt may include large irrelevant paragraphs, wasting tokens and diluting answer quality. Compression extracts only the relevant sentences, improving both cost and accuracy.

---

## 📄 License

This project is for educational and portfolio purposes.
