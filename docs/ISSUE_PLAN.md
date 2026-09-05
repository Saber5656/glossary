# glossary — v1 Issue Plan

- Derived from: [DESIGN.md](DESIGN.md) (canonical). GitHub Issues are generated
  from `docs/issues/NN-*.md`; if they disagree, these files win.
- Issue files: `docs/issues/NN-short-title.md`, one implementation unit each,
  written for a lower-capability implementation agent (no guessing required).

## 1. v1 completion statement

When issues **01–42** are all closed with their Acceptance Criteria met and
their Validation steps executed, the v1 product defined in DESIGN.md §3.1 is
complete: a TypeScript CLI that deterministically extracts Japanese/English
term candidates from a repository, supports two-tier curation, exports
`GLOSSARY.md`, builds a self-contained searchable static site, offers
consent-gated LLM definition drafting, passes the hostile-fixture security
suite in CI, and has been manually validated on the owner's real repository —
with no product behavior specified outside DESIGN.md and these issues, except
newly discovered implementation unknowns (§7).

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | 01-repo-scaffolding.md | Repository scaffolding: package, TS strict, lint, test, license | W0 |
| 02 | 02-ci-workflow.md | CI workflow with SHA-pinned actions and least privilege | W0 |
| 03 | 03-errors-logging.md | Error taxonomy, exit codes, logger, injectable clock | W0 |
| 04 | 04-yaml-io.md | Safe YAML read/write module with hard limits and atomic writes | W0 |
| 05 | 05-config-loader.md | Config schema (zod) and loader with fail-closed validation | W0 |
| 06 | 06-cli-skeleton.md | CLI skeleton: commander program, global flags, JSON plumbing | W0 |
| 07 | 07-init-command.md | `glossary init` scaffolding command | W0 |
| 08 | 08-term-identity.md | Normalization and term identity module (normalizeSurface/termKey/termId) | W1 |
| 09 | 09-terms-store.md | Curated terms store (`terms/<id>.yaml`) and schema | W1 |
| 10 | 10-candidates-store.md | Candidates store (`candidates.yaml`) and schema | W1 |
| 11 | 11-rejected-store.md | Rejected list store (`rejected.yaml`) and schema | W1 |
| 12 | 12-validate-command.md | `glossary validate`: invariants V1–V5 | W1 |
| 13 | 13-fixture-repos.md | Fixture repositories: ja-mixed, hostile, empty | W2 |
| 14 | 14-file-scanner.md | File scanner with ignore rules, guards, deterministic order | W2 |
| 15 | 15-markdown-extractor.md | Markdown content extractor (text blocks, structure facts, positions) | W2 |
| 16 | 16-code-extractor.md | Code content extractor (comments, strings, declared identifiers) | W2 |
| 17 | 17-ja-tokenizer.md | Japanese tokenizer wrapper (kuromoji) with graceful fallback | W2 |
| 18 | 18-identifier-splitter.md | Identifier splitter (camel/snake/kebab/acronym) | W2 |
| 19 | 19-extractor-ja-domain.md | Extractor E1: Japanese domain terms with FLR scoring | W3 |
| 20 | 20-extractor-identifiers.md | Extractor E2: code identifiers and API names | W3 |
| 21 | 21-extractor-abbreviations.md | Extractor E3: abbreviations and expansion pairing | W3 |
| 22 | 22-extractor-doc-definitions.md | Extractor E4: definitions already present in docs | W3 |
| 23 | 23-merge-pipeline.md | Candidate merge, combined scoring, drift data | W3 |
| 24 | 24-extract-command.md | `glossary extract` orchestration, summary, drift report | W3 |
| 25 | 25-read-commands.md | `glossary list` / `show` / `status` | W4 |
| 26 | 26-curation-commands.md | `glossary approve` / `reject` (lifecycle T2/T3/T5) | W4 |
| 27 | 27-export-command.md | `glossary export`: deterministic GLOSSARY.md | W4 |
| 28 | 28-e2e-golden-harness.md | E2E harness + golden files for extract→curate→export; determinism test | W4 |
| 29 | 29-search-tokenizer.md | Shared search tokenizer (CJK bigram + Latin words) | W5 |
| 30 | 30-site-data-build.md | `glossary build` shell: site data (terms.json, search-index.json), outDir hygiene | W5 |
| 31 | 31-site-templates.md | Auto-escaping template engine and site pages (index/term/tag/about, i18n, CSP) | W5 |
| 32 | 32-site-client.md | Browser search client + committed prebuilt bundle + rebuild-diff CI check | W5 |
| 33 | 33-site-hardening.md | Site security tests: hostile fixture rendering, no-external-URL scan | W5 |
| 34 | 34-pages-example.md | Example GitHub Pages deployment workflow + docs | W5 |
| 35 | 35-llm-client.md | OpenAI-compatible fetch client with consent-safe error handling | W6 |
| 36 | 36-llm-redaction.md | Snippet selection, redaction, payload builder | W6 |
| 37 | 37-draft-command.md | `glossary draft`: consent gate, dry-run preview, batching, writeback | W6 |
| 38 | 38-llm-e2e.md | LLM e2e with mock server: goldens, schema retry, injection fixture | W6 |
| 39 | 39-readme-docs.md | README.md (en) + README.ja.md + usage guide | W7 |
| 40 | 40-security-contributing-docs.md | SECURITY.md, CONTRIBUTING.md, third-party notices | W7 |
| 41 | 41-smoke-script.md | `scripts/smoke.sh` and public-repo smoke protocol | W7 |
| 42 | 42-dogfood-validation.md | Dogfood on this repo + owner manual validation protocol | W7 |

