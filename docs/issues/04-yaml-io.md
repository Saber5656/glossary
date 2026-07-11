# 04 — Safe YAML read/write module with hard limits and atomic writes

## Title

`src/store/yaml-io.ts`: bounded YAML parsing, stable serialization, atomic file writes

## Summary

Single module through which ALL glossary YAML files are read and written,
implementing the parser-boundary defenses of DESIGN.md §13-B2 and the
deterministic serialization required by §4.

## Context

Glossary YAML crosses a trust boundary: files can arrive via malicious PR
(abuse case AC2, billion-laughs). Every store (issues 09–11) and config
loader (05) must use this module — no direct `yaml` imports elsewhere.

## Scope

- `readYamlFile`, `writeYamlFile`, `serializeYaml` + unit tests including
  attack inputs. No schemas here (zod lives in each store).

## Detailed Requirements

1. `readYamlFile(absPath: string, opts?: {maxBytes?: number}): unknown`
   - Default `maxBytes` 1 MiB (1_048_576). Check `stat.size` BEFORE reading
     (fast reject), AND re-check the actual read bytes
     (`Buffer.byteLength`) after reading (TOCTOU guard); either over ⇒
     `RuntimeError(E_YAML_TOO_LARGE)` naming the file and limit.
   - Parse with the `yaml` package: `parse(text, {schema: 'core', version: '1.2', maxAliasCount: 100, uniqueKeys: true})`.
   - Reject documents nested deeper than 20 levels: walk the parsed value
     iteratively; deeper ⇒ `RuntimeError(E_YAML_INVALID, 'nesting too deep')`.
   - Multiple documents (`---`) ⇒ `E_YAML_INVALID`.
   - Parse errors wrap into `E_YAML_INVALID`; message format
     `<absPath>:<line>:<col>: <yaml error message>` (1-based, from the yaml
     error's position) or `<absPath>: unknown location: <message>` when the
     library reports no position.
2. `serializeYaml(value: JsonLike): string`
   - `JsonLike` = null | boolean | **finite** number | string | JsonLike[] |
     plain records (own enumerable string keys only, prototype
     `Object.prototype` or `null`). Runtime-assert recursively; `NaN`,
     `±Infinity`, BigInt, symbols, functions, Dates, Maps, Sets, class
     instances, and `undefined` values ⇒ `RuntimeError(E_YAML_INVALID)` —
     callers convert first.
   - Options: `lineWidth: 0` (no folding), `indent: 2`, `defaultStringType`
     plain with automatic quoting as the yaml lib decides deterministically;
     block literals (`|`) for strings containing `\n`.
   - **Key order = insertion order of the object** (stores construct objects in
     schema order); no sorting inside this module.
   - Output always ends with exactly one `\n`; LF only.
3. `writeYamlFile(absPath, value, opts?: {header?: string})`
   - `header` is RAW text WITHOUT comment markers (e.g.
     `"GENERATED — do not edit"`); the module splits it on newlines and emits
     each non-empty line as `# ${line}` before the YAML body.
   - Atomic algorithm (exact): create parent dirs → write
     `${absPath}.tmp-<pid>` with mode 0644 → `fsync` the temp fd → close →
     `rename` over `absPath` → best-effort `fsync` of the parent directory
     (ignore platforms/errors where unsupported). On any failure before a
     successful rename, best-effort unlink the temp file in `finally`.
4. Module exports `YAML_LIMITS` constants for reuse in docs/tests.

## Acceptance Criteria

- [ ] Attack tests: (a) alias-bomb doc (`&a`… ×101 aliases) rejected; (b) 21-deep nesting rejected; (c) 2 MiB file rejected by size before parse (test with sparse/generated file); (d) duplicate keys rejected; (e) multi-doc rejected.
- [ ] Round-trip test: serialize → read → deep-equal for a value using JA strings, multiline strings, arrays of objects.
- [ ] Determinism test: serializing the same object twice is byte-identical; output ends with single `\n`.
- [ ] Atomicity test: no `.tmp-` file remains after success or after injected rename failure.
- [ ] JsonLike assertion tests: NaN, Infinity, BigInt, Date, Map, class instance, undefined-valued key, symbol key — each rejected; null-prototype record accepted.
- [ ] Source-scan test: no file under `src/` except `src/store/yaml-io.ts` matches `/from ['"]yaml['"]|require\(['"]yaml['"]\)|import\(['"]yaml['"]\)/` (vitest source scan, portable).

## Validation

`npm run test` green including the attack suite; show the alias-bomb fixture
inline in the test file (small, self-contained).

## Dependencies

01, 03.

## Non-goals

Schema validation (per-store issues), config discovery (05), YAML comments
preservation on rewrite (machine files are regenerated; human files are only
appended by tools that construct full objects).

## Design References

DESIGN.md §4 (determinism), §7 (schemas use this module), §13-B2 (limits),
ADR-002 §5.
