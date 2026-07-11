# 23 — Candidate merge, combined scoring, drift data

## Title

`src/extract/merge.ts`: merge RawCandidates into Candidate[], apply exclusions/thresholds, produce drift data

## Summary

Implement the merge stage of DESIGN.md §9.6 exactly: group by termKey, resolve
kind/score/surface, apply curated/rejected exclusion and thresholds, cap the
list, pick evidence, and compute drift keys for the report.

## Context

This is the convergence point of the pipeline; every rule here is part of the
frozen deterministic contract that goldens (28) protect.

## Scope

- Merge module + tests. No file I/O (24 wires stores).

## Detailed Requirements

1. Signature:
   ```ts
   mergeCandidates(raw: RawCandidate[], ctx: {
     curatedKeys: Set<string>, rejectedKeys: Set<string>,
     cfg: {minOccurrences, maxCandidates, definitionLanguage: 'ja'|'en'},
   }): { candidates: Candidate[], allKeys: Set<string>, stats: MergeStats }
   ```
2. Steps (§9.6 numbering):
   1. `key = termKey(normalizeSurface(raw.surface))`; drop empty keys.
   2. Group; per group compute:
      - kind: priority `doc-defined > domain > abbreviation > code` among
        contributing raws.
      - per-extractor score = max score among that extractor's raws;
        **normalization**: E1 raw FLR and E2 spread scores are used as-is;
        E3 = occurrences count; E4 = flat 5.0 (documented table in code).
      - merged score = max(per-extractor) + 0.5 × (distinctExtractors − 1),
        round half-up 2 decimals.
      - surface = most frequent raw surface; tie → compareCodepoint first.
        surfaces = distinct normalized-input surfaces sorted.
      - occurrences = raw count (per-occurrence emission contract).
   3. suggestedDefinition: among the group's E4 raws, pick the one with the
      highest-priority `definitionKind` (priority order = the union order in
      issue 22 / DESIGN §9.4; tie → (path, line) ascending) and use its
      `definition`; else, if any E3 raw carries `expansion`, format
      `"${expansion} の略。"` (ja) or `"Abbreviation of ${expansion}."` (en)
      per `cfg.definitionLanguage`. `suggestedDefinitionSource: 'doc'` in
      both cases (expansions are doc evidence — DESIGN §7.3/§9.6 step 7).
      Else null/null.
   4. Exclusions: key ∈ curatedKeys ⇒ drop (count into stats.curatedHits);
      key ∈ rejectedKeys ⇒ drop (stats.rejectedHits).
   5. Threshold: occurrences < minOccurrences ⇒ drop UNLESS the group contains
      an E4 raw or an E3 expansion (exemptions per §9.6/issues 21–22).
   6. Cap: sort (score desc, key asc) → slice maxCandidates
      (stats.capped = dropped count).
   7. Evidence: ≤ 5 sources — SELECT by ordering raws by (kind-priority
      extractor first, then path, line) and deduping (path, line); the final
      emitted `sources` array is then RE-SORTED by (path, line) per §7.3.
      Snippet hygiene enforced here before emission: strip control chars,
      cap 200 chars (the store re-asserts as last line of defense).
   8. Candidate mapping (every §7.3 field, explicit):
      `key` (step 1) · `surface`/`surfaces` (step 2) · `kind` (step 2) ·
      `score` = merged score rounded half-up to 2 decimals (a number
      satisfying `Number.isInteger(score*100)`) · `extractors` = contributing
      extractor ids sorted in the fixed order [ja-domain, identifiers,
      abbreviations, doc-definitions] · `occurrences` = deduped raw count ·
      `sources` (step 7) · `suggestedDefinition`/`suggestedDefinitionSource`
      (step 3).
   3′. `allKeys` = every key seen BEFORE exclusions/thresholds (drift input:
      24 compares curatedKeys against allKeys).
3. MergeStats: `{groups, emitted, curatedHits, rejectedHits, belowThreshold,
   capped}` for the extract summary.
4. Pure & deterministic; property test with shuffled input order ⇒ identical
   output.

## Acceptance Criteria

- [ ] Kind priority: group with E2+E4 raws ⇒ doc-defined; E1+E3 ⇒ domain.
- [ ] Score: constructed case where two extractors contribute → +0.5 bonus asserted; rounding half-up verified (e.g. 3.715 → 3.72).
- [ ] Surface tie-break and surfaces sorting asserted with JA variants (支払予約/支払い予約).
- [ ] Definition priority: a `table` E4 raw beats a `headingSection` raw for the same key regardless of input order (definitionKind priority; construct raws in both orders — identical output).
- [ ] Snippet hygiene: an over-long/control-char snippet in a raw is sanitized in the emitted Candidate.
- [ ] Evidence: selection by kind-priority, final sources sorted by (path, line) — asserted on a constructed multi-extractor group.
- [ ] Exclusions/threshold/exemptions/cap each unit-tested; stats counts exact.
- [ ] Shuffle-invariance property test (≥100 random permutations of a 20-raw corpus).

## Validation

`npm run test`.

## Dependencies

08, 09, 10, 11 (types/key sets), 19–22 (RawCandidate shapes).

## Non-goals

Store writes (24), synonym clustering across different keys (v2), FLR
recomputation (E1 owns it).

## Design References

DESIGN.md §9.6 (steps 1–7), §7.3 (output shape), §8 (T1/T4 semantics).
