# USE_CASES.md — Prototype Verdance: Detailed Use Cases

**Version:** 1.1
**Date:** 2026-08-16
**Related:** PRD.md, TRD.md, ARCHITECTURE.md

## Change History
| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-16 | Initial draft |
| 1.1 | post-review | Rebrand; corrected OOM scenario (UC-701) to match RAM reality; standardized 50-page truncation; tagged deferred UCs; aligned test file names with TRD §13 |

## Overview
This document details each use case for **Prototype Verdance** (Maintenance RAG MVP), organized by persona and workflow. Each use case includes preconditions, steps, success criteria, edge cases, and acceptance tests.

## Persona Definitions
| Persona | Description | Technical Skill | Access Level |
|---|---|---|---|
| Maintenance Engineer (ME) | Hands-on technician fixing machines daily | Low-Medium | Read/Write (upload, query) |
| Plant Manager (PM) | Oversees multiple lines, reviews failure trends | Low | Read (query) |
| QA/Compliance Officer (QC) | Audits maintenance records, ensures traceability | Medium | Read (query, export) |
| System Admin (SA) | Deploys, configures, monitors the system | High | Admin |

## Use Case Categories
- UC-1xx: Machine Management
- UC-2xx: Document Ingestion
- UC-3xx: Query & Answer
- UC-4xx: Citation & Traceability
- UC-5xx: Settings & Configuration
- UC-6xx: Admin & Operations
- UC-7xx: Edge Cases & Error Handling

> **Status tags:** `[MVP]` = build now, `[v1.1]` = deferred.

---

## UC-100: Machine Management

### UC-101: Create Machine `[MVP]`
**Persona:** ME, PM | **Priority:** P0
**Description:** Register a new machine before uploading documents.

**Steps:**
1. User clicks "Add Machine"
2. Modal: Name (required), Serial Number (optional, unique), Location, Description
3. User fills form, clicks "Create"
4. Validate: name ≤100 chars, serial ≤100 unique, location ≤200, description ≤1000
5. Create machine record; UI refreshes table with document count = 0

**Success Criteria:** Machine appears <1s; serial uniqueness enforced (409); fields persisted.

**Edge Cases:**
- Empty name → 422 inline
- Duplicate serial → 409 "Serial number already registered"
- Network error → toast "Failed to create machine. Try again."

**Acceptance Test:**
```
Given I am on the machines page
When I create machine "CNC-17" with serial "DMG-2023-0042"
Then I see "CNC-17" in the table with 0 documents
And the database has a record in machines table
```

### UC-102: List Machines `[MVP]`
**Persona:** ME, PM, QC | **Priority:** P0
**Steps:** Navigate to `/machines` → `GET /api/machines?limit=50&offset=0` → table (Name, Serial, Location, Doc Count, Actions).

**Success Criteria:** Loads <500ms; counts accurate; pagination works.
**Edge Cases:** 0 machines → empty state CTA; 100+ → pagination.

### UC-103: View Machine Details `[v1.1]`
**Persona:** ME, PM, QC | **Priority:** P1
**Description:** See machine metadata and recent documents. Deferred to v1.1.

### UC-104: Delete Machine `[MVP]`
**Persona:** ME (with confirmation), SA | **Priority:** P1
**Steps:** Click delete → confirmation modal ("removes all N chunks, cannot be undone") → `DELETE /api/machines/{id}` → cascade deletes → toast.

**Success Criteria:** Cascade works (no orphans); confirmation prevents accidents.

---

## UC-200: Document Ingestion

### UC-201: Upload Single File `[MVP]`
**Persona:** ME | **Priority:** P0
**Description:** Upload a document (TXT/MD in Tier 1, PDF in Tier 2) and process it.

**Steps:**
1. Navigate to `/upload`, select machine "CNC-17"
2. Drop file (≤10MB)
3. Validate: extension, size, magic bytes
4. Create `IngestionJob` (pending), return `job_id`
5. Background: extract → chunk (500/50) → embed (batch 32) → batch insert → completed
6. UI polls `GET /api/ingest/{job_id}` every 2s

**Success Criteria:** Job created <1s; text extracted; chunks tagged correctly; embedding dim 384; page metadata present (PDF).

**Edge Cases:**
- Scanned PDF (no text) → 0 chunks, warning "No extractable text. Try OCR."
- Password-protected PDF → error "Cannot extract: PDF is encrypted"
- **>50-page PDF → process first 50, warning "Truncated to 50 pages"** *(canonical behavior)*
- >10MB → error "Max 10MB. Compress and re-upload."

