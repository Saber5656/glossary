# Research: Japanese term extraction for Node.js (2026-07)

Status: Accepted input for [ADR-003](../decisions/ADR-003-japanese-tokenization.md).
Scope: choose the v1 Japanese tokenization library and the term-candidate scoring
approach used by the `ja-domain` extractor (DESIGN.md §9).

## 1. Problem

The core value of glossary is extracting **domain terms from Japanese documents**
(README, design docs, ADRs) mixed with English source code. Japanese has no word
boundaries, so naive whitespace tokenization does not work. We need:

1. A tokenizer with part-of-speech (POS) tagging to build **compound noun
   candidates** (e.g. 支払 + 予約 → 支払予約).
2. A **deterministic, offline, dependency-light** implementation (secure default:
   no network, no native build toolchain, reproducible CI).
3. A scoring method to rank multi-word/compound candidates.

## 2. Candidates evaluated

| Library | Type | Maintenance (2026-07) | Dictionary | Native/WASM | Notes |
|---|---|---|---|---|---|
| `kuromoji` (kuromoji.js) | Pure JS port of Kuromoji | Unmaintained (last release ~2018) | IPADIC (2011) bundled | None (pure JS) | De-facto standard for JS; huge install base; deterministic; dictionary is old but stable |
| `lindera` / lindera-wasm | Rust morphological analyzer compiled to WASM | Actively maintained | IPADIC / UniDic variants | WASM binary in package | Modern dictionaries; adds WASM loading + larger artifacts; API less established in JS ecosystem |
| Sudachi (WASM builds) | Rust (sudachi.rs) WASM experiments | Repo archived / not maintained for JS | SudachiDict | WASM | Not viable as a supported JS package |
| SudachiDict-for-kuromoji | Dictionary rebuild for kuromoji.js | Community recipe | SudachiDict (updated) | None | Escape hatch if IPADIC vocabulary gaps hurt precision |
| TinySegmenter / budoux | Segmenter only (no POS) | Maintained (budoux) | n/a | None | No POS tags → cannot build noun-phrase candidates reliably; rejected |

Sources:

- kuromoji.js repository: <https://github.com/takuyaa/kuromoji.js/>
- kuromoji.js status and JS NLP constraints: <https://end0tknr.hateblo.jp/entry/20240514/1715664493>
- lindera-wasm packaging for npm: <https://zenn.dev/higumachan/articles/a42f1ee50bbb8d>
- SudachiDict built for kuromoji.js (IPADIC/UniDic comparison): <https://qiita.com/piijey/items/2517af039bbedddec7b8>
- kuromoji.js custom dictionary entries: <https://eieito.hatenablog.com/entry/2025/07/31/090000>

## 3. Decision inputs

- **Pure JS beats freshness for v1.** kuromoji.js is unmaintained but frozen and
  widely deployed; it has no install scripts, no native compilation, and fully
  deterministic output — all properties we rank above dictionary recency.
- **Old dictionary risk is bounded.** Team-specific jargon is *always* OOV
  (out-of-vocabulary) regardless of dictionary age. The extractor therefore must
  not depend solely on dictionary hits: katakana runs, kanji runs, and quoted
  strings (「…」) are harvested by heuristics independent of the tokenizer.
- **Swap must stay cheap.** All tokenizer access goes through a `JaTokenizer`
  interface (one module). If precision on real repos is unsatisfying, lindera-wasm
  (or a SudachiDict rebuild) replaces the default without touching extractors.
- **Supply chain**: `kuromoji` bundles its dictionary as static gzipped files
  loaded from the package directory — no network fetch at runtime. License:
  Apache-2.0 (code), IPADIC license (dictionary; redistribution permitted with
  notice — must be listed in NOTICE/credits).

## 4. Scoring approach for compound-noun candidates

Use the classic **FLR score** (Nakagawa/Mori) as used by the `termextract`
family of tools, simplified:

```
FLR(CN) = f(CN) * ( Π_{i=1..L} (FL(Ni)+1)(FR(Ni)+1) )^(1/2L)
```

- `CN` = compound noun candidate composed of nouns `N1..NL`
- `f(CN)` = frequency of the exact candidate
- `FL(N)` / `FR(N)` = number of distinct nouns appearing directly left/right of
  `N` inside other candidates (connectivity)

Rationale: FLR needs no training data, is deterministic, cheap (single pass +
count maps), and is the standard baseline for Japanese technical-term
extraction. Exact formula, tie-breaking, and a worked example are specified in
the extractor issue (docs/issues) so an implementation agent does not guess.

## 5. Decision

- v1 default tokenizer: **`kuromoji` (npm), IPADIC, lazy-loaded singleton**.
- Heuristic extractors (katakana/kanji runs, quoted terms) run independently so
  a tokenizer failure degrades, not disables, extraction.
- Scoring: FLR as above, combined with occurrence thresholds from config.

## 6. Revisit triggers (v2 candidates)

- Precision/recall on real repos measurably poor for OOV-heavy vocabularies →
  evaluate lindera-wasm or SudachiDict-for-kuromoji.
- kuromoji install breakage on a supported Node line → pin fork or vendor.
- Need for reading (よみ) auto-fill → tokenizer already returns readings for
  dictionary words; expose later.
