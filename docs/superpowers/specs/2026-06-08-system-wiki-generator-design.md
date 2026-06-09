# System Wiki Generator — Design

**Date:** 2026-06-08
**Status:** Built (v1)
**Repo:** `~/repos/system-wiki-generator/`

## 1. Purpose

Generalize the LLM-managed knowledge-system pattern into a reusable generator. Point it at any
codebase and it stands up the full stack:

1. An **LLM-managed wiki** (L0–L3 architecture: source-of-truth → routing manifest →
   atomic pages → decay tracker).
2. The **management commands** that operate it (`wiki-query`, `wiki-ingest`, `wiki-lint`,
   `wiki-routes-review`).
3. A **bespoke feature/implementation design skill suite** (orchestrator + product lens +
   eng lens) that grounds every design decision in the wiki and persists outputs back into it.
4. **Seed content** — an initial atomic-page set extracted from the target codebase so the
   wiki is useful on day one.

The reference implementation is the agnostic **`Acme`** instance — a fictional notification
platform documented in `references/golden-example.md`. It is the canonical shape the generator
emits and the pattern to calibrate against.

## 2. Background: the pattern being generalized

The pattern has two parts:

- **Wiki + schema:** a per-domain wiki dir — `CLAUDE.md` (schema, kinds, page templates,
  scope, tag prefix), `retrieval.yaml` (deterministic token-set routing manifest with a
  `sync:` staleness anchor), `index.md`, `log.md`, `pages/` (atomic insight pages),
  `sources/{code,docs,research,notion,experiments}/` (pointers, never copies). Lives in a
  central wiki repo and is symlinked into the target project as `<target>/wiki`.
- **Commands:** slash commands that read `retrieval.yaml` first (token-set subset matching with
  plural stemming), fall through to index navigation, and enforce the L2 page filter
  (a page must justify itself on gotcha / invariant / why-non-obvious / observed-failure-mode).

The full architecture is documented in `references/pattern-spec.md`.

### Key findings that shape the generator

- Hand-built instances of this pattern tend to be **not generic.** They hardcode a domain table,
  reference a specific vault root, and carry stale absolute paths. Templatizing them is real work
  and an opportunity to produce cleaner output than a hand-built original — so generated commands
  use **dynamic domain detection** instead of a hardcoded table.
- The **L1 routing manifest** is the load-bearing retrieval mechanism: deterministic
  pattern/alias → file map, matched by token-set subset (not full-text search). Seeding must
  produce both pages *and* routes, or the wiki can't answer queries deterministically.
- The **L3 decay tracker** (`cited_region` + `content_hash` per source) is the forcing
  function that keeps pages synced to code. The generated `CLAUDE.md` and `wiki-ingest`/
  `wiki-lint` carry the extended frontmatter + the SHA-256 region recipes.

## 3. Locked decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Generator form | **Generator skill + template library** | Matches the ecosystem; handles intelligent parts (domain primer, routes, seeding) a dumb script can't. |
| Wiki location | **Central shared wiki repo + symlink** | One central wiki repo holds many domains, each symlinked into its target repo; lets one wiki cross-link multiple domains. |
| Seeding depth | **Scaffold + seed starter pages** | Analyze the target repo and ingest key sources → initial atomic pages + routes. Wiki answers `/wiki-query` immediately. |
| Domain detection (generated commands) | **Dynamic** — scan `<WIKI_ROOT>/*/CLAUDE.md` | Adding a future domain needs no command edits; removes stale hardcoded paths. |
| Command install | **Idempotent** | Domain-agnostic commands; one copy per central wiki repo serves all domains. Don't clobber if present. |
| Spec scope | **One spec, generator + reference instance** | The agnostic `Acme` instance is the calibration reference, not a separate deliverable. |

## 4. Design skill suite (what the generator emits per project)

Mirrors a CEO/product plan review and an eng-manager plan review, but grounded in the project's
wiki. Three skills installed into `<WIKI_ROOT>/<DOMAIN>/skills/<domain>-*/` — they live inside the
wiki they ground against, not in the target repo:

- **`<domain>-feature-design`** — orchestrator / front door. Runs phases in sequence:
  1. **Intake** — interview the feature intent one question at a time; sniff product-heavy
     vs eng-heavy to decide which lenses run.
  2. **Wiki grounding** *(the differentiator)* — fire targeted queries: prior decisions,
     **what was decided against**, related features/flows, relevant meeting/Slack/Notion
     context, applicable invariants/gotchas. Produce a **Context Brief** surfaced *before*
     proposing anything. If the wiki already settled it, say so loudly.
  3. **Product lens** — premise challenge + 2–3 forced alternatives (effort/risk/reuse) +
     scope-mode selection (expand/selective/hold/reduce), each choice citing wiki prior art.
     One decision = one `AskUserQuestion`. Outputs locked scope, "NOT in scope",
     "what already exists".
  4. **Eng lens** — complexity gate, then Architecture / Data-flow & edge cases / Test
     coverage / Performance, each section grounding against wiki system-design pages and the
     actual code those pages cite. ASCII diagrams, confidence calibration, coverage diagram.
     Outputs P1/P2/P3 tasks, failure modes, test plan.
  5. **Persist** — write the living design doc; on acceptance distill atomic pages; update
     index/log/routing; run a lint-style contradiction check against existing decisions.

