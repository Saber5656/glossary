# ADR-003: Japanese tokenization via kuromoji behind a swappable interface

- Status: Accepted (2026-07-11)
- Deciders: designer (owner delegated library choice)
- Related: DESIGN.md §9.3–§9.4, docs/research/japanese-term-extraction.md

## Context

Extracting Japanese domain terms requires POS tagging to build compound-noun
candidates. Constraints from ADR-001: pure JS, no native/WASM modules, offline,
deterministic. Research (see research doc) compared kuromoji.js, lindera-wasm,
Sudachi WASM builds, and segmenter-only options.

## Decision

1. Default tokenizer: **`kuromoji`** (npm, pure JS, bundled IPADIC), loaded
   lazily as a process-wide singleton.
2. All access goes through a `JaTokenizer` interface in `src/tokenize/` so the
   implementation can be swapped (lindera-wasm is the documented successor if
   quality/health triggers fire — research doc §6).
3. Tokenizer failure degrades gracefully: extractor E1 falls back to
   heuristics-only mode (katakana runs, kanji runs, quoted phrases) with a
   warning; the CLI never hard-fails because of the tokenizer.
4. Candidate scoring uses the FLR method (research doc §4); the exact formula
   and worked example are frozen in the E1 issue file.
5. IPADIC's license notice is included in the repository NOTICE section
   (README credits) at implementation time.

## Consequences

- Dictionary vocabulary is dated (IPADIC 2011); acceptable because team jargon
  is OOV for any dictionary — heuristic extractors carry OOV coverage.
- `kuromoji` is unmaintained: risk is pinned versions + interface isolation;
  known unknown U1 tracks install/runtime health on Node 22–26.
- Dictionary assets (~MB scale) ship via npm install only to the CLI side;
  they are never sent to the browser bundle.

## Alternatives considered

- **lindera-wasm**: maintained, modern dictionaries; rejected for v1 only
  because it introduces WASM loading/artifact weight; first-line replacement.
- **SudachiDict-built-for-kuromoji**: dictionary refresh path without code
  change; documented escape hatch.
- **budoux/TinySegmenter**: no POS info → cannot form noun-phrase candidates.
