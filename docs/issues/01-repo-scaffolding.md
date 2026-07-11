# 01 — Repository scaffolding: package, TS strict, lint, test, license

## Title

Repository scaffolding: package.json, strict TypeScript, lint/format, vitest, MIT license

## Summary

Create the Node.js/TypeScript project skeleton exactly as specified in
DESIGN.md §5, §15, §17 and ADR-001, with no product logic yet.

## Context

This is the first implementation issue; every other issue builds on the
toolchain and directory layout fixed here. The package stays private in v1
(distribution = git clone, ADR-006).

## Scope

- `package.json`, `tsconfig.json`, `eslint.config.js`, `.prettierrc.json`,
  `vitest.config.ts`, `.editorconfig`, `.gitignore`, `LICENSE`,
  `src/` directory skeleton with placeholder modules, npm scripts.
- No CI (issue 02), no CLI behavior (issue 06).

## Detailed Requirements

1. `package.json`:
   - `"name": "glossary"`, `"version": "0.1.0"`, `"private": true`,
     `"type": "module"`, `"license": "MIT"`.
   - `"engines": { "node": ">=22" }`.
   - `"bin": { "glossary": "./dist/cli/main.js" }`.
   - Runtime dependencies (exact set, latest stable versions, saved exact or
     caret — use caret): `commander`, `zod`, `yaml`, `fast-glob`, `ignore`,
     `unified`, `remark-parse`, `mdast-util-to-string`, `kuromoji`,
     `minisearch`. **No other runtime dependencies** (ADR-001 allowlist).
   - Dev dependencies: `typescript`, `vitest`, `@types/node`, `esbuild`,
     `eslint` + `typescript-eslint`, `prettier`, `tsx`.
   - Scripts: `build` (tsc -p tsconfig.build.json), `typecheck` (tsc --noEmit),
     `lint` (eslint .), `format` / `format:check` (prettier), `test`
     (vitest run), `test:watch`.
   - Commit `package-lock.json`.
2. `tsconfig.json`: `strict: true`, `module: NodeNext`,
   `moduleResolution: NodeNext`, `target: ES2023`,
   `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`,
   `outDir: dist`, `rootDir: src`, `sourceMap: true`,
   `forbidden: no \`any\` escapes` via eslint rule (see 4).
   Provide `tsconfig.build.json` extending it that excludes `**/*.test.ts` and
   `src/site/client/**` (client is bundled separately, issue 32).
3. Directory skeleton (each with a placeholder `index.ts` exporting nothing, so
   imports referenced by later issues resolve): `src/cli/`, `src/config/`,
   `src/store/`, `src/scan/`, `src/content/`, `src/tokenize/`, `src/extract/`,
   `src/export/`, `src/site/`, `src/site/client/`, `src/llm/`, `src/util/`.
   Also create empty dirs with `.gitkeep`: `fixtures/`, `assets/site/`,
   `scripts/`, `examples/`, `test/e2e/`. (`examples/pages.yml` itself is
   issue 34's deliverable; this issue creates only the placeholder directory.)
4. ESLint (flat config) with typescript-eslint recommended-type-checked;
   rules: `@typescript-eslint/no-explicit-any: error`,
   `@typescript-eslint/no-floating-promises: error`, `eqeqeq: error`.
   Prettier as formatter (no eslint style rules).
5. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `*.tsbuildinfo`,
   `.DS_Store`. Do NOT ignore `assets/site/` (committed bundle, ADR-004).
6. `LICENSE`: MIT, copyright `2026 glossary contributors`.
7. `vitest.config.ts`: node environment, include `src/**/*.test.ts` and
   `test/**/*.test.ts`, coverage provider v8 (thresholds set later).
8. README is NOT rewritten here (issue 39); leave as-is.
9. `src/cli/main.ts` placeholder: prints exactly `glossary CLI placeholder`
   to stdout and exits 0 — no argument parsing, no command handling (the real
   CLI is issue 06).
10. Supply-chain verifier `scripts/check-dep-scripts.mjs` (wired into
   `npm run lint`): reads the production dependency tree
   (`npm ls --omit=dev --all --json` or walking `node_modules` from
   package-lock) and exits non-zero if ANY production dependency's
   package.json declares `preinstall`, `install`, `postinstall`, or `prepare`.
   This repo's own package.json must declare none of those lifecycle scripts.

## Acceptance Criteria

- [ ] `npm ci && npm run typecheck && npm run lint && npm run test && npm run build` all exit 0 on Node 22, 24, and 26.
- [ ] `node dist/cli/main.js` prints `glossary CLI placeholder` and exits 0.
- [ ] Direct runtime dependencies in package.json are exactly the ADR-001 allowlist (name-for-name).
- [ ] `node scripts/check-dep-scripts.mjs` exits 0 on the installed tree, and fails when pointed at a fixture tree containing a dep with a `postinstall` script (unit-test the verifier itself).
- [ ] `package-lock.json` committed; LICENSE present; all skeleton dirs exist.

## Validation

Run the command chain above on Node 22, 24, and 26 (nvm/volta locally; CI in
issue 02). Paste outputs in the PR. `git status` clean after build (dist
ignored).

## Dependencies

None.

## Non-goals

CI (02), CLI behavior (06), client bundle build script (32), README (39).

## Design References

DESIGN.md §5 (layout), §15 (dependency allowlist), §17 (platform);
ADR-001; ADR-006 (license).
