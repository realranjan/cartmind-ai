<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="CartMind AI — Autonomous E-Commerce Shopping Assistant">
</p>

<p align="center">
  <a href="https://github.com/realranjan/cartmind-ai/actions"><img src="https://img.shields.io/badge/CI-Passing-34D399?style=flat-square&logo=githubactions&logoColor=white" alt="CI Status"></a>
  <a href="https://fastapi.tiangolo.com/"><img src="https://img.shields.io/badge/FastAPI-0.115+-009688?style=flat-square&logo=fastapi&logoColor=white" alt="FastAPI"></a>
  <a href="https://react.dev/"><img src="https://img.shields.io/badge/React-19.2+-61DAFB?style=flat-square&logo=react&logoColor=black" alt="React 19"></a>
  <a href="https://ai.google.dev/"><img src="https://img.shields.io/badge/Google%20Gemini-2.5%20Flash-8E75B2?style=flat-square&logo=googlegemini&logoColor=white" alt="Gemini 2.5"></a>
  <a href="https://www.postgresql.org/"><img src="https://img.shields.io/badge/PostgreSQL-Neon%20Serverless-4169E1?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL"></a>
  <a href="https://www.pinecone.io/"><img src="https://img.shields.io/badge/Pinecone-Vector%20DB-000000?style=flat-square&logo=pinecone&logoColor=white" alt="Pinecone"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue?style=flat-square" alt="MIT License"></a>
</p>

---

## ⚡ Overview

**CartMind AI** is an autonomous, full-stack AI shopping assistant built on a live scraped catalogue of over 3,600+ footwear SKUs. Unlike simple chat wrappers that hallucinate catalogue items or invent store policies, CartMind pairs an **agentic LangChain router** with deterministic, schema-safe tools:

1. **Text-to-SQL Engine**: Generates real SQL queries against Postgres to verify stock, price, and specs before recommending.
2. **Shop by Photo**: Multimodal visual search via Gemini 2.5 Vision that maps uploaded shoe images to catalogue items while rejecting non-footwear photos.
3. **Policy RAG**: Semantic vector lookup across store returns, delivery, and warranty rules via Pinecone and Gemini embeddings.
4. **Transactional Cart & Orders**: Handles add-to-cart, cancellations, and order placement with atomic stock validation.
5. **Persistent User Memory**: Integrates **Supermemory** to retain shopper sizing, favorite brands, and price constraints across sessions.

<p align="center">
  <img src="./assets/readme/features.svg" width="100%" alt="CartMind AI Core Capabilities">
</p>

---

## 🏗️ Architecture & Request Lifecycle

CartMind isolates AI reasoning from web traffic using an asynchronous, lease-managed PostgreSQL queue (`FOR UPDATE SKIP LOCKED`). Shopper questions never block HTTP workers, and results stream incrementally via Server-Sent Events (SSE).

<p align="center">
  <img src="./assets/readme/architecture.svg" width="100%" alt="CartMind AI System Architecture">
</p>

### The 5 Architectural Layers

1. **Client Experience (React 19 + Vite)**:
   - Real-time SSE streaming reader with markdown parsing and incremental token rendering.
   - Interactive comparison drawer to shortlist and compare 2–4 shoes side-by-side.
   - Multimodal photo uploader with client-side canvas compression.
   - Shopper Settings modal for Bring-Your-Own-Key (BYOK) provider credentials (stored AES-encrypted).

2. **Application Gateway (FastAPI)**:
   - JWT authentication (`python-jose`), bcrypt password hashing, and login rate limiting.
   - `POST /api/chats/{chat_id}/message` immediately returns `202 Accepted` with a `job_id`.
   - `GET /api/jobs/{id}/events` serves real-time SSE event streams.
   - Fair per-user claiming and background lease-reaper to prevent zombie jobs under heavy load.

3. **Cognitive Agent Core (LangChain)**:
   - Tool routing dynamically selects the right tool based on semantic intention.
   - Multi-part requests (e.g. *"Show Nike running shoes under 3000 and what is your return policy?"*) execute sequentially across tools.
   - Single-request queries return verbatim tool results without unnecessary extra model hops.

4. **Data & State Layer**:
   - **Neon Cloud PostgreSQL**: Houses catalogue products, customer accounts, chats, job queues, carts, orders, and LLM response cache.
   - **Pinecone Vector Database**: Indexes store policy chunks using 1,024-dimensional Gemini vector embeddings for semantic retrieval.

5. **External AI Fabric & Resilience**:
   - **Google Gemini 2.5 Flash / Pro**: Primary agent routing, text-to-SQL, FAQ answering, vision extraction, and SQL fallback.
   - **Cloudflare Workers AI (`@cf/openai/gpt-oss-20b`)**: Alternative text provider selectable via `LLM_MODEL=CLOUDFLARE` or BYOK.
   - **Circuit Breaker Failover**: Automatically flips provider on repeated upstream failures.

---

## 🧭 Agent Tool Matrix

The agent router (`create_agent`) maps incoming shopper prompts to four deterministic tools:

| Tool | Module | Trigger Condition | Deterministic Guarantee |
|:---|:---|:---|:---|
| **Text-to-SQL** | `app/sql.py` | Price queries, brand filters, stock checks, sorting, "cheaper than X" | Read-only SQL execution; recommends only `stock > 0` items. |
| **FAQ RAG** | `app/faq.py` | Shipping timeline, returns, refunds, payment options, cancellations | Pure vector retrieval; zero hallucinated policies. |
| **Compare** | `app/compare.py` | *"Compare saved"*, *"Which of my shortlisted is best?"*, *"Save #2"* | Joins saved IDs against live database prices and specs. |
| **Cart & Orders** | `app/orders.py` | *"Add to cart"*, *"Place order"*, *"Cancel order #X"*, *"View cart"* | Atomic database transaction; checks remaining stock. |
| *(Direct Reply)* | — | Greetings, small talk, chit-chat, conversational follow-ups | Zero tool overhead; answers directly in conversation. |

