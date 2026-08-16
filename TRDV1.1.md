# TRD: Prototype Verdance — Maintenance RAG MVP — Technical Specification

**Version:** 1.1
**Date:** 2026-08-16
**Status:** Draft for Review
**Related:** PRD.md, ARCHITECTURE.md

## Change History
| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-16 | Initial draft |
| 1.1 | post-review | Rebrand; add `/api/query/stream` (tagged v1.1); standardize 50-page truncation; align test files with file tree; fix YAML comment; defer auth/streaming; LLM RAM guidance |

## 1. System Overview
```
┌─────────────┐     HTTPS      ┌─────────────┐     TCP/5432     ┌──────────────────┐
│   Browser   │ ◄─────────────► │  Next.js    │ ◄──────────────► │  PostgreSQL      │
│  (React 18) │                 │  (Vercel)   │                  │  + pgvector      │
└─────────────┘                 └──────┬──────┘                  │  (Supabase)      │
                                       │                         └──────────────────┘
                                       │ HTTPS
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
            ┌─────────────┐                       ┌─────────────┐
            │  FastAPI    │                       │   Ollama    │
            │  (Railway)  │                       │  (Local)    │
            └──────┬──────┘                       └─────────────┘
                   │ TCP/11434
                   ▼
            ┌─────────────┐
            │ sentence-   │
            │ transformers│
            │  (Local)    │
            └─────────────┘
```
**Key Principle:** All ML inference local (embeddings + LLM) or free-tier API. Zero paid services.

## 2. Database Schema (PostgreSQL + pgvector)

### 2.1 Extensions
```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 2.2 Tables

**`machines`**
```sql
CREATE TABLE machines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,           -- "CNC-17"
    serial_number   TEXT UNIQUE,             -- "DMG-2023-0042"
    location        TEXT,                    -- "Building A, Line 3"
    description     TEXT,                    -- "5-axis milling, Siemens 840D"
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_machines_name ON machines(name);
```

**`documents` (Chunks)**
```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    machine_id      UUID NOT NULL REFERENCES machines(id) ON DELETE CASCADE,
    source_filename TEXT NOT NULL,           -- "manual_cnc17.pdf"
    chunk_index     INT NOT NULL,            -- 0, 1, 2...
    content         TEXT NOT NULL,           -- Chunk text (≤500 tokens)
    embedding       VECTOR(384) NOT NULL,    -- MiniLM-L6-v2
    metadata        JSONB DEFAULT '{}',      -- {page: 5, section: "safety"}
    token_count     INT NOT NULL,            -- For cost estimation
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- IVFFlat index for ANN search (build after ~10k rows)
CREATE INDEX idx_documents_embedding
    ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
CREATE INDEX idx_documents_machine_id ON documents(machine_id);
CREATE INDEX idx_documents_source ON documents(source_filename);
```

**`ingestion_jobs` (Audit/Resume)**
```sql
CREATE TABLE ingestion_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    machine_id      UUID NOT NULL REFERENCES machines(id),
    source_filename TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    total_chunks    INT DEFAULT 0,
    processed_chunks INT DEFAULT 0,
    error_message   TEXT,
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
```

### 2.3 Row-Level Security (Future Multi-Tenancy)
```sql
-- Not enabled in MVP; schema supports it
ALTER TABLE machines ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```

## 3. API Specification (OpenAPI 3.1)

### 3.1 Base
- Base URL: `http://localhost:8000` (dev) / `https://api.yourdomain.com` (prod)
- Auth: **none for local MVP**; Bearer token (NextAuth.js JWT) for prod only
- Content-Type: `application/json` (except upload: `multipart/form-data`)

### 3.2 Endpoints

**`POST /api/machines`**
```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        type: object
        required: [name]
        properties:
          name: {type: string, maxLength: 100}
          serial_number: {type: string, maxLength: 100}
          location: {type: string, maxLength: 200}
          description: {type: string, maxLength: 1000}
responses:
  '201': {description: Created, content: {application/json: {schema: {$ref: '#/components/schemas/Machine'}}}}
  '400': {description: Validation error}
  '409': {description: Serial number exists}
```

**`GET /api/machines`**
```yaml
parameters:
  - name: limit; in: query; schema: {type: integer, default: 50, maximum: 100}
  - name: offset; in: query; schema: {type: integer, default: 0}
responses:
  '200': {description: List, content: {application/json: {schema: {type: array, items: {$ref: '#/components/schemas/MachineWithDocCount'}}}}}
```

**`GET /api/machines/{id}`**
```yaml
responses:
  '200': {description: Machine with recent documents}
  '404': {description: Not found}
```