## 3. Dependency table

`A ← B` means B depends on A. Only direct dependencies listed; transitive
implied. "13*" = fixtures are a test-time dependency.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01, 03 |
| 05 | 01, 03, 04 |
| 06 | 01, 03, 05 |
| 07 | 04, 06 |
| 08 | 01, 03 |
| 09 | 04, 08 |
| 10 | 04, 08 |
| 11 | 04, 08 |
| 12 | 06, 09, 10, 11 |
| 13 | 01 |
| 14 | 03, 05, 13* |
| 15 | 01, 03, 13* |
| 16 | 01, 03, 13* |
| 17 | 01, 03 |
| 18 | 01, 03, 08 |
| 19 | 15, 17, 08, 13* |
| 20 | 16, 18, 08, 13* |
| 21 | 08, 15, 16, 18, 13* |
| 22 | 15, 08, 13* |
| 23 | 08, 09, 10, 11, 19, 20, 21, 22 |
| 24 | 06, 14, 15, 16, 23 |
| 25 | 06, 09, 10, 11 |
| 26 | 06, 09, 10, 11 |
| 27 | 06, 09 |
| 28 | 12, 13, 24, 26, 27 |
| 29 | 01, 08, 18 |
| 30 | 06, 09, 29 |
| 31 | 30 |
| 32 | 29, 30, 31 |
| 33 | 13, 28, 31, 32 |
| 34 | 02, 31 |
| 35 | 03, 05 |
| 36 | 03, 05, 10, 13* |
| 37 | 06, 09, 10, 35, 36 |
| 38 | 13, 28, 33, 37 |
| 39 | 02, 07, 12, 24, 25, 26, 27, 30, 34, 37, 40 (documented behavior must exist) |
| 40 | 01 |
| 41 | 24, 30, 33 |
| 42 | 28, 33, 38, 39, 40, 41 (final gate runs on finished docs/UX) |

## 4. Implementation waves

| Wave | Issues | Goal / gate |
|---|---|---|
| W0 Foundation | 01–07 | `glossary init` runs; CI green on an almost-empty codebase |
| W1 Stores | 08–12 | Stores round-trip; `validate` enforces V1–V5 |
| W2 Ingestion | 13–18 | Scanner + content + tokenizers unit-tested on fixtures |
| W3 Extraction | 19–24 | `glossary extract` produces deterministic candidates on fixtures |
| W4 Curation | 25–28 | Full extract→approve→export loop covered by golden e2e |
| W5 Site | 29–34 | `glossary build` emits hardened searchable site; hostile fixture inert |
| W6 LLM | 35–38 | Consent-gated drafting with dry-run; mock-server e2e green |
| W7 Release readiness | 39–42 | Docs complete; smoke + dogfood + owner validation executed |

