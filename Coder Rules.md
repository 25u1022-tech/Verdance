# CODER_RULES.md — Prototype Verdance

You are helping build **Prototype Verdance**, a local-first Maintenance RAG MVP. These rules govern every change you make. Follow them exactly.

## Source of Truth (v1.1)

Use these documents as the single source of truth:
- `docs/PRD.md` (v1.1)
- `docs/TRD.md` (v1.1)
- `docs/ARCHITECTURE.md` (v1.1)
- `docs/USE_CASES.md` (v1.1)
- `docs/ROADMAP.md` (v1.1)

If anything conflicts between them, **STOP and ask** before coding.

## Strict Working Rules

1. Do **NOT** build the entire application at once.
2. Work **ONLY** on the milestone the user requests (see `ROADMAP.md`).
3. Before writing or changing any code, list:
   - The exact files you will create or modify
   - The exact commands to run or test the change
   - Any dependencies you will add
4. **WAIT** for user approval before making changes.
5. Do **NOT** modify files outside the requested milestone.
6. Do **NOT** add features not in the milestone.
7. Do **NOT** refactor unrelated code.
8. Do **NOT** delete or rewrite documentation files.
9. Keep every change minimal and reviewable (small diff).
10. Never push directly to `main`. Always work on a milestone branch (`milestone/<n>-<slug>`) and open a PR.
11. After completing a milestone, **STOP** and report:
    - Files changed
    - How to run it
    - How to verify it
    - What comes next

## Scope Guardrails (v1.1 Trimmed Scope)

Build **only** what belongs to the current milestone's tier. The following are **deferred to v1.1+** and must NOT be implemented unless explicitly requested:

- ❌ Authentication / NextAuth (none for local MVP)
- ❌ Streaming / SSE (`POST /api/query/stream`)
- ❌ Settings page UI (provider configured via env)
- ❌ Citation hover tooltips & answer export
- ❌ Multi-turn conversation context
- ❌ Ingestion history view
- ❌ Landing page
- ❌ Health/metrics dashboards

If a requested change would require a deferred feature, **stop and flag it** instead of building it.

### Tier Discipline
- **Tier 1 (vertical slice):** machines CRUD, TXT/MD ingestion, embeddings, retrieval, query, basic 3-page frontend (non-streaming). Build this fully first.
- **Tier 2 (MVP complete):** PDF ingestion + OpenRouter fallback. Only after Tier 1 works.

## Technical Rules

- **LLM sizing:** Never hardcode a 27B model for an 8GB host. Respect `PRD.md §7.1` RAM sizing. Use the env-configured `OLLAMA_MODEL`; size to host RAM.
- **Embeddings:** Always local `sentence-transformers/all-MiniLM-L6-v2` (384-dim, normalized). Never a cloud embedding API.
- **50-page rule:** PDFs >50 pages are **truncated to the first 50 with a warning** (canonical v1.1 behavior). Do not hard-reject them.
- **Citations:** Enforce `[doc_<uuid>:<chunk_idx>]` format; validate every citation against retrieved chunks; strip hallucinated citations.
- **Insufficient info:** When no chunks exceed the 0.3 threshold, return `insufficient_info:true`, `confidence:0.0`, and populated `missing_info`. Never hallucinate an answer.
- **Secrets:** Never hardcode API keys, passwords, or connection strings. Always read from environment variables.
- **Type safety:** Pydantic v2 (backend), TypeScript strict (frontend), SQLAlchemy 2.0 async.

## Definition of a Complete Milestone

A milestone is done **only** when it:
- Runs locally without errors
- Can be verified with a concrete command (from `ROADMAP.md`)
- Includes tests where the milestone expects them
- Does not touch deferred features
- Has been approved by the user

## When Unsure

- If a requirement is ambiguous → **ask** before coding.
- If two docs disagree → **ask** which takes priority.
- If the change risks scope creep → **flag it** and propose the minimal version.