# PRD: Prototype Verdance — Maintenance RAG MVP (Local-First, Free-Tier)

**Version:** 1.1
**Date:** 2026-08-16
**Status:** Draft for Review
**Owner:** Krishkara V Hotti
**Target Release:** Weekend MVP (Option A parallel track)

## Change History
| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-16 | Initial draft |
| 1.1 | post-review | Rebrand to Prototype Verdance; fix OOM/RAM feasibility; trim & phase scope; align auth/streaming decisions |

## 1. Problem Statement
Manufacturing engineers waste 20–30 minutes per query searching scattered PDFs (manuals, failure logs, SOPs, maintenance reports) to answer questions like *"Why did Machine 17 fail last time?"* Existing solutions (Glean, Notion AI, custom RAG) are either too expensive, require cloud data egress, or lack citation traceability for regulated environments.

**Opportunity:** **Prototype Verdance** is a local-first, free-tier RAG that ingests documents, retrieves relevant passages, generates answers with inline citations, and honestly admits when context is insufficient — all running on developer hardware with zero API costs.

## 2. Target Users & Personas
| Persona | Role | Pain Point | Success Metric |
|---|---|---|---|
| **Primary:** Maintenance Engineer | Fixes machines, reads manuals | "Finding the right procedure takes 30 min" | Answer in <30 sec with citation |
| **Secondary:** Plant Manager | Reviews failure trends | "No centralized failure knowledge" | Query failure patterns across machines |
| **Tertiary:** QA/Compliance | Audits maintenance records | "Can't trace answers to source docs" | Every answer cites source chunk |

## 3. MVP Scope (v1.1 — Trimmed & Phased)

Scope is split into two build phases so a working vertical slice ships first. Auth, streaming, and polish are deferred.

### Tier 1 — Vertical Slice (build first)
The smallest end-to-end path that proves the product works:
- Create machine, upload **TXT/MD** documents, ask a question, get a cited answer.

### Tier 2 — MVP Complete
Adds the features that make it genuinely useful:
- **PDF ingestion** (manuals are PDFs — core value)
- OpenRouter fallback + provider switch
- Full "I don't know" / insufficient-info UX
- Basic frontend pages

### In Scope ✅ (MVP)
- Upload PDF/TXT/MD (≤10MB) tagged to a machine
- Natural language query → answer with `[doc_id:chunk_idx]` citations
- Machine-scoped search (filter by `machine_id`)
- "I don't know" response when confidence < threshold
- Local-only: Ollama (Qwen model sized to RAM) + local embeddings + pgvector
- OpenRouter as optional fallback provider
- Deploy: Vercel (FE) + Railway/Render free tier (BE)

### Deferred to v1.1+ ⏸️ (explicitly NOT in MVP)
- **Auth** — none for local MVP; add NextAuth email magic link for prod only
- **Streaming (SSE)** — sync responses for MVP; add SSE later
- **Settings page UI** — provider/model configured via env for MVP
- Citation hover tooltips & answer export
- Multi-turn conversation context (single-turn for MVP)
- Ingestion history view
- Health/metrics dashboards

### Out of Scope ❌ (not planned for v1)
- Multi-tenancy / org isolation (single-tenant MVP)
- Real-time sync (watch folders, S3 events)
- Advanced reranking (cross-encoder, ColBERT)
- Agentic workflows (multi-step tool use)
- OCR for scanned PDFs (text-based only)
- Multi-language (English only v1)
- Kubernetes, Redis, message queues
- Paid vector DBs (Pinecone, Weaviate Cloud)
- Custom fine-tuning

## 4. Functional Requirements

### FR-1: Document Ingestion
| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | Accept multipart/form-data upload: TXT, MD (Tier 1), PDF (Tier 2) | P0 |
| FR-1.2 | Extract text via `pdfplumber` (PDF) / direct read (TXT/MD) | P0 |
| FR-1.3 | Chunk: 500 tokens, 50 overlap, recursive splitter | P0 |
| FR-1.4 | Embed chunks locally: `sentence-transformers/all-MiniLM-L6-v2` (384-dim) | P0 |
| FR-1.5 | Persist to PostgreSQL: `documents` table with `embedding vector(384)` | P0 |
| FR-1.6 | Associate chunks with `machine_id` (FK to `machines` table) | P0 |
| FR-1.7 | Return ingestion summary: `{chunks_created, chars_processed, time_ms}` | P1 |
| FR-1.8 | Files >50 pages: process first 50, set warning "Truncated to 50 pages" | P1 |

### FR-2: Query & Retrieval
| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | POST `/api/query` with `{question, machine_id?, top_k=5}` | P0 |
| FR-2.2 | Embed question with same local model | P0 |
| FR-2.3 | Vector search: cosine similarity, pgvector IVFFlat index | P0 |
| FR-2.4 | Filter by `machine_id` if provided (pre-filter) | P0 |
| FR-2.5 | Return top-k chunks with scores, snippets, metadata | P0 |
| FR-2.6 | POST `/api/query/stream` (SSE) — **deferred to v1.1** | P2 |

### FR-3: Answer Generation
| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | Build prompt: system + context chunks + user question | P0 |
| FR-3.2 | Call LLM (Ollama/OpenRouter): Qwen, thinking mode off | P0 |
| FR-3.3 | Enforce citation format: `[doc_<uuid>:<chunk_idx>]` | P0 |
| FR-3.4 | If confidence low (no relevant chunks > 0.3 score): return structured "insufficient info" response | P0 |
| FR-3.5 | Return: `{answer, citations[], confidence, latency_ms}` | P0 |

