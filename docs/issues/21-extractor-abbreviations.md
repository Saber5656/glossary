# 21 — Extractor E3: abbreviations and expansion pairing

## Title

`src/extract/e3-abbreviations.ts`: ALL-CAPS/katakana abbreviation candidates with expansion detection

## Summary

Implement extractor E3 per DESIGN.md §9.4-E3 over doc text, code comments, and
identifier acronym tokens, detecting `Full Phrase (ABBR)` style expansions.

## Context

Team jargon is often abbreviations (R1: 略語・チーム内用語). Expansion pairing
turns a cryptic token into a self-explanatory candidate.

## Scope

- E3 module + built-in abbreviation stoplist + tests.

## Detailed Requirements

1. Signature:
   `extractAbbreviations(inputs: {blocks: PositionedText[], comments: PositionedText[], acronymTokens: {raw, path, line}[]}, cfg: {minOccurrences, stopwords: Set<string>}): RawCandidate[]`.
2. Candidate detection (kind `abbreviation`):
   - ALL-CAPS tokens `\b[A-Z][A-Z0-9]{1,5}\b` in prose/comments (2–6 chars),
     excluding tokens immediately preceded/followed by `_` (identifier parts
     come via acronymTokens instead).
   - Acronym tokens from identifiers (18's `isLikelyAcronymToken` output,
     provided by the pipeline).
   - Katakana short forms: katakana runs of 3–5 chars that also appear as a
     strict PREFIX of a longer katakana run elsewhere in the corpus
     (`オーソリ` ⊂ `オーソリゼーション`) — pair them (expansion = the longer
     form).
3. Expansion pairing (sets `snippet` to the pairing evidence and records the
   expansion): patterns searched within the same text block —
   - `Full Phrase (ABBR)`: `([A-Z][A-Za-z]+(?: [A-Za-z][A-Za-z]+){0,6}) \((ABBR)\)`
     where initials of the phrase words (case-insensitive) spell ABBR.
   - `ABBR（日本語正式名）` / `ABBR (日本語正式名)`: ABBR followed by a
     parenthesized non-ASCII phrase ≤ 30 chars.
   - Katakana prefix rule above.
   When found, RawCandidate gains `expansion: string` (extend RawCandidate
   with optional field; merge (23) copies the first expansion into
   `suggestedDefinition` ONLY IF E4 supplied none, formatted as
   `"<expansion> の略。"` (ja) / `"Abbreviation of <expansion>."` (en by
   definitionLanguage) — implement the formatting in 23, E3 only records
   `expansion`).
4. Stoplist `src/extract/stopwords-abbr.ts` (≥ 40): HTTP, HTTPS, HTML, CSS,
   JSON, YAML, XML, API, URL, URI, UUID, ID, DB, SQL, CLI, GUI, UI, UX, OS,
   CPU, GPU, RAM, TCP, UDP, IP, DNS, TLS, SSL, SSH, README, TODO, FIXME, NOTE,
   WARN, INFO, DEBUG, ERROR, OK, NG, PR, CI, CD, npm-ish tokens (NPM), GET,
   POST, PUT, DELETE, PATCH. User stopwords merge (same file as E1's user
   list — a single user stopword file feeds all extractors).
5. minOccurrences (default 2) applied per surface across the corpus BEFORE
   emission; paired-with-expansion candidates are exempt (explicit definition
   evidence beats frequency, mirroring E4's exemption).
6. Determinism: outputs sorted (surface, path, line).

## Acceptance Criteria

- [ ] repo-ja-mixed: emits `SLO` with expansion `Service Level Objective` (pairing pattern) and `オーソリ` with expansion `オーソリゼーション`; does NOT emit HTTP/API-style stoplisted tokens (add 2 to fixture if absent).
- [ ] Initials check: `Central Processing Unit (CPU)` pairs but CPU still dropped by stoplist (pairing≠bypass stoplist — stoplist wins; test).
- [ ] `AUTH` from AUTH_TIMEOUT_MS arrives via acronymTokens path and counts occurrences with prose `AUTH` mentions.
- [ ] Single occurrence without expansion dropped; single WITH expansion kept.
- [ ] Determinism double-run; regex linearity documented.

## Validation

Unit tests + fixture output table in PR.

## Dependencies

08, 15, 16, 18 (acronym tokens); fixtures 13.

## Non-goals

Cross-document coreference, nested abbreviation chains, JA kanji abbreviations
(第一四半期→Q1 style) — v2.

## Design References

DESIGN.md §9.4-E3; §9.6 step 7 (expansion→suggestedDefinition rule executes in 23).
