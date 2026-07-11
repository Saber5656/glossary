# 24 — `glossary extract` orchestration, summary, drift report

## Title

`glossary extract`: full pipeline run, candidates rewrite, drift report, JSON summary

## Summary

Wire scanner → content extraction → tokenizers → E1–E4 → merge → candidates
store into the `extract` command per DESIGN.md §10.2, with the human summary
and `--json` contract.

## Context

This command is the product's heartbeat; its output diff is how teams review
extraction changes (ADR-002). It must be safe to run repeatedly and fast
enough for daily use.

## Scope

- `src/cli/commands/extract.ts` + `src/extract/pipeline.ts` (orchestrator) +
  integration tests. Stopword-file loading lives here.

## Detailed Requirements

1. Flow:
   1. `scanRepo` (14). 2. Read file contents (utf-8; single read per file).
   3. Per file kind: doc → markdown extractor (15); code → code extractor (16).
   4. Load user stopwords (config.extract.stopwordsPath; one term per line,
      `#` comments, termKey-normalized; missing file ⇒ UsageError
      E_CONFIG_INVALID naming the path).
   5. Get JaTokenizer (17; may be null).
   6. Run enabled extractors (config.extract.extractors.*.enabled) with their
      config thresholds: E1(19), E2(20), E3(21), E4(22).
   7. Read curated key set (09 indexes) + rejected key set (11).
   8. `mergeCandidates` (23) with definitionLanguage = config.llm.definitionLanguage.
   9. `writeCandidates` (10) with tool version + clock.
   10. Drift: `curatedKeys − allKeys` (term key or ANY alias key found ⇒ not
       drifted) → list of `{id, term}`.
2. Command contract: `glossary extract` accepts NO positional args; honors
   the global flags (`--dir`, `--repo`, `--json`, `--verbose`, `--no-color`);
   usage/config errors exit 2; under `--json`, stdout carries ONLY the JSON
   envelope — logs/warnings go to stderr (issue 06 conventions).
   Store discipline (B2): curated/rejected are read ONLY via the issue-09/11
   store APIs, candidates written ONLY via issue-10 `writeCandidates`; no
   direct YAML parse/serialize anywhere in this command.
3. Human output (stdout), exactly this shape:
   ```
   scanned 123 files (5 skipped: 2 size, 1 binary, 1 ignored, 1 symlink)
   extractors: ja-domain 45, identifiers 23, abbreviations 8, doc-definitions 6
   candidates: 61 written (12 below threshold, 3 curated, 2 rejected, 0 capped)
   drift: 1 curated term no longer found: 旧用語 (t-a1b2c3d4)
   ```
   (drift section omitted when empty; zero-count skip reasons omitted;
   counts from ScanResult + MergeStats.)
4. `--json` data (frozen; the envelope's `data` field — the envelope itself
   carries `warnings` = logger.warnings()):
   ```json
   {"scanned": n,
    "skipped": {"size": n, "binary": n, "ignored": n, "escape": n, "symlink": n, "unreadable": n},
    "extractorCounts": {"ja-domain": n, "identifiers": n, "abbreviations": n, "doc-definitions": n},
    "written": n, "dropped": {"belowThreshold": n, "curated": n, "rejected": n, "capped": n},
    "drift": [{"id": "...", "term": "..."}],
    "tokenizer": "kuromoji" | "heuristics-only"}
   ```
   ("ignored" aggregates the scanner's denylist/gitignore/exclude reasons.)
5. Drift (deterministic): computed from the FULL pre-drop key set (23's
   `allKeys`); a curated term drifts iff NEITHER its term key NOR any alias
   key is in allKeys; one entry per curated term, sorted by id
   (compareCodepoint).
6. Failure behavior: single unreadable file (EACCES etc.) ⇒ warning +
   SkipRecord reason `unreadable`, never aborts the run; store write errors
   abort with exit 1. Tokenizer fallback (17) emits its warning — visible in
   the envelope warnings.
7. Performance: the e2e test asserts repo-ja-mixed cold extract completes
   < 10 s (hard, CI-safe) and logs the measured duration; the 5 s target is
   informational. Dict-load ms logged at debug level.
8. `extract` writes ONLY candidates.yaml (ownership contract §6) — add an
   fs-spy test asserting no other write paths.

## Acceptance Criteria

- [ ] Integration (runCli) on repo-ja-mixed: exit 0; candidates.yaml exists, schema-valid, contains 支払予約 / payment reservation / SLO / 与信枠 with correct kinds; 無視用語 and DecoyTerm absent.
- [ ] Re-run without changes ⇒ candidates.yaml byte-identical except NOTHING (generatedAt uses injected clock in test ⇒ fully identical).
- [ ] Approve one term (via store call), re-extract ⇒ key excluded, drift empty; delete its source lines from fixture copy, re-extract ⇒ drift lists it.
- [ ] Reject a key ⇒ excluded on next run.
- [ ] `--json` snapshot matches frozen shape incl. `symlink`/`unreadable` buckets; human output matches template (regex-based assertions).
- [ ] Warnings: tokenizer-fallback run (bogus dict via seam) and an unreadable file both surface stable warning strings in the envelope `warnings`.
- [ ] Disabled extractor (config) produces zero of that kind and its count key = 0.
- [ ] Perf: e2e asserts < 10 s hard bound and logs duration.
- [ ] fs-spy: only candidates.yaml written.

## Validation

Integration suite green (perf bound is an automated assertion inside it).

## Dependencies

06, 14, 15, 16, 23 (and 17–22 transitively).

## Non-goals

Watch/incremental mode (v2), parallel file parsing (measure first; only
if budget fails), curated `sources` refresh (§8-T7: curated untouched).

## Design References

DESIGN.md §10.2, §9 (whole pipeline), §8 (T1/T4/T7), §6 (ownership), §4
(determinism).