### FR-4: Machine Management
| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | CRUD `/api/machines` (name, serial, location, description) | P0 |
| FR-4.2 | List machines with document count | P1 |

### FR-5: Frontend Pages (MVP = 3 pages)
| Page | Route | Key Features | Priority |
|---|---|---|---|
| Machines | `/machines` | Table + create modal | P0 |
| Upload | `/upload` | Drag-drop, machine tag dropdown, progress | P0 |
| Chat | `/machines/[id]` | Non-streaming Q&A, citation list, insufficient-info banner | P0 |
| Landing | `/` | Hero + machine selector | P2 (deferred) |
| Settings | `/settings` | Provider toggle | P2 (deferred) |

## 5. Non-Functional Requirements
| Category | Requirement | Target |
|---|---|---|
| Performance | Ingestion (10MB PDF) | < 60 sec end-to-end |
| Performance | Query latency (p95) | < 3 sec (local LLM) / < 2 sec (OpenRouter) |
| Performance | Concurrent users | 5 (free tier) |
| Availability | Uptime | Best-effort for pilot (free tier; not a hard SLA) |
| Security | Data egress | Zero (Ollama) / Minimal (OpenRouter) |
| Security | Auth | None (local MVP) → NextAuth email (prod only) |
| Observability | Logging | Structured JSON (stdout) |
| Compliance | Citations | 100% answers cite source chunks |
| Compliance | Honesty | Explicit "I don't know" when confidence < 0.3 |

## 6. Success Metrics (KPIs)
| Metric | Target | Measurement |
|---|---|---|
| Time-to-Answer | < 30 sec | Median query latency |
| Citation Accuracy | 100% | Manual spot-check: every citation maps to real chunk |
| Hallucination Rate | < 2% | Adversarial eval: 50 "unknown" questions → "I don't know" |
| Ingestion Success | > 95% | Valid files → chunks stored |
| User Adoption | 3+ engineers | Weekly active users in pilot plant |

## 7. Risks & Mitigations
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Ollama OOM (model too big for RAM)** | High | High | **Size model to RAM (see §7.1).** On 8GB use a 7–8B model or OpenRouter. 27B requires 16GB+ (24GB for q4). |
| pgvector IVFFlat index not used | Low | Medium | `SET enable_seqscan = OFF`; verify `EXPLAIN ANALYZE` |
| LLM ignores citation instruction | Medium | High | Few-shot examples in prompt; post-process validation |
| Supabase free tier limits (500MB, 2GB RAM) | Medium | Medium | Monitor usage; purge old chunks; compress embeddings |
| PDF extraction fails on complex layouts | Medium | Low | Log failed pages; manual TXT fallback |

### 7.1 Local LLM RAM Sizing (CRITICAL)
The model must fit in available RAM. **A 27B model cannot run on 8GB RAM under any usable quant.**

| Model / Quant | Approx. Size | Minimum RAM | Notes |
|---|---|---|---|
| qwen3.8-27b-q3_k_m | ~12 GB | 16 GB | Tight; expect swapping |
| qwen3.8-27b-q4_k_m | ~16 GB | 24 GB | Recommended for 27B |
| 7–8B class (q4) | ~5 GB | 8 GB | Correct choice for 8GB machines |
| OpenRouter (any) | n/a | any | Required when local RAM is insufficient |

> **Rule:** 27B is the default **only** when ≥16 GB RAM is available. On 8GB machines, select a 7–8B local model or switch provider to OpenRouter. Verify actual Ollama model tags with `ollama list` before configuring.

## 8. Dependencies
| Dependency | Version | Purpose |
|---|---|---|
| Python | 3.11+ | Backend runtime |
| FastAPI | 0.110+ | API framework |
| SQLAlchemy | 2.0+ | Async ORM |
| pgvector (PostgreSQL extension) | 0.5+ | Vector index/search in Postgres |
| pgvector (Python client) | latest | SQLAlchemy vector type |
| sentence-transformers | 3.0+ | Local embeddings |
| pdfplumber | 0.11+ | PDF text extraction |
| ollama / openai | Latest | LLM client (local / OpenRouter) |
| Next.js | 14.2+ | Frontend framework |
| Tailwind CSS | 3.4+ | Styling |
| shadcn/ui | Latest | Component library |

## 9. Acceptance Criteria (Definition of Done)
- [ ] `docker compose up` starts PostgreSQL + pgvector + backend + frontend
- [ ] Create machine "CNC-17" via UI
- [ ] Upload TXT/MD (Tier 1) and PDF (Tier 2) tagged to CNC-17
- [ ] Ask: "Why did CNC-17 fail on 2024-03-15?"
- [ ] Answer cites ≥2 chunks with `[doc_xxx:y]` format
- [ ] Ask: "What's the warranty on CNC-17?" → Returns "I don't have enough information. Missing: warranty documentation."
- [ ] Switch provider Ollama → OpenRouter → answers still work
- [ ] All endpoints return structured JSON (no 500s on valid input)
- [ ] Zero external network calls when using Ollama (verified via `netstat`)
- [ ] Code passes `ruff check . && mypy . && pytest` (backend) / `npm run lint && tsc --noEmit` (frontend)