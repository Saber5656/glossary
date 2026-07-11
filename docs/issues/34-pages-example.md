# 34 — Example GitHub Pages deployment workflow + docs

## Title

`examples/pages.yml`: least-privilege GitHub Pages deploy example and hosting guide

## Summary

Provide a copy-paste GitHub Actions workflow that builds the glossary site in
a USER's repository and deploys it to GitHub Pages, plus a short hosting guide
— documentation artifacts only, per DESIGN.md §13-B6.

## Context

Users asked "make it viewable by the team" (R7); Pages is the default answer.
The example runs in user repos with their permissions, so it must model
least-privilege and SHA-pinning (it will be copied verbatim, mistakes
propagate).

## Scope

- `examples/pages.yml`, `docs/guides/hosting.md`. NOT installed into this
  repo's workflows (this repo dogfoods separately in 42 if desired).

## Detailed Requirements

1. `examples/pages.yml`:
   - Trigger: `push` to `main` with `paths: ['glossary/**']` +
     `workflow_dispatch`.
   - `permissions:` block exactly: `contents: read`, `pages: write`,
     `id-token: write` (the Pages deploy minimum) — top level, single job
     chain `build` → `deploy` using `actions/upload-pages-artifact` +
     `actions/deploy-pages`, `environment: github-pages`.
   - Build job steps: checkout → setup-node 24 → install glossary from git
     (v1 distribution: `npm install github:Saber5656/glossary#<pinned-sha>` —
     placeholder with a comment telling users to pin a commit) → `npx
     glossary build --out _site` → upload artifact `_site`.
   - All actions SHA-pinned with version comments; `concurrency` group
     `pages`; timeout 10 min.
   - Comment header: what it does, prerequisites (Pages enabled, Settings →
     Pages → Source: GitHub Actions), and a warning: the site publishes
     glossary content — confirm the repo/site visibility matches the
     information sensitivity (public Pages on a private repo leaks terms).
2. `docs/guides/hosting.md` (English):
   - GitHub Pages setup steps (with the visibility warning above, prominent).
   - Alternative: any static host — copy `glossary/site/`; note CSP is a meta
     tag; recommended extra HTTP headers if the host allows
     (X-Content-Type-Options, Referrer-Policy) with a table.
   - Local preview: `python3 -m http.server -d glossary/site` (or
     `npx serve`), and why file:// won't work (fetch of JSONs).
3. Workflow YAML must pass `actionlint` (add as a dev script `lint:actions`
   scanning `.github/workflows` AND `examples/` — actionlint binary via
   `npx`? NO binary deps: instead validate in CI with the official
   actionlint docker/binary is NOT acceptable either → use
   `actionlint`'s GitHub Action in OUR ci.yml, SHA-pinned, scanning both
   dirs — amend ci.yml here).

## Acceptance Criteria

- [ ] actionlint (CI step) passes over ci.yml + examples/pages.yml.
- [ ] pages.yml has the exact permissions block, SHA-pinned actions, path filter, concurrency, and the visibility warning comment.
- [ ] hosting.md covers Pages setup, generic hosts + header table, local preview, and the sensitivity warning.
- [ ] Dry validation: the workflow runs green in a scratch fork/testing repo with a sample glossary (paste run link in the PR — one-time manual proof).

## Validation

actionlint in CI; the scratch-repo deployment run link; docs reviewed for
step-by-step reproducibility.

## Dependencies

31 (site exists to deploy); 02 (ci.yml to amend).

## Non-goals

Auto-installing the workflow via `init` (v2 candidate), Netlify/Vercel
configs, custom domains, publishing THIS repo's Pages (owner decision, 42).

## Design References

DESIGN.md §13-B6, §11 (site as artifact), §3.1 item 8; ADR-006 §5.
