# PROJECT_STATE.md — Prototype Verdance

**Last Updated:** [date/time]
**Current Phase:** Milestone 1 — Repository + PostgreSQL + pgvector
**Current Branch:** `<your-branch-name>`
**Docs Version:** v1.1
**Build Mode:** Qwen Coder writes code to GitHub; Architect reviews; Human approves.

---

## Current Milestone

**Milestone 1:** Repository structure + PostgreSQL + pgvector

### Allowed Scope
Create ONLY:

- `docker-compose.yml`
- `init.sql`

### Not Allowed
- Do NOT create backend application code.
- Do NOT create frontend application code.
- Do NOT add authentication.
- Do NOT add ingestion logic.
- Do NOT add query endpoints.
- Do NOT add embeddings.
- Do NOT modify docs unless explicitly requested.

---

## Completed

- [ ] Project branch created from main
- [ ] PRD v1.1 drafted
- [ ] TRD v1.1 drafted
- [ ] ARCHITECTURE v1.1 drafted
- [ ] USE_CASES v1.1 drafted
- [ ] ROADMAP drafted
- [ ] CODER_RULES drafted
- [ ] README drafted
- [ ] Docs committed to repo
- [ ] Qwen Coder connected to repo
- [ ] Milestone 1 prompt given to Qwen Coder
- [ ] Milestone 1 PR opened
- [ ] Milestone 1 verified locally
- [ ] Milestone 1 merged

---

## In Progress

- Commit all v1.1 docs, ROADMAP, CODER_RULES, README, and continuity files.
- Run Milestone 1 in Qwen Coder.
- Review Qwen Coder’s proposed files before approving.

---

## Blocked

- None currently known.

---

## Next Exact Action

1. Commit all planning/docs files to the project branch.
2. Paste the Milestone 1 prompt into Qwen Coder.
3. Wait for Qwen Coder to list the exact files it wants to create.
4. Verify it proposes only:
   - `docker-compose.yml`
   - `init.sql`
5. Approve only if scope is correct.
6. Open PR.
7. Run verification commands.
8. Paste verification output back to Architect chat.

---

## Milestone 1 Verification Commands

```bash
docker compose up -d postgres
docker compose exec postgres psql -U postgres -d maintenance_rag -c "\dx"
docker compose exec postgres psql -U postgres -d maintenance_rag -c "\dt"