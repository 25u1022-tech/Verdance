# ROADMAP — Prototype Verdance

**Project:** Prototype Verdance — Maintenance RAG MVP (Local-First, Free-Tier)
**Version:** 1.1
**Status:** Active build plan
**Related:** `PRD.md` (v1.1), `TRD.md` (v1.1), `ARCHITECTURE.md` (v1.1), `USE_CASES.md` (v1.1), `CODER_RULES.md`

## Change History
| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-16 | Initial roadmap |
| 1.1 | post-review | Rebrand to Prototype Verdance; align to v1.1 trimmed scope (Tier 1/Tier 2); add RAM-sizing gate; defer auth/streaming/settings; branch-per-milestone discipline |

---

## 1. Build Philosophy

> Build a **thin vertical slice** end-to-end first. Do **not** build all features at once.

Every milestone must produce something that **runs and can be verified locally** before moving on. No milestone is complete until it runs.

### Scope Tiers (from PRD v1.1 §3)
- **Tier 1 — Vertical Slice:** create machine → upload TXT/MD → ask question → get cited answer.
- **Tier 2 — MVP Complete:** PDF ingestion + OpenRouter fallback.
- **Deferred to v1.1+:** auth, streaming/SSE, settings UI, citation hover/export, multi-turn context, ingestion history, landing page.

**Rule:** Build Tier 1 fully, then Tier 2. Do not touch deferred features unless explicitly requested.

### Collaboration Model
| Role | Who | Responsibility |
|---|---|---|
| Architect | Lead assistant (chat) | Plans each milestone, reviews Qwen Coder output, decides next step |
| Hands | Qwen 3.8 Max Coder | Writes code into GitHub, one milestone branch at a time |
| Conductor | You | Approves each step, runs & verifies, merges PRs |

---

## 2. Hardware & Model Sizing Gate (DO THIS FIRST)

Before any LLM-dependent milestone, size the runtime model to RAM per **PRD v1.1 §7.1**:

| Host RAM | Runtime LLM | Notes |
|---|---|---|
| 8 GB | 7–8B class (q4) **or** OpenRouter | 27B will NOT fit |
| 16 GB | 27B at q3, or 14B at q4 | Tight |
| 24 GB+ | 27B at q4 | Comfortable |
| Any (no GPU) | OpenRouter fallback | No local VRAM needed |

> Embeddings always run locally via `all-MiniLM-L6-v2` (384-dim, CPU). Only the **generation** LLM varies. Verify actual Ollama tags with `ollama list`.

---

## 3. Milestone Overview

| # | Milestone | Tier | Layer | Depends On | Priority |
|---|---|---|---|---|---|
| 0 | Environment & model sizing | — | Setup | — | P0 |
| 1 | Repo + PostgreSQL + pgvector | T1 | Infra | 0 | P0 |
| 2 | Backend skeleton + health | T1 | Backend | 1 | P0 |
| 3 | Machine CRUD API | T1 | Backend | 2 | P0 |
| 4 | Local embedding service | T1 | Backend | 2 | P0 |
| 5 | TXT/MD ingestion | T1 | Backend | 3, 4 | P0 |
| 6 | Retrieval service | T1 | Backend | 4, 5 | P0 |
| 7 | Query API with citations (core RAG) | T1 | Backend | 6 | P0 |
| 8 | Insufficient-info handling | T1 | Backend | 7 | P0 |
| 9 | Basic frontend (3 pages, non-streaming) | T1 | Frontend | 3, 5, 7, 8 | P0 |
| 10 | PDF ingestion | T2 | Backend | 5 | P0 |
| 11 | OpenRouter fallback + provider abstraction | T2 | Backend | 7 | P0 |
| 12 | Testing + golden eval | T2 | QA | 7, 8 | P1 |
| 13 | Production deploy | post-MVP | Infra | all | P2 |

### Critical Path
```
0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9   (Tier 1 done)
                          ↘ 10 → 11 → 12     (Tier 2 done)
```
Milestones 10 & 11 can start once their dependencies are met. Everything after 12 is post-MVP.

