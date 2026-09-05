# 03 — Error taxonomy, exit codes, logger, injectable clock

## Title

Utility layer: typed errors with stable codes, exit-code mapping, leveled logger, injectable clock

## Summary

Implement `src/util/errors.ts`, `src/util/logger.ts`, `src/util/clock.ts`
exactly as DESIGN.md §14 specifies, so every later module reports failures and
logs uniformly and tests can control time.

## Context

Exit codes and stderr/stdout separation are part of the CLI contract (§10);
deterministic output (§4) requires that no module calls `Date.now()` directly.

## Scope

- The three modules + unit tests. No CLI wiring yet (issue 06 maps errors to
  exit codes at the top level).

## Detailed Requirements

1. `errors.ts`:
   - `abstract class GlossaryError extends Error { readonly code: string; readonly exitCode: 1|2|3 }`.
   - `class UsageError extends GlossaryError` → exitCode 2.
   - `class ValidationFailed extends GlossaryError` → exitCode 3.
   - `class RuntimeError extends GlossaryError` → exitCode 1.
   - Constructor signature: `(code: string, message: string, opts?: {cause?: unknown, hint?: string})`.
     `GlossaryError` declares `readonly hint?: string` and passes `cause` to
     `super(message, {cause})`.
   - Export `const ErrorCodes` object freezing the initial code list:
     `E_CONFIG_INVALID`, `E_CONFIG_NOT_FOUND`, `E_PATH_ESCAPE`,
     `E_YAML_TOO_LARGE`, `E_YAML_INVALID`, `E_SCHEMA_INVALID`,
     `E_KEY_NOT_FOUND`, `E_ID_CONFLICT`, `E_LLM_CONSENT`, `E_LLM_HTTP`,
     `E_LLM_SCHEMA`, `E_OUTDIR_UNSAFE`, `E_NOT_INITIALIZED`, `E_USAGE`,
     `E_UNEXPECTED`. Later issues may append codes here (single source of
     truth).
   - `toGlossaryError(err: unknown): GlossaryError` — returns `err` if it
     already is one; otherwise wraps as
     `RuntimeError(ErrorCodes.E_UNEXPECTED, 'unexpected error: ' + String(message ?? err), {cause: err})`.
   - `formatError(err: unknown, verbose: boolean): string` — applies
     `toGlossaryError` first, then renders:
     non-verbose = `error <code>: <message>` plus `hint: <hint>` on a second
     line when present; verbose = the same plus the error's stack and, when
     `cause` is an Error, `caused by:` + the cause's stack.
2. `logger.ts`:
   - Interface (exported):
     ```ts
     interface Logger {
       error(message: string): void; warn(message: string): void;
       info(message: string): void; debug(message: string): void;
       warnings(): string[];
     }
     ```
   - Levels `error|warn|info|debug`; default threshold `info`; `--verbose` ⇒
     `debug`. All output to **stderr** (colors: error red, warn yellow only).
   - Factory `createLogger(opts: {verbose: boolean; color: boolean; env?: NodeJS.ProcessEnv})`
     (env defaults to `process.env`); effective color =
     `opts.color && !env.NO_COLOR`.
   - `warnings()` returns every `warn(...)` message verbatim (plain text, no
     ANSI), in emission order — consumed by command summaries and `--json`
     embedding (DESIGN §14).
   - `redactEnvValue(name: string, text: string, env?: NodeJS.ProcessEnv): string`
     — when `env[name]` is a non-empty string, replaces every occurrence of
     that value in `text` with `[REDACTED:<name>]`; no-op when unset/empty.
     Used by LLM issues (35/37); unit test proves a fake key value never
     survives.
3. `clock.ts`: `interface Clock { now(): Date }`, `systemClock`, and
   `fixedClock(iso: string)` for tests. Helper `isoDate(clock)` → `YYYY-MM-DD`
   and `isoDateTime(clock)` → `YYYY-MM-DDTHH:mm:ssZ` (UTC, seconds precision,
   no millis) — the only timestamp formats used in stores (§7.3–7.5).

## Acceptance Criteria

- [ ] Unit tests cover: exit-code mapping per class; toGlossaryError wrapping (string throw, Error throw, GlossaryError passthrough); formatError non-verbose/verbose × with/without hint × with/without cause; logger level filtering; injected-env NO_COLOR handling; warnings collection order and plainness; redactEnvValue (set/unset/empty/multiple occurrences); fixedClock formatting (both helpers, UTC).
- [ ] Source-scan test: a vitest that reads all files under `src/` (excluding `src/util/clock.ts`) and asserts none matches `/Date\.now\(|new Date\(/` (portable; no grep dialect dependency).
- [ ] No module here imports from outside `src/util/` + node builtins.

## Validation

`npm run test` green; run a small script with `fixedClock('2026-01-02T03:04:05Z')`
asserting `isoDateTime` output equals the input formatting exactly.

## Dependencies

01.

## Non-goals

CLI arg parsing (06), i18n of messages (v2), file logging.

## Design References

DESIGN.md §4 (determinism), §10 (exit codes, stdout/stderr), §14.
