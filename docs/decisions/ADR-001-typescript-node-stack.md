# ADR-001: TypeScript on Node.js, ESM, minimal dependency policy

- Status: Accepted (2026-07-11)
- Deciders: product owner (stack choice), designer (details)
- Related: DESIGN.md §5, §15, §17

## Context

The product is a CLI + static site generator targeting Japanese-doc/English-code
repositories, implemented by delegated agents (Codex), intended for OSS release.
Owner selected TypeScript/Node over Python and Go/Rust in the requirements
interview (R8). We must fix runtime, module system, and a dependency policy so
implementation issues are unambiguous.

## Decision

1. **TypeScript (strict) on Node.js ≥ 22**, ESM only (`"type": "module"`),
   `moduleResolution: NodeNext`. CI matrix: Node 22 / 24 / 26 (as of 2026-07,
   24 = Active LTS, 22 = Maintenance LTS, 26 = Current; source:
   endoflife.date/nodejs).
2. Build: `tsc` for the CLI; `esbuild` (dev-dependency) bundles the browser
   client into `assets/site/app.js`, committed so the published package/clone
   needs no browser build step.
3. Package remains `"private": true` in v1 (distribution = git clone; npm
   naming is a v2 decision, known unknown U5). Binary name: `glossary`.
4. **Runtime dependency allowlist** (DESIGN.md §15): commander, zod, yaml,
   fast-glob, ignore, unified/remark-parse/mdast-util-to-string, kuromoji,
   minisearch. Adding a runtime dependency requires amending this ADR in the
   same PR. No dependencies with install scripts.
5. No native modules, no WASM in v1 (pure JS/TS end to end).

## Consequences

- Pure-JS constraint drives tokenizer (ADR-003) and search (ADR-004) choices.
- Windows support is best-effort (CI `continue-on-error`), revisited via U3.
- Node < 22 users are unsupported; acceptable for a dev tool in 2026.
- Committed client bundle must be reproducible; a CI check rebuilds and diffs
  it (site issues include this).

## Alternatives considered

- **Python**: strongest JA NLP (SudachiPy) but weaker fit for the static-site
  client and future npm/Action/MCP distribution.
- **Go/Rust**: single-binary distribution is attractive, but JA morphological
  analysis and HTML tooling would raise implementation cost sharply for v1.
