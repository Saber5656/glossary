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

1. `splitIdentifier(name: string): string[]` rules, applied in order:
   1. Split on `_`, `-`, `$`, `.`, and any non-alphanumeric separator; drop
      empty segments.
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
3. `isLikelyAcronymToken(tok): boolean` — original segment was an
   all-caps run of length 2–6 (E3 helper; expose original-case tokens via a
   second return shape: `splitIdentifierDetailed(name): {lower: string, raw: string}[]`).
4. Guard: names longer than 256 chars → truncate with no error (defensive).
5. No locale APIs; ASCII-only logic (identifiers with non-ASCII pass through
   as single tokens, lowercased where applicable).

## Acceptance Criteria

- [ ] Table-driven tests (minimum set): `PaymentReservation`→[payment,reservation]; `creditLimit`→[credit,limit]; `AUTH_TIMEOUT_MS`→[auth,timeout,ms]; `HTTPServer`→[http,server]; `parseJSONValue`→[parse,json,value]; `create_payment_reservation`→[create,payment,reservation]; `kebab-case-name`→[kebab,case,name]; `utf8Decoder`→[utf,8,decoder]; `X`→[x]; `__private`→[private]; `名前`→[名前].
- [ ] `identifierPhrase` examples asserted.
- [ ] Detailed variant preserves raw `HTTP` for acronym detection.
- [ ] Fuzz: 500 random ASCII identifiers — output tokens rejoin (ignoring separators/case) to the alnum content of input.

## Validation

`npm run test`.

## Dependencies

01, 03, 08 (uses compareCodepoint? not needed — then just 01, 03; keep 08 out
if unused).

## Non-goals

Spelling correction, abbreviation expansion (E3), natural-language stemming.

## Design References

DESIGN.md §9.3 (splitIdentifier), §9.4-E2; research/client-side-search.md §3
(shared with search tokenization).
