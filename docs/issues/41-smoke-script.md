# 41 — `scripts/smoke.sh` and public-repo smoke protocol

## Title

Smoke script: run the full pipeline against arbitrary public repositories

## Summary

Implement the smoke tool of DESIGN.md §16 (R12): a script that shallow-clones
a given repo, runs init → extract → export → build, and prints a quality/
health report — for local use against public JA-doc OSS repos (never in CI).

## Context

Fixtures prove correctness; smoke runs prove robustness against real-world
messiness (huge trees, odd encodings, unexpected Markdown). Findings feed U2
(threshold tuning) and may spawn new issues.

## Scope

- `scripts/smoke.sh` + `docs/validation/smoke-protocol.md`.

## Detailed Requirements

1. `scripts/smoke.sh <git-url-or-local-path> [--keep]`:
   - bash, `set -euo pipefail`; requires: git, node ≥ 22, built CLI
     (`dist/` present — error with hint `npm run build` otherwise).
   - Workspace: `mktemp -d`; shallow clone `--depth 1 --single-branch`
     (or copy if local path); NEVER runs inside the current repo.
   - Steps with timing (`time -p` equivalents captured):
     `init` → `extract --json` → `validate --json` → `export` →
     `build` → 33's site scanner invoked via a node one-liner
     (`node dist/... ` — expose the scanner as `npm run scan:site -- <dir>`
     utility script; add that wiring here).
   - Report (stdout, also written to `smoke-report-<repo>-<date>.md` in cwd):
     repo, commit, file counts (scanned/skipped by reason), extractor counts,
     top 20 candidates table (key/kind/score/occ), drift n/a, timings per
     step, site scanner verdict, warnings summary.
   - Exit non-zero if any step fails; `--keep` retains the temp dir and
     prints its path.
   - Safety: workspace paths quoted everywhere; no `eval`; cleanup via trap;
     the cloned repo's content is DATA (we never execute its scripts — the
     CLI only reads).
2. `docs/validation/smoke-protocol.md`:
   - Candidate public repos (JA docs, active): suggest 3 concrete ones with
     rationale placeholders for the runner to confirm current state (e.g. a
     JA-documented OSS CLI, a JA tech-doc repo, one mostly-EN repo as
     contrast) — the protocol instructs choosing 2–3, running, and recording:
     command output file, top-20 subjective precision notes (how many of top
     20 are "real terms" — target ≥ 12/20 as a soft bar per U2), crashes ⇒
     file issues.
   - Explicitly: smoke never runs in CI (results depend on upstream repos);
     reports are attached to the tracking issue, not committed.
3. Script passes shellcheck (add `lint:sh` dev script using shellcheck via
   `npx shellcheck`? — shellcheck is a binary; instead: document manual
   shellcheck run in the issue validation, no new toolchain dep; CI skips it).

## Acceptance Criteria

- [ ] Smoke run against `fixtures/repo-ja-mixed` (local-path mode) completes green and produces the report file with all sections.
- [ ] Smoke run against ONE real public JA-doc repo completes (or failures are triaged into filed issues); report attached to the GitHub issue.
- [ ] `--keep` and cleanup-trap behavior verified; no temp dirs leak on failure (test by killing mid-run).
- [ ] Manual shellcheck run: zero errors (paste output).
- [ ] scan:site utility wired and reused (33's scanner, not a copy).

## Validation

Two smoke transcripts (fixture + real repo) attached to the PR/issue; timings
noted against the §24 budget expectations.

## Dependencies

24, 30 (pipeline + build), 33 (scanner reuse).

## Non-goals

CI integration, multi-repo batch orchestration, automatic issue filing,
Windows shell support (bash only; documented).

## Design References

DESIGN.md §16 (smoke row), R12, U2; §13-B1 (untrusted repo posture).
