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
   3. suggestedDefinition: first E4 definition by source priority order (22's
      ordering is encoded by array order — merge takes the first
      RawCandidateWithDef in input order after sorting raws by
      (extractor==doc-definitions first, path, line)); else if any E3
      expansion: format `"${expansion} の略。"` (ja) or
      `"Abbreviation of ${expansion}."` (en). Source: 'doc' in both cases
      (abbreviation expansions ARE doc evidence). Else null.
   4. Exclusions: key ∈ curatedKeys ⇒ drop (count into stats.curatedHits);
      key ∈ rejectedKeys ⇒ drop (stats.rejectedHits).
   5. Threshold: occurrences < minOccurrences ⇒ drop UNLESS the group contains
      an E4 raw or an E3 expansion (exemptions per §9.6/issues 21–22).
   6. Cap: sort (score desc, key asc) → slice maxCandidates
      (stats.capped = dropped count).
   7. Evidence: ≤ 5 sources — order raws by (extractor priority as in kind,
      then path, line), dedupe (path,line), take 5; snippets carried through.
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
- [ ] Definition priority: table-def beats とは-sentence when both exist for 締め処理-style case (construct raws in both orders — output identical due to internal sort).
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
