# system-wiki-generator

Stand up a complete LLM-managed knowledge system for any codebase, from one skill.

Point `/generate-system-wiki` at a repo and it builds:

1. **An LLM-only wiki** — the L0–L3 architecture: source-of-truth (the real code) → a deterministic
   routing manifest → atomic insight pages → a decay tracker that flags pages when the code they
   cite moves.
2. **The four management commands** that operate it: `/wiki-query`, `/wiki-ingest`, `/wiki-lint`,
   `/wiki-routes-review`.
3. **A bespoke feature-design skill suite** — an orchestrator plus a product lens and an engineering
   lens that ground every design decision in the wiki and persist outputs back into it, so designs
   compound into the same knowledge base.
4. **Seed content** — starter atomic pages extracted from the target codebase, so the wiki answers
   queries on day one.

## Why

A codebase carries knowledge that isn't in the code: why a path was chosen, what was rejected, what
invariant must hold, what bites in production. Normal docs rot because nothing keeps them synced.
This pattern is an agent-readable navigation layer: small pages that cite code regions, a router
that finds them deterministically, and a hash-based decay check that flags drift. See
`references/pattern-spec.md`.

## Usage

```
/generate-system-wiki
```

- **Full generate** (default): scaffold a fresh wiki, install commands + design skills, seed pages.
- **Skills-only**: install the design skills against a wiki that already exists, without
  re-scaffolding or clobbering it.

The skill interviews you for the target repo, domain name, and central wiki location, scans the
codebase, shows you the resolved manifest for confirmation, then writes everything.

## Layout

```
SKILL.md                     # /generate-system-wiki — the generator
templates/
  wiki/                      # CLAUDE.md, retrieval.yaml, index.md, log.md, .last-sync (.tmpl)
  commands/                  # wiki-query/ingest/lint/routes-review (.tmpl) — domain-agnostic
  skills/                    # feature-design, product-review, eng-review, _shared/ (.tmpl)
references/
  pattern-spec.md            # the L0–L3 architecture explained
  golden-example.md          # a fully-worked agnostic instance (the `Acme` wiki)
  manifest-schema.md         # every variable the generator collects
```

Every template carries `{{VARIABLE}}` markers resolved from the manifest before writing; the
generator self-reviews for leftover markers in Phase 6.

## Design notes

- **Central shared wiki + symlink.** One wiki repo (`~/repos/wiki` by default) holds many domains,
  each symlinked into its target repo as `wiki/`. Lets one knowledge base cross-link multiple
  systems.
- **Domain-agnostic commands.** Generated commands discover domains by scanning
  `<WIKI_ROOT>/*/CLAUDE.md` at run time — one copy serves every domain, and adding a domain needs
  no command edits.
- **Idempotent + safe.** Never clobbers an existing wiki, command, symlink, or skill without
  asking.
