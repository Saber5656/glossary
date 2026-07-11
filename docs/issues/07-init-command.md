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

1. Behavior (in `glossaryDir` from context; create it if missing). The five
   managed entries (repo-relative; `<dir>` = glossary dir name):
   `<dir>/config.yaml`, `<dir>/terms/`, `<dir>/terms/.gitkeep`,
   `<dir>/rejected.yaml`, `<dir>/.gitignore`.
   - `config.yaml`: when missing, write `defaultConfigYaml(<dir>)` (05).
     Existing file is left untouched and reported in `skipped` (idempotent
     re-run, exit 0). With `--force` it is overwritten and reported in
     `overwritten` — `--force` affects config.yaml ONLY.
   - `terms/` + `.gitkeep`: create if missing; never touched by `--force`.
   - `rejected.yaml`: if missing, write `{schemaVersion: 1, rejected: []}`
     with header `Managed by 'glossary reject' — edit via CLI`. Existing file
     untouched even with `--force`.
   - `.gitignore`: if missing, write `site/\n`. Existing untouched.
   - Symlink guard: `lstat` `glossaryDir` and each existing managed entry
     before writing; any symlink ⇒ `UsageError(E_PATH_ESCAPE)` naming the
     path (§13-B1 defense; realpath containment of the dir itself is already
     enforced by 05's resolvePaths).
2. Output (human): checklist of created/overwritten/skipped paths, then a
   next-steps hint (`glossary extract`). `--json` data (the envelope `data`
   field, issue 06): `{created: string[], overwritten: string[], skipped:
   string[]}` — repo-relative paths, directories with trailing `/`, each
   array sorted. Example full stdout:
   `{"ok":true,"command":"init","data":{"created":["glossary/.gitignore","glossary/config.yaml","glossary/rejected.yaml","glossary/terms/","glossary/terms/.gitkeep"],"overwritten":[],"skipped":[]},"warnings":[]}`
3. `init` must not require config (needsConfig=false). It performs no config
   loading, no store reads, and no repository scanning — only issue 05/06
   path resolution (repo-root discovery from cwd/`.git`) plus the
   managed-entry checks above.
4. Re-run without changes ⇒ exit 0, all five entries in `skipped`.
5. If `glossaryDir` exists but is a file, or resolves outside repoRoot ⇒
   errors per issue 05's `resolvePaths` (already enforced; test it end-to-end).

## Acceptance Criteria

- [ ] Fresh temp repo: `init` exits 0; `created` equals exactly the five managed entries of Req 1; second run exits 0 with all five in `skipped` and zero filesystem writes (mtime/fs-spy check).
- [ ] `--force`: `overwritten == ["glossary/config.yaml"]`; content and mtimes of the other four entries unchanged.
- [ ] Symlinked `glossary/` dir or symlinked existing `config.yaml` ⇒ exit 2 `E_PATH_ESCAPE` naming the path, zero writes.
- [ ] Written config.yaml round-trips through `loadConfig` equal to `DEFAULT_CONFIG`.
- [ ] `--json` output matches the frozen envelope example (snapshot via runCli).
- [ ] `--dir docs/glossary`: entries created under `docs/glossary/`, generated config contains `site.outDir: docs/glossary/site`, `.gitignore` content is `site/`.

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
