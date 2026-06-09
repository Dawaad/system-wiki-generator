# Manifest Schema — variables the generator collects

The generator resolves these from intake (asked) or analysis (a codebase-scan subagent), then
substitutes them into every `*.tmpl`. **Self-review must confirm zero `{{VAR}}` markers remain in
emitted files.** Examples use the agnostic `Acme` domain (see `golden-example.md`).

| Variable | Source | Example (Acme) |
|----------|--------|----------------|
| `DOMAIN` | intake | `Acme` |
| `DOMAIN_LOWER` | derived (lowercase of `DOMAIN`) | `acme` |
| `WIKI_ROOT` | intake (default `~/repos/wiki`) | `~/repos/wiki` |
| `TARGET_REPO` | intake | `~/repos/acme` |
| `REPO_ROOTS` | analysis | `~/repos/acme/acme-api`, `~/repos/acme/acme-web` |
| `REPO_ROOTS_BULLETED` | derived (markdown bullets of `REPO_ROOTS`, one per line) | `- \`~/repos/acme/acme-api\` — Python notification API` |
| `DOMAIN_SUMMARY` | analysis | "A transactional + scheduled notification platform: an API ingests send requests, a worker fans them out across email/SMS/push providers, and a Next.js console manages templates and delivery logs." |
| `KINDS` | template default, tunable | `decision, flow, feature, interaction, sop, insight, framework` |
| `KINDS_PIPE` | derived (`KINDS` joined with ` | `) | `decision \| flow \| feature \| interaction \| sop \| insight \| framework` |
| `TAG_PREFIX` | intake (default `#<domain_lower>/`) | `#acme/` |
| `SCOPE_IN` | analysis + confirm (markdown bullets) | architecture decisions, send/fan-out flows, provider interactions, ops SOPs |
| `SCOPE_OUT` | analysis + confirm (markdown bullets) | GTM/pricing, onboarding tutorials, status updates |
| `SEED_SOURCES` | analysis | entrypoints, READMEs, ADRs, key subsystems (the Phase-5 ingest list) |
| `SYNC_BLOCKS` | analysis (one YAML block per repo, for `retrieval.yaml`) | `acme-api:\n    last_sync_sha: null\n    last_sync_date: 2026-06-08` |
| `SYNC_REPO_BLOCKS` | analysis (one YAML block per repo, for `.last-sync`) | see `.last-sync.tmpl` shape |
| `REPO_NAME` / `REPO_ROOT` | analysis (per repo, used inside sync blocks) | `acme-api` / `~/repos/acme/acme-api` |
| `DATE` | run time (`git`/host date, passed in) | `2026-06-08` |

## Notes

- `DOMAIN` is the wiki folder name and the `wiki:` frontmatter value; keep it stable once chosen
  (renaming means rewriting every page's frontmatter).
- Derived variables (`DOMAIN_LOWER`, `KINDS_PIPE`, `REPO_ROOTS_BULLETED`, the sync blocks) are
  computed by the generator from the collected ones — they are not separately asked.
- `DATE` is passed in at run time rather than read inside a template; templates never call the
  clock themselves.
- Multi-repo systems repeat `SYNC_BLOCKS` / `SYNC_REPO_BLOCKS` once per repo. Single-repo systems
  emit one block.
- Everything here is confirmed with the user in Phase 2 (manifest resolution) **before** any file
  is written.