**Acceptance Test:**
```
Given machine "CNC-17" exists
When I upload "manual_cnc17.pdf" (5MB, 30 pages)
Then ingestion job completes with status "completed"
And 47 documents exist in DB with machine_id=CNC-17
And each document has embedding vector(384)
And metadata includes page numbers 1-30
```

### UC-202: Upload Multiple Files `[MVP]`
**Persona:** ME | **Priority:** P0
**Steps:** Select 3 files → 1 job per file → process in parallel (max 4) → individual progress → "All files processed".

**Success Criteria:** 3 jobs complete; correct `source_filename`; total chunks = sum.
**Edge Cases:** One fails → partial success, failed file red; 10 files → queue, 4 parallel.

### UC-203: Upload Text/Markdown Files `[MVP]`
**Persona:** ME | **Priority:** P0
**Description:** Ingest non-PDF documents (the Tier 1 vertical-slice path).

**Steps:** Upload `failure_log.txt` → direct read (no pdfplumber) → chunk by paragraphs → embed/store.

**Success Criteria:** No pdfplumber needed; Markdown headers in metadata; logical chunk boundaries.

### UC-204: View Ingestion History `[v1.1]`
**Persona:** ME, QC | **Priority:** P1
**Description:** See all past uploads for a machine. Deferred to v1.1.

---

## UC-300: Query & Answer

### UC-301: Basic Question Answering `[MVP]`
**Persona:** ME | **Priority:** P0
**Description:** Ask a maintenance question, get a cited answer.

**Steps:**
1. Type: "What is the oil change interval for CNC-17?"
2. Enter → `POST /api/query` `{question, machine_id, top_k=5}`
3. System: embed → vector search (cosine, filter, threshold 0.3) → top 5 → build prompt → LLM (temp=0, thinking=off) → parse citations → confidence
4. Return `QueryResponse`; UI renders answer + citation list + confidence badge

**Example Response:**
```json
{
  "answer": "The oil change interval for CNC-17 is every 500 operating hours or 6 months, whichever comes first [doc_a1b2c3d4:2]. Use ISO VG 68 hydraulic oil [doc_a1b2c3d4:3].",
  "citations": [
    {"document_id": "a1b2c3d4", "chunk_index": 2, "snippet": "Oil change: every 500 hours...", "score": 0.87},
    {"document_id": "a1b2c3d4", "chunk_index": 3, "snippet": "Recommended oil: ISO VG 68...", "score": 0.82}
  ],
  "confidence": 0.85,
  "latency_ms": 1240,
  "model": "qwen3.8-27b",
  "insufficient_info": false
}
```

**Success Criteria:** ≥1 relevant citation; citations match retrieved chunks; confidence >0.7 for known info; latency <3s (local)/<2s (OpenRouter).

### UC-302: Machine-Scoped vs Global Search `[MVP]`
**Persona:** ME, PM | **Priority:** P0
**Steps:** Machine chat page → includes `machine_id`; global search → omits it.

**Success Criteria:** Machine page = that machine only; citations identify source.

### UC-303: Insufficient Information Handling `[MVP]`
**Persona:** ME, QC | **Priority:** P0
**Description:** System honestly admits when it doesn't know.

**Steps:** Ask "What is the warranty period for CNC-17?" → no chunks >0.3 → empty context → structured insufficient response.

**Expected Response:**
```json
{
  "answer": "I don't have enough information to answer. Missing: warranty documentation for CNC-17.",
  "citations": [],
  "confidence": 0.0,
  "latency_ms": 450,
  "model": "qwen3.8-27b",
  "insufficient_info": true,
  "missing_info": "warranty documentation for CNC-17"
}
```

**UI:** Yellow warning banner "⚠️ Insufficient information"; italic answer; suggested action "Upload warranty documents".

**Success Criteria:** `insufficient_info:true` when no relevant chunks; `missing_info` populated; no hallucination; confidence 0.0.

### UC-304: Follow-up Questions `[v1.1]`
**Persona:** ME | **Priority:** P1
**Description:** Multi-turn conversation context. Deferred to v1.1 (MVP is single-turn).

### UC-305: Streaming Response `[v1.1]`
**Persona:** ME | **Priority:** P1
**Description:** Token-by-token SSE answers via `POST /api/query/stream`. Deferred to v1.1 (MVP is sync).

---

## UC-400: Citation & Traceability

### UC-401: Citation Display `[MVP]`
**Persona:** ME, QC | **Priority:** P0
**Description:** Show the list of citations (document, chunk, snippet, score) under each answer.

**Success Criteria:** Citations render with snippet + score; match retrieved chunks.

### UC-401b: Citation Hover Preview `[v1.1]`
**Persona:** ME, QC | **Priority:** P1
**Description:** Hover a citation to see page/section/snippet tooltip. Deferred to v1.1.

