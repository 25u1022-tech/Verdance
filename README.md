# Prototype Verdance

> **Local-first Maintenance RAG.** Ask your machines questions. Get cited answers. Keep your data private.

Prototype Verdance is a free-tier, privacy-first Retrieval-Augmented Generation (RAG) system for manufacturing maintenance teams. It ingests machine manuals, failure logs, and SOPs, then answers natural-language questions with inline citations — and honestly says *"I don't know"* when the information isn't there.

Everything runs on developer hardware with **zero API costs** and **zero data egress** (when using local Ollama).

---

## Why Prototype Verdance?

Maintenance engineers waste 20–30 minutes per incident hunting through scattered PDFs to answer questions like *"Why did CNC-17 fail last time?"*

Existing tools are either too expensive, require sending sensitive factory data to the cloud, or can't prove where an answer came from.

**Prototype Verdance is different:**

| Concern | Our approach |
|---|---|
| **Cost** | Free-tier hosting + local models. No API bills. |
| **Privacy** | All inference local via Ollama. Manufacturing data never leaves your network. |
| **Trust** | Every answer cites its source chunk. Hallucinated citations are stripped. |
| **Honesty** | Explicit "I don't know" when confidence is low — no made-up answers on a factory floor. |

---

## Features (MVP Scope)

### ✅ Tier 1 — Vertical Slice
Create a machine → upload TXT/MD documents → ask a question → get a cited answer.

- Machine CRUD
- TXT/Markdown ingestion (chunk → embed → store)
- Local embeddings (`all-MiniLM-L6-v2`, 384-dim)
- Machine-scoped vector search (pgvector)
- Query with `[doc_id:chunk_idx]` citations
- "I don't know" handling for insufficient context
- Basic 3-page frontend (Machines, Upload, Chat)

### ✅ Tier 2 — MVP Complete
- PDF ingestion (manuals are PDFs)
- OpenRouter fallback when Ollama is unavailable

### ⏸️ Deferred to v1.1+
Auth, streaming/SSE, settings UI, citation hover tooltips, export, multi-turn context, ingestion history, landing page. See [ROADMAP.md](docs/ROADMAP.md) for the full plan.

---

## Architecture at a Glance

```
Browser (Next.js 14)
    │  HTTPS / REST
    ▼
FastAPI Backend
    ├── Machine Service     → CRUD
    ├── Ingestion Service   → chunk + embed + store
    ├── Retrieval Service   → pgvector cosine search
    └── Generation Service  → Qwen via Ollama / OpenRouter
    │
    ▼
PostgreSQL + pgvector   ←── local embeddings (MiniLM-L6-v2)
```

- **Backend:** FastAPI + SQLAlchemy 2.0 (async) + pgvector
- **Frontend:** Next.js 14 (App Router) + Tailwind + shadcn/ui + TanStack Query
- **Embeddings:** `sentence-transformers/all-MiniLM-L6-v2` (local, CPU)
- **LLM:** Ollama (local Qwen) primary, OpenRouter fallback
- **DB:** PostgreSQL 16 + pgvector (IVFFlat)

Full design: [ARCHITECTURE.md](docs/ARCHITECTURE.md)

---

## Quick Start

### Prerequisites
- Docker + Docker Compose
- Python 3.11+
- Node 20+
- Git

### 1. Clone & configure
```bash
git clone https://github.com/<you>/prototype-verdance.git
cd prototype-verdance
cp backend/.env.example backend/.env   # then edit
```

### 2. Choose your LLM (important — see sizing below)
```bash
ollama pull <model-sized-to-your-ram>
```

### 3. Start the stack
```bash
docker compose up -d postgres
cd backend && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

### 4. Verify
```bash
curl http://localhost:8000/api/health
```

---

## ⚠️ LLM Sizing (Read This First)

The generation model **must fit in your RAM**. A 27B model cannot run on 8GB under any usable quantization.

| Host RAM | Recommended runtime LLM | Notes |
|---|---|---|
| 8 GB | 7–8B class (q4) **or** OpenRouter | 27B will NOT fit |
| 16 GB | 27B at q3, or 14B at q4 | Tight |
| 24 GB+ | 27B at q4 | Comfortable |
| Any (no GPU) | OpenRouter fallback | No local VRAM needed |

> Embeddings always run locally (MiniLM, CPU). Only the **generation** LLM varies. Verify available Ollama tags with `ollama list`. Details in [PRD.md §7.1](docs/PRD.md).

---

## Project Structure

```
prototype-verdance/
├── docs/
│   ├── PRD.md              # Product requirements (v1.1)
│   ├── TRD.md              # Technical spec (v1.1)
│   ├── ARCHITECTURE.md     # System design (v1.1)
│   ├── USE_CASES.md        # Use cases (v1.1)
│   ├── ROADMAP.md          # Milestone build plan (v1.1)
│   └── CODER_RULES.md      # Rules for AI-assisted builds
├── backend/                # FastAPI + pgvector
│   ├── app/
│   │   ├── api/            # machines, ingest, query, health
│   │   ├── services/       # machine, ingestion, embedding, retrieval, generation
│   │   ├── models/         # SQLAlchemy models
│   │   ├── schemas/        # Pydantic schemas
│   │   └── utils/          # chunking, pdf_extract, citations
│   └── tests/
├── frontend/               # Next.js 14
│   └── src/
│       ├── app/            # machines, upload, chat
│       ├── components/
│       └── lib/
├── docker-compose.yml
└── README.md
```

---

## Documentation

| Document | Purpose |
|---|---|
| [PRD.md](docs/PRD.md) | Product requirements, personas, scope, KPIs |
| [TRD.md](docs/TRD.md) | API spec, DB schema, pipelines, testing, infra |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | System design, services, data flows, ADRs |
| [USE_CASES.md](docs/USE_CASES.md) | Detailed use cases, edge cases, acceptance tests |
| [ROADMAP.md](docs/ROADMAP.md) | Milestone-by-milestone build plan |
| [CODER_RULES.md](docs/CODER_RULES.md) | Guardrails for AI-assisted development |

---

## Development Workflow

This project is built **milestone by milestone** using a plan-first, branch-per-milestone discipline (see [ROADMAP.md](docs/ROADMAP.md)).

1. Each milestone lives on its own branch: `milestone/<n>-<slug>`
2. Changes are proposed as a PR, reviewed, and verified before merge
3. A milestone is done only when it **runs locally** and passes its verify command
4. Deferred features (v1.1+) are **not** built until explicitly requested

If you're an AI coding assistant working on this repo, **read [CODER_RULES.md](docs/CODER_RULES.md) first.**

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI, SQLAlchemy 2.0 (async), Pydantic v2 |
| Vector DB | PostgreSQL 16 + pgvector (IVFFlat) |
| Embeddings | sentence-transformers / all-MiniLM-L6-v2 |
| LLM | Ollama (local) + OpenRouter (fallback) |
| Frontend | Next.js 14, React 18, TypeScript, Tailwind, shadcn/ui |
| Data fetching | TanStack Query |
| Infra | Docker Compose, Vercel, Railway, Supabase (free tiers) |

---

## Status

🚧 **Active development** — building Tier 1 (vertical slice). See [ROADMAP.md](docs/ROADMAP.md) for current milestone.

---

## License

[Your license here — e.g., MIT]