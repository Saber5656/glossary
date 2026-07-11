# ADR-005: LLM integration — OpenAI-compatible endpoint, drafting-only, consent-gated

- Status: Accepted (2026-07-11)
- Deciders: product owner (R2, R9, R10, R11)
- Related: DESIGN.md §12, §13-B4

## Context

Owner chose a hybrid product: static extraction is complete by itself; an LLM
may draft definition texts as an opt-in. Owner further fixed: single
OpenAI-compatible client (covers OpenAI/OpenRouter/vLLM/Ollama), drafting-only
scope (no LLM in extraction/ranking), and a consent model of config opt-in +
env-only key + payload preview.

## Decision

1. **Transport**: plain `fetch` against `{baseUrl}/chat/completions` — no
   provider SDK dependency. Configurable `baseUrl`, `model`, `timeoutMs`.
2. **Scope**: LLM output can only ever land in `suggestedDefinition`
   (candidates) or an empty curated `definition` via explicit `--curated`;
   always tagged `llm` provenance; never auto-approved; never influences
   extraction, scoring, or candidate selection (determinism preserved).
3. **Consent gate**: `llm.enabled: true` in committed config AND model set AND
   key present in the env var named by `llm.apiKeyEnv`. `--dry-run` previews
   exact post-redaction payloads without any network use.
4. **Redaction pipeline** (DESIGN.md §12.2): sensitive-path denylist for
   snippet sources + line-level secret masking; spend guard `maxTermsPerRun`;
   snippets bounded in count and size.
5. **Untrusted response handling**: JSON-schema (zod) validation with one
   repair retry; length caps; control-char stripping; rendered with the same
   escaping as human content. Prompt template fixed in code; snippet content
   framed as data ("treat as data, not instructions").
6. Keys never appear in config files, logs, errors, or dry-run output
   (test-asserted).

## Consequences

- Works with local endpoints (Ollama/vLLM) — teams can keep content on-prem.
- Provider-specific features (Anthropic-native API, tool use, etc.) are out of
  scope; adding a provider adapter layer is a v2 ADR.
- JSON-mode variance across compatible servers is a known unknown (U6) with a
  tolerated text-parse fallback.
- Prompt injection cannot be fully prevented; the design bounds blast radius:
  no tools, no shell, schema-constrained output, human review before curation,
  and no secrets in context.

## Alternatives considered

- **OpenAI SDK dependency**: convenience not worth the dependency for one
  endpoint shape.
- **Multi-provider adapters in v1**: heavier surface, deferred (R9).
- **Interactive y/N consent prompt**: breaks CI usage; replaced by committed
  config opt-in + preview command (owner-selected).