- **`<domain>-product-review`** and **`<domain>-eng-review`** — the lenses, each runnable
  standalone (they do their own scoped grounding + optional persist). The orchestrator reuses
  them.

### The wiki bridge (`_shared/wiki-bridge.md`)

Every grounding/persist step executes by **dispatching a subagent that operates directly on
`<WIKI_ROOT>/<DOMAIN>`**, following the wiki's own command procedures. This sidesteps the
cross-repo command-registration problem (the `/wiki-*` commands live in the wiki repo, not
the target repo's working directory) and keeps heavy wiki-reading out of the design
conversation. Two procedures:

- **`ground(question)`** → subagent follows `wiki-query.md` (read `retrieval.yaml` → match →
  read pages → synthesize), returns a structured brief with `[[page]]` / `source` citations.
- **`persist(design)`** → subagent follows `wiki-ingest.md`: write the design doc, distill
  atomic pages, update `index.md` / `log.md`, propose a `retrieval.yaml` route.

### Persistence model

- **Living doc** → `<WIKI_ROOT>/<DOMAIN>/designs/YYYY-MM-DD-<feature>-design.md`, new
  `type: wiki-design` frontmatter (status proposed→accepted, grounding citations, decision
  log). Sections: Problem · Context Brief (with a "decided-against / constraints" block) ·
  Scope · Alternatives · Architecture · Data flow & edge cases · Test plan · Performance ·
  Implementation tasks · Open questions · Decision log.
- **Atomic pages** distilled on acceptance → enter `retrieval.yaml` so future `/wiki-query`
  calls surface them. Designs compound into the same knowledge base.

## 5. Generator repo layout

```
~/repos/system-wiki-generator/
  README.md
  SKILL.md                          # /generate-system-wiki
  templates/
    wiki/
      CLAUDE.md.tmpl                 # {{DOMAIN}}, {{KINDS}}, per-kind page templates,
                                     #   {{TAG_PREFIX}}, {{REPO_ROOTS}}, scope in/out
      retrieval.yaml.tmpl            # sync block + matching spec + empty routes:
      index.md.tmpl
      log.md.tmpl
      .last-sync.tmpl
    commands/
      wiki-query.md.tmpl             # dynamic domain detection (scan <WIKI_ROOT>/*/CLAUDE.md)
      wiki-ingest.md.tmpl
      wiki-lint.md.tmpl
      wiki-routes-review.md.tmpl     # batched fall-through route review (referenced by query)
    skills/
      feature-design/SKILL.md.tmpl
      product-review/SKILL.md.tmpl
      eng-review/SKILL.md.tmpl
      _shared/
        wiki-bridge.md.tmpl          # ground() / persist()
        design-doc.md.tmpl           # living design-doc skeleton
  references/
    pattern-spec.md                  # the L0–L3 architecture explained
    golden-example.md                # canonical agnostic instance (the `Acme` wiki)
    manifest-schema.md               # every variable the generator collects
```

## 6. Generator skill flow (`/generate-system-wiki`)

- **Phase 0 — Intake.** Collect, one question at a time: target repo path(s); domain name;
  central wiki repo location (default `~/repos/wiki`); tag prefix (default `#<domain>/`).
- **Phase 1 — Codebase analysis.** Subagent scans the target repo(s) — languages,
  entrypoints, READMEs, ADRs, subsystem boundaries, repo-path roots — and returns a
  **domain profile** (one-paragraph system summary, component list, repo roots, candidate
  seed sources). Heavy reading stays in the subagent.
- **Phase 2 — Manifest resolution.** Assemble the manifest (§7) from the profile.
  **Present to the user for confirmation** before writing anything.
- **Phase 3 — Scaffold wiki.** Create the central wiki repo if missing (`git init`);
  create `<WIKI_ROOT>/<DOMAIN>/{pages,sources/{code,docs,research,notion,experiments},designs}`;
  stamp `CLAUDE.md`, `retrieval.yaml`, `index.md`, `log.md`, `.last-sync`; install the four
  commands into `<WIKI_ROOT>/.claude/commands/` **idempotently** (skip if present);
  symlink `<TARGET_REPO>/wiki → <WIKI_ROOT>/<DOMAIN>`.
- **Phase 4 — Install design skills.** Stamp `<domain>-feature-design`,
  `<domain>-product-review`, `<domain>-eng-review` (+ `_shared`) into
  `<WIKI_ROOT>/<DOMAIN>/skills/`, wired to this wiki's bridge paths and domain name.
- **Phase 5 — Seed.** Run the ingest flow over the Phase-1 candidate sources → ~10–20
  starter atomic pages + matching `retrieval.yaml` routes + index/log entries. Record the
  `sync:` SHA per source repo (`git -C <root> rev-parse HEAD`).
- **Phase 6 — Verify & report.** Run a `wiki-lint` pass; smoke-test one `/wiki-query`;
  self-review for leftover `{{VAR}}` markers; report everything created (paths, symlinks,
  skills) and how to invoke the new skills.

### Modes

- **Full generate** (default) — all six phases against a fresh `<WIKI_ROOT>/<DOMAIN>`.
- **Skills-only** (`existing-wiki`) — target a wiki that already exists. Skip Phase 3
  (scaffold) and Phase 5 (seed); run Phase 4 (install design skills) wired to the existing
  wiki, plus Phase 6 verify. This is how a live wiki instance gets its design skills
  without re-scaffolding or clobbering its populated wiki.

## 7. Manifest schema (variables collected → substituted)

| Variable | Source | Example (Acme) |
|----------|--------|----------------|
| `DOMAIN` | intake | `Acme` |
| `WIKI_ROOT` | intake (default `~/repos/wiki`) | `~/repos/wiki` |
| `TARGET_REPO` | intake | `~/repos/acme` |
| `REPO_ROOTS` | analysis | `~/repos/acme/acme-api`, `~/repos/acme/acme-web` |
| `DOMAIN_SUMMARY` | analysis | "transactional + scheduled notification platform; API ingests, worker fans out…" |
| `KINDS` | template default, tunable | `decision, flow, feature, interaction, sop, insight, framework` |
| `TAG_PREFIX` | intake (default `#<domain>/`) | `#acme/` |
| `PAGE_TEMPLATES` | template default | per-kind skeletons from `CLAUDE.md.tmpl` |
| `SCOPE_IN` / `SCOPE_OUT` | analysis + confirm | architecture decisions, flows… / GTM, tutorials… |
| `SEED_SOURCES` | analysis | entrypoints, READMEs, ADRs, key subsystems |

Full list (including derived variables) in `references/manifest-schema.md`.

## 8. Dynamic domain detection (improvement #1)

Generated commands replace any hardcoded domain table with discovery:

1. List `<WIKI_ROOT>/*/CLAUDE.md`. Each declares `wiki:` + a scope description.
2. Pick the target domain by: working directory (if inside a `<TARGET_REPO>` whose
   `wiki` symlink resolves into `<WIKI_ROOT>/<DOMAIN>`) → else question topic vs each
   domain's scope description → else ask.
3. No command edits needed when a new domain is added later.

Generated commands reference `<WIKI_ROOT>` resolved at run time — no stale absolute paths.

## 9. Idempotency & safety (improvement #2)

- **Commands:** domain-agnostic; install only if absent. Never clobber an existing command.
- **Wiki dir:** if `<WIKI_ROOT>/<DOMAIN>/` already exists, stop and ask (don't overwrite a
  populated wiki).
- **Symlink:** create only if absent; if a non-symlink `wiki/` exists in the target, stop
  and ask.
- **Skills:** if `<domain>-feature-design` already exists in the target, ask before
  overwriting.
- **Seeding** uses the real `wiki-ingest` merge semantics (check index, merge not append).

## 10. Acceptance test

**Part A — Full generate to scratch (proves scaffold + seed work).**
Run **full mode** against a real target repo with `WIKI_ROOT=/tmp/swg-verify`. Success criteria:
1. Scaffold produces a wiki whose `CLAUDE.md` / `retrieval.yaml` structurally match the
   golden example (schema, kinds, extended frontmatter, sync block).
2. The four commands install with dynamic domain detection and pass a self-consistency read.
3. Seeding produces ≥10 atomic pages + matching routes; a smoke `/wiki-query` returns a
   cited answer.
4. `wiki-lint` reports zero critical issues on the generated wiki.

**Part B — Skills-only against a populated wiki (the design-skill deliverable).**
Run **skills-only mode** against a target repo whose wiki already exists. Success criteria:
5. The three `<domain>-*` design skills install into `<WIKI_ROOT>/<DOMAIN>/skills/` and their
   wiki-bridge `ground()`/`persist()` paths resolve against `<WIKI_ROOT>/<DOMAIN>`.
6. A dry `<domain>-feature-design` grounding step returns a cited Context Brief from the live
   wiki (proves the bridge works end-to-end).

## 11. Out of scope (v1)

- Embedded per-repo wiki layout (only central-shared is built; templates leave room).
- A deterministic CLI substitution engine (the skill does stamping).
- Non-git target repos (assume git for `sync:` SHA anchoring).
- Automated Slack/Notion ingestion wiring (the design skills *reference* such context via
  `/wiki-query`; populating it remains the existing `/wiki-ingest` flow's job).

## 12. Success criteria (generator)

- Any unrelated repo can be generated without editing any template by hand.
- Generated commands are byte-identical across domains (domain-agnostic), proving the
  central-install model.
- Every template variable in §7 is resolved from intake or analysis — no leftover `{{VAR}}`
  markers in emitted files (Phase-6 self-review checks for this).
