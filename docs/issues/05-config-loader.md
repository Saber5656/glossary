# 05 — Config schema (zod) and loader with fail-closed validation

## Title

`src/config/`: `glossary/config.yaml` zod schema, defaults, discovery, friendly errors

## Summary

Implement the configuration contract of DESIGN.md §7.2: a zod schema with the
exact keys/defaults, a loader that locates the glossary directory, and
fail-closed validation (unknown keys are errors).

## Context

Every command starts by resolving `(repoRoot, glossaryDir, config)`. Getting
this wrong corrupts all downstream behavior, so the contract is frozen here.

## Scope

- `src/config/schema.ts` (zod + inferred `Config` type), `src/config/load.ts`
  (`resolvePaths`, `loadConfig`), `DEFAULT_CONFIG` export, unit tests.

## Detailed Requirements

1. Schema: implement DESIGN.md §7.2 verbatim — top-level keys `schemaVersion`,
   `include`, `exclude`, `scan`, `extract`, `export`, `site`, `llm` with the
   exact sub-keys, types, and defaults listed there. Constraints:
   - `schemaVersion` literal `1`.
   - `.strict()` at every object level (unknown key ⇒ error, fail closed).
   - `include` non-empty array of strings; defaults
     `["**/*.md", "**/*.mdx", "src/**"]`.
   - `site.locale` enum `ja|en`; `llm.definitionLanguage` enum `ja|en`.
   - Numeric bounds: `scan.maxFileSizeKB` 1–10240; `extract.maxCandidates`
     1–5000; `llm.maxTermsPerRun` 1–200; `llm.snippetContextLines` 0–10;
     `llm.maxSnippetsPerTerm` 1–20; `llm.timeoutMs` 1000–120000.
   - `llm.apiKeyEnv` matches `^[A-Z][A-Z0-9_]*$`.
   - `export.path` and `site.outDir` are repo-relative paths; reject absolute
     paths and any path containing `..` segments (`E_PATH_ESCAPE`).
2. `resolvePaths(cwd, flags: {repo?: string, dir?: string})`:
   - `repoRoot` = `--repo` if given, else nearest ancestor of `cwd` containing
     `.git` (dir or file), else `cwd`. Always `path.resolve`d.
   - `glossaryDir` = resolve(`repoRoot`, `--dir` ?? `glossary`); MUST stay
     under `repoRoot` after resolution (symlink-free `path.resolve` check) or
     `UsageError(E_PATH_ESCAPE)`.
   - Returns `{repoRoot, glossaryDir, configPath: glossaryDir/config.yaml}`.
3. `loadConfig(configPath): Config`:
   - Missing file ⇒ `UsageError(E_NOT_INITIALIZED, hint: "run 'glossary init'")`.
   - Read via yaml-io (issue 04); zod-parse; on failure throw
     `UsageError(E_CONFIG_INVALID)` whose message lists every issue as
     `<yamlPath>: <problem>` (use zod issue paths joined with `.`), max 20
     lines.
   - Returns fully-defaulted `Config` (all optionals resolved).
4. Export `DEFAULT_CONFIG: Config` (used by `init`, issue 07) and
   `defaultConfigYaml(): string` rendering the commented default file content
   (comments explaining each section, ≤ 60 lines, matches schema).

## Acceptance Criteria

- [ ] Unit tests: defaults application (empty file ⇒ error? NO — missing keys fine, `{}` with `schemaVersion: 1` yields full defaults); unknown key rejected at top and nested levels; every numeric bound rejected outside range; absolute/`..` paths rejected; `resolvePaths` finds repo root from a nested dir; `--dir` escape attempt rejected.
- [ ] `defaultConfigYaml()` parses back through `loadConfig` byte-safe (write to temp, load, deep-equal DEFAULT_CONFIG).
- [ ] Error message for a config with 3 problems lists all 3 with YAML paths.

## Validation

`npm run test`; additionally `tsx` one-shot: load a hand-written valid config
from a temp dir and print the resolved object.

## Dependencies

01, 03, 04.

## Non-goals

CLI flag parsing (06), `init` writing files (07), env-var reading (LLM key is
read in 35, not here — config stores only the env var NAME).

## Design References

DESIGN.md §7.2 (schema), §6 (dir rules), §13-B1 (path guard), §14 (errors).
