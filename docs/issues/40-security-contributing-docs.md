# 40 — SECURITY.md, CONTRIBUTING.md, third-party notices

## Title

Project policy docs: security policy, contribution guide, notices

## Summary

Write the security policy (threat-model-backed, per DESIGN.md §13),
contribution guide (workflow + quality gates), and consolidate third-party
license notices.

## Context

Public OSS repo from day one (ADR-006): a real SECURITY.md channels reports
away from public issues; CONTRIBUTING.md encodes the docs-first workflow so
future contributors (and agents) follow the same source-of-truth discipline.

## Scope

- `SECURITY.md`, `CONTRIBUTING.md`, `NOTICE.md`. Repo settings (advisories,
  scanning toggles) remain owner-managed — documented as prerequisites only.

## Detailed Requirements

1. `SECURITY.md`:
   - Reporting: GitHub Security Advisories (private) on this repo; no email
     published in v1; expected first response ≤ 7 days (best-effort OSS
     wording); supported version: latest main (v1 has no releases).
   - Security model summary FOR USERS (from DESIGN §13, rewritten plainly):
     what the tool does/never does (no network by default; LLM opt-in with
     consent + redaction + dry-run; generated site is static/self-contained);
     out-of-scope, EXACTLY §13's list: a fully malicious local user;
     sandboxing Node itself; secrets already committed to the target repo.
     Separately, a "Deployment caution" paragraph (NOT out-of-scope):
     hosting visibility must match content sensitivity — link
     docs/guides/hosting.md's warning.
   - Hardening tips for users: run in CI with least privilege, review
     candidates.yaml diffs, keep `llm.enabled` false unless needed, pin the
     install SHA.
2. `CONTRIBUTING.md`:
   - Source-of-truth rule: behavior changes require updating DESIGN.md (and
     the relevant docs/issues file) in the same PR; GitHub Issues derive from
     docs/issues.
   - Dev setup with the exact issue-01 commands: `npm ci`,
     `npm run typecheck`, `npm run lint`, `npm run format:check`,
     `npm run test`, `npm run build`; golden updates:
     `GOLDEN_UPDATE=1 npm run test` (local only — refused in CI).
   - PR rules: CI green required; no new runtime deps without ADR-001
     amendment; security-sensitive paths — changes there require test
     additions or an explicit reviewer-visible justification:
     `src/store/yaml-io.ts`, `src/scan/**`, `src/extract/**` (regex
     discipline), `src/site/templates/**`, `src/site/client/**`,
     `src/llm/**`, `package.json`, `package-lock.json`,
     `.github/workflows/**`, `examples/pages.yml`.
   - Conventional, English commit messages; PR to main only (protected).
3. `NOTICE.md`: kuromoji (Apache-2.0), IPADIC dictionary license (full
   required notice text), MiniSearch (MIT), other runtime deps table with
   licenses (generate once manually; a script is v2). README links here
   (39 coordinates — if 39 already merged, update its credits link).
4. All three in English (ADR-006), concise (< 150 lines each).

## Acceptance Criteria

- [ ] SECURITY.md contains: reporting channel, response expectation, user-facing security model, the EXACT §13 out-of-scope triple, the separate deployment caution, hardening tips — each verifiable against DESIGN §13 (no invented promises).
- [ ] CONTRIBUTING.md encodes docs-first + dependency-ADR + security-area test rules.
- [ ] NOTICE.md includes the verbatim IPADIC notice and a complete direct-runtime-deps license table (verify against package.json).
- [ ] GitHub renders the security policy tab (file recognized).
- [ ] Links among SECURITY.md / CONTRIBUTING.md / NOTICE.md and already-existing docs resolve; README links are added only if the README files already exist on the branch (otherwise 39 wires them — coordinate via a note in this issue's PR).

## Validation

Link checker; manual check of the repo Security tab; license table
cross-checked against `npm ls --omit=dev`.

## Dependencies

01 only (dep list exists). Coordinates with 39 by documenting intended README
link targets; does not depend on it.

## Non-goals

Automated license scanning, CLA/DCO setup, code of conduct (owner decision,
add on request), release/security-patch process (v2 with versioning).

## Design References

DESIGN.md §13 (model + out-of-scope), §16; ADR-001 (deps), ADR-003 §5
(IPADIC), ADR-006.