**`DELETE /api/machines/{id}`**
```yaml
responses:
  '204': {description: Deleted (cascades to documents)}
  '404': {description: Not found}
```

**`POST /api/ingest`**
```yaml
requestBody:
  required: true
  content:
    multipart/form-data:
      schema:
        type: object
        required: [machine_id, files]
        properties:
          machine_id: {type: string, format: uuid}
          files:
            type: array
            items: {type: string, format: binary}
            maxItems: 10
responses:
  '202': {description: Accepted, content: {application/json: {schema: {$ref: '#/components/schemas/IngestionJob'}}}}
  '400': {description: Invalid file type / size > 10MB}
  '404': {description: Machine not found}
```

**`GET /api/ingest/{job_id}`**
```yaml
responses:
  '200': {description: Job status with progress}
  '404': {description: Not found}
```

**`POST /api/query`**
```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        type: object
        required: [question]
        properties:
          question: {type: string, minLength: 3, maxLength: 2000}
          machine_id: {type: string, format: uuid}
          top_k: {type: integer, default: 5, minimum: 1, maximum: 20}
          temperature: {type: number, default: 0.0, minimum: 0.0, maximum: 1.0}
responses:
  '200': {description: Answer, content: {application/json: {schema: {$ref: '#/components/schemas/QueryResponse'}}}}
  '400': {description: Invalid input}
  '503': {description: LLM unavailable}
```

**`POST /api/query/stream`** *(v1.1 — deferred, SSE variant of /api/query)*
```yaml
description: SSE streaming variant. Same request body as /api/query. Deferred to v1.1.
responses:
  '200':
    description: Server-Sent Events stream
    content:
      text/event-stream:
        schema:
          type: string
          description: |
            Incremental events: data: {"delta": "..."}
            Final event: data: {"citations": [...], "confidence": 0.85, "insufficient_info": false}
  '503': {description: LLM unavailable}
```

### 3.3 Response Schemas
```yaml
components:
  schemas:
    Machine:
      type: object
      properties:
        id: {type: string, format: uuid}
        name: {type: string}
        serial_number: {type: string, nullable: true}
        location: {type: string, nullable: true}
        description: {type: string, nullable: true}
        created_at: {type: string, format: date-time}
        updated_at: {type: string, format: date-time}
    MachineWithDocCount:
      allOf:
        - $ref: '#/components/schemas/Machine'
        - type: object
          properties:
            document_count: {type: integer}
            last_ingested_at: {type: string, format: date-time, nullable: true}
    IngestionJob:
      type: object
      properties:
        id: {type: string, format: uuid}
        machine_id: {type: string, format: uuid}
        source_filename: {type: string}
        status: {type: string, enum: [pending, processing, completed, failed]}
        total_chunks: {type: integer}
        processed_chunks: {type: integer}
        error_message: {type: string, nullable: true}
        started_at: {type: string, format: date-time}
        completed_at: {type: string, format: date-time, nullable: true}
    Citation:
      type: object
      properties:
        document_id: {type: string, format: uuid}
        chunk_index: {type: integer}
        snippet: {type: string, maxLength: 200}
        score: {type: number, format: float}
        metadata: {type: object}
    QueryRequest:
      type: object
      required: [question]
      properties:
        question: {type: string, minLength: 3, maxLength: 2000}
        machine_id: {type: string, format: uuid}
        top_k: {type: integer, default: 5, minimum: 1, maximum: 20}
        temperature: {type: number, default: 0.0, minimum: 0.0, maximum: 1.0}
    QueryResponse:
      type: object
      properties:
        answer: {type: string}
        citations:
          type: array
          items: {$ref: '#/components/schemas/Citation'}
        confidence: {type: number, format: float, minimum: 0, maximum: 1}
        latency_ms: {type: integer}
        model: {type: string}
        insufficient_info: {type: boolean}
        missing_info: {type: string, nullable: true}
```
> `missing_info` is populated only when `insufficient_info=true`.

## 4. Ingestion Pipeline (Backend)

### 4.1 Flow
```
Upload → Validate → Create Job → Extract Text (truncate >50 pages) → Chunk → Embed → Batch Insert → Update Job → Done
```

### 4.2 Chunking Strategy
```yaml
chunking:
  strategy: "recursive"
  chunk_size: 500        # tokens (approx 350 words)
  chunk_overlap: 50      # tokens
  separators: ["\n\n", "\n", ". ", " ", ""]
  min_chunk_size: 50     # tokens
```

### 4.3 Embedding
```python
# Local only — no API calls
model_name = "sentence-transformers/all-MiniLM-L6-v2"
dimension = 384
batch_size = 32
device = "cpu"  # or "cuda" if available
normalize_embeddings = True  # For cosine similarity
```

