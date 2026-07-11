# 39 — README.md (en) + README.ja.md + usage guide

## Title

User-facing documentation: bilingual README and the usage guide

## Summary

Replace the placeholder README with the English README + full Japanese
README.ja.md per ADR-006 §2, documenting install, the core workflow, every
command, configuration, and the LLM opt-in with its security posture.

## Context

v1 distribution is `git clone` (R4) — the README IS the installer UX. The
primary persona is Japanese (README.ja.md must be complete, not a summary).

## Scope

- `README.md`, `README.ja.md`, `docs/guides/usage.md` (single detailed
  reference both READMEs link to). Hosting guide exists (34).

## Detailed Requirements

1. `README.md` (English) sections, in order:
   - Title + tagline (keep the original Japanese line as the subtitle:
     「リポジトリからチーム用語集を自動構築する」) + CI badge (02).
   - What it does (3 bullets + the §1.2 journey block, copy-adapted).
   - Install (v1): `git clone` → `npm ci` → `npm run build` → `npm link`;
     Node ≥ 22 requirement; verification `glossary --version`.
   - Quick start: init → extract → list/show → approve/reject → export →
     build → preview locally with the exact command
     `python3 -m http.server -d glossary/site 8000` (link hosting.md for
     deployment).
   - Configuration: table of every config key (§7.2) with defaults — GENERATE
     faithfully from the design (drift here is a doc bug).
   - LLM drafting (opt-in): consent model, dry-run example, redaction summary,
     provider examples (OpenAI, OpenRouter, Ollama local with http
     localhost note), "what is sent / what is never sent" table (§12.2).
   - Security posture summary linking SECURITY.md (40): no network by
     default, no telemetry, self-contained site.
   - Language note (JA-focused extraction, EN identifiers), link README.ja.md.
   - Credits/licenses: MIT + kuromoji Apache-2.0 + IPADIC notice (ADR-003 §5).
2. `README.ja.md`: full Japanese equivalent (not a stub) — same section set,
   natural Japanese, identical command examples.
3. `docs/guides/usage.md`: starts with a **Global flags** section (`--dir`,
   `--repo`, `--json` incl. the envelope shape, `--verbose`, `--no-color`,
   `--version`, `--help` — defaults and stdout/stderr conventions from
   DESIGN §10); then a per-command reference — for EACH of the 11 commands:
   synopsis, flags table, exit codes, `--json` data shape (copied from the
   frozen shapes in issues 07/12/24–27/30/37), 2+ realistic examples (JA
   terms); then a troubleshooting section (tokenizer fallback warning,
   E_NOT_INITIALIZED, E_OUTDIR_UNSAFE, consent errors — each with
   cause/fix).
4. Cross-link check `scripts/check-docs-links.mjs` (run in CI): scans
   `README.md`, `README.ja.md`, `docs/**/*.md`; validates relative FILE
   links and intra-file heading anchors; ignores `http(s)://` URLs entirely
   (no network); exits non-zero printing `<source>: <broken link>` lines.
5. All examples must be copy-paste runnable against `fixtures/repo-ja-mixed`
   (state this and verify once via the quick-start transcript below).

## Acceptance Criteria

- [ ] Both READMEs complete with the section sets above; README.ja.md has owner (native-Japanese) review sign-off recorded in the PR, requested fixes applied.
- [ ] Config table matches DESIGN §7.2 key-for-key (review diff side by side).
- [ ] usage.md covers the global-flags section + all 11 commands × all flags; JSON shapes match the issue-frozen ones.
- [ ] Link-check script green in CI; badge renders.
- [ ] Quick-start transcript: from a CLEAN copy of `fixtures/repo-ja-mixed`, run the exact README quick-start commands under `set -e` through `glossary build`; fenced transcript (commands + key output lines) pasted in the PR.

## Validation

Manual execution of quick start; link-check CI step; owner reads README.ja.md
(42's session includes doc feedback).

## Dependencies

02 (CI step add), 07, 12, 24, 25, 26, 27, 30, 34 (hosting.md exists to link),
37, 40 (SECURITY.md exists to link) — documentation must describe shipped
behavior.

## Non-goals

Hosting guide (34 owns), SECURITY/CONTRIBUTING (40), API/library docs (no
public API in v1), docs site.

## Design References

DESIGN.md §1, §3.1 item 8, §7.2, §10, §12; ADR-003 §5; ADR-006 §2/§4.
