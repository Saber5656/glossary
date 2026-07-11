# 11 — Rejected list store (`rejected.yaml`) and schema

## Title

`src/store/rejected.ts`: rejected-keys schema, add/remove operations, key set

## Summary

Implement the rejected tier per DESIGN.md §7.5: a small human-owned (via CLI)
list of term keys that extraction must permanently exclude until removed.

## Context

Rejection is what makes re-extraction livable (noise doesn't resurface).
Operations must be idempotent and keep the file sorted for clean diffs.

## Scope

- Schema + store + unit tests.

## Detailed Requirements

1. Zod schema per §7.5, `.strict()`: `schemaVersion: 1`;
   `rejected: {key: string (non-empty), reason: string|null, rejectedAt: YYYY-MM-DD}[]`.
   ALL reads/writes go through yaml-io (issue 04) — no direct `yaml` import,
   no ad-hoc fs serialization; `E_YAML_*` errors propagate unchanged.
   Serialized order: top-level `schemaVersion` then `rejected`; per entry
   `key`, `reason`, `rejectedAt`.
2. `readRejected(glossaryDir)` — missing file ⇒ empty list (init normally
   creates it; tolerate absence). zod failure ⇒ `E_SCHEMA_INVALID` with
   hint to fix by hand or re-init (file is small and human-readable).
3. `addRejected(glossaryDir, entries: {key: string, reason?: string | null}[], clock)`:
   - Keys are stored ALREADY termKey-normalized (callers pass raw; this module
     applies `termKey` — single place). A key that is EMPTY after
     normalization ⇒ `UsageError(E_INVALID_KEY)` naming the original input;
     nothing is written (add `E_INVALID_KEY` to 03's ErrorCodes).
   - Duplicates within one call: normalize first, process in input order —
     each normalized key ends in exactly one return bucket; the last
     non-undefined `reason` wins before writing.
   - Reason semantics: NEW key — omitted/undefined stores `null`; EXISTING
     key — undefined leaves reason unchanged, a string updates it, explicit
     `null` clears it. `rejectedAt` of existing keys is always preserved.
   - Returns `{added: string[], updated: string[], unchanged: string[]}`.
   - Write sorted by key (compareCodepoint), header
     `Managed by 'glossary reject' — edit via CLI`.
4. `removeRejected(glossaryDir, keys: string[])` — normalize (same
   `E_INVALID_KEY` rule), remove; returns `{removed: string[], notFound:
   string[]}`; file rewritten sorted.
5. `rejectedKeySet(glossaryDir): Set<string>`.
6. Scope guard: this module mutates ONLY `rejected.yaml`. Candidate existence
   checks, `--force`, and removal from `candidates.yaml` are issue 26's work.

## Acceptance Criteria

- [ ] Schema tests (strict, date format, null reason).
- [ ] add: new + duplicate-in-call + reason-update + reason-clear (`null`) + reason-unchanged (undefined) paths return the right buckets; file byte-golden (incl. field order) and sorted after mixed JA/EN inserts.
- [ ] Empty-after-normalization key (`「」`) ⇒ `E_INVALID_KEY`, zero writes.
- [ ] remove: existing + missing keys; empty-file write remains schema-valid (`rejected: []`).
- [ ] Normalization: adding `「システム」` then removing `システム` works (same key).
- [ ] Round-trip read/write deterministic; module never touches candidates.yaml (fs-spy).

## Validation

`npm run test`.

## Dependencies

04, 08.

## Non-goals

reject CLI UX (26), matching against candidates (23/24 consume the key set).

## Design References

DESIGN.md §7.5, §8 (T3/T5), §9.6 step 5.