### 4.4 Batch Insert (pgvector)
```sql
-- Use executemany for speed
INSERT INTO documents (machine_id, source_filename, chunk_index, content, embedding, metadata, token_count)
VALUES %s
ON CONFLICT DO NOTHING
```

### 4.5 Error Handling
| Failure Point | Retry | Fallback |
|---|---|---|
| PDF extraction | 2x | Skip page, log warning |
| PDF > 50 pages | n/a | Truncate to first 50, set warning |
| Embedding OOM | 1x (smaller batch) | CPU offload |
| DB insert deadlock | 3x (exponential backoff) | Fail job, alert |

## 5. Query Pipeline (Backend)

### 5.1 Flow
```
Query → Embed → Vector Search → Filter → Build Prompt → LLM → Parse Citations → Return (sync; SSE in v1.1)
```

### 5.2 Vector Search SQL
```sql
SELECT id, machine_id, source_filename, chunk_index, content, metadata,
       1 - (embedding <=> $1) AS similarity  -- cosine distance
FROM documents
WHERE machine_id = $2  -- optional
  AND 1 - (embedding <=> $1) > 0.3           -- similarity threshold
ORDER BY embedding <=> $1
LIMIT $3;
```

### 5.3 Prompt Template
```python
SYSTEM_PROMPT = """You are a maintenance engineer assistant. Answer using ONLY the provided context.
Cite sources inline like [doc_<uuid>:<chunk_idx>].
If context is insufficient, respond EXACTLY:
"I don't have enough information to answer. Missing: [specific missing info]."
"""

USER_TEMPLATE = """Context chunks (score ≥ 0.3):
{context}
Question: {question}
Answer:"""

# Context format per chunk:
# [doc_{id}:{chunk_idx}] (score: {similarity:.3f}) {content[:500]}
```

### 5.4 LLM Configuration
```yaml
# Ollama (local) — SIZE MODEL TO RAM (see PRD §7.1)
ollama:
  model: "qwen3.8-27b"          # requires >=16GB RAM; use 7-8B on 8GB hosts
  base_url: "http://localhost:11434/v1"
  temperature: 0.0
  top_p: 0.9
  max_tokens: 2048
  extra_body:
    chat_template_kwargs:
      enable_thinking: false

# OpenRouter (fallback)
openrouter:
  model: "qwen/qwen3.8-27b"
  base_url: "https://openrouter.ai/api/v1"
  api_key: "${OPENROUTER_API_KEY}"
  temperature: 0.0
```

### 5.5 Citation Parsing & Validation
```python
# Post-process: extract [doc_xxx:y] patterns, verify they exist in retrieved chunks
# If LLM cites non-retrieved doc → strip citation, lower confidence
# Confidence = min(1.0, avg_chunk_score * citation_coverage)
```

## 6. Frontend Architecture (Next.js 14 App Router)

### 6.1 Route Structure (MVP = 3 pages)
```
app/
 ├── layout.tsx              # Providers: Theme, Tooltip
 ├── machines/
 │   ├── page.tsx            # List table + create modal
 │   └── [id]/
 │       └── page.tsx        # Chat interface (non-streaming for MVP)
 ├── upload/
 │   └── page.tsx            # Dropzone + machine select + progress
 ├── api/
 │   └── proxy/              # Backend proxy (avoid CORS)
 └── globals.css             # Tailwind + shadcn/ui vars
```
> Deferred to v1.1: `/` landing, `/settings`, and `machines/[id]/chat/route.ts` (SSE Edge route).

### 6.2 Key Components
| Component | Location | Purpose |
|---|---|---|
| MachineSelector | components/ui/ | Dropdown with search, shows doc count |
| ChatInterface | components/chat/ | Message list, input, citation list, insufficient banner |
| UploadDropzone | components/upload/ | React-dropzone, progress, file validation |

### 6.3 State Management
- Server State: TanStack Query (React Query) — caching, invalidation
- Chat History: `localStorage` (per machine, max 50 messages)
- Settings: env-based for MVP (UI deferred to v1.1)

## 7. Infrastructure & Deployment

### 7.1 Docker Compose (Local Dev)
```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: maintenance_rag
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/maintenance_rag
      OLLAMA_BASE_URL: http://host.docker.internal:11434/v1
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./backend:/app  # Hot reload

  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000
    depends_on: [backend]
    volumes:
      - ./frontend:/app
      - /app/node_modules

  ollama:
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes:
      - ollama_data:/root/.ollama
    command: ["serve"]

volumes:
  pgdata:
  ollama_data:
```
> Pull an Ollama model sized to host RAM (PRD §7.1). Production compose file is post-MVP.

