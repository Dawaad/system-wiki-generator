---
name: generate-system-wiki
description: Stand up a full LLM-managed knowledge system for any codebase — an L0–L3 atomic-page wiki, the four management commands that operate it (wiki-query/ingest/lint/routes-review), and a bespoke feature-design skill suite (orchestrator + product lens + eng lens) grounded in that wiki, then seed it with starter pages from the target repo. Use when someone wants to "set up a wiki / knowledge base for this repo", "generate the design skills for project X", or bootstrap the system-wiki pattern somewhere new.
---

# Generate System Wiki

Point this at a codebase and it scaffolds the whole stack: an LLM-only wiki (L0–L3), the commands
that run it, and a project-specific design-skill suite, seeded so it answers `/wiki-query` on day
one.

**Read first:** `references/pattern-spec.md` (the architecture), `references/golden-example.md`
(a worked instance), `references/manifest-schema.md` (the variables). Templates live under
`templates/`; every `{{VAR}}` is resolved from the manifest before writing.

## Modes

- **Full generate** (default) — all six phases against a fresh `<WIKI_ROOT>/<DOMAIN>`.
- **Skills-only** (`existing-wiki`) — the wiki already exists. Skip Phase 3 (scaffold) and Phase 5
  (seed); run Phase 4 (install design skills) wired to the existing wiki, plus Phase 6 verify.
  This installs design skills onto a live, populated wiki without re-scaffolding or clobbering it.

Confirm the mode in Phase 0 if it isn't obvious from the request.

## Phase 0 — Intake

Ask **one question at a time**: target repo path(s); domain name (`DOMAIN`); central wiki repo
location (`WIKI_ROOT`, default `~/repos/wiki`); tag prefix (default `#<domain_lower>/`). Confirm
full vs skills-only.

## Phase 1 — Codebase analysis

Dispatch a subagent to scan the target repo(s) — languages, entrypoints, READMEs, ADRs, subsystem
boundaries, repo-path roots. It returns a **domain profile**: a one-paragraph `DOMAIN_SUMMARY`, a
component list, `REPO_ROOTS`, and candidate `SEED_SOURCES`. Heavy reading stays in the subagent.

## Phase 2 — Manifest resolution

Assemble the full manifest (`references/manifest-schema.md`) from intake + the profile, including
derived variables (`DOMAIN_LOWER`, `KINDS_PIPE`, `REPO_ROOTS_BULLETED`, the sync blocks).
**Present it to the user for confirmation before writing anything.** Resolve `SCOPE_IN` / `SCOPE_OUT`
with them.

## Phase 3 — Scaffold wiki (skip in skills-only)

Safety first (see Idempotency below). Then:
1. Create `<WIKI_ROOT>` if missing (`git init`).
2. **If `<WIKI_ROOT>/<DOMAIN>/` already exists, STOP and ask** — don't overwrite a populated wiki.
3. Create `<WIKI_ROOT>/<DOMAIN>/{pages,sources/{code,docs,research,notion,experiments},designs}`.
4. Stamp `CLAUDE.md`, `retrieval.yaml`, `index.md`, `log.md`, `.last-sync` from `templates/wiki/`;
   create an empty `retrieval.suggested.log`.
5. Install the four commands from `templates/commands/` into `<WIKI_ROOT>/.claude/commands/`
   **idempotently** — skip any that already exist; never clobber.
6. Symlink `<TARGET_REPO>/wiki → <WIKI_ROOT>/<DOMAIN>` (only if absent; if a non-symlink `wiki/`
   exists, STOP and ask).

## Phase 4 — Install design skills

Stamp `<domain>-feature-design`, `<domain>-product-review`, `<domain>-eng-review`, and `_shared/`
from `templates/skills/` into `<WIKI_ROOT>/<DOMAIN>/skills/` — the skills live inside the wiki they
ground against, not in the target repo. Wire them to the bridge paths (`WIKI_ROOT`, `DOMAIN`,
`TARGET_REPO`). If `<domain>-feature-design` already exists there, ask before overwriting.

## Phase 5 — Seed (skip in skills-only)

Run the `/wiki-ingest` flow over the Phase-1 `SEED_SOURCES` → ~10–20 starter atomic pages +
matching `retrieval.yaml` routes + index/log entries. Use the real ingest merge semantics (check
the index, merge don't append). Record each repo's `last_sync_sha` (`git -C <root> rev-parse HEAD`)
into `.last-sync`, `retrieval.yaml` `sync:`, and `index.md` `synced_sha`.

## Phase 6 — Verify & report

1. Run a `/wiki-lint` pass over the new wiki; expect zero critical issues.
2. Smoke-test one `/wiki-query` against a seeded topic; confirm it returns a cited answer.
3. **Self-review for leftover `{{VAR}}` markers** in every emitted file — fail loudly if any remain.
4. Report everything created (paths, symlinks, skills) and how to invoke the new design skills.

## Idempotency & safety

- **Commands:** domain-agnostic; install only if absent. Never clobber.
- **Wiki dir:** if `<WIKI_ROOT>/<DOMAIN>/` exists, stop and ask.
- **Symlink:** create only if absent; a non-symlink `wiki/` in the target → stop and ask.
- **Skills:** if `<domain>-feature-design` exists in `<WIKI_ROOT>/<DOMAIN>/skills/`, ask before overwriting.
- **Seeding:** real ingest merge semantics (check index, merge not append).
- Assume git target repos (the `sync:` SHA anchoring needs it).

## Out of scope (v1)

- Embedded per-repo wiki layout (only central-shared is built; templates leave room).
- A deterministic CLI substitution engine (the skill does the stamping).
- Non-git target repos.
- Automated Slack/Notion ingestion wiring (the design skills *reference* such context via
  `/wiki-query`; populating it stays the `/wiki-ingest` flow's job).
