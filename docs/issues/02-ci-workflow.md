# 02 — CI workflow with SHA-pinned actions and least privilege

## Title

GitHub Actions CI: lint/typecheck/test matrix with pinned actions and minimal permissions

## Summary

Add `.github/workflows/ci.yml` running lint, typecheck, unit and e2e tests on
Node 22/24/26 across ubuntu/macos (+ best-effort windows), following the
security posture in DESIGN.md §13-B5/B6, plus Dependabot configuration.

## Context

The repo is public from day one. CI is both the quality gate for every later
issue and part of the supply-chain security surface (abuse case AC6).

## Scope

- `.github/workflows/ci.yml`, `.github/dependabot.yml`.
- No release/publish workflows (v2), no Pages deployment (issue 34).

## Detailed Requirements

1. Trigger: `push` to `main` and `pull_request` (no `pull_request_target`).
2. Top-level `permissions: contents: read`. No job-level `permissions` blocks
   (or, if present, exactly `contents: read`); no `id-token`,
   `pull-requests`, `actions`, or any write scope anywhere. No secrets used
   anywhere (no `${{ secrets.` references).
3. `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
4. Job `test`: matrix `node: [22, 24, 26]` × `os: [ubuntu-latest, macos-latest]`;
   steps: checkout → setup-node (with `cache: npm`) → `npm ci` →
   `npm run lint` → `npm run typecheck` → `npm run test` → `npm run build`.
5. Job `windows`: same steps, `windows-latest`, Node 24 only,
   `continue-on-error: true` (known unknown U3 — failures visible, not blocking).
6. **Every action reference pinned to a full commit SHA** with a trailing
   version comment, e.g.
   `uses: actions/checkout@<40-char-sha> # v4.x.y`. No tag/branch refs.
7. Timeout: `timeout-minutes: 15` per job.
8. `.github/dependabot.yml`, exactly this structure:
   ```yaml
   version: 2
   updates:
     - package-ecosystem: npm
       directory: "/"
       schedule: { interval: weekly }
       groups:
         minor-and-patch:
           update-types: ["minor", "patch"]
     - package-ecosystem: github-actions
       directory: "/"
       schedule: { interval: weekly }
   ```
9. Workflow `name: ci` (issue 39 adds the README badge referencing this
   name; no README change in this issue).

## Acceptance Criteria

- [ ] CI runs green on a PR touching only a comment (matrix 6 jobs + windows).
- [ ] Every `uses:` reference in ci.yml matches `@[0-9a-f]{40}` — verify with `grep -E 'uses:' .github/workflows/ci.yml | grep -vE '@[0-9a-f]{40}'` returning nothing.
- [ ] `grep -E 'pull_request_target|\$\{\{ secrets\.' .github/workflows/ci.yml` returns nothing; the only `permissions` lines are `contents: read`.
- [ ] Dependabot config validates (GitHub UI shows both ecosystems active).
- [ ] (actionlint validation is deferred to issue 34, which adds the SHA-pinned actionlint step covering `.github/workflows/` and `examples/`.)

## Validation

Open a draft PR after merging this workflow; screenshot/paste the run summary.
Intentionally break lint locally to confirm the job fails (do not push the
broken commit).

## Dependencies

01.

## Non-goals

Pages deploy example (34), coverage upload services, release automation (v2),
CodeQL/secret-scanning repo settings (owner-managed, ADR-006 §5).

## Design References

DESIGN.md §13-B5/B6, §16 (CI row), §17; ADR-006 §5.
