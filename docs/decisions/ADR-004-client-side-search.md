# ADR-004: Self-contained static site with MiniSearch + CJK bigram tokenizer

- Status: Accepted (2026-07-11)
- Deciders: product owner (R7), designer (library choice)
- Related: DESIGN.md §11, docs/research/client-side-search.md

## Context

R7 requires a static glossary site with client-side full-text search usable by
non-engineers, deployable on GitHub Pages. Security posture requires the built
site to be fully self-contained (no CDN, no external requests). Content is
Japanese-heavy, so whitespace tokenization is insufficient. Scale is small
(10^2–10^3 terms).

## Decision

1. Search library: **MiniSearch** (pure JS) with a custom tokenizer shared
   between index build (Node) and query (browser): NFKC → script-run split →
   Latin word tokens (+identifier sub-splitting) and **CJK character bigrams**;
   katakana runs also emitted whole. One module, used on both sides.
2. Index and term data are serialized at `glossary build` time to
   `search-index.json` / `terms.json`, fetched same-origin by the client.
3. The client bundle (`assets/app.js`) is prebuilt with esbuild in this repo
   and committed; `build` copies it. No user-side bundling.
4. Site is progressive enhancement: index.html statically lists all terms
   (works with JS disabled); search/filter activate with JS.
5. Hard security rules (DESIGN.md §11.2): auto-escaping templates, CSP meta,
   no inline scripts/styles, no external URLs (test-enforced), client renders
   via `textContent` only.

## Consequences

- Bigram indexing inflates the index versus morphological indexing; at
  glossary scale this is negligible (known unknown U4 guards the limit).
- Search quality: substring-like recall for JA (bigram standard); good enough
  for term lookup; definition-body long-text search is v2 (Pagefind trigger).
- No framework dependency; templates are typed functions — testable and
  XSS-safe by construction.

## Alternatives considered

- **Pagefind**: best-in-class CJK for static sites, but ships a Rust binary
  via npm wrapper (supply-chain + platform surface) and its own UI; overkill
  at our scale. Documented v2 trigger.
- **lunr + lunr-languages**: dormant, mediocre JA handling.
- **Fuse.js**: linear scan fuzzy match; poor ranking, no field weighting fit.
