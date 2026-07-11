# 35 — OpenAI-compatible fetch client with consent-safe error handling

## Title

`src/llm/client.ts`: minimal chat-completions client (fetch), timeouts, bounded retries, secret hygiene

## Summary

Implement the transport layer of DESIGN.md §12 / ADR-005: a dependency-free
OpenAI-compatible chat client with strict error taxonomy and guaranteed
key-non-disclosure. No drafting logic here (36/37).

## Context

This is trust boundary B4's outbound half. Keeping it tiny, SDK-free, and
test-injectable is the security and testability posture chosen in ADR-005.

## Scope

- Client module + unit tests with undici MockAgent (dev-only test dep is
  builtin to Node's undici? Node ships fetch via undici but MockAgent needs
  the `undici` package — ADD `undici` as devDependency ONLY; document in
  ADR-001 dev list).

## Detailed Requirements

1. Types:
   ```ts
   interface LlmClientConfig { baseUrl: string; model: string; apiKey: string;
     timeoutMs: number }
   interface ChatRequest { system: string; user: string; jsonMode: boolean;
     maxTokens?: number }
   interface ChatResult { content: string; model: string;
     usage?: {promptTokens: number, completionTokens: number} }
   chat(cfg: LlmClientConfig, req: ChatRequest,
        deps?: { fetchImpl?: (input: string | URL, init: RequestInit) => Promise<Response>,
                 sleep?: (ms: number) => Promise<void> }): Promise<ChatResult>
   // deps default to global fetch and real sleep — injectable for tests
   ```
2. Request: POST `${baseUrl}/chat/completions` — baseUrl normalization trims
   TRAILING SLASHES ONLY and preserves any path prefix
   (`https://host/v1/` → `https://host/v1/chat/completions`). Headers
   `Authorization: Bearer <key>`, `Content-Type: application/json`; body
   serialized with `JSON.stringify` in exactly this field order:
   `{model, messages: [{role:'system',content},{role:'user',content}],
     ...(jsonMode ? {response_format: {type: 'json_object'}} : {}),
     ...(maxTokens !== undefined ? {max_tokens: maxTokens} : {})}`
   (issue 36's PreviewEntry.body must byte-equal this wire body — asserted
   in 38). `maxTokens`, when provided, must be a positive integer (else
   UsageError E_USAGE).
   - Endpoint validation (UsageError `E_LLM_ENDPOINT_INVALID` — add to 03's
     ErrorCodes): `baseUrl` must be http(s); plain `http:` allowed ONLY for
     localhost/127.0.0.1/[::1] (local model servers); anything else refused
     ('refusing non-TLS remote endpoint').
3. Timeout via AbortController: `timeoutMs` applies PER HTTP ATTEMPT (each
   attempt gets a fresh controller); a whole `chat()` call may therefore
   take up to 2×timeoutMs + 1s. Abort ⇒ RuntimeError E_LLM_HTTP
   `timeout after Nms`.
4. Retries: network errors and HTTP 429/5xx retried ONCE after a fixed
   `sleep(1000)` (deterministic, no jitter; injectable); 4xx (except 429)
   never retried. Final failure ⇒ RuntimeError E_LLM_HTTP with status +
   first 200 chars of response body **after secret scrub**.
5. Secret hygiene: a scrub function removes the apiKey string and any
   `Bearer …` token from EVERY error path — message, hint, AND `cause`
   (errors thrown by this module either omit cause or wrap it in a
   sanitized Error whose message/stack passed the scrub). The key is
   accepted as an argument — client never reads env directly (37 resolves
   env).
6. Response parsing: `choices[0].message.content` string required; missing ⇒
   E_LLM_SCHEMA (transport-level shape only; drafting schema is 37's).
   If HTTP 200 but `response_format` unsupported-server returned plain text —
   that IS content; pass through (U6 tolerance). `usage`: map
   `usage.prompt_tokens`/`usage.completion_tokens` →
   `promptTokens`/`completionTokens`; missing or malformed usage is IGNORED
   (undefined), never an error.
7. No streaming, no tools, no multi-turn. `user-agent: glossary/<version>`.

## Acceptance Criteria

- [ ] MockAgent tests: happy path (json body asserted byte-exact incl. response_format presence/absence and field order); 429→retry→success (sleep injected, called with 1000); 500→retry→fail E_LLM_HTTP; 401 no-retry; per-attempt timeout abort; non-TLS remote refused with E_LLM_ENDPOINT_INVALID; localhost http allowed.
- [ ] Key-scrub: an error embedding the key in message AND in a nested cause passes `formatError(err, true)` with zero occurrences of the key or `Bearer <key>`.
- [ ] baseUrl: `https://host/v1` and `https://host/v1/` both hit `/v1/chat/completions`; usage snake→camel mapping and malformed-usage tolerance tested.
- [ ] `npm ls --omit=dev --depth=0` unchanged; `undici` present only under devDependencies in package.json.

## Validation

`npm run test`; optional manual hit against a local Ollama if available (not
required for close).

## Dependencies

03, 05 (config types).

## Non-goals

Payload construction/redaction (36), consent logic (37), streaming, provider
adapters (v2, ADR-005).

## Design References

DESIGN.md §12.2 (request), §12.5 (failures), §13-B4; ADR-005 §1/§6; U6.
