# 32 — Browser search client + committed prebuilt bundle + rebuild-diff CI check

## Title

`src/site/client/`: search/filter UI, esbuild bundling to `assets/site/app.js`, drift check

## Summary

Implement the browser half of the site per DESIGN.md §11.3 and ADR-004: load
the index, run MiniSearch with the shared tokenizer, render results safely,
plus the build script producing the committed bundle and a CI check that the
bundle matches the source.

## Context

The client runs in every glossary reader's browser under the site CSP; it must
render untrusted strings only via DOM text APIs. The bundle is a committed
artifact (users don't run esbuild), so CI must prove it's honest.

## Scope

- `src/site/client/app.ts` (+ small modules), `scripts/build-client.mjs`,
  `assets/site/app.js` + `style.css`, CI step addition, tests (node-side unit
  + jsdom DOM tests).

## Detailed Requirements

1. Client behavior (progressive enhancement; script is `type=module`; all
   element lookups via issue 31's `src/site/contract.ts` DOM constants —
   never hand-typed selectors):
   1. Fetch `./search-index.json` + `./terms.json` (same-origin relative;
      any failure ⇒ leave static list working, `console.warn`, keep search
      disabled with title = `<body data-msg-search-disabled>`).
   2. `MiniSearch.loadJSON(indexJson, miniSearchOptions())` (29 — bundled).
      Build a `Map<id, term-record>` from terms.json — hits carry ONLY `id`
      (30's storeFields); all display data comes from that map.
   3. Enable search input; on input (debounced 150 ms): query; empty query ⇒
      restore static list visibility; else hide static list and render
      results container (empty results show `data-msg-no-results` text).
   4. Result rendering: for each hit (max 50) build DOM via
      `document.createElement` + `textContent` ONLY; each result: link with
      href from the KNOWN-SAFE template
      `terms/${encodeURIComponent(id)}.html`, term, kind badge, definition
      first 120 chars — all from the terms.json map.
   5. Filters: kind CHIPS and tag CHIPS (31's contract; data from
      terms.json). Behavior: at most one active kind chip and one active tag
      chip (click toggles; clicking another replaces); active filters AND
      together and AND with the query; they apply to both search results and
      the static list (hide/show via a class on `termItem` elements using
      their data-kind/data-tags).
   6. No state in URL (v1), no history API, no storage APIs, no workers.
2. Accessibility: results region `aria-live="polite"`; input labeled; chips
   are buttons with `aria-pressed`; keyboard operability (tab/enter).
3. `scripts/build-client.mjs`: esbuild — entry app.ts, bundle, format=esm,
   target `es2022`, minify, `define` nothing, sourcemap=false. Banner comment
   `/* built from src/site/client @ <hash12> */` where `hash12` = first 12
   hex of sha256 over the concatenation of every file under
   `src/site/client/**/*.ts` PLUS the two shared imports
   (`src/tokenize/search.ts`, `src/tokenize/identifier.ts`,
   `src/site/contract.ts`), sorted by relPath, each framed as
   `relPath + '\0' + LF-normalized content + '\0'`.
   Output `assets/site/app.js`. `style.css` is hand-written static (no
   preprocessor), lives directly in `assets/site/`. No URLs anywhere in the
   bundle or banner (33's scanner enforces).
4. npm scripts: `build:client`; CI (02 workflow amended here) adds step:
   run build:client then `git diff --exit-code assets/site/` (drift check).
5. Sink lint fence for `src/site/client/**` (exact bans, with a failing
   fixture proving each fires): `innerHTML`, `outerHTML`,
   `insertAdjacentHTML`, `document.write`, `document.writeln`,
   `DOMParser.prototype.parseFromString`, `Range.prototype.createContextualFragment`
   — via eslint `no-restricted-properties`/`no-restricted-syntax` entries.
6. jsdom tests: inject a 3-term fixture DOM+JSONs; assert: XSS-bait term
   renders as textContent (no element injection — query for img/script in
   results = none); kind+tag chip AND-filtering on both list and results;
   fetch-failure path leaves static list usable.
7. Bundle budget: app.js ≤ 60 KB minified (MiniSearch ~30 KB) — assert size in
   test.

## Acceptance Criteria

- [ ] jsdom suite green incl. XSS textContent assertions and fetch-failure fallback.
- [ ] Rebuild-diff CI step passes; touching client source without rebuilding fails CI (prove once locally, describe in PR).
- [ ] Bundle ≤ 60 KB; banner hash present and matches recomputation.
- [ ] Every banned DOM sink fails lint under `src/site/client/**` (fixture-file proof for each of the 7 bans).
- [ ] Manual smoke (validation aid, exact recipe): `node dist/cli/main.js build --repo fixtures/repo-ja-mixed` on a fixture copy after scripted curation → `python3 -m http.server -d <outDir>` → open `http://localhost:8000/` → JA query `支払` and EN query `payment` both list the 支払予約 result linking `terms/payment-reservation.html`.

## Validation

Unit/jsdom tests; manual browser check via local static server; CI link.

## Dependencies

29, 30 (data shapes), 31 (imports the DOM contract `src/site/contract.ts`
that 31 creates).

## Non-goals

Framework adoption, URL state/deep-linking (v2), offline service worker,
analytics (forbidden by ADR-006).

## Design References

DESIGN.md §11.2–11.3, §13-B3; ADR-004; research/client-side-search.md.
