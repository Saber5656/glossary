# 18 — Identifier splitter (camel/snake/kebab/acronym)

## Title

`src/tokenize/identifier.ts`: `splitIdentifier` with acronym and digit handling

## Summary

Implement the deterministic identifier-to-words splitter of DESIGN.md §9.3,
used by E2 (phrase building) and the search tokenizer (29).

## Context

Splitting quality directly drives identifier-term readability
(`PaymentReservation` → "payment reservation"). The rules below are frozen so
goldens stay stable.

## Scope

- One pure module + exhaustive table-driven tests.

## Detailed Requirements

0. Input guard FIRST: slice input to its first 256 UTF-16 code units before
   ALL processing (no error). Implementation must be a single linear scan or
   bounded linear regexes only (B2' discipline); adversarial tests below.
1. `splitIdentifier(name: string): string[]` rules, applied in order:
   1. Split on separators = ASCII characters outside `[A-Za-z0-9]`
      (`_`, `-`, `$`, `.`, whitespace, etc.); non-ASCII characters are TOKEN
      characters, not separators; drop empty segments.
   2. Within each segment, split camel boundaries:
      - lower→Upper (`paymentReservation` → payment|Reservation)
      - Upper-run→Upper+lower keeps acronym: `HTTPServer` → HTTP|Server;
        `parseJSONValue` → parse|JSON|Value.
      - letter↔digit boundaries: `utf8Decoder` → utf|8|Decoder; digit runs are
        their own token.
   3. Lowercase all tokens (`HTTP` → `http`).
   4. Return in order; no dedup.
2. `identifierPhrase(name): string` = tokens joined with single spaces
   (`PaymentReservation` → `payment reservation`); used by E2 as candidate
   surface.
3. Detailed API (exact):
   ```ts
   export type IdentifierToken = { raw: string; lower: string }
   export function splitIdentifierDetailed(name: string): IdentifierToken[]
   // splitIdentifier(name) === splitIdentifierDetailed(name).map(t => t.lower)
   export function isLikelyAcronymToken(tok: IdentifierToken): boolean
   //   === /^[A-Z][A-Z0-9]{1,5}$/.test(tok.raw)
   ```
   Token order in `splitIdentifierDetailed` is source order, identical to
   `splitIdentifier`.
4. (Truncation is requirement 0 — before all processing.)
5. No locale APIs. Non-ASCII runs pass through as single tokens
   (`名前` → [`名前`]), lowercased via the same `toLowerCase()` as everything
   else.

## Acceptance Criteria

- [ ] Table-driven tests (minimum set): `PaymentReservation`→[payment,reservation]; `creditLimit`→[credit,limit]; `AUTH_TIMEOUT_MS`→[auth,timeout,ms]; `HTTPServer`→[http,server]; `parseJSONValue`→[parse,json,value]; `create_payment_reservation`→[create,payment,reservation]; `kebab-case-name`→[kebab,case,name]; `utf8Decoder`→[utf,8,decoder]; `X`→[x]; `__private`→[private]; `名前`→[名前].
- [ ] `identifierPhrase` examples asserted.
- [ ] Detailed variant preserves raw `HTTP` for acronym detection; `isLikelyAcronymToken` true for raw `HTTP`/`SLO2`, false for `Http`/`H`/`TOOLONGX`.
- [ ] Fuzz: 500 identifiers from a seeded PRNG (mulberry32, seed 42), alphabet `[A-Za-z0-9_$.-]`, lengths 1–64 — invariant: output tokens rejoin (ignoring separators/case) to the alnum content of the (truncated) input.
- [ ] Adversarial perf: 10 inputs of 10_000 chars (all separators / all caps / alternating case) complete in < 100 ms total (truncation makes this trivial — the test guards the requirement-0 ordering).

## Validation

`npm run test`.

## Dependencies

01, 03, 08.

## Non-goals

Spelling correction, abbreviation expansion (E3), natural-language stemming.
`searchTokenize` is issue 29's module — this issue only exports identifier
helpers that 29 reuses.

## Design References

DESIGN.md §9.3 (splitIdentifier), §9.4-E2; research/client-side-search.md §3
(shared with search tokenization).
