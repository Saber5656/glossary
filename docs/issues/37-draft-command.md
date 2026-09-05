# 37 — `glossary draft`: consent gate, dry-run preview, batching, writeback

## Title

`glossary draft`: LLM definition drafting flow (T8/T9) with consent enforcement

## Summary

Implement the `draft` command per DESIGN.md §10.11/§12: enforce the consent
gate, resolve targets, run redacted batches through the client, validate
responses, and write results with `llm` provenance.

## Context

The only command with network egress. Its refusal paths are as much a part of
the product as its success path (R11); its writeback rules keep human
curation authoritative (T8/T9).

## Scope

- `src/cli/commands/draft.ts` + response-schema module + tests (client
  mocked). Real-network e2e is 38.

## Detailed Requirements

1. Consent gate (checked in order; all three are ValidationFailed
   E_LLM_CONSENT, exit 3, with these exact messages):
   1. `llm.enabled !== true` ⇒ message `llm drafting is disabled`, hint
      `set 'llm.enabled: true' in glossary/config.yaml`.
   2. `llm.model` empty ⇒ message `llm.model is not set`, hint
      `set 'llm.model' in glossary/config.yaml`.
   3. Env `config.llm.apiKeyEnv` unset/empty ⇒ message
      `API key not found in $<VARNAME>` (the variable NAME only — its value
      never appears anywhere), hint `export <VARNAME>=…` — **skipped when
      `--dry-run`**.
2. Target resolution (the three forms are mutually exclusive — combining
   them or giving none ⇒ UsageError E_USAGE):
   - `draft <key...>`: candidate keys (termKey-normalized; all-or-nothing —
     any unknown key ⇒ UsageError E_KEY_NOT_FOUND, nothing sent). Explicit
     keys DO overwrite an existing non-null `suggestedDefinition` (the user
     asked for exactly these keys).
   - `draft --all-pending`: all candidates with `suggestedDefinition == null`
     (never overwrites).
   - `draft --curated <id...>`: curated ids (all-or-nothing on unknown ids ⇒
     E_KEY_NOT_FOUND); ids with EMPTY definition are drafted; non-empty ⇒
     per-id warning + skip (never overwrite, §12.4).
   - Zero targets after resolution/skips ⇒ exit 0 with `nothing to draft`
     (not an error).
   - THEN apply `maxTermsPerRun` truncation with a warning listing dropped
     keys (this command owns the cut; 36 receives the already-limited list).
   - Map targets into 36's `PayloadTermInput`; provide the read callback with
     repoRoot containment + no-symlink-follow guarantees (36 Req 3).
3. Notice line (non-dry runs) to stderr BEFORE any request:
   `llm: sending N terms (M snippets) to <host> model <model>` (§12.1).
4. `--dry-run`: print per batch the PreviewEntry (36): endpoint, model,
   byteSize, and the EXACT `body` string — labeled but NEVER reserialized or
   reformatted (DESIGN §12.3: the preview must be byte-identical to the wire
   body). `--json` ⇒ `{batches: PreviewEntry[]}` (body as the same string).
   Exit 0. NO network, NO key requirement, NO writes.
5. Execution: per batch (36) → `chat` (35, jsonMode: true) → parse content as
   JSON → zod `{definitions: [{key: string, definition: string}]}`:
   - Parse/schema failure ⇒ ONE repair retry: same request + user-message
     suffix `Return ONLY the JSON object.`; second failure ⇒ batch fails
     (E_LLM_SCHEMA warning), other batches continue; ≥1 failed batch ⇒ exit 1
     at end (partial writes stand — idempotent to re-run).
   - Transport failures (35's final E_LLM_HTTP incl. timeout): that batch's
     requested keys are marked failed with the SCRUBBED error message as
     reason; other batches continue; ≥1 failed batch ⇒ exit 1 (DESIGN §12.5).
   - Returned keys must ⊆ requested; unknown keys ignored with warning;
     requested-but-missing keys counted as failed (`reason: 'not returned'`).
   - Definition post-processing: trim, strip control chars, collapse
     whitespace, cap 1000 chars (§12.4).
6. Writeback:
   - Candidate targets: `updateSuggestedDefinitions` (10) with source 'llm'.
   - Curated targets: set `definition`, `definitionSource: 'llm'`,
     `updatedAt` = clock date, via `writeTerm` overwrite mode (09).
7. Output human: per-term line `drafted 支払予約 (142 chars)` + summary
   `drafted X, skipped Y, failed Z`; `--json` data (frozen):
   ```json
   {"drafted": [{"type": "candidate", "key": "支払予約", "chars": 142},
                {"type": "curated", "id": "payment-reservation", "chars": 98}],
    "skipped": [{"type": "candidate", "key": "…", "reason": "no safe evidence"}],
    "failed":  [{"type": "candidate", "key": "…", "reason": "…scrubbed…"}],
    "dryRun": false}
   ```
   (`key` present iff type candidate; `id` iff curated.)
8. Secret hygiene: command never prints the key (35's scrub + no env echo);
   `--verbose` prints request metadata only, never bodies (bodies are
   dry-run's job).

## Acceptance Criteria

- [ ] Consent matrix tests: each of the three gates fires with the exact message/hint above; dry-run works with no key set.
- [ ] Dry-run golden: body strings match 36's committed payload golden byte-exact (no reformatting); fs-spy asserts zero writes; no fetch called (mock asserts).
- [ ] Happy path (mock): candidates updated with source llm; explicit key overwrites an existing suggestion; --all-pending does not; curated empty-def filled with updatedAt bump; non-empty curated skipped with warning.
- [ ] Flag exclusivity: `draft key --all-pending`, `draft --all-pending --curated x`, and bare `draft` each exit 2 E_USAGE; unknown key/id ⇒ exit 2 E_KEY_NOT_FOUND with zero requests; zero-target run exits 0 with `nothing to draft`.
- [ ] Repair-retry path: first response invalid JSON, second valid ⇒ success; both invalid ⇒ exit 1, other batch still written. Transport-failure batch (mock 500×2) ⇒ its keys failed with scrubbed reason, exit 1.
- [ ] maxTermsPerRun truncation warning lists dropped keys; batches of ≤10 asserted.
- [ ] Unknown returned key ignored+warned; requested-but-missing key counted as failed; `--json` snapshot matches the frozen shape.

## Validation

runCli tests with injected mock client; consent-refusal transcript pasted in
PR.

## Dependencies

06, 09, 10, 35, 36.

## Non-goals

Interactive confirmation prompts (ADR-005 rejected), streaming progress,
auto-approve of drafts (never), curated non-empty overwrite.

## Design References

DESIGN.md §10.11, §12.1–12.5, §8 (T8/T9); ADR-005; R11.