---

## 4. Milestones (Detail)

### Milestone 0 — Environment & Model Sizing
**Goal:** Confirm toolchain; lock the runtime LLM per §2.
**Tasks:** Verify Docker, Python 3.11+, Node 20+, Git. Install Ollama and pull the RAM-appropriate model (or prepare OpenRouter key).
**Verify:**
```bash
docker --version && python3 --version && node --version && git --version
ollama list
```
**Done when:** all tools report versions; chosen model pulled or OpenRouter key ready.

### Milestone 1 — Repo Structure + PostgreSQL + pgvector
**Goal:** Database foundation per **TRD v1.1 §2**.
**Scope (ONLY):** folder skeleton (`prototype-verdance/`, `backend/`, `frontend/`, `docs/`), `docker-compose.yml` (pgvector/pgvector:pg16, DB `maintenance_rag`, port 5432, healthcheck), `init.sql` (extensions + `machines`, `documents`, `ingestion_jobs` + indexes).
**Out of scope:** backend code, frontend code, auth, ingestion logic.
**Verify:**
```bash
docker compose up -d postgres
docker compose exec postgres psql -U postgres -d maintenance_rag -c "\dx"   # vector present
docker compose exec postgres psql -U postgres -d maintenance_rag -c "\dt"   # 3 tables
```
**Qwen Coder prompt:**
```
Read CODER_RULES.md and docs/ (v1.1). Implement ONLY Milestone 1 from ROADMAP.md:
repo structure + PostgreSQL + pgvector. Create only docker-compose.yml and init.sql
(schema from TRD.md §2). Do NOT create backend/ or frontend/ code, auth, or ingestion.
List exact files first and wait for approval. Branch "milestone/1-db-setup", open a PR.
Then give commands to start the DB and verify the vector extension + tables.
```

### Milestone 2 — Backend Skeleton + Health
**Goal:** FastAPI boots and connects to the DB.
**Scope:** `backend/requirements.txt`, `app/main.py` (FastAPI + CORS), `app/config.py` (Pydantic Settings incl. `MAX_PDF_PAGES=50` and RAM-sized `OLLAMA_MODEL`), `app/database.py` (async engine/session), `GET /api/health` → `{status, db: connected}`.
**Out of scope:** machines, ingestion, query, embeddings, frontend.
**Verify:**
```bash
cd backend && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
curl http://localhost:8000/api/health
```
**Qwen Coder prompt:**
```
Implement ONLY Milestone 2: FastAPI skeleton with /api/health checking the DB
connection. Async SQLAlchemy 2.0 + Pydantic Settings per ARCHITECTURE.md §7.1.
Include MAX_PDF_PAGES and a RAM-sized OLLAMA_MODEL placeholder in config.
Do NOT implement machines, ingestion, query, or embeddings. List files first,
wait for approval, branch "milestone/2-backend-skeleton", open a PR, give run+verify.
```

### Milestone 3 — Machine CRUD API
**Goal:** Machine management per **PRD FR-4** / **USE_CASES UC-101, UC-102, UC-104**.
**Scope:** `models/machine.py`, `schemas/machine.py`, `services/machine.py`, `api/machines.py` — `POST`, `GET` (list), `GET /{id}`, `DELETE /{id}`; duplicate serial → 409, missing → 404; pytest for create/list/delete.
**Out of scope:** documents, ingestion, query.
**Verify:**
```bash
curl -X POST http://localhost:8000/api/machines -H "Content-Type: application/json" \
  -d '{"name":"CNC-17","serial_number":"DMG-2023-0042"}'
pytest backend/tests/test_machines.py -v
```

### Milestone 4 — Local Embedding Service
**Goal:** Text → 384-dim vectors locally (no API). Per **PRD FR-1.4**.
**Scope:** add `sentence-transformers`; `services/embedding.py` (load `all-MiniLM-L6-v2` once, batch embed, normalize); unit test asserting dim == 384.
**Out of scope:** chunking, ingestion endpoints, retrieval.
**Verify:** `pytest backend/tests/test_embedding.py -v`