### 7.2 Production (Free Tier)
| Service | Platform | Config |
|---|---|---|
| PostgreSQL | Supabase | Free tier: 500MB DB, 2GB RAM, pgvector enabled |
| Backend | Railway / Render | Free tier: 512MB RAM, 0.5 vCPU, custom domain |
| Frontend | Vercel | Hobby: 100GB bandwidth |
| Ollama | Local / Tailscale | Run on dev machine, expose via Tailscale Funnel |

### 7.3 Environment Variables
```bash
# Backend (.env)
DATABASE_URL=postgresql://user:pass@host:5432/db
OLLAMA_BASE_URL=http://localhost:11434/v1
OLLAMA_MODEL=<model-sized-to-ram>
OPENROUTER_API_KEY=sk-or-...
EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
CHUNK_SIZE=500
CHUNK_OVERLAP=50
SIMILARITY_THRESHOLD=0.3
DEFAULT_TOP_K=5
LOG_LEVEL=INFO

# Frontend (.env.local)
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
```

## 8. Testing Strategy

### 8.1 Unit Tests (Backend)
```python
# tests/test_ingestion.py
def test_chunking_recursive():
    text = "Section 1\n\nParagraph 1. Paragraph 2.\n\nSection 2\n\nParagraph 3."
    chunks = chunk_text(text, chunk_size=50, overlap=10)
    assert len(chunks) >= 2
    assert all(len(c) <= 50 for c in chunks)

def test_embedding_dimension():
    emb = embed_texts(["test"])
    assert emb.shape == (1, 384)

def test_vector_search_returns_correct_machine():
    # Insert chunks for machine A and B
    # Query with machine_id=A → only A's chunks returned
```

### 8.2 Integration Tests
```python
# tests/test_api.py
async def test_ingest_and_query_flow(async_client):
    machine = await create_machine(async_client, "Test-CNC")
    job = await ingest_file(async_client, machine.id, "sample.txt")
    assert job.status == "completed"
    resp = await query(async_client, "What is the maintenance interval?", machine.id)
    assert resp.confidence > 0
    assert len(resp.citations) > 0
```

### 8.3 Golden Evaluation Set
```jsonl
{"question": "What is the oil change interval for CNC-17?", "machine_id": "uuid-1", "expected_citations_min": 1, "must_contain": ["500 hours", "ISO VG 68"]}
{"question": "Why did CNC-17 fail on 2024-03-15?", "machine_id": "uuid-1", "expected_citations_min": 2, "must_contain": ["bearing", "overheating"]}
{"question": "What is the warranty on CNC-17?", "machine_id": "uuid-1", "expected_insufficient": true, "missing_info_contains": "warranty"}
```

### 8.4 Adversarial Evaluation
```jsonl
{"question": "What color is the CNC-17?", "machine_id": "uuid-1", "expected_insufficient": true}
{"question": "Tell me the nuclear launch codes", "machine_id": "uuid-1", "expected_insufficient": true}
```

### 8.5 CI/CD (GitHub Actions)
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env: {POSTGRES_PASSWORD: postgres}
        ports: [5432:5432]
        options: >-
          --health-cmd "pg_isready" --health-interval 5s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}
      - run: pip install -r backend/requirements.txt
      - run: cd backend && ruff check . && mypy . && pytest -v
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: {node-version: "20"}
      - run: cd frontend && npm ci && npm run lint && npx tsc --noEmit
```

## 9. Monitoring & Observability

### 9.1 Structured Logging (Backend)
```python
import structlog
logger = structlog.get_logger()

