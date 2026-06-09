# Pattern Spec — the L0–L3 LLM-only wiki

This is the architecture the generator stands up. Read it to understand *why* each emitted
file exists. The canonical concrete instance is `golden-example.md`.

## The problem it solves

A codebase carries knowledge that isn't in the code: why a path was chosen, what was rejected,
what invariant must hold, what bites you in production. Traditional docs rot because nothing
forces them to stay synced. This pattern is an **LLM-only navigation layer** over the code:
small atomic pages that cite code regions, a deterministic router that finds them, and a decay
tracker that flags pages when the code they cite moves.

It is LLM-only by design: the pages are written to be read by an agent answering a question,
not by a human browsing. No tutorials, no restating code.

## The four layers

```
L0  Source-of-truth ......... code, ADRs, linked docs in the real repos. Cited, never copied.
        │  (pages point down into it; lint hashes regions of it)
L1  Routing manifest ........ retrieval.yaml — deterministic pattern → page map. Read FIRST.
        │  (token-set subset match, not full-text search)
L2  Atomic insight pages .... pages/*.md — one concept each, ~80–150 words, must pass the filter.
        │  (each page's frontmatter cites L0 regions)
L3  Decay tracker ........... cited_region + content_hash per source, + per-repo sync SHA.
            (lint re-hashes on demand; flags drift between page and code)
```

### L0 — Source-of-truth
The real repos and linked docs. The wiki **never duplicates** them — a page records a path and
a cited region, and the reader follows it. This is what keeps the wiki small and prevents the
"two copies drift apart" failure.

### L1 — Routing manifest (`retrieval.yaml`)
A list of `route`s, each a `pattern` (+ optional `aliases`) → `load:` paths. Matching is
**token-set subset**: tokenize and plural-stem the query, and a route fires when its pattern's
tokens are a subset of the query's tokens. Deterministic — the same query always routes the same
way, with no embedding model in the loop. A query that matches nothing is logged to
`retrieval.suggested.log` and later promoted into a route by `/wiki-routes-review`. This is the
load-bearing retrieval mechanism; seeding must produce routes, not just pages.

### L2 — Atomic insight pages (`pages/*.md`)
One concept per page. A page may exist ONLY if it justifies itself on at least one axis:
**gotcha**, **invariant**, **why-non-obvious**, or **observed failure mode**. If it can't in
~150 words, it restates source and should be deleted. Each page has a `kind` (decision, flow,
feature, interaction, sop, insight, framework) that drives retrieval weighting.

### L3 — Decay tracker
The forcing function that keeps pages honest. Each source entry carries `cited_region` (a heading,
a symbol, or `<whole-file>`) and `content_hash` (SHA-256 of that region). `/wiki-lint` re-extracts
the region and re-hashes; a mismatch is flagged `STALE`. Region-level hashing keeps the
false-positive rate low (unrelated edits elsewhere in the file don't trigger it). A separate
per-repo `sync:` SHA (in `.last-sync` and `retrieval.yaml`) anchors whole-repo staleness: every
code-grounded query first compares the wiki's last-synced commit to repo `HEAD`.

## The commands that operate it

- `/wiki-query` — read `retrieval.yaml` → match → read pages → synthesize a cited answer.
- `/wiki-ingest` — capture a source, extract atomic insights, classify kind, write/update pages
  with L3 frontmatter, update index + routes + log.
- `/wiki-lint` — dead links, duplicates, orphans, stale, **L3 decay**, kind violations, contradictions.
- `/wiki-routes-review` — batch-promote logged fall-through queries into routes.

Generated commands are **domain-agnostic**: they discover domains by scanning
`<WIKI_ROOT>/*/CLAUDE.md` at run time, so one copy in the central wiki repo serves every domain
and adding a domain needs no command edits.

## Layout of one domain wiki

```
<WIKI_ROOT>/<DOMAIN>/
  CLAUDE.md                 # schema: kinds, page templates, scope, tag prefix, frontmatter spec
  retrieval.yaml            # L1 routing manifest (+ sync: anchor)
  retrieval.suggested.log   # fall-through queries awaiting routes-review
  index.md                  # catalog of pages, grouped by section
  log.md                    # append-only ingest/lint history
  .last-sync                # per-repo git SHA staleness anchor
  pages/*.md                # L2 atomic pages (flat — no subfolders)
  sources/{code,docs,research,notion,experiments}/   # pointers, never copies
  designs/                  # living design docs written by the design skills
```

The target repo gets a `wiki` symlink → `<WIKI_ROOT>/<DOMAIN>`, so the design skills and commands
can resolve the wiki from inside the codebase.
