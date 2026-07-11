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
     what it does NOT protect against (out-of-scope list from §13: secrets
     already in your repo, malicious local user, hosting misconfiguration —
     link hosting.md visibility warning).
   - Hardening tips for users: run in CI with least privilege, review
     candidates.yaml diffs, keep `llm.enabled` false unless needed, pin the
     install SHA.
2. `CONTRIBUTING.md`:
   - Source-of-truth rule: behavior changes require updating DESIGN.md (and
     the relevant docs/issues file) in the same PR; GitHub Issues derive from
     docs/issues.
   - Dev setup (clone/ci/build/test), test conventions (unit/e2e/golden
     update workflow `GOLDEN_UPDATE=1`), lint/format.
   - PR rules: CI green required; no new runtime deps without ADR-001
     amendment; security-sensitive areas list (yaml-io, scanner, templates,
     client, llm/*) → require explicit test additions.
   - Conventional, English commit messages; PR to main only (protected).
3. `NOTICE.md`: kuromoji (Apache-2.0), IPADIC dictionary license (full
   required notice text), MiniSearch (MIT), other runtime deps table with
   licenses (generate once manually; a script is v2). README links here
   (39 coordinates — if 39 already merged, update its credits link).
4. All three in English (ADR-006), concise (< 150 lines each).

## Acceptance Criteria

- [ ] SECURITY.md contains: reporting channel, response expectation, user-facing security model, out-of-scope list, hardening tips — each verifiable against DESIGN §13 (no invented promises).
- [ ] CONTRIBUTING.md encodes docs-first + dependency-ADR + security-area test rules.
- [ ] NOTICE.md includes the verbatim IPADIC notice and a complete direct-runtime-deps license table (verify against package.json).
- [ ] GitHub renders the security policy tab (file recognized).
- [ ] Links from/to README resolve (39's link checker passes).

## Validation

Link checker; manual check of the repo Security tab; license table
cross-checked against `npm ls --omit=dev`.

## Dependencies

01 (dep list exists); coordinates with 39.

## Non-goals

Automated license scanning, CLA/DCO setup, code of conduct (owner decision,
add on request), release/security-patch process (v2 with versioning).

## Design References

DESIGN.md §13 (model + out-of-scope), §16; ADR-001 (deps), ADR-003 §5
(IPADIC), ADR-006.