### UC-402: Citation Validation `[MVP]`
**Persona:** System (automatic) | **Priority:** P0
**Steps:** Parse `[doc_xxx:y]` patterns → verify each exists in retrieved chunks → strip mismatches, reduce confidence → `confidence = min(1.0, avg_chunk_score × citation_coverage)`.

**Success Criteria:** 100% validated; hallucinated citations removed; confidence reflects quality.

### UC-403: Export Answer with Citations `[v1.1]`
**Persona:** QC | **Priority:** P2
**Description:** Export Q&A for audit trail. Deferred to v1.1.

---

## UC-500: Settings & Configuration

### UC-501: Switch LLM Provider `[v1.1]`
**Persona:** ME, SA | **Priority:** P1 (MVP uses env config)
**Description:** Toggle Ollama ↔ OpenRouter via `/settings` UI. For MVP, provider/model are set via environment variables; the UI toggle is deferred to v1.1.

### UC-502: Configure Model Parameters `[v1.1]`
**Persona:** SA | **Priority:** P2
**Description:** Adjust temperature, top_k, max_tokens. Deferred to v1.1.

### UC-503: View System Health `[v1.1]`
**Persona:** SA | **Priority:** P1
**Description:** Monitor backend, database, LLM status. Deferred to v1.1.

---

## UC-600: Admin & Operations

### UC-601: Deploy to Production `[v1.1]`
**Persona:** SA | **Priority:** P1
**Description:** Deploy FE to Vercel, BE to Railway, DB to Supabase. Post-MVP.

### UC-602: Monitor Free Tier Usage `[v1.1]`
**Persona:** SA | **Priority:** P1

| Platform | Metric | Free Limit | Alert At |
|---|---|---|---|
| Supabase | DB Size | 500 MB | 400 MB |
| Supabase | RAM | 2 GB | 1.5 GB |
| Railway | RAM | 512 MB | 400 MB |
| Vercel | Bandwidth | 100 GB | 80 GB |
| OpenRouter | Credits | $1/day | $0.50 |

### UC-603: Backup & Restore `[v1.1]`
**Persona:** SA | **Priority:** P2

---

## UC-700: Edge Cases & Error Handling

### UC-701: Local LLM Memory Exhaustion (OOM) `[MVP]`
**Persona:** System | **Priority:** P0
**Description:** Handle insufficient RAM for the selected model. *(Corrected in v1.1.)*

**Reality:** `qwen3.8-27b-q4_k_m` ≈ 16 GB and **cannot** load on an 8 GB host.

**Behavior:**
1. Backend health check detects Ollama 500/timeout
2. Retry once with smaller context
3. If persistent → 503: "Local model unavailable. Reduce model size or switch to OpenRouter in Settings."
4. Increment `rag_llm_errors_total`

**Correct guidance (PRD §7.1):**
| Host RAM | Recommended model |
|---|---|
| 8 GB | 7–8B class q4, or OpenRouter fallback |
| 16 GB | qwen3.8-27b-q3_k_m (tight) |
| 24 GB+ | qwen3.8-27b-q4_k_m |

### UC-702: pgvector Index Not Used `[MVP]`
**Persona:** System | **Priority:** P1
**Detection:** `EXPLAIN ANALYZE` shows `Seq Scan`.
**Behavior:** If seq_scan >10% → warn, suggest `ANALYZE documents`; admin can `REINDEX`.

### UC-703: PDF Extraction Failures `[MVP]`
**Persona:** ME | **Priority:** P1

| Failure Type | Behavior |
|---|---|
| Encrypted | Skip, log "PDF encrypted: manual.pdf", job status=failed |
| Scanned (no text) | Extract 0 chunks, warn "No extractable text. Use OCR tool first." |
| Corrupted | Skip, log error, continue other files |
| >50 pages | Process first 50, warn "Truncated to 50 pages" *(canonical behavior)* |

### UC-704: Concurrent Query Load `[MVP]`
**Persona:** System | **Priority:** P1
**Behavior:** Railway queues requests; Ollama sequential inference; 503 + retry-after when overloaded; frontend exponential backoff (max 3).

### UC-705: Network Partition (Ollama Unreachable) `[MVP]`
**Persona:** System | **Priority:** P0
**Behavior:** Health check fails (timeout 5s) → auto-fallback to OpenRouter if key set → banner "Local model unreachable. Using OpenRouter." → on recovery "Local model restored."

### UC-706: Malicious Query Injection `[MVP]`
**Persona:** Attacker | **Priority:** P1
**Mitigations:** Pydantic validation (max 2000 chars, no control chars); chunk content sanitized; system prompt "Answer ONLY from context"; post-process strips non-citation markdown.

