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

1. `scanRepo(repoRoot, config, logger): ScanResult` where
   `ScanResult = {files: ScannedFile[], skipped: SkipRecord[], warnings}` and
   `ScannedFile = {relPath, absPath, kind: 'doc'|'code', sizeBytes}`.
2. Enumeration: `fast-glob` with `config.include` patterns, `dot: false`,
   `followSymbolicLinks: false`, `onlyFiles: true`, cwd=repoRoot.
3. Filters applied in order (record every exclusion with a reason in
   `skipped` when `--verbose`):
   1. Built-in denylist globs (export const): `.git/**`, `node_modules/**`,
      `dist/**`, `build/**`, `out/**`, `vendor/**`, `coverage/**`,
      `**/*.min.*`, `**/package-lock.json`, `**/yarn.lock`,
      `**/pnpm-lock.yaml`, `<glossaryDir>/**` except nothing (the whole
      managed dir is excluded from scanning — candidates must not
      self-extract), plus binary extensions:
      `png,jpg,jpeg,gif,webp,ico,svg?` — NOT svg (text) — use list:
      `png jpg jpeg gif webp ico woff woff2 ttf otf eot zip gz tgz bz2 7z rar
      pdf mp3 mp4 mov avi wasm class jar exe dll so dylib`.
   2. `.gitignore` semantics via `ignore` package: root `.gitignore` +
      nested ones (collect all `.gitignore` files first, apply per-directory
      scope as the package documents).
   3. User `config.exclude` globs.
   4. Size guard: `sizeBytes > scan.maxFileSizeKB*1024` ⇒ skip, reason `size`.
   5. Binary sniff: read first 8 KiB; contains byte 0x00 ⇒ skip, reason
      `binary`.
   6. Path safety: `path.resolve(absPath)` must start with `repoRoot + sep`
      (defense-in-depth; fast-glob shouldn't escape, assert anyway) else skip
      with reason `escape` + warning (`E_PATH_ESCAPE` semantics, non-fatal).
4. `kind`: `doc` iff extension ∈ {md, mdx, txt}; else `code`.
5. Ordering: `files` sorted by `relPath` with compareCodepoint. Warning (not
   error) when file count > 20_000: continue but log.
6. No file CONTENT reading here beyond the 8 KiB sniff.

## Acceptance Criteria

- [ ] Against `fixtures/repo-ja-mixed`: includes README.md, docs/*, src/**; excludes `tmp/ignored.md` (.gitignore) and `node_modules/**` (denylist); order matches sorted relPaths (snapshot).
- [ ] Against `fixtures/repo-hostile`: `big/huge.txt` skipped(size), `bin/blob.dat` skipped(binary), `links/escape` not followed/absent, no path outside repoRoot ever returned (assert all prefixes).
- [ ] glossaryDir contents never scanned even when include is `**/*`.
- [ ] Determinism: two runs → deep-equal results.
- [ ] Unit tests for the denylist list itself (svg included as text, wasm excluded).

## Validation

`npm run test`; `--verbose` run over hostile fixture pasted in PR showing skip
reasons.

## Dependencies

03, 05; fixtures 13 for tests.

## Non-goals

Content parsing (15/16), incremental caching (v2), configurable
followSymlinks=true (reserved key, rejected in v1).

## Design References

DESIGN.md §9.1, §13-B1 (guards, AC3), §4 (determinism).
