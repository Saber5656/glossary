# 14 — File scanner with ignore rules, guards, deterministic order

## Title

`src/scan/`: repository file enumeration with denylist, .gitignore, size/binary/symlink guards

## Summary

Implement the scanner of DESIGN.md §9.1: given (repoRoot, config), return the
deterministic list of files to analyze, applying built-in denylist, user
include/exclude, `.gitignore` semantics, and the B1 security guards.

## Context

The scanner is trust boundary B1; every downstream stage assumes its output is
safe to read wholesale. It also fixes pipeline determinism via ordering.

## Scope

- `src/scan/scanner.ts`, `src/scan/builtin-excludes.ts` + tests against
  fixtures.

## Detailed Requirements

1. `scanRepo(opts: {repoRoot: string, glossaryDirRel: string, config: Config, logger: Logger}): ScanResult`
   — `glossaryDirRel` is the normalized repo-relative managed directory
   (from `--dir`; the CLI passes it — it is NOT part of config).
   Types (exported):
   ```ts
   type SkipReason = 'denylist'|'gitignore'|'exclude'|'size'|'binary'|'symlink'|'escape'
   type SkipRecord = { relPath: string; reason: SkipReason; detail?: string }
   type ScanWarning = { code: string; message: string; relPath?: string }
   type ScannedFile = { relPath: string; absPath: string; kind: 'doc'|'code'; sizeBytes: number }
   type ScanResult = { files: ScannedFile[]; skipped: SkipRecord[]; warnings: ScanWarning[] }
   ```
   `skipped` is ALWAYS fully populated (deterministic, testable); `--verbose`
   only controls whether it is logged.
2. Enumeration: `fast-glob` with `config.include` patterns, `dot: false`
   (dot-directories/dotfiles are out of v1 scan scope — DESIGN §9.1),
   `followSymbolicLinks: false`, `onlyFiles: true`, cwd=repoRoot.
3. Filters applied in order (every exclusion appends a SkipRecord):
   1. Built-in denylist globs (export const): `.git/**`, `node_modules/**`,
      `dist/**`, `build/**`, `out/**`, `vendor/**`, `coverage/**`,
      `**/*.min.*`, `**/package-lock.json`, `**/yarn.lock`,
      `**/pnpm-lock.yaml`, `<glossaryDirRel>/**` (the whole managed dir —
      candidates must not self-extract), `config.export.path` (the exported
      GLOSSARY.md) and `config.site.outDir/**` (generated outputs must never
      be re-scanned), plus the binary-extension list:
      `png jpg jpeg gif webp ico woff woff2 ttf otf eot zip gz tgz bz2 7z rar
      pdf mp3 mp4 mov avi wasm class jar exe dll so dylib`.
      `svg` is deliberately NOT in the list (it is text; other rules may
      still exclude it).
   2. Symlink guard: `lstat` each candidate; `isSymbolicLink()` ⇒ skip,
      reason `symlink` (explicit check — do not rely on fast-glob options
      alone).
   3. `.gitignore` semantics via the `ignore` package — exact algorithm:
      discover every `.gitignore` under repoRoot (skipping denylisted dirs);
      parse each into an `ignore()` matcher scoped to its containing
      directory; a candidate is ignored iff any ancestor-scope matcher
      ignores its path relative to that scope (Git semantics incl. `!`
      negations are delegated to the package per scope; deeper scopes are
      consulted after shallower ones).
   4. User `config.exclude` globs.
   5. Size guard: `sizeBytes > scan.maxFileSizeKB*1024` ⇒ skip, reason `size`.
   6. Binary sniff: read first 8 KiB; contains byte 0x00 ⇒ skip, reason
      `binary`.
   7. Path safety: `path.resolve(absPath)` must start with `repoRoot + sep`
      (defense-in-depth; fast-glob shouldn't escape, assert anyway) else skip
      with reason `escape` + ScanWarning (`E_PATH_ESCAPE` semantics,
      non-fatal).
4. `kind`: `doc` iff extension ∈ {md, mdx, txt}; else `code`.
5. Ordering: `files` sorted by `relPath` with compareCodepoint. ScanWarning
   (not error) when file count > 20_000: continue but log.
6. No file CONTENT reading here beyond the 8 KiB sniff.

## Acceptance Criteria

- [ ] Against `fixtures/repo-ja-mixed`: includes README.md, docs/*, src/**; excludes `tmp/ignored.md` (.gitignore) and `node_modules/**` (denylist); order matches sorted relPaths (snapshot).
- [ ] A committed GLOSSARY.md and a populated site.outDir inside a test repo are excluded (self-artifact rule).
- [ ] Against `fixtures/repo-hostile`: `big/huge.txt` skipped(size), `bin/blob.dat` skipped(binary), `links/escape` recorded skipped(symlink) — never read/followed; no path outside repoRoot ever returned (assert all prefixes).
- [ ] `skipped` is fully populated regardless of verbosity (snapshot the hostile fixture's SkipRecords — this replaces any manual verbose-output review).
- [ ] glossaryDir contents never scanned even when include is `**/*` (test with `--dir docs/glossary` variant too).
- [ ] Nested `.gitignore` test: `sub/.gitignore` ignoring `sub/skip.md` works while `skip.md` at root is still scanned; a `!keep.md` negation is honored.
- [ ] Determinism: two runs → deep-equal results.
- [ ] Unit tests for the denylist list itself (svg treated as text, wasm excluded).

## Validation

`npm run test` (the hostile-fixture SkipRecord snapshot is the objective
evidence; no manual transcript required).

## Dependencies

03, 05; fixtures 13 for tests.

## Non-goals

Content parsing (15/16), incremental caching (v2), configurable
followSymlinks=true (reserved key, rejected in v1).

## Design References

DESIGN.md §9.1, §13-B1 (guards, AC3), §4 (determinism).
