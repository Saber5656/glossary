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
   - Export `const ErrorCodes` object freezing the initial code list:
     `E_CONFIG_INVALID`, `E_CONFIG_NOT_FOUND`, `E_PATH_ESCAPE`,
     `E_YAML_TOO_LARGE`, `E_YAML_INVALID`, `E_SCHEMA_INVALID`,
     `E_KEY_NOT_FOUND`, `E_ID_CONFLICT`, `E_LLM_CONSENT`, `E_LLM_HTTP`,
     `E_LLM_SCHEMA`, `E_OUTDIR_UNSAFE`, `E_NOT_INITIALIZED`. Later issues may
     append codes here (single source of truth).
   - `formatError(err: unknown, verbose: boolean): string` — one-line message
     `error <code>: <message>` plus `hint:` line if present; stack trace only
     when `verbose`.
2. `logger.ts`:
   - Levels `error|warn|info|debug`; default threshold `info`; `--verbose` ⇒
     `debug`. All output to **stderr**. `NO_COLOR` env or `--no-color` disables
     ANSI (colors: error red, warn yellow only; keep minimal).
   - Factory `createLogger(opts: {verbose: boolean, color: boolean})`; logger
     collects warnings: `logger.warnings(): string[]` for command summaries
     and `--json` embedding (DESIGN §14).
   - Never logs values of env vars; add a guard helper
     `redactEnvValue(name, text)` used by LLM issues.
3. `clock.ts`: `interface Clock { now(): Date }`, `systemClock`, and
   `fixedClock(iso: string)` for tests. Helper `isoDate(clock)` → `YYYY-MM-DD`
   and `isoDateTime(clock)` → `YYYY-MM-DDTHH:mm:ssZ` (UTC, seconds precision,
   no millis) — the only timestamp formats used in stores (§7.3–7.5).

## Acceptance Criteria

- [ ] Unit tests cover: exit-code mapping per class; formatError with/without verbose and hint; logger level filtering; NO_COLOR handling; warnings collection; fixedClock formatting (both helpers, UTC).
- [ ] `grep -rn "Date.now\|new Date()" src --include='*.ts' | grep -v util/clock` returns nothing (enforced later by lint note; document in module header).
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