### Milestone 5 — TXT/MD Ingestion (Vertical Slice)
**Goal:** Upload `.txt`/`.md`, chunk, embed, store. **Skip PDFs.** Per **PRD FR-1** / **USE_CASES UC-201, UC-203**.
**Scope:** `utils/chunking.py` (recursive, 500/50, TRD §4.2); `POST /api/ingest` (`.txt`, `.md`), `GET /api/ingest/{job_id}`; `services/ingestion.py` (create job → chunk → embed → batch insert → update job); store in `documents` with `machine_id`, `source_filename`, `metadata`.
**Out of scope:** PDF parsing, frontend, streaming.
**Verify:**
```bash
curl -X POST http://localhost:8000/api/ingest -F "machine_id=<uuid>" -F "files=@sample.txt"
curl http://localhost:8000/api/ingest/<job_id>
docker compose exec postgres psql -U postgres -d maintenance_rag -c "SELECT count(*) FROM documents;"
```

### Milestone 6 — Retrieval Service
**Goal:** Retrieve the right chunks for a question. Per **PRD FR-2**.
**Scope:** `services/retrieval.py` (embed query, cosine search, `machine_id` filter, threshold 0.3, top_k) using TRD §5.2 SQL; test proving machine-scoped isolation.
**Out of scope:** LLM calls, prompt building, query endpoint.
**Verify:** `pytest backend/tests/test_retrieval.py -v`

### Milestone 7 — Query API with Citations (Core RAG) ⭐
**Goal:** Answer a question with cited evidence. Per **PRD FR-2, FR-3** / **USE_CASES UC-301**.
**Scope:** `services/llm/client.py` (Ollama + OpenRouter abstraction, ARCHITECTURE §3.2); `services/generation.py` (prompt per TRD §5.3, call LLM, parse citations); `utils/citations.py` (extract `[doc_id:chunk_index]`, validate); `POST /api/query` returning `QueryResponse`; confidence = `min(1.0, avg_chunk_score × citation_coverage)`; temp=0, thinking off.
**Out of scope:** streaming, frontend, PDFs.
**Verify:**
```bash
curl -X POST http://localhost:8000/api/query -H "Content-Type: application/json" \
  -d '{"question":"What is the oil change interval?","machine_id":"<uuid>","top_k":5}'
```
Expect `answer` with `[doc_...:n]` citations and populated `citations[]`.

### Milestone 8 — Insufficient-Info Handling
**Goal:** Honest "I don't know". Per **PRD FR-3.4** / **USE_CASES UC-303**.
**Scope:** if no chunks > threshold → `insufficient_info:true`, `confidence:0.0`, populated `missing_info`; post-process guard against hallucination; golden + adversarial eval entries (TRD §8.3, §8.4).
**Verify:**
```bash
curl -X POST http://localhost:8000/api/query -H "Content-Type: application/json" \
  -d '{"question":"What is the warranty on CNC-17?","machine_id":"<uuid>"}'
# expect insufficient_info=true and missing_info mentioning "warranty"
```

### Milestone 9 — Basic Frontend (3 pages, non-streaming)
**Goal:** First usable UI. Create machine → upload text → ask → see cited answer. Per **PRD FR-5** (MVP pages).
**Scope:** Next.js 14 App Router + Tailwind + shadcn/ui + TanStack Query; `/machines` (list+create), `/upload` (dropzone), `/machines/[id]` (chat); **non-streaming** `POST /api/query`; render answer + citation list + confidence badge + insufficient banner; backend proxy route for CORS.
**Out of scope:** streaming, settings page, auth, landing page, citation hover.
**Verify:** create CNC-17 in UI → upload `.txt` → ask → see answer with citations.
> 🎉 End of Milestone 9 = **Tier 1 (vertical slice) complete.**

