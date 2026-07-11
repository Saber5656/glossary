# ADR-002: In-repo YAML canonical store with two-tier candidate/curated model

- Status: Accepted (2026-07-11)
- Deciders: product owner (R5, R6), designer (layout details)
- Related: DESIGN.md §6, §7, §8

## Context

The glossary must live with the code it describes, survive re-extraction
without destroying human edits, and be reviewable through normal Git workflows.
Owner chose "structured file in the target repo is canonical; Markdown/site are
generated" and "two-tier candidates/curated with approve/reject CLI".

## Decision

1. All managed files live under one directory (default `glossary/`) in the
   target repo (layout table: DESIGN.md §6).
2. **Ownership split is absolute**: `candidates.yaml` is machine-owned and
   rewritten wholesale by `extract`; `terms/<id>.yaml` and `rejected.yaml` are
   human-owned and only modified by explicit curation commands (`approve`,
   `reject`, `draft --curated`). `extract` never touches human-owned files.
3. One file per curated term (`terms/<id>.yaml`) — minimizes merge conflicts
   between teammates and gives clean per-term Git history.
4. Term identity = `termKey` (NFKC + trim + Latin lowercase + space→`-`;
   DESIGN.md §7.1), implemented in exactly one module. File ids are ASCII
   (`t-<hash8>` or user slug) to avoid macOS NFD/Windows encoding pitfalls.
5. YAML (not JSON) for human readability of the curated tier; parsed with the
   `yaml` package restricted to the core schema plus hard limits
   (1 MiB, alias ≤ 100, depth ≤ 20) and validated with zod (DESIGN.md §13-B2).
6. Generated site is git-ignored; `GLOSSARY.md` export is committed (that is
   the shareable artifact).

## Consequences

- Re-extraction is always safe to run; the diff of `candidates.yaml` is the
  review surface for extraction changes.
- Lifecycle needs an explicit state machine and `validate` invariants
  (DESIGN.md §8) — specified once, tested.
- Hand-editing `candidates.yaml` is unsupported by contract (documented).
- A malicious/malformed PR editing glossary YAML cannot exploit the parser
  (limits + schema validation) — required because the store crosses the PR
  trust boundary.

## Alternatives considered

- **Single terms.yaml**: simpler I/O but merge-conflict-prone and noisy diffs.
- **Markdown-as-canonical**: friendlier editing, but structure validation and
  deterministic tooling suffer; Markdown remains a generated view.
- **SQLite**: not diffable/reviewable in Git; rejected for a Git-native tool.
- **Central glossary repo (multi-repo)**: matches larger orgs but front-loads
  sync/conflict design; deferred to v2 (DESIGN.md §3.3).