---

## Use Case Traceability Matrix
| UC ID | PRD FR | TRD Section | API Endpoint | Test File |
|---|---|---|---|---|
| UC-101 | FR-4.1 | 3.2 | POST /api/machines | test_machines.py |
| UC-102 | FR-4.2 | 3.2 | GET /api/machines | test_machines.py |
| UC-104 | FR-4.1 | 3.2 | DELETE /api/machines/{id} | test_machines.py |
| UC-201 | FR-1.1-1.8 | 4.1-4.5 | POST /api/ingest | test_ingestion.py |
| UC-202 | FR-1.1-1.8 | 4.1-4.5 | POST /api/ingest (multi) | test_ingestion.py |
| UC-203 | FR-1.1-1.8 | 4.1-4.5 | POST /api/ingest | test_ingestion.py |
| UC-301 | FR-2.1-2.5, FR-3.1-3.5 | 5.1-5.5 | POST /api/query | test_query.py |
| UC-302 | FR-2.4 | 5.2 | POST /api/query | test_query.py |
| UC-303 | FR-3.4 | 5.3-5.5 | POST /api/query | test_query.py + eval/golden.jsonl |
| UC-305 | FR-2.6 | 3.2, 4.3 | POST /api/query/stream | test_streaming.py (v1.1) |
| UC-401 | FR-3.3 | 5.5 | - | test_citations.py |
| UC-402 | FR-3.3 | 5.5 | - | test_citations.py |
| UC-503 | - | 9.2 | GET /api/health | test_health.py |
| UC-701 | Risk #1 | 8, 10 | - | chaos test |
| UC-702 | Risk #2 | 5, 10 | - | perf test |
| UC-703 | Risk #5 | 4.5 | POST /api/ingest | test_ingestion.py |
| UC-705 | - | 11 | - | integration test |
| UC-706 | Security | 10 | - | security test |

---

## Acceptance Test Scenarios (Golden Path)

### Scenario 1: First-Time User Success
```gherkin
Feature: New user sets up and queries in 10 minutes
Background:
  - Docker Compose running
  - Ollama model pulled for host RAM (PRD §7.1)
Scenario: Complete workflow
  Given I open http://localhost:3000
  When I create machine "CNC-17" with serial "DMG-2023-0042"
  And I upload "manual.pdf" (5MB)
  And I upload "failure_log.txt"
  And I wait for ingestion to complete
  And I ask "Why did CNC-17 fail on 2024-03-15?"
  Then I get an answer citing failure_log.txt chunks
  And confidence > 0.7
  And I ask "What is the warranty?"
  Then I get "I don't have enough information. Missing: warranty documentation."
```

### Scenario 2: Provider Failover
```gherkin
Scenario: Ollama down → OpenRouter works
  Given Ollama is stopped
  And OpenRouter API key configured
  When I ask a question
  Then answer comes from OpenRouter
  And banner shows "Using OpenRouter"
  When I start Ollama
  Then banner shows "Local model restored"
```

### Scenario 3: Multi-Machine Isolation
```gherkin
Scenario: Query only searches selected machine
  Given machine "CNC-17" has manual mentioning "ISO VG 68"
  And machine "LATHE-42" has manual mentioning "ISO VG 32"
  When I query on CNC-17 page "What oil?"
  Then answer cites "ISO VG 68"
  And no mention of "ISO VG 32"
```

---

## Out-of-Scope Use Cases (Future)
| UC | Description | Phase |
|---|---|---|
| UC-801 | OCR for scanned PDFs | v2 |
| UC-802 | Cross-encoder reranking | v2 |
| UC-803 | Multi-tenancy (orgs, RBAC) | v2 |
| UC-804 | Scheduled folder watch (S3, network share) | v2 |
| UC-805 | Agentic: "Create work order from this failure" | v3 |
| UC-806 | Multi-language (Spanish, German manuals) | v3 |
| UC-807 | Table extraction from PDFs | v3 |
| UC-808 | Voice query (speech-to-text) | v3 |
| UC-809 | Mobile app (offline sync) | v3 |
| UC-810 | Integration: CMMS (Maximo, SAP PM) | v3 |

## Maintenance & Updates
| Trigger | Action |
|---|---|
| New PRD requirement | Add UC, update traceability matrix |
| Bug found in prod | Add edge case UC-7xx |
| Performance issue | Add UC-7xx with mitigation |
| New model released | Update UC-501, UC-701 |
| Free tier limits change | Update UC-602 |

**Document Owner:** [Your Name]
**Review Cadence:** Per sprint / per release