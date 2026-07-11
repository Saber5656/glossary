# 07 — `glossary init` scaffolding command

## Title

`glossary init`: create the managed directory, commented default config, and companion files

## Summary

Implement the `init` command creating the target-repo layout of DESIGN.md §6:
`glossary/config.yaml` (commented defaults), `glossary/terms/`,
`glossary/rejected.yaml`, `glossary/.gitignore` — idempotent, `--force` aware.

## Context

First command a user runs; must be safe to re-run and must never clobber an
existing curated glossary.

## Scope

- `src/cli/commands/init.ts` replacing the stub. Uses `defaultConfigYaml()`
  (05) and yaml-io (04).

## Detailed Requirements

1. Behavior (in `glossaryDir` from context; create it if missing):
   - `config.yaml`: write `defaultConfigYaml()`. If it exists: without
     `--force` ⇒ `UsageError(E_ALREADY_INITIALIZED)` (add code) listing what
     exists; with `--force` ⇒ overwrite config.yaml ONLY.
   - `terms/`: create dir (+ `.gitkeep`) if missing. NEVER touched by --force.
   - `rejected.yaml`: if missing, write `{schemaVersion: 1, rejected: []}`
     with header `# Managed by 'glossary reject' — safe to review, edit via CLI`.
     Existing file untouched even with `--force`.
   - `.gitignore` (inside glossaryDir): if missing, write `site/\n`. Existing
     untouched.
2. Output (human): checklist of created/skipped paths, then next-steps hint
   (`glossary extract`). `--json` data:
   `{created: string[], skipped: string[]}` (repo-relative paths, sorted).
3. `init` must not require config (needsConfig=false) and must not read
   anything outside `glossaryDir`.
4. Re-run without changes ⇒ exit 0, everything in `skipped`.
5. If `glossaryDir` exists but is a file, or resolves outside repoRoot ⇒
   errors per issue 05's `resolvePaths` (already enforced; test it end-to-end).

## Acceptance Criteria

- [ ] Fresh temp repo: `init` exits 0 and creates exactly the 4 paths; second run exits 0 with all-skipped; `--force` rewrites only config.yaml (mtimes/content of others unchanged).
- [ ] Existing config without `--force` ⇒ exit 2, message names config.yaml and suggests `--force`.
- [ ] Written config.yaml round-trips through `loadConfig` equal to `DEFAULT_CONFIG`.
- [ ] `--json` output matches the frozen shape (snapshot test via runCli).
- [ ] `--dir docs/glossary` variant works and `.gitignore` content is `site/`.

## Validation

runCli-based integration tests in a temp dir; manual: `node dist/cli/main.js
init` inside a scratch git repo, inspect files.

## Dependencies

04, 06 (and 05 transitively).

## Non-goals

Interactive prompts/wizard; git operations (no auto-commit); sample stopwords
file (documented in README instead).

## Design References

DESIGN.md §6 (layout & ownership), §7.2 (config defaults), §10.1.