Within a wave, issues sharing no dependency edge may proceed in parallel
(e.g. 09/10/11; 14–18; 19–22; 25/26/27; 35/36).

## 5. Coverage table (DESIGN.md → issues)

| DESIGN.md section | Covered by |
|---|---|
| §4 design principles | determinism → 03 (clock), 23 (shuffle-invariance), 28 (byte goldens), 30/31 (site determinism); untrusted-input posture → 13, 33, 38; dependency-ADR discipline → 01 |
| §5 product repo layout | 01 |
| §6 target-repo layout & ownership rules | 04, 07, 24 (write discipline), 26 |
| §7.1 normalization/identity | 08 |
| §7.2 config schema | 05 |
| §7.3 candidates schema | 10 |
| §7.4 curated term schema | 09 |
| §7.5 rejected schema | 11 |
| §8 lifecycle & invariants | 12, 24 (T4/T7), 26 (T2/T3/T5), 37 (T8/T9) |
| §9.1 scanner | 14 |
| §9.2 content extraction | 15, 16 |
| §9.3 tokenization | 17, 18, 29 |
| §9.4 extractors E1–E4 | 19, 20, 21, 22 |
| §9.6 scoring & merge, drift | 23, 24 |
| §10 CLI commands & conventions | 06, 07, 12, 24, 25, 26, 27, 30, 37 |
| §11 site | 29, 30, 31, 32, 33, 34 |
| §12 LLM subsystem | 35, 36, 37, 38 |
| §13 security model B1 | 14 |
| §13 B2/B2' | 04, 12, 19–22 (regex discipline), 03 |
| §13 B3 | 31, 32, 33 |
| §13 B4 | 35, 36, 37, 38 |
| §13 B5 | 01 (lockfile, no lifecycle scripts, runtime-dep allowlist + verifier), 02 (npm ci, SHA-pinned actions, Dependabot) |
| §13 B6 | 02, 34 |
| §14 errors/logging | 03, 06 |
| §15 dependency allowlist | 01 |
| §16 testing strategy | 02, 13, 28, 33, 38, 41, 42 |
| §17 platform | 01, 02 |
| §19 dogfooding | 42 |

Every DESIGN.md behavioral section maps to at least one issue; no v1 behavior
lives only in prose.

## 6. Validation strategy (whole product)

1. **Per-issue**: each issue's Validation section is mandatory before close;
   CI (issue 02) runs lint + typecheck + unit + e2e on Node 22/24/26.
2. **Cross-cutting gates**:
   - Golden e2e (28) freezes extract/curate/export behavior; any diff is a
     reviewed decision.
   - Determinism test (28) guards DESIGN §4's core principle.
   - Security abuse cases map to tests: AC1→33, AC2→04/12, AC3→14, AC4→36,
     AC5→38, AC6→02 (DESIGN §13).
   - Client bundle rebuild-diff check (32) keeps the committed artifact honest.
3. **Real-world**: smoke script on public OSS repos (41), dogfood on this repo
   and owner's manual protocol on a real team repo (42) — the owner sign-off in
   42 is the final v1 gate.

## 7. Known unknowns (may create additional issues)

Tracked in DESIGN.md §3.4: U1 kuromoji health (17), U2 extraction precision
defaults (42 feedback loop), U3 Windows behavior (02's non-blocking job),
U4 index size at scale (30 perf budget), U5 npm naming (v2), U6 JSON-mode
variance across OpenAI-compatible servers (35/38 fallback). Any trigger firing
during implementation spawns a new numbered issue rather than silently widening
an existing one.

## 8. Deferred to v2 (not planned here)

npm packaging/naming, GitHub Action wrapper, MCP server, multi-repo
aggregation, tree-sitter extractors, LLM candidate re-ranking/clustering,
glossary lint (prose consistency checks), incremental extraction/watch mode,
Pagefind-scale search, reading auto-fill, CLI i18n (DESIGN.md §3.3, §14).
