# 28 — E2E harness + golden files for extract→curate→export; determinism test

## Title

End-to-end golden test harness over the fixture repos

## Summary

Build the e2e test harness of DESIGN.md §16: run the BUILT CLI against copies
of fixture repos through the full extract → approve/reject → validate →
export flow, compare against committed goldens, and enforce determinism.
Site build goldens and scans are issue 33's layer on top of this harness
(DESIGN §16 split).

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
   - `cli(cwd, args, opts?: {env?, expectExit?: number, allowStderrErrors?: boolean}): Promise<{exitCode: number, stdout: string, stderr: string, json?: unknown}>`
     — spawns `node dist/cli/main.js` with `--repo <cwd>`; `json` is the
     parsed stdout when `--json` was passed. The call FAILS the test when
     exitCode ≠ (expectExit ?? 0), or when stderr contains a line matching
     `/^error /m` and `allowStderrErrors` is not set.
   - Injectable clock via env `GLOSSARY_FAKE_NOW` — ADD this test-only seam
     by amending issue 03's `src/util/clock.ts` + its unit tests:
     `systemClock` reads the env var once at first use; accepted value =
     exactly the issue-03 `isoDateTime` format `YYYY-MM-DDTHH:mm:ssZ`;
     when set, both helpers derive from that fixed instant; unset ⇒ real
     time; any other value ⇒ `UsageError(E_USAGE, 'invalid GLOSSARY_FAKE_NOW')`.
     Unit tests: unset / valid / invalid.
   - Golden compare helper: byte compare with readable diff output; UPDATE
     mode via `GOLDEN_UPDATE=1`, which REFUSES to run when `CI=true`
     (update is a local-only workflow, documented in test/e2e/README.md).
2. Scenario `repo-ja-mixed` (the canonical run):
   1. `init` → assert created files.
   2. `extract` (fixed now) → golden `golden/ja-mixed.candidates.yaml`.
   3. Exact curation script (in this order):
      `approve 支払予約 --id payment-reservation --tag billing` (has doc
      suggestion), `approve 与信枠 --tag billing`, `approve slo`,
      `reject auth-timeout-ms --reason "constant, not a domain term"`.
   4. `extract` again → golden `ja-mixed.candidates2.yaml` (approved/rejected
      excluded; byte-stable).
   5. `export` → golden `ja-mixed.GLOSSARY.md`.
   6. `validate` → exit 0.
3. Determinism test (single CI job, no external reruns): fresh copy, run
   `extract` twice (same fake now) → byte-identical candidates.yaml; then the
   FULL scenario runs 3 times in-process and every run's outputs
   byte-compare equal to the goldens.
4. Scenario `repo-empty`: init+extract+export succeed; zero candidates;
   GLOSSARY.md "no terms" golden.
5. Scenario `repo-hostile` (pre-site part): extract completes; `--json`
   skipped buckets assert size ≥ 1, binary ≥ 1, symlink ≥ 1 (issue 24's
   shape), and every returned path stays under the repo copy; `<img…>` term
   string appears in candidates as plain text; export after approving the
   XSS-bait term → golden `hostile.GLOSSARY.md` with escaped output
   (extends in 33 for site).
6. CI: e2e runs in the matrix after build (02's `npm run test` includes it);
   temp dirs under os.tmpdir.

## Acceptance Criteria

- [ ] All goldens committed and green on Node 22/24/26, ubuntu+macos.
- [ ] GOLDEN_UPDATE workflow documented in test/e2e/README.md; update mode refuses under CI=true (tested).
- [ ] Determinism: 3 in-job scenario repetitions byte-match the goldens (single CI run proves it).
- [ ] Fake-now seam: unset/valid/invalid unit tests added to 03's suite (invalid ⇒ E_USAGE).
- [ ] Hostile scenario asserts the exact skip buckets and the escaped export bytes.
- [ ] `validate` exits 0 at scenario end (dependency 12).

## Validation

CI run link; intentionally perturb a fixture locally to show a golden diff
fails loudly (do not commit).

## Dependencies

12, 13, 24, 26, 27.

## Non-goals

Site goldens (33), LLM flows (38), performance benchmarking (24 owns budget),
coverage thresholds.

## Design References

DESIGN.md §16 (golden e2e + determinism rows), §4, §13 (AC3 assertions).
