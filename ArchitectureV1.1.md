# Architecture: Prototype Verdance — Maintenance RAG MVP

**Version:** 1.1
**Date:** 2026-08-16
**Status:** Draft for Review
**Architecture Style:** Modular Monolith (Backend) + SPA (Frontend)
**Deployment:** Containerized, Free-Tier Cloud + Local ML

## Change History
| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-16 | Initial draft |
| 1.1 | post-review | Rebrand; corrected LLM RAM sizing (ADR-008); added /api/query/stream to flows; scope phasing (ADR-009) |

## 1. Architectural Goals
| Goal | Decision | Rationale |
|---|---|---|
| Zero Cost | Local embeddings + Ollama LLM + Free-tier hosting | No API bills, no GPU cloud |
| Data Privacy | All ML inference local (optional OpenRouter fallback) | Manufacturing data never leaves network |
| Simplicity | Single FastAPI service, no microservices | 1-person team, weekend build |
| Extensibility | Clean service layer, pluggable providers | Swap LLM, embedder, vector DB later |
| Observability | Structured logging, Prometheus metrics | Debug production issues free |
| Type Safety | Pydantic v2 + TypeScript strict + SQLAlchemy 2.0 | Catch bugs at compile time |

## 2. High-Level Architecture
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
│  │  Machines   │  │   Chat      │  │  Upload     │                         │
│  │  List       │  │  Interface  │  │  Page       │                         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                         │
│         └────────────────┼────────────────┘                                │
│                          ▼                                                 │
│                   ┌─────────────────────────────┐                          │
│                   │      Next.js 14 (Vercel)    │                          │
│                   │  App Router + TanStack Query│                          │
│                   └──────────────┬──────────────┘                          │
└─────────────────────────────────┼─────────────────────────────────────────┘
                                  │ HTTPS (REST)
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BACKEND (FastAPI on Railway)                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  Machines    │ │  Ingestion   │ │   Query      │ │   Health     │       │
│  │  Router      │ │  Router      │ │   Router     │ │   Router     │       │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘       │
│         └────────────────┼────────────────┼────────────────┘               │
│                          ▼                ▼                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        SERVICE LAYER                                │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │   │
│  │  │  Machine    │ │ Ingestion   │ │ Retrieval   │ │ Generation  │   │   │
│  │  │  Service    │ │  Service    │ │  Service    │ │  Service    │   │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                │                │                │               │
│         ▼                ▼                ▼                ▼               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  PostgreSQL  │ │  sentence-   │ │   pgvector   │ │   LLM        │       │
│  │  + pgvector  │ │  transformers│ │  (IVFFlat)   │ │   Client     │       │
│  │  (Supabase)  │ │  (Local CPU) │ │              │ │  (Ollama/    │       │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. Component Design

### 3.1 Backend Services

**`MachineService` (`backend/app/services/machine.py`)**
```python
class MachineService:
    async def create(self, data: MachineCreate) -> Machine
    async def get(self, id: UUID) -> Machine | None
    async def list(self, limit: int, offset: int) -> list[MachineWithDocCount]
    async def delete(self, id: UUID) -> bool
    async def get_doc_count(self, id: UUID) -> int
```

**`IngestionService` (`backend/app/services/ingestion.py`)**
```python
class IngestionService:
    async def create_job(self, machine_id: UUID, filenames: list[str]) -> IngestionJob
    async def process_job(self, job_id: UUID) -> None
    async def extract_text(self, file: UploadFile) -> str
    async def chunk_text(self, text: str, source: str) -> list[TextChunk]
    async def embed_chunks(self, chunks: list[TextChunk]) -> list[EmbeddedChunk]
    async def store_chunks(self, machine_id: UUID, chunks: list[EmbeddedChunk]) -> int
```
Responsibility: End-to-end ingestion pipeline (incl. 50-page truncation).
Concurrency: `ProcessPoolExecutor` for CPU-bound extract/chunk/embed.

**`RetrievalService` (`backend/app/services/retrieval.py`)**
```python
class RetrievalService:
    async def search(
        self,
        query_embedding: list[float],
        machine_id: UUID | None,
        top_k: int,
        threshold: float
    ) -> list[RetrievedChunk]
```

**`GenerationService` (`backend/app/services/generation.py`)**
```python
class GenerationService:
    async def generate(
        self,
        question: str,
        context_chunks: list[RetrievedChunk],
        provider: LLMProvider,
        model: str,
        stream: bool = False
    ) -> GenerationResult | AsyncIterator[str]
```

### 3.2 LLM Provider Abstraction
```python
# backend/app/services/llm/client.py
from abc import ABC, abstractmethod

class LLMClient(ABC):
    @abstractmethod
    async def complete(
        self,
        messages: list[ChatMessage],
        temperature: float,
        max_tokens: int,
        stream: bool
    ) -> AsyncIterator[str] | str:
        ...

class OllamaClient(LLMClient):
    def __init__(self, base_url: str, model: str): ...

class OpenRouterClient(LLMClient):
    def __init__(self, api_key: str, model: str): ...

# Factory
def get_llm_client(provider: str, config: Settings) -> LLMClient:
    if provider == "ollama":
        return OllamaClient(config.OLLAMA_BASE_URL, config.OLLAMA_MODEL)
    return OpenRouterClient(config.OPENROUTER_API_KEY, config.OPENROUTER_MODEL)
```

## 4. Data Flow Details

### 4.1 Ingestion Flow (Async, Background)
1. `POST /api/ingest` (multipart) → IngestionRouter
2. Validate files (type, size) → Create IngestionJob (status=pending)
3. Return 202 Accepted with job_id
4. Background task: `IngestionService.process_job(job_id)`
   a. Update job status=processing
   b. For each file (parallel, max 4 workers):
      i. Extract text (pdfplumber / direct read); **truncate PDFs >50 pages with warning**
      ii. Chunk (recursive, 500/50 tokens)
      iii. Embed (MiniLM-L6-v2, batch=32)
      iv. Batch INSERT to documents table
      v. Update job.processed_chunks
   c. Update job status=completed, total_chunks
5. Client polls `GET /api/ingest/{job_id}` for progress

### 4.2 Query Flow (Sync, <3s target — MVP)
1. `POST /api/query` → QueryRouter
2. Validate input (Pydantic)
3. Embed question → `RetrievalService.search()` (cosine, IVFFlat, machine filter, threshold 0.3)
4. Build prompt (system + context + question)
5. Call `LLMClient.complete()` (sync)
6. Parse response: extract citations, validate, compute confidence
7. Return `QueryResponse` (JSON)

### 4.3 Streaming Query Flow (SSE — v1.1)
1. `POST /api/query/stream` → QueryRouter (same retrieval as 4.2)
2. Call `LLMClient.complete(stream=True)`
3. Emit SSE events: `data: {"delta": "..."}\n\n`
4. Final event carries `citations`, `confidence`, `insufficient_info`

## 5. Database Design Decisions
| Decision | Choice | Justification |
|---|---|---|
| Vector Index | IVFFlat (lists=100) | Fast build, good recall for <100k vectors; upgrade to HNSW at scale |
| Similarity | Cosine (1 - distance) | Standard for normalized embeddings |
| Chunk PK | UUID + chunk_index | Natural ordering, no separate ID needed |
| Metadata | JSONB | Flexible: page, section, headers, custom fields |
| Cascade Delete | ON DELETE CASCADE | Machine delete → chunks auto-cleaned |
| Connection Pool | SQLAlchemy async pool (size=10) | Matches Railway free tier limits |

## 6. Frontend Architecture

### 6.1 Component Hierarchy (MVP)
```
App (Providers)
 ├── Layout (Header, Footer)
 ├── MachinesPage
 │   ├── MachineTable (TanStack Query)
 │   └── CreateMachineModal
 ├── MachineChatPage
 │   ├── ChatHeader (machine info)
 │   ├── MessageList
 │   │   ├── UserMessage
 │   │   └── AssistantMessage
 │   │       ├── AnswerText (with inline citations)
 │   │       ├── CitationList
 │   │       ├── ConfidenceBadge
 │   │       └── InsufficientBanner
 │   └── ChatInput
 └── UploadPage
     ├── Dropzone (react-dropzone)
     ├── FileList (progress, status)
     ├── MachineSelect
     └── IngestButton
```
> Deferred to v1.1: Landing page, Settings page, CitationCard hover tooltips.

### 6.2 Data Fetching (TanStack Query)
```ts
// lib/api.ts
export const useMachines = () => useQuery({
  queryKey: ['machines'],
  queryFn: () => api.get<Machine[]>('/machines'),
});

export const useQueryMutation = () => useMutation({
  mutationFn: (data: QueryRequest) => api.post<QueryResponse>('/query', data),
});

export const useIngestMutation = () => useMutation({
  mutationFn: (formData: FormData) => api.post<IngestionJob>('/ingest', formData),
});
```

## 7. Configuration Management

### 7.1 Backend (Pydantic Settings)
```python
# backend/app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=True)

    # Database
    DATABASE_URL: str

    # Embeddings
    EMBEDDING_MODEL: str = "sentence-transformers/all-MiniLM-L6-v2"
    EMBEDDING_DIMENSION: int = 384
    EMBEDDING_BATCH_SIZE: int = 32
    EMBEDDING_DEVICE: str = "cpu"

    # Chunking
    CHUNK_SIZE: int = 500
    CHUNK_OVERLAP: int = 50
    MIN_CHUNK_SIZE: int = 50
    MAX_PDF_PAGES: int = 50          # pages beyond this are truncated w/ warning

    # Retrieval
    DEFAULT_TOP_K: int = 5
    MAX_TOP_K: int = 20
    SIMILARITY_THRESHOLD: float = 0.3

    # LLM — size model to host RAM (PRD §7.1)
    DEFAULT_PROVIDER: str = "ollama"  # ollama | openrouter
    OLLAMA_BASE_URL: str = "http://localhost:11434/v1"
    OLLAMA_MODEL: str = "qwen3.8-27b"  # requires >=16GB RAM; use 7-8B on 8GB hosts
    OPENROUTER_API_KEY: str | None = None
    OPENROUTER_MODEL: str = "qwen/qwen3.8-27b"
    LLM_TEMPERATURE: float = 0.0
    LLM_MAX_TOKENS: int = 2048

    # Ingestion
    MAX_FILE_SIZE_MB: int = 10
    MAX_FILES_PER_REQUEST: int = 10
    ALLOWED_EXTENSIONS: list[str] = [".pdf", ".txt", ".md"]
    INGESTION_WORKERS: int = 4

    # Logging
    LOG_LEVEL: str = "INFO"

settings = Settings()
```

### 7.2 Frontend (Environment)
```ts
// src/lib/config.ts
export const CONFIG = {
  API_URL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000',
  DEFAULT_PROVIDER: 'ollama' as const,
  MAX_HISTORY: 50,
} as const;
```

## 8. Error Handling Strategy
| Layer | Approach |
|---|---|
| Validation | Pydantic (backend) + Zod (frontend) — 422 on invalid input |
| Business Logic | Custom exceptions → mapped to HTTP status in middleware |
| LLM Failure | Retry 2x (exponential backoff) → 503 with user-friendly message |
| DB Failure | SQLAlchemy retry on transient errors (deadlock, timeout) → 503 |
| File Processing | Per-file try/catch → partial success reported in job status |
| Frontend | Error boundaries + toast notifications (sonner) |

### 8.1 Exception Hierarchy
```python
# backend/app/exceptions.py
class RAGException(Exception):
    status_code = 500
    detail = "Internal server error"

class ValidationError(RAGException):
    status_code = 422

class NotFoundError(RAGException):
    status_code = 404

class LLMUnavailableError(RAGException):
    status_code = 503
    detail = "LLM service temporarily unavailable. Try again or switch provider."

class IngestionError(RAGException):
    status_code = 500
```

## 9. Performance Optimization
| Area | Technique | Target |
|---|---|---|
| Embedding | Batch (32), CPU threads=4, normalize once | < 50ms/chunk |
| Vector Search | IVFFlat + `SET enable_seqscan=OFF` | < 100ms (10k chunks) |
| DB Inserts | `executemany` + `COPY` for bulk | 1000 chunks/sec |
| LLM | Ollama keep-alive, quant model sized to RAM | < 2s first token |
| Frontend | React Query caching (5min), code splitting | < 1s TTI |

## 10. Scaling Path (Post-MVP)
| Bottleneck | Solution | Effort |
|---|---|---|
| Ingestion throughput | Celery + Redis queue, horizontal workers | Medium |
| Vector search latency | HNSW index, partition by machine_id | Low |
| LLM latency | vLLM/SGLang self-hosted, batch requests | Medium |
| Multi-tenancy | RLS policies, org_id on all tables | Low (schema ready) |
| Reranking | Local cross-encoder (bge-reranker-base) | Low |
| Large PDFs | OCR pipeline (Tesseract), table extraction | High |

## 11. Security Architecture

### 11.1 Data Flow Classification
| Data | Classification | Path |
|---|---|---|
| Machine metadata | Internal | HTTPS → Backend → PostgreSQL |
| Document content | Confidential | HTTPS → Backend → PostgreSQL (pgvector) |
| Embeddings | Derived/Confidential | Local CPU → PostgreSQL |
| Query + Context | Confidential | HTTPS → Backend → Local LLM (Ollama) OR OpenRouter API |
| LLM Response | Internal | Local LLM/OpenRouter → Backend → HTTPS → Frontend |

### 11.2 Local-First Guarantee
- **Ollama mode:** Zero network egress for ML inference (verified via `netstat -an | grep 11434`)
- **OpenRouter mode:** Only question + top-k chunks sent (no raw documents, no embeddings)
- **Supabase:** Encrypted at rest (AES-256), in transit (TLS 1.3)

## 12. Deployment Architecture

### 12.1 Development (docker-compose)
```
# All services local, hot reload enabled.
# Pull an Ollama model sized to host RAM (PRD §7.1):
#   >=24GB RAM -> qwen3.8-27b-q4_k_m
#   16GB RAM   -> qwen3.8-27b-q3_k_m
#   8GB RAM    -> 7-8B q4 model, or use OpenRouter fallback
```

### 12.2 Production (Free Tier)
| Component | Platform | Connection |
|---|---|---|
| Frontend | Vercel | `https://app.domain.com` |
| Backend | Railway | `https://api.domain.com` |
| Database | Supabase | `postgresql://user:pass@host:5432/db` (pooled) |
| LLM | Local (Ollama) + Tailscale Funnel | `http://tailscale-ip:11434` |
| LLM Fallback | OpenRouter | `https://openrouter.ai/api/v1` |

## 13. Observability Architecture

### 13.1 Logging Pipeline
```
Structured JSON (structlog)
    ├─► stdout (Docker/Railway captures)
    └─► (Optional) Loki/Grafana Cloud free tier
```

### 13.2 Key Dashboards
| Dashboard | Panels |
|---|---|
| API Health | Request rate, latency p50/p95/p99, error rate by endpoint |
| RAG Quality | Avg confidence, insufficient info rate, citation count/query |
| Ingestion | Throughput (chunks/sec), success rate, queue depth |
| LLM | Token usage, latency, provider distribution, error types |
| Database | Connections, query duration, index usage, storage |

## 14. Decision Log (ADRs)
| ADR | Title | Decision | Date |
|---|---|---|---|
| ADR-001 | Vector Index | IVFFlat for MVP, HNSW at 100k+ vectors | 2026-08-16 |
| ADR-002 | LLM Provider | Ollama primary, OpenRouter fallback | 2026-08-16 |
| ADR-003 | Embedding Model | all-MiniLM-L6-v2 (384-dim, CPU-friendly) | 2026-08-16 |
| ADR-004 | Chunking | Recursive 500/50 tokens | 2026-08-16 |
| ADR-005 | Auth | None for local MVP, NextAuth email for prod | 2026-08-16 |
| ADR-006 | Streaming | Non-streaming for MVP; SSE via Edge Runtime in v1.1 | 2026-08-16 |
| ADR-007 | Multi-tenancy | Schema-ready (RLS), single-tenant MVP | 2026-08-16 |
| ADR-008 | Local LLM sizing | Model quant/size chosen by host RAM (PRD §7.1) | post-review |
| ADR-009 | Scope phasing | Vertical slice (TXT/MD) first; PDF + fallback in Tier 2 | post-review |

## 15. File Reference
| Document | Purpose |
|---|---|
| PRD.md | Product requirements, user stories, success metrics |
| TRD.md | Technical specs: API, DB, pipelines, testing, infra |
| ARCHITECTURE.md | This doc: system design, components, data flows, decisions |
| USE_CASES.md | Detailed use cases, edge cases, acceptance tests |
| ROADMAP.md | Milestone build plan for Qwen Coder |
| backend/app/config.py | All tunable parameters (single source of truth) |
| docker-compose.yml | Local dev environment |