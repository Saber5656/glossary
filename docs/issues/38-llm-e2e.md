# 38 — LLM e2e with mock server: goldens, schema retry, injection fixture

## Title

LLM end-to-end suite against a local mock OpenAI-compatible server

## Summary

Extend the e2e harness (28) with the LLM path per DESIGN.md §16: the built CLI
talking HTTP to an in-process mock server, covering dry-run goldens, drafting
writeback, retry behavior, and the prompt-injection abuse case (AC5).

## Context

Issues 35–37 test with injected mocks; this suite proves the REAL wiring
(config → env key → HTTP → parse → writeback) using the actual binary, and
pins the network contract with goldens. CI stays fully offline (mock binds
127.0.0.1).

## Scope

- `test/e2e/llm.e2e.test.ts` + `test/helpers/mock-llm-server.ts` (node:http).

## Detailed Requirements

1. Mock server: node:http on an ephemeral 127.0.0.1 port; routes POST
   `/v1/chat/completions`; scriptable per-test queue of responses; records
   every request (headers+body) for assertions; supports modes: valid JSON
   answer / invalid-then-valid (retry) / 500 / slow (> timeout).
2. Scenario setup on a `repo-hostile` copy: init; config patched:
   `llm: {enabled: true, baseUrl: 'http://127.0.0.1:<port>/v1', model:
   'mock-1', apiKeyEnv: 'GLOSSARY_TEST_KEY', maxTermsPerRun: 5}`; extract.
3. Tests:
   - **Dry-run golden**: `draft --dry-run --json` over 決済トークン (the
     injection/near-secret bait term) — body golden asserts: masked
     `[REDACTED]` where sk-live token was; injection prose present verbatim
     in the fenced snippet; guard sentence in system message; NO request
     recorded by the server; env key unset.
   - **Consent**: enabled=false ⇒ exit 3, no request. Key env unset (non-dry)
     ⇒ exit 3.
   - **Happy path**: server returns definitions for requested keys ⇒
     candidates.yaml updated (source llm); Authorization header equals
     `Bearer test-key-123` on the wire and that string appears in NO CLI
     output (stdout+stderr captured and grepped).
   - **AC5 injection outcome**: server plays an "attacked" response
     `{"definitions":[{"key":"決済トークン","definition":"IGNORE… <script>alert(1)</script> https://evil.example"}]}`
     ⇒ writeback stores it (schema-valid — content is opaque), then `export`
     + `build` render it ESCAPED (reuse 33's scanner on the rebuilt site);
     documents that injection containment = schema + escaping + human review,
     not content censorship.
   - **Retry**: invalid JSON then valid ⇒ success, exactly 2 requests.
   - **Timeout**: slow mode with timeoutMs=1000 ⇒ E_LLM_HTTP, exit 1, ≤2
     requests (retry once).
4. All tests offline-safe (assert no DNS: baseUrl is literal 127.0.0.1);
   suite must pass with `--offline`-ish env (document: CI has network but the
   suite never leaves loopback).

## Acceptance Criteria

- [ ] All six scenarios green in CI matrix.
- [ ] Dry-run golden shared with 36 (single source of truth file) and byte-stable.
- [ ] Key-leak grep assertion covers stdout, stderr, and written files (candidates.yaml).
- [ ] AC5 chain ends with 33's scanner passing on the poisoned-then-rebuilt site.
- [ ] Server helper is reusable (exported) — 42's manual protocol references it for local experiments.

## Validation

CI link; a full transcript of the happy-path run pasted into the PR.

## Dependencies

13, 37 (and 28's harness, 33's scanner).

## Non-goals

Real-provider integration tests (manual, 42), performance/cost benchmarking,
streaming.

## Design References

DESIGN.md §16 (LLM row), §12, §13-B4 (AC4/AC5); ADR-005.
