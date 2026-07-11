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
   chat(cfg: LlmClientConfig, req: ChatRequest, deps?: {fetchImpl?}): Promise<ChatResult>
   ```
2. Request: POST `${baseUrl}/chat/completions` (baseUrl trailing-slash
   normalized), headers `Authorization: Bearer <key>`,
   `Content-Type: application/json`; body:
   `{model, messages: [{role:'system'...},{role:'user'...}],
     ...(jsonMode ? {response_format: {type: 'json_object'}} : {}),
     ...(maxTokens ? {max_tokens} : {})}`.
   - `baseUrl` MUST be http(s); plain `http:` allowed ONLY for
     localhost/127.0.0.1/[::1] (local model servers), otherwise
     UsageError E_LLM_HTTP ('refusing non-TLS remote endpoint').
3. Timeout via AbortController (`timeoutMs`); abort ⇒ RuntimeError E_LLM_HTTP
   `timeout after Nms`.
4. Retries: network errors and HTTP 429/5xx retried ONCE after 1s fixed delay
   (deterministic, no jitter); 4xx (except 429) never retried. Final failure
   ⇒ E_LLM_HTTP with status + first 200 chars of response body **after secret
   scrub**.
5. Secret hygiene: a scrub function removes the apiKey string and any
   `Bearer …` token from EVERY error message/log path; unit test constructs
   an error embedding the key and asserts absence. The key is accepted as an
   argument — client never reads env directly (37 resolves env).
6. Response parsing: `choices[0].message.content` string required; missing ⇒
   E_LLM_SCHEMA (transport-level shape only; drafting schema is 37's).
   If HTTP 200 but `response_format` unsupported-server returned plain text —
   that IS content; pass through (U6 tolerance).
7. No streaming, no tools, no multi-turn. `user-agent: glossary/<version>`.

## Acceptance Criteria

- [ ] MockAgent tests: happy path (json body asserted byte-exact incl. response_format presence/absence); 429→retry→success; 500→retry→fail E_LLM_HTTP; 401 no-retry; timeout abort; non-TLS remote refused, localhost http allowed.
- [ ] Key-scrub test passes (key never in message/stack/hint).
- [ ] Trailing-slash baseUrl variants normalize to a single form.
- [ ] Zero new runtime dependencies (`npm ls` check).

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