### Milestone 10 — PDF Ingestion (Tier 2)
**Goal:** Add PDF support. Per **PRD FR-1.2** / **USE_CASES UC-703**.
**Scope:** add `pdfplumber`; `utils/pdf_extract.py`; extract page text, store `page` in metadata; encrypted PDFs fail cleanly; scanned PDFs warn "no extractable text"; **>50 pages → truncate to first 50 with warning** (canonical v1.1 behavior).
**Verify:** upload a real PDF → chunks appear with page metadata.

### Milestone 11 — OpenRouter Fallback + Provider Abstraction (Tier 2)
**Goal:** Ollama ↔ OpenRouter fallback. Per **USE_CASES UC-705**, **ARCHITECTURE ADR-002**.
**Scope:** provider selection via env (`DEFAULT_PROVIDER`); auto-fallback to OpenRouter when Ollama unreachable (if key set); banner messaging. Config only — no settings UI (deferred).
**Verify:** stop Ollama → query succeeds via OpenRouter; restart Ollama → local restored.
> 🎉 End of Milestone 11 = **Tier 2 (MVP) complete.**

### Milestone 12 — Testing + Golden Eval
**Goal:** Prove correctness. Per **TRD v1.1 §8**.
**Scope:** unit + integration tests (`test_machines`, `test_ingestion`, `test_retrieval`, `test_generation`, `test_query`, `test_citations`, `test_health`); golden + adversarial eval; GitHub Actions CI (TRD §8.5).
**Verify:** `pytest` green; `ruff check . && mypy .` clean; CI passes.

### Milestone 13 — Production Deploy (post-MVP)
**Goal:** Ship free-tier stack per **TRD v1.1 §7.2**.
**Scope:** DB → Supabase; backend → Railway/Render; frontend → Vercel; LLM → local Ollama (Tailscale) or OpenRouter; env vars; smoke test.
**Verify:** full flow works on prod URLs; no secret leakage.

---

## 5. Deferred Features (v1.1+) — DO NOT BUILD YET

These are intentionally out of the MVP. Do not implement unless explicitly requested:
- Auth (NextAuth) — none for local MVP
- Streaming / SSE (`POST /api/query/stream`)
- Settings page UI (provider configured via env for MVP)
- Citation hover tooltips & answer export
- Multi-turn conversation context
- Ingestion history view
- Landing page
- Health/metrics dashboards

---

## 6. Review Checklist (before merging ANY PR)

| Check | What to look for |
|---|---|
| Scope | Only milestone-related files changed? |
| Tier | Is it a deferred (v1.1+) feature? If yes, reject. |
| Dependencies | Any unnecessary new packages? |
| Secrets | Any hardcoded API keys / passwords? |
| Runs | Does the verify command actually pass? |
| Tests | Are tests included where expected? |
| Diff size | Small and reviewable? |
| Docs | Did it modify PRD/TRD/ARCHITECTURE without permission? |

If the diff is too broad, reject with:
> "This is too large. Redo it with only the files in this milestone's scope."

---

## 7. Definition of Done (Trimmed MVP)

The MVP is complete when **all** are true (mirrors PRD v1.1 §9):
- [ ] `docker compose up` starts PostgreSQL + pgvector + backend + frontend
- [ ] Can create machine `CNC-17` via UI
- [ ] Can upload TXT/MD (Tier 1) and PDF (Tier 2) tagged to `CNC-17`
- [ ] A known question returns an answer with valid `[doc_xxx:y]` citations
- [ ] An unknown question returns `insufficient_info:true` with `missing_info`
- [ ] Machine-scoped search returns only that machine's chunks
- [ ] Provider switch Ollama → OpenRouter works
- [ ] Backend tests pass (`pytest`)
- [ ] Zero external network calls when using local Ollama

---

## 8. Maintenance
| Trigger | Action |
|---|---|
| Milestone completed | Check it off, update this file |
| New requirement | Add a milestone, update overview table |
| Blocker found | Note it in the relevant milestone section |
| Model/hardware change | Re-check §2 sizing table |
| Doc version bump | Verify milestones still reference correct v1.x sections |

**Owner:** You
**Review cadence:** Per milestone