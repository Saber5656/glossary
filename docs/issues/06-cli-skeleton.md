# 06 — CLI skeleton: commander program, global flags, JSON plumbing

## Title

`src/cli/`: program wiring, global flags, exit-code mapping, `--json` output convention

## Summary

Build the CLI frame per DESIGN.md §10: command registry, global flags, unified
error → exit-code handling, and the machine-output convention every command
issue (07, 12, 24–27, 30, 37) plugs into. Commands land as stubs here.

## Context

All later command issues assume this frame exists so they only implement a
`run(ctx, args)` body. Freezing conventions now prevents per-command drift.

## Scope

- `src/cli/main.ts` (entry), `src/cli/context.ts`, `src/cli/output.ts`,
  stub registrations for: init, extract, list, show, status, approve, reject,
  validate, export, build, draft.

## Detailed Requirements

1. commander program `glossary`, version from package.json.
   Global options: `--repo <path>`, `--dir <path>` (default `glossary`),
   `--json`, `--verbose`, `--no-color`.
   - Path resolution delegates to `resolvePaths` (issue 05): `--repo` defaults
     to the nearest ancestor of cwd containing `.git`, else cwd; an explicit
     `--repo` that does not exist or is not a directory ⇒
     `UsageError(E_USAGE, '--repo path does not exist: <path>')`; `--dir`
     resolves under repoRoot with escape rejected as `E_PATH_ESCAPE` (05's
     contract — do not reimplement here).
2. `CommandContext` built once per invocation:
   `{repoRoot, glossaryDir, configPath, config | null, logger, clock, flags}`.
   - Config is loaded lazily: commands declare `needsConfig: boolean`
     (`init` = false, everything else = true).
   - Honor `NO_COLOR` env as if `--no-color`.
3. Error handling at top level ONLY: wrap via `toGlossaryError` (03), print
   `formatError(err, verbose)` to stderr, `process.exitCode = err.exitCode`.
   commander parse failures (unknown command/flag, missing arg): use
   `program.exitOverride()` and convert to
   `UsageError(E_USAGE, <commander message>)` — usage/help text goes to
   stderr; with `--json`, the standard error object (see 4) still goes to
   stdout. No `process.exit()` calls inside commands (let streams flush).
4. `output.ts` convention:
   - `emit(ctx, humanText: string, jsonValue: unknown)` — prints exactly one of
     the two to **stdout** depending on `--json`.
   - JSON envelope for ALL commands:
     `{"ok": boolean, "command": string, "data": <command-specific>, "warnings": string[]}`
     — single line, UTF-8, no ANSI; `warnings` = `logger.warnings()`.
     The command-specific shapes frozen in each command's issue file (and the
     `--json` shapes quoted in DESIGN.md §10) are the **`data` field** of this
     envelope, not the whole stdout document (DESIGN.md §10 states this
     envelope rule — single source of truth for the envelope is THIS issue).
   - On error with `--json`: envelope `{"ok": false, "command": string, "error": {"code": string, "message": string}}`
     to stdout AND human message to stderr; exit code per §10.
5. Each stub command registers with its final name/flags/help text (copy the
   command table DESIGN.md §10) but `run()` throws
   `RuntimeError('E_NOT_IMPLEMENTED', 'implemented in issue NN')` — add
   `E_NOT_IMPLEMENTED` to ErrorCodes.
6. `glossary --help` groups commands in the §10 order; each has a one-line
   description matching the design table.
7. Integration-test helper `runCli(argv, opts)` (in `test/helpers/`) spawning
   the built CLI (or calling main with injected ctx) capturing
   {stdout, stderr, exitCode} — used by all later command tests.

## Acceptance Criteria

- [ ] `glossary --help` lists all 11 commands with correct descriptions; `glossary <cmd> --help` shows the declared flags.
- [ ] In a temp repo WITH a valid `glossary/config.yaml`: `glossary extract` (stub) exits 1 with `error E_NOT_IMPLEMENTED…`; with `--json` prints the one-line error envelope on stdout.
- [ ] In a temp repo WITHOUT config: `glossary list` exits 2 with `E_NOT_INITIALIZED` and the init hint (lazy config load precedes the stub).
- [ ] `glossary bogus` and `glossary list --bogus` both exit 2 via exitOverride→`E_USAGE`; with `--json` the error envelope appears on stdout.
- [ ] `--repo /nonexistent` exits 2 with `E_USAGE` naming the path.
- [ ] A committed vitest spec calls `runCli(['--help'])` and `runCli(['extract'], {cwd: <initialized temp repo>})`, asserting `{stdout, stderr, exitCode}` exactly and that no code path calls `process.exit()` (spawned process exits via exitCode).

## Validation

`npm run build && node dist/cli/main.js --help`; run the stub matrix test.

## Dependencies

01, 03, 05.

## Non-goals

Real command behavior (later issues); shell completions; CLI i18n (v2).

## Design References

DESIGN.md §10 (table, flags, exit codes, stdout/stderr), §14.
