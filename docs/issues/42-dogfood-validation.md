# 42 — Dogfood on this repo + owner manual validation protocol

## Title

Dogfooding setup and the owner's real-repo acceptance protocol (final v1 gate)

## Summary

Run glossary on its own repository (committed config + GLOSSARY.md per
DESIGN.md §19), write the manual validation protocol, and execute the owner
acceptance session on a real team repository — the last v1 gate (R12).

## Context

Everything before this proves the design to machines; this issue proves it to
the actual user. Its output (precision notes, friction list) closes U2 or
spawns tuning issues.

## Scope

- This repo's own `glossary/` setup + committed export;
  `docs/validation/manual-protocol.md`; the executed protocol record.

## Detailed Requirements

1. Dogfood this repository:
   - `glossary init`; tune config: include `docs/**`, `src/**`, `README*`;
     add stopwords as needed after a first pass.
   - `extract` → curate ≥ 15 real terms of this project (candidate, curated,
     drift, extractor, FLR, …) → `export` (commit GLOSSARY.md) → `build`
     (gitignored) → `validate` green.
   - Add npm script `dogfood` chaining extract+validate+export; CONTRIBUTING
     gains one line: "run `npm run dogfood` when your PR changes extraction
     behavior; review the GLOSSARY.md diff".
2. `docs/validation/manual-protocol.md` — a runbook the owner executes on a
   private team repo (results stay private; only aggregate notes return):
   - Preconditions (Node, clone, build, link).
   - Session script (60–90 min): init → extract; record: total candidates,
     top-30 precision count (real term? y/n), top-5 missing expected terms
     (recall probes chosen beforehand by the owner), noisiest extractor;
     approve 10+ / reject 10+; re-extract (exclusion + drift check);
     hand-edit 2 definitions; optional: draft --dry-run reviewed, then (if
     owner consents with their own endpoint/key) draft 3 terms; export +
     build; open site, search 5 terms JA/EN, click through; validate.
   - Recording template (Markdown table) + verdict line:
     `accept / accept-with-tuning-issues / reject` with reasons.
   - Explicit sensitivity note: never paste private repo content into public
     issues; aggregate numbers only.
3. Acceptance session: owner runs the protocol (agent may assist). File
   resulting tuning issues (label them; reference U2). v1 is DONE when the
   verdict is `accept` or all `accept-with-tuning-issues` issues are closed.

## Acceptance Criteria

- [ ] This repo's glossary/ committed: config + ≥15 curated terms + GLOSSARY.md; CI green with dogfood script.
- [ ] manual-protocol.md complete and executable without asking questions (dry-run it on fixtures first).
- [ ] Owner session executed; record (aggregates) attached to this issue; verdict recorded.
- [ ] Tuning issues filed for every noted problem (or none needed).
- [ ] ISSUE_PLAN.md §1 completion statement satisfied — cross-check every issue closed; final comment lists the check.

## Validation

The executed protocol record IS the validation. Reviewer confirms the
completion-statement cross-check.

## Dependencies

28, 33, 38 (full product), 39–41 (docs/smoke first so the session uses final
UX).

## Non-goals

Public Pages deployment of this repo's glossary (owner decision, separate),
performance tuning beyond noted issues, v2 planning.

## Design References

DESIGN.md §19, §16 (manual row), §3.4-U2, R12; ISSUE_PLAN.md §1/§6.
