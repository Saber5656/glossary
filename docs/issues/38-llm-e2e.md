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

1. Mock server (`node:http` on 127.0.0.1 — a SPAWNED CLI process cannot be
   intercepted by undici MockAgent, hence a real loopback server for e2e;
   DESIGN §16 records this two-level split). Exact helper API:
   ```ts
   type MockLlmResponse =
     | { kind: 'json', definitions: {key: string, definition: string}[] }
     | { kind: 'raw', status: number, body: string }
     | { kind: 'invalid-json' }
     | { kind: 'slow', delayMs: number }
   type RecordedRequest = { headers: Record<string,string>, body: string }
   startMockLlmServer(script: MockLlmResponse[]): Promise<{
     baseUrl: string,            // http://127.0.0.1:<port>/v1
     requests: RecordedRequest[],
     close(): Promise<void>,     // always awaited in afterEach
   }>
   ```
   Responses are consumed from `script` in order; an exhausted script
   returns 500 (fails the test loudly).
2. Scenario setup on a `repo-hostile` copy: init; config patched:
   `llm: {enabled: true, baseUrl: 'http://127.0.0.1:<port>/v1', model:
   'mock-1', apiKeyEnv: 'GLOSSARY_TEST_KEY', maxTermsPerRun: 5}`; extract.
3. Tests:
   - **Dry-run golden**: `draft --dry-run --json` over 決済トークン (the
     injection/near-secret bait term) — body golden asserts: masked
     `[REDACTED]` where the sk-live token was; the `secrets/.env` content
     (incl. its 決済トークン comment line) absent from the payload
     (deny-listed path — AC4); injection prose present verbatim in the
     fenced snippet; guard sentence in system message; NO request recorded
     by the server; env key unset.
   - **Consent**: enabled=false ⇒ exit 3, no request. Key env unset (non-dry)
     ⇒ exit 3. (Two separate named tests.)
   - **Happy path**: server returns definitions for requested keys ⇒
     candidates.yaml updated (source llm); Authorization header equals
     `Bearer test-key-123` on the wire; that key string appears in NO CLI
     output (stdout+stderr) AND in NO recorded request BODY
     (`JSON.stringify(body)` contains neither `test-key-123` nor `Bearer`)
     — the key must never enter model context (AC5 guarantee).
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

- [ ] Exactly these named tests green in CI: `dry-run-golden`, `consent-disabled`, `consent-no-key`, `happy-path`, `injection-containment`, `schema-retry`, `timeout-retry` (7 tests).
- [ ] Dry-run golden shared with 36 (single source-of-truth file) and byte-stable.
- [ ] Key-leak assertions cover stdout, stderr, written files (candidates.yaml), AND every recorded request body.
- [ ] AC5 chain ends with 33's scanner passing on the poisoned-then-rebuilt site.
- [ ] Server helper exported for reuse by later local experimentation.

## Validation

CI link; a full transcript of the happy-path run pasted into the PR.

## Dependencies

13, 28 (harness), 33 (scanner), 37.

## Non-goals

Real-provider integration tests (manual, 42), performance/cost benchmarking,
streaming.

## Design References

DESIGN.md §16 (LLM row), §12, §13-B4 (AC4/AC5); ADR-005.
