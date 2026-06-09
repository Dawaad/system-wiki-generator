# Golden Example — the `Acme` wiki

A canonical, fully-worked instance of the pattern, for pattern-matching when generating a new
wiki. `Acme` is a fictional, domain-agnostic system: a **transactional + scheduled notification
platform**. An API ingests send requests; a worker fans them out across email/SMS/push providers;
a Next.js console manages templates and delivery logs.

Two repos:
- `~/repos/acme/acme-api` — Python core (ingest API, fan-out worker, provider adapters)
- `~/repos/acme/acme-web` — Next.js console

This is what a healthy generated wiki looks like a few ingests in. Use it to calibrate tone,
granularity, and frontmatter — not as content to copy.

---

## `CLAUDE.md` (excerpt)

```markdown
---
type: wiki-schema
wiki: Acme
Created: 2026-06-08
Updated: 2026-06-08
---

# Acme Wiki

A transactional + scheduled notification platform: an API ingests send requests, a worker fans
them out across email/SMS/push providers, and a Next.js console manages templates and logs.

Source repos:
- `~/repos/acme/acme-api` — Python ingest API, fan-out worker, provider adapters
- `~/repos/acme/acme-web` — Next.js console

## Scope
In: architecture decisions (provider abstraction, queue choice), send/fan-out flows, provider
interactions, delivery-log SOPs, observed insights (provider failure modes).
Out: GTM/pricing, onboarding tutorials, status updates.

## Conventions
- Tag prefix: `#acme/*`
- Reference repo code by path — never paste
```

## `retrieval.yaml` (excerpt)

```yaml
domain: Acme

sync:
  acme-api:
    last_sync_sha: 4f2c9ad1e0b7c3a59d8e2f1a6b4c0d9e8f7a6b5c
    last_sync_date: 2026-06-08
  acme-web:
    last_sync_sha: null            # console ingest pending
    last_sync_date: 2026-06-08

routes:
  # ─── System overview ────────────────────────────────────────────
  - pattern: "system overview"
    aliases: ["what is acme", "acme architecture"]
    load:
      - Acme/pages/acme-system-overview.md
      - Acme/pages/ingest-vs-fanout-split.md
    why: Mental model — ingest accepts, worker fans out, console observes.

  # ─── Delivery ───────────────────────────────────────────────────
  - pattern: "provider failover"
    aliases: ["failover", "provider fallback", "retry across providers"]
    load:
      - Acme/pages/provider-failover-ladder.md
      - Acme/pages/idempotent-send-key.md
    why: How a send survives one provider being down; where double-sends creep in.
```

## A page — `pages/ingest-vs-fanout-split.md` (kind: decision)

```markdown
---
type: wiki-page
wiki: Acme
kind: decision
sources:
  - path: ~/repos/acme/acme-api/acme/api/ingest.py
    cited_region: "<whole-file>"
    content_hash: 9c1f4b7e2a8d6035c4e1b9f0a7d2c6e8b3f5a1d409e7c2b6f8a0d3e5c7b1f9a24
Updated: 2026-06-08
tags: [#acme, #acme/architecture]
---

# Ingest vs. Fan-out Split

## Choice
Split the API into a synchronous **ingest** path (validate + enqueue, returns a send id) and an
asynchronous **fan-out** worker (selects provider, delivers, records the result).

## Why
Ingest is on the caller's critical path and must be fast and durable; fan-out is retryable and
latency-tolerant. Coupling them would force the caller to wait on a flaky third-party provider.

## Invariant
Ingest is the only writer of the `sends` row; the worker only transitions its status. A send id
is returned only after the row is durably enqueued.

## Gotcha
The idempotency key is checked at ingest, not fan-out — a retried fan-out must not create a second
send. See [[idempotent-send-key]].

## Tradeoff
Two code paths and an eventual-consistency window between "accepted" and "delivered".

## Cross-links
- [[acme-system-overview]]
- [[provider-failover-ladder]] · [[idempotent-send-key]]
```

## `index.md` (excerpt)

```markdown
## Architecture & decisions
- [[acme-system-overview]] [insight] — One-page mental model of ingest → fan-out → console.
- [[ingest-vs-fanout-split]] [decision] — Why the sync API and async worker are separate.
- [[idempotent-send-key]] [decision] — Dedup at ingest; the key contract callers must honor.

## Flows
- [[provider-failover-ladder]] [flow] — How a send walks the provider ladder on failure.
```

## `log.md` (excerpt)

```markdown
## [2026-06-08] init | wiki scaffolded
- Created CLAUDE.md, index.md, log.md, retrieval.yaml, retrieval.suggested.log, .last-sync.

## [2026-06-08] ingest | acme-api first-pass (Phase-5 seed)
- [[acme-system-overview]] [insight]
- [[ingest-vs-fanout-split]] [decision]
- [[idempotent-send-key]] [decision]
- [[provider-failover-ladder]] [flow]
- skipped: per-provider SDK wrappers (restate the SDK — grep the code)
```

## `.last-sync` (excerpt)

```yaml
repos:
  acme-api:
    path: ~/repos/acme/acme-api
    branch: main
    last_sync_sha: 4f2c9ad1e0b7c3a59d8e2f1a6b4c0d9e8f7a6b5c
    last_sync_short: 4f2c9ad
    last_sync_date: 2026-06-08
    last_sync_subject: "Add provider failover ladder + idempotency key"
  acme-web:
    path: ~/repos/acme/acme-web
    branch: main
    last_sync_sha: null
    note: "Console ingest pending — do NOT treat acme-web pages as SHA-anchored."
```

---

## What makes this a *good* instance

- Pages pass the L2 filter: each leads with a gotcha/invariant/why, none restate the SDK.
- Frontmatter carries L3 decay fields (`cited_region` + `content_hash`) against real code paths.
- Routes are reachable by phrases a developer would actually type, with aliases for synonyms.
- The seed `log.md` records what was **skipped** and why — the skip discipline is the point.
- One repo is SHA-anchored; the other is honestly marked `null` (partial ingest), not faked.
