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
   - npm script `dogfood`, exactly:
     `glossary extract && glossary validate && glossary export && glossary build && glossary validate`
     (build included so site generation is exercised on real content; final
     validate guards the post-export state). CONTRIBUTING gains one line:
     "run `npm run dogfood` when your PR changes extraction behavior; review
     the GLOSSARY.md diff".
2. `docs/validation/manual-protocol.md` — a runbook the owner executes on a
   private team repo (results stay private; only aggregate notes return):
   - Preconditions (Node, clone, build, link).
   - Session script (60–90 min): init → extract; record: total candidates,
     top-30 precision count (real term? y/n), top-5 missing expected terms
     (recall probes chosen beforehand by the owner), noisiest extractor;
     approve 10+ / reject 10+; re-extract (exclusion + drift check);
     hand-edit 2 definitions; export + build; open site, search 5 terms
     JA/EN, click through; validate.
   - Optional LLM step with an explicit safeguards CHECKLIST in the
     protocol (R11/§13-B4): owner's own endpoint/key; key via env var only;
     `llm.enabled: true` set consciously; `draft --dry-run` payload REVIEWED
     before any network call; deny-listed paths confirmed absent from the
     preview; then draft EXACTLY 3 terms (`maxTermsPerRun` friction noted);
     no payload or private snippet is ever copied into public records.
   - Recording template — the committed aggregate-only record at
     `docs/validation/manual-session-YYYY-MM-DD.md` with EXACTLY these
     fields: repo profile (size/lang mix, no name required), candidate
     count, top-30 precision as `n/30`, top-5 recall probes hit as `n/5`,
     noisiest extractor + example category (no verbatim private terms),
     approve/reject counts, re-extract exclusion OK (y/n), drift result,
     site search results `n/5`, validate result, verdict
     `accept / accept-with-tuning-issues / reject` + reasons.
   - Explicit sensitivity note: never paste private repo content into public
     issues; aggregate numbers only.
3. Acceptance session: owner runs the protocol (agent may assist). File
   resulting tuning issues with label `tuning` (create the label if absent),
   each containing: observed aggregate, expected behavior, affected
   extractor/config default, reproduction constraints (privacy), and a `U2`
   reference. v1 is DONE when the verdict is `accept` or all
   `accept-with-tuning-issues` issues are closed.

## Acceptance Criteria

- [ ] This repo's glossary/ committed: config + ≥15 curated terms + GLOSSARY.md; CI green with dogfood script.
- [ ] manual-protocol.md objective completeness: all required sections present, every command written out verbatim, the recording template contains every field listed in Req 2, the LLM safeguards checklist present, and a fixture dry-run transcript of the protocol is linked from the file.
- [ ] Owner session executed; `docs/validation/manual-session-<date>.md` committed with every template field filled; verdict recorded.
- [ ] Tuning issues filed per the Req 3 template with the `tuning` label (or "none needed" recorded).
- [ ] ISSUE_PLAN.md §1 completion statement satisfied — cross-check every issue closed; final comment lists the check.

## Validation

The executed protocol record IS the validation. Reviewer confirms the
completion-statement cross-check.

## Dependencies

28, 33, 38 (full product), 39, 40, 41 (docs/smoke land first so the session
exercises the final UX — mirrored in ISSUE_PLAN §3).

## Non-goals

Public Pages deployment of this repo's glossary (owner decision, separate),
performance tuning beyond noted issues, v2 planning.

## Design References

DESIGN.md §19, §16 (manual row), §3.4-U2, R12; ISSUE_PLAN.md §1/§6.
