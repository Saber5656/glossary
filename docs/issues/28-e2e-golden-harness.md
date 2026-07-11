# 28 — E2E harness + golden files for extract→curate→export; determinism test

## Title

End-to-end golden test harness over the fixture repos

## Summary

Build the e2e test harness of DESIGN.md §16: run the BUILT CLI against copies
of fixture repos through the full extract → approve/reject → export flow,
compare against committed goldens, and enforce determinism.

## Context

Goldens freeze the pipeline's observable behavior; from this issue on, any
behavior change is a reviewed golden diff, not an accident. Site (33) and LLM
(38) e2e extend this harness.

## Scope

- `test/e2e/harness.ts`, `test/e2e/pipeline.e2e.test.ts`,
  `test/e2e/golden/` files, npm script `test:e2e` (included in `test`).

## Detailed Requirements

1. Harness:
   - `withFixtureCopy(name, fn)`: copy `fixtures/<name>` to a temp dir
     (preserving symlinks), yield path, always cleanup.
   - `cli(cwd, args, env?)`: spawn `node dist/cli/main.js` with
     `--repo <cwd>`; capture stdout/stderr/exit; fail test on unexpected
     stderr errors. Injectable clock via env `GLOSSARY_FAKE_NOW` — ADD this
     test-only seam: `src/util/clock.ts` systemClock checks the env var
     (ISO string) and, when set, returns that fixed time (documented as
     test-only; acceptable because output timestamps must be controllable
     in e2e; refuse non-ISO values).
   - Golden compare helper: byte compare with readable diff output; UPDATE
     mode via `GOLDEN_UPDATE=1`.
2. Scenario `repo-ja-mixed` (the canonical run):
   1. `init` → assert created files.
   2. `extract` (fixed now) → golden `golden/ja-mixed.candidates.yaml`.
   3. `approve 支払予約 --id payment-reservation --tag billing`,
      `approve payment-reservation`? (key for E2 phrase: `payment-reservation`
      termKey) — script approves 3 known keys incl. one with doc suggestion;
      `reject システム` style one key with reason.
   4. `extract` again → golden `ja-mixed.candidates2.yaml` (approved/rejected
      excluded; byte-stable).
   5. `export` → golden `ja-mixed.GLOSSARY.md`.
   6. `validate` → exit 0.
3. Determinism test: fresh copy, run `extract` twice (same fake now) → the two
   candidates.yaml files byte-identical; and full scenario re-run ⇒ all
   goldens match again.
4. Scenario `repo-empty`: init+extract+export succeed; zero candidates;
   GLOSSARY.md "no terms" golden.
5. Scenario `repo-hostile` (pre-site part): extract completes; skipped-file
   assertions (size/binary/symlink) via `--json`; `<img…>` term string appears
   in candidates as plain text; export after approving the XSS-bait term →
   golden `hostile.GLOSSARY.md` with escaped output (extends in 33 for site).
6. CI: e2e runs in the matrix after build (02's `npm run test` includes it);
   temp dirs under os.tmpdir.

## Acceptance Criteria

- [ ] All goldens committed and green on Node 22/24/26, ubuntu+macos.
- [ ] GOLDEN_UPDATE workflow documented in test/e2e/README.md (3 lines).
- [ ] Determinism assertions pass 3 consecutive CI runs (link).
- [ ] Fake-now seam refuses invalid ISO and is inert when unset (unit test in 03's suite — amend it here).
- [ ] Hostile scenario asserts the exact skip reasons and the escaped export bytes.

## Validation

CI run link; intentionally perturb a fixture locally to show a golden diff
fails loudly (do not commit).

## Dependencies

13, 24, 26, 27.

## Non-goals

Site goldens (33), LLM flows (38), performance benchmarking (24 owns budget),
coverage thresholds.

## Design References

DESIGN.md §16 (golden e2e + determinism rows), §4, §13 (AC3 assertions).