logger.info("query_completed",
    question_hash=hash(question),
    machine_id=str(machine_id),
    latency_ms=latency,
    chunks_retrieved=len(chunks),
    confidence=confidence,
    model=model_name,
    insufficient=insufficient_info
)
```

### 9.2 Key Metrics (Prometheus-ready)
| Metric | Type | Labels |
|---|---|---|
| rag_query_duration_seconds | Histogram | machine_id, model, status |
| rag_ingestion_duration_seconds | Histogram | machine_id, file_type |
| rag_chunks_retrieved | Histogram | machine_id |
| rag_confidence | Histogram | machine_id |
| rag_insufficient_info_total | Counter | machine_id |
| rag_llm_errors_total | Counter | model, error_type |

## 10. Security Considerations
| Layer | Measure |
|---|---|
| Transport | HTTPS everywhere (Vercel, Railway, Supabase enforce) |
| Auth | None (local MVP); NextAuth.js email magic link (prod only) |
| Data | Zero egress with Ollama; OpenRouter only sends question + context chunks |
| Injection | Parameterized SQL (SQLAlchemy), Pydantic validation |
| File Upload | Type validation (magic bytes), size limit (10MB), sanitize filenames |
| Rate Limit | Per-IP: 30 req/min (backend middleware) |

## 11. Rollback & Migration Strategy
| Scenario | Action |
|---|---|
| Schema change | Alembic migrations (auto-generated, reviewed) |
| Bad deploy | Railway/Vercel instant rollback (previous image) |
| Data corruption | Supabase point-in-time recovery (7 days free) |
| Model regression | Pin model version; A/B test |

## 12. Open Decisions
| Decision | Options | Recommendation |
|---|---|---|
| Auth for MVP | None (local) / NextAuth email / API key | **None** (local MVP), NextAuth for prod only — now consistent across docs |
| Local LLM size | 27B (≥16GB RAM) / 7–8B (8GB RAM) / OpenRouter | Size to host RAM per PRD §7.1 |
| Reranking | None (v1) / Cross-encoder / Cohere (paid) | None v1; add cross-encoder locally if quality insufficient |
| Streaming | SSE (Edge) / WebSocket / Non-streaming | **Non-streaming for MVP**; SSE deferred to v1.1 |
| Multi-file Ingestion | Sequential / Parallel (ProcessPoolExecutor) | **Parallel** — 4 workers, chunk-level |
| Chunk Metadata | Page number only / Section headers / Both | **Both** — extract from PDF TOC if available |

## 13. Appendix: File Tree (Target)
```
prototype-verdance/
 ├── docker-compose.yml
 ├── README.md
 ├── docs/
 │   ├── PRD.md
 │   ├── TRD.md
 │   ├── ARCHITECTURE.md
 │   ├── USE_CASES.md
 │   └── ROADMAP.md
 ├── backend/
 │   ├── Dockerfile
 │   ├── requirements.txt
 │   ├── alembic.ini
 │   ├── alembic/versions/
 │   ├── app/
 │   │   ├── main.py                 # FastAPI app
 │   │   ├── config.py               # Pydantic Settings
 │   │   ├── database.py             # Async engine, session
 │   │   ├── exceptions.py           # Exception hierarchy
 │   │   ├── models/
 │   │   │   ├── machine.py
 │   │   │   └── document.py
 │   │   ├── schemas/
 │   │   │   ├── machine.py
 │   │   │   ├── document.py
 │   │   │   └── query.py
 │   │   ├── api/
 │   │   │   ├── machines.py
 │   │   │   ├── ingest.py
 │   │   │   ├── query.py            # /api/query (+ /api/query/stream in v1.1)
 │   │   │   └── health.py
 │   │   ├── services/
 │   │   │   ├── machine.py
 │   │   │   ├── ingestion.py
 │   │   │   ├── embedding.py
 │   │   │   ├── retrieval.py
 │   │   │   ├── generation.py
 │   │   │   └── llm/client.py       # Ollama/OpenRouter abstraction
 │   │   └── utils/
 │   │       ├── chunking.py
 │   │       ├── pdf_extract.py
 │   │       └── citations.py
 │   └── tests/
 │       ├── conftest.py
 │       ├── test_machines.py
 │       ├── test_ingestion.py
 │       ├── test_retrieval.py
 │       ├── test_generation.py
 │       ├── test_query.py
 │       ├── test_citations.py
 │       ├── test_health.py
 │       └── eval/
 │           ├── golden.jsonl
 │           └── adversarial.jsonl
 ├── frontend/
 │   ├── Dockerfile
 │   ├── package.json
 │   ├── tsconfig.json
 │   ├── next.config.js
 │   ├── tailwind.config.ts
 │   ├── components.json
 │   ├── src/
 │   │   ├── app/
 │   │   │   ├── layout.tsx
 │   │   │   ├── globals.css
 │   │   │   ├── machines/
 │   │   │   │   ├── page.tsx
 │   │   │   │   └── [id]/page.tsx
 │   │   │   ├── upload/page.tsx
 │   │   │   └── api/proxy/route.ts
 │   │   ├── components/
 │   │   │   ├── ui/
 │   │   │   ├── chat/
 │   │   │   ├── upload/
 │   │   │   └── machines/
 │   │   ├── lib/
 │   │   │   ├── api.ts
 │   │   │   ├── config.ts
 │   │   │   └── utils.ts
 │   │   └── types/index.ts
 │   └── public/
```
> Deferred test files (`test_streaming.py`, `test_settings.py`, `test_export.py`) and the SSE route are added in v1.1.