# Research: client-side search for the generated static site (2026-07)

Status: Accepted input for [ADR-004](../decisions/ADR-004-client-side-search.md).
Scope: choose the search mechanism for the generated glossary site (DESIGN.md §11),
under the constraints: **fully static output, no external requests, no native
binaries in the build chain, Japanese + English content**.

## 1. Requirements

| Requirement | Why |
|---|---|
| Runs entirely in the browser from static files | Site must be deployable to GitHub Pages / any static host with zero backend |
| Japanese substring/word search that works without spaces | Terms and definitions are mostly Japanese |
| Index built at `glossary build` time, deterministic output | Golden-file testing, reproducible builds |
| No network access from the built site | Security posture: generated site makes no external requests |
| No extra binary dependency in the build toolchain | Supply-chain minimalism; npm-only install |
| Scale target: 10^2–10^3 terms | A team glossary is small; we do not need web-scale chunked indexes |

## 2. Candidates evaluated

| Candidate | CJK handling | Build-time deps | Runtime | Verdict |
|---|---|---|---|---|
| **MiniSearch** | Custom `tokenize` hook → implement character **bigram** tokenization for CJK ourselves | Pure JS (npm) | Pure JS, small (~7 KB gz) | **Chosen**: full control, deterministic, self-contained |
| Pagefind | Excellent: segmentation at index time; since v1.5.0 query-side CJK segmentation via `Intl.Segmenter` | **Rust binary** (downloaded platform binaries via npm wrapper) | Its own JS UI + chunked index fetches | Rejected for v1: binary dependency + framework-shaped UI; overkill for ≤10^3 terms |
| Lunr + lunr-languages (ja) | TinySegmenter-based `lunr.ja` | Pure JS | Pure JS | Rejected: project dormant; ja support quality mediocre; index size larger |
| Fuse.js | Fuzzy matching over raw strings; no real tokenization | Pure JS | Pure JS | Rejected: O(n·m) scan per query; poor ranking for JA compounds |
| Roll-our-own substring scan | Trivial | None | Trivial | Fallback only; no ranking, no field weighting |

Sources:

- MiniSearch documentation (custom tokenizer, in-browser usage): <https://lucaong.github.io/minisearch/>
- MiniSearch on npm: <https://www.npmjs.com/package/minisearch>
- CJK bigram approach as used by mainstream search engines: <https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-cjk-bigram-tokenfilter.html>
- Pagefind multilingual support: <https://pagefind.app/docs/multilingual/>
- Pagefind v1.5.0 query-side CJK segmentation via Intl.Segmenter: <https://github.com/Pagefind/pagefind/releases/tag/v1.5.0>
- Pagefind CJK substring-search limitations discussion: <https://github.com/Pagefind/pagefind/issues/987>

## 3. Tokenization strategy (shared build/query module)

One tokenizer module used **both** at index build time (Node) and query time
(browser) so index and query agree:

1. NFKC-normalize, lowercase Latin.
2. Split into runs by script class: Latin/digit runs → word tokens (plus
   camelCase/snake_case sub-splitting); CJK runs → **character bigrams**
   (a single CJK char run of length 1 emits a unigram).
3. Katakana runs are kept both as a whole token and as bigrams (improves exact
   matches for loanwords).

Bigram indexing is the standard engine-side technique for CJK when no
morphological analyzer is available client-side; it trades index size for
recall, which is acceptable at glossary scale. We deliberately do not ship
kuromoji to the browser (dictionary is ~MB-scale and unnecessary).

## 4. Decision

- **MiniSearch** with the shared bigram tokenizer; fields: `term` (boost 3),
  `aliases`/`reading` (boost 2), `definition`, `tags`.
- Index serialized to `search-index.json` at build time; loaded with a
  same-origin `fetch` by the client bundle.
- Site works without JavaScript: the index page statically lists all terms;
  search/filter is progressive enhancement.

## 5. Revisit triggers (v2 candidates)

- Glossaries ≥ ~5k terms or index JSON ≥ ~2 MB → evaluate Pagefind (chunked
  index) despite the binary dependency.
- Need cross-page full-text search of definitions bodies → Pagefind again.
- `Intl.Segmenter`-based query segmentation as a lighter alternative to bigrams
  once baseline ships.
