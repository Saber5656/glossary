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

1. Client behavior (progressive enhancement; script is `type=module`):
   1. Fetch `./search-index.json` + `./terms.json` (same-origin relative;
      any failure ⇒ leave static list working, log console.warn, keep search
      disabled with tooltip string from i18n embedded in DOM data-attrs by 31).
   2. `MiniSearch.loadJSON(indexJson, searchOptions())` (29 — bundled).
   3. Enable search input; on input (debounced 150 ms): query; empty query ⇒
      restore static list visibility; else hide static list and render
      results container.
   4. Result rendering: for each hit (max 50) build DOM via
      `document.createElement` + `textContent` ONLY (lint rule: no innerHTML/
      insertAdjacentHTML/outerHTML in client code — eslint no-restricted-
      properties); each result: link to `terms/<id>.html` (id from
      storeFields; href built from a KNOWN-SAFE template `terms/${encodeURIComponent(id)}.html`),
      term, kind badge, definition first 120 chars.
   5. Filters: kind chips + tag select (data from terms.json); filters apply
      to both search results and static list (hide/show via class).
   6. No state in URL (v1), no history API, no storage APIs, no workers.
2. Accessibility: results region `aria-live="polite"`; input labeled; chips
   are buttons with `aria-pressed`; keyboard operability (tab/enter).
3. `scripts/build-client.mjs`: esbuild — entry app.ts, bundle, format=esm,
   target `es2022`, minify, `define` nothing, sourcemap=false (committed
   artifact stays reviewable-small), banner comment with source hash:
   `/* built from src/site/client @ <sha256 of sources, 12 hex> */`.
   Output `assets/site/app.js`. `style.css` is hand-written static (no
   preprocessor), lives directly in `assets/site/`.
4. npm scripts: `build:client`; CI (02 workflow amended here) adds step:
   run build:client then `git diff --exit-code assets/site/` (drift check).
5. jsdom tests: inject a 3-term fixture DOM+JSONs; assert: XSS-bait term
   renders as textContent (no element injection — query for img/script in
   results = none); filter interaction; fetch-failure path leaves static list.
6. Bundle budget: app.js ≤ 60 KB minified (MiniSearch ~30 KB) — assert size in
   test.

## Acceptance Criteria

- [ ] jsdom suite green incl. XSS textContent assertions and fetch-failure fallback.
- [ ] Rebuild-diff CI step passes; touching client source without rebuilding fails CI (prove once locally, describe in PR).
- [ ] Bundle ≤ 60 KB; banner hash present and matches recomputation.
- [ ] eslint restriction on innerHTML-family active for `src/site/client/**` (violation fails lint — test).
- [ ] Manual smoke on the fixture site: JA query `支払` and EN query `payment` both hit 支払予約 page (screenshot in PR).

## Validation

Unit/jsdom tests; manual browser check via local static server; CI link.

## Dependencies

29, 30 (data shapes), 31 (DOM contract: element ids/classes documented in a
shared `src/site/contract.ts` — create it here, 31 imports).

## Non-goals

Framework adoption, URL state/deep-linking (v2), offline service worker,
analytics (forbidden by ADR-006).

## Design References

DESIGN.md §11.2–11.3, §13-B3; ADR-004; research/client-side-search.md.
