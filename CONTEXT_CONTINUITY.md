# CONTEXT_CONTINUITY.md — Prototype Verdance

This project uses AI assistants with limited context windows. The repository is the permanent source of truth.

## Core Rule

If any AI context window ends, resume from the repository, not from memory.

## Permanent Source Files

Always read these first:

1. `CODER_RULES.md`
2. `ROADMAP.md`
3. `PROJECT_STATE.md`
4. `SESSION_LOG.md`
5. `docs/PRD.md`
6. `docs/TRD.md`
7. `docs/ARCHITECTURE.md`
8. `docs/USE_CASES.md`

## Before Ending Any Session

Before closing any chat or stopping work:

1. Stop at a safe checkpoint.
2. Update `PROJECT_STATE.md`.
3. Append a new entry to `SESSION_LOG.md`.
4. Commit and push changes.
5. Write the next exact action.
6. Write the resume prompt for the next session.

Do NOT end a session while a half-finished task exists without recording it.

## If Architect Chat Context Ends

Start a new Architect chat and provide:

- `PROJECT_STATE.md`
- `ROADMAP.md`
- `CODER_RULES.md`
- Latest relevant error or Qwen Coder output

Then use the Architect Resume Prompt.

## If Qwen Coder Context Ends

Start a new Qwen Coder session on the same repo/branch.

Make it read:

- `CODER_RULES.md`
- `ROADMAP.md`
- `PROJECT_STATE.md`
- Current branch changes
- Current PR, if one exists

Then use the Qwen Coder Resume Prompt.

## Safe Stop Points

Stop and save state when:

- A milestone is complete.
- A PR is opened.
- A verify command passes or fails.
- A confusing error appears.
- The AI starts forgetting constraints.
- The AI starts expanding scope.
- The conversation becomes long or slow.

## Never Do These

- Do NOT rely on chat memory alone.
- Do NOT let Qwen Coder rewrite docs without approval.
- Do NOT let Qwen Coder build deferred features.
- Do NOT continue after scope drift without resetting constraints.
- Do NOT merge without local verification.