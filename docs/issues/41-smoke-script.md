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
   - Workspace: `work=$(mktemp -d)`; git URL ⇒ shallow clone
     `git clone --depth 1 --single-branch <url> "$work/repo"`; LOCAL path ⇒
     copy into `"$work/repo"` preserving symlinks WITHOUT dereferencing
     (`cp -R -P` semantics), excluding `.git`; the CLI's own scanner guards
     (no symlink follow, path-prefix checks) handle traversal from there.
     NEVER runs inside the current repo.
   - Exact step commands (cwd = the PRODUCT repo; `$R="$work/repo"`), each
     timed:
     1. `node dist/cli/main.js init --repo "$R"`
     2. `node dist/cli/main.js extract --repo "$R" --json`
     3. `node dist/cli/main.js validate --repo "$R" --json`
     4. `node dist/cli/main.js export --repo "$R"`
     5. `node dist/cli/main.js build --repo "$R"`
     6. `npm run scan:site -- "$R/glossary/site"` (command provided by issue
        33 — consumed here, not created here)
   - Report — written to
     `"$PWD/smoke-report-${slug}-$(date +%Y%m%d-%H%M%S).md"` where `slug` =
     URL/path basename minus `.git`, with every char outside `[A-Za-z0-9._-]`
     replaced by `_`. Required H2 sections in order: `Repository`, `Commit`,
     `File Counts`, `Extractor Counts`, `Top 20 Candidates`, `Timings`,
     `Site Scanner`, `Warnings`.
   - Exit non-zero if any step fails. Cleanup: trap on EXIT/SIGINT/SIGTERM
     removes `$work` UNLESS `--keep` was given (then print the path and skip
     removal in all paths, including failure).
   - Safety: workspace paths quoted everywhere; no `eval`; the cloned repo's
     content is DATA (we never execute its scripts — the CLI only reads).
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

- [ ] Smoke run against `fixtures/repo-ja-mixed` (local-path mode) completes green; the report file contains exactly the 8 required H2 sections in order; the filename matches the slug+timestamp pattern.
- [ ] Smoke run against ONE real public JA-doc repo completes (or failures are triaged into filed issues); report attached to the GitHub issue.
- [ ] Cleanup semantics verified: normal failure removes the temp dir; SIGINT mid-run removes it; `--keep` preserves it in success, failure, AND signal paths (record before/after `ls` of the temp path for each case).
- [ ] Manual shellcheck run: zero errors (paste output).
- [ ] scan:site utility wired and reused (33's scanner, not a copy).

## Validation

Two smoke transcripts (fixture + real repo) attached to the PR/issue; timings
noted against the §24 budget expectations.

## Dependencies

24, 30 (pipeline + build), 33 (provides `npm run scan:site`).

## Non-goals

CI integration, multi-repo batch orchestration, automatic issue filing,
Windows shell support (bash only; documented).

## Design References

DESIGN.md §16 (smoke row), R12, U2; §13-B1 (untrusted repo posture).
