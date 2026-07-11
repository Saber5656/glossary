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

1. Consent gate (checked in order, each with its own message):
   1. `llm.enabled !== true` ⇒ ValidationFailed E_LLM_CONSENT (exit 3), hint:
      set `llm.enabled: true` in glossary/config.yaml.
   2. `llm.model` empty ⇒ E_LLM_CONSENT.
   3. Env `config.llm.apiKeyEnv` unset/empty ⇒ E_LLM_CONSENT naming the VAR
      NAME only — **skipped when `--dry-run`**.
2. Target resolution (mutually exclusive; none given ⇒ UsageError):
   - `draft <key...>`: candidate keys (termKey-normalized, must exist).
   - `draft --all-pending`: all candidates with `suggestedDefinition == null`.
   - `draft --curated <id...>`: curated terms with EMPTY definition (non-empty
     ⇒ per-id warning + skip; never overwrite, §12.4).
   - Apply `maxTermsPerRun` truncation with warning listing dropped keys.
3. Notice line (non-dry runs) to stderr BEFORE any request:
   `llm: sending N terms (M snippets) to <host> model <model>` (§12.1).
4. `--dry-run`: print每 batch PreviewEntry (36): endpoint, model, byteSize,
   pretty-printed body; `--json` ⇒ `{batches: PreviewEntry[]}` (body as
   string). Exit 0. NO network, NO key requirement, NO writes.
5. Execution: per batch (36) → `chat` (35, jsonMode: true) → parse content as
   JSON → zod `{definitions: [{key: string, definition: string}]}`:
   - Parse/schema failure ⇒ ONE repair retry: same request + user-message
     suffix `Return ONLY the JSON object.`; second failure ⇒ batch fails
     (E_LLM_SCHEMA warning), other batches continue; ≥1 failed batch ⇒ exit 1
     at end (partial writes stand — idempotent to re-run).
   - Returned keys must ⊆ requested; unknown keys ignored with warning.
   - Definition post-processing: trim, strip control chars, collapse
     whitespace, cap 1000 chars (§12.4).
6. Writeback:
   - Candidate targets: `updateSuggestedDefinitions` (10) with source 'llm'.
   - Curated targets: set `definition`, `definitionSource: 'llm'`,
     `updatedAt` = clock date, via `writeTerm` overwrite mode (09).
7. Output human: per-term line `drafted 支払予約 (142 chars)` + summary
   `drafted X, skipped Y (no safe evidence), failed Z`; `--json`:
   `{drafted: [{key|id, chars}], skipped: [...], failed: [...], dryRun: bool}`.
8. Secret hygiene: command never prints the key (35's scrub + no env echo);
   `--verbose` prints request metadata only, never bodies (bodies are
   dry-run's job).

## Acceptance Criteria

- [ ] Consent matrix tests: each gate fires with exact code/message; dry-run works with no key set.
- [ ] Dry-run golden: matches 36's committed payload golden byte-exact; fs-spy asserts zero writes; no fetch called (mock asserts).
- [ ] Happy path (mock): candidates updated with source llm; curated empty-def filled with updatedAt bump; non-empty curated skipped with warning.
- [ ] Repair-retry path: first response invalid JSON, second valid ⇒ success; both invalid ⇒ exit 1, other batch still written.
- [ ] maxTermsPerRun truncation warning lists dropped keys; batches of ≤10 asserted.
- [ ] Unknown returned key ignored+warned; requested-but-missing key counted as failed.

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