---

## 🚀 Quickstart

### Option 1: Docker Compose (Recommended)

Run the full stack with one command:

```bash
# 1. Clone repository
git clone https://github.com/realranjan/cartmind-ai.git
cd cartmind-ai

# 2. Configure environment
cp backend/app/.env.example backend/app/.env
# Edit backend/app/.env with your Gemini API key and Neon/Pinecone credentials

# 3. Launch stack
docker compose up --build
```

- **Frontend**: [http://localhost:5173](http://localhost:5173)
- **Backend API**: [http://localhost:8000](http://localhost:8000)
- **API Documentation**: [http://localhost:8000/docs](http://localhost:8000/docs)

---

### Option 2: Local Development Setup

#### Backend Setup

```bash
cd backend
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

pip install -r requirements.txt
cp app/.env.example app/.env

# Run FastAPI backend with hot-reload
uvicorn main:app --reload --port 8000
```

#### Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

---

## ⚙️ Environment Variables

Configure these keys in `backend/app/.env`:

| Variable | Description | Default / Example |
|:---|:---|:---|
| `LLM_MODEL` | Default LLM provider | `GEMINI` (or `CLOUDFLARE`) |
| `GEMINI_API_KEY` | Google Gemini API key | `AIzaSy...` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@ep-xyz.neon.tech/cartmind_db?sslmode=require` |
| `PINECONE_API_KEY` | Pinecone vector DB key | `pcsk_...` |
| `PINECONE_INDEX_NAME` | Pinecone index name | `faq-index` |
| `PINECONE_HOST` | Pinecone host URL | `https://faq-index-xyz.svc.pinecone.io` |
| `JWT_SECRET` | Secret for signing auth tokens | Random 64-char hex string |
| `SUPERMEMORY_API_KEY` | Optional: user memory sync | `sm_...` |
| `CLOUDFLARE_ACCOUNT_ID` | Optional: Cloudflare account ID | `...` |
| `CLOUDFLARE_API_TOKEN` | Optional: Cloudflare Workers AI token | `...` |

---

## 📊 Evaluation & Verification Suite

CartMind includes an automated evaluation harness with 160 realistic shopper questions evaluated via DeepEval GEval:

```bash
# Run the evaluation benchmark
python backend/evals/run_eval_v2.py
```

### Benchmark Metric Highlights

| Evaluation Category | Target Standard | Observed Performance |
|:---|:---|:---|
| **Stock Faithfulness** | Only recommend in-stock SKUs (`stock > 0`) | **100%** |
| **Tool Routing Precision** | Route SQL vs. FAQ vs. Orders accurately | **98.2%** |
| **Photo Hallucination Defense** | Refuse non-shoe photos cleanly | **100%** |
| **PII & User Isolation** | Zero leaking of user tokens, credentials, or IDs | **100%** |

Run offline unit and regression tests without third-party network dependencies:

```bash
# Fast logging and observability tests
python tests/test_logging.py

# Offline agent regression checks
python tests/test_agent_regression_v1.py
```

---

## 📁 Repository Structure

```text
cartmind-ai/
├── assets/
│   └── readme/
│       ├── hero.svg            # Native visual hero banner & execution specimen
│       ├── architecture.svg    # 5-layer system & queue topology diagram
│       └── features.svg        # 4 core capability pillars
├── backend/
│   ├── app/
│   │   ├── db/                 # SQLAlchemy models and database session
│   │   ├── resources/          # Seed FAQ dataset and catalog CSVs
│   │   ├── agent.py            # LangChain routing agent definition
│   │   ├── api_keys.py         # Encrypted BYOK credential management
│   │   ├── compare.py          # Shortlist and saved items compare tool
│   │   ├── discover_products.py# Live Flipkart catalog crawler
│   │   ├── faq.py              # Pinecone vector RAG implementation
│   │   ├── jobs.py             # Postgres-backed queue & lease management
│   │   ├── llm_provider.py     # Gemini and Cloudflare provider abstraction
│   │   ├── memory.py           # Long-term context and query rewriting
│   │   ├── orders.py           # Transactional cart and order tool
│   │   ├── sql.py              # Text-to-SQL catalog querying tool
│   │   ├── vision.py           # Gemini 2.5 Multimodal shoe matcher
│   │   └── worker.py           # Background job consumer
│   ├── evals/                  # DeepEval 160-question benchmark suite
│   ├── main.py                 # FastAPI application and route endpoints
│   └── Dockerfile              # Production backend container
├── frontend/
│   ├── src/
│   │   ├── components/         # Chat, Auth, Landing, Compare, and Key modals
│   │   ├── App.jsx             # Root React component
│   │   └── api.js              # Axios and SSE client helpers
│   ├── index.html              # Main HTML entrypoint
│   └── Dockerfile              # Multi-stage NGINX static container
├── tests/                      # Unit and regression test suites
├── docker-compose.yml          # Local full-stack orchestration
└── LICENSE                     # MIT License
```

---

## 📄 License

Distributed under the MIT License. See [`LICENSE`](./LICENSE) for full terms.
