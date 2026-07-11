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
2. `readRejected(glossaryDir)` — missing file ⇒ empty list (init normally
   creates it; tolerate absence). Schema failure ⇒ `E_SCHEMA_INVALID` with
   hint to fix by hand or re-init (file is small and human-readable).
3. `addRejected(glossaryDir, entries: {key, reason?}[], clock)`:
   - Keys are stored ALREADY termKey-normalized (callers pass raw; this module
     applies `termKey` — single place).
   - Idempotent: existing key updates `reason` only if a new one is given;
     `rejectedAt` preserved. Returns `{added: string[], updated: string[],
     unchanged: string[]}`.
   - Write sorted by key (compareCodepoint), header
     `# Managed by 'glossary reject' — edit via CLI`.
4. `removeRejected(glossaryDir, keys: string[])` — normalize, remove; returns
   `{removed: string[], notFound: string[]}`; file rewritten sorted.
5. `rejectedKeySet(glossaryDir): Set<string>`.

## Acceptance Criteria

- [ ] Schema tests (strict, date format, null reason).
- [ ] add: new + duplicate + reason-update paths return the right buckets; file byte-golden and sorted after mixed JA/EN inserts.
- [ ] remove: existing + missing keys; empty-file write remains schema-valid (`rejected: []`).
- [ ] Normalization: adding `「システム」` then removing `システム` works (same key).
- [ ] Round-trip read/write deterministic.

## Validation

`npm run test`.

## Dependencies

04, 08.

## Non-goals

reject CLI UX (26), matching against candidates (23/24 consume the key set).

## Design References

DESIGN.md §7.5, §8 (T3/T5), §9.6 step 5.
