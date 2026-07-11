# 30 — `glossary build` shell: site data files and outDir hygiene

## Title

`glossary build` command frame: terms.json, search-index.json, asset copy, safe output directory handling

## Summary

Implement the data half of the site builder per DESIGN.md §11.1: load curated
terms, produce `terms.json` and the serialized MiniSearch `search-index.json`,
copy static assets, and manage the output directory safely. Page rendering
plugs in via issue 31.

## Context

`build` writes and DELETES files in a user-configurable directory — the most
dangerous filesystem surface in the product (E_OUTDIR_UNSAFE). The data files
are also the site's API; their shapes freeze here.

## Scope

- `src/cli/commands/build.ts`, `src/site/data.ts`, outDir management +
  tests. Templates (31) and client bundle content (32) are consumed as
  black boxes (31/32 register: pages renderer + asset list).

## Detailed Requirements

1. Input: curated terms via 09 (schema problems ⇒ RuntimeError, refuse
   partial site). Empty glossary builds a valid site with an empty list.
2. `terms.json` (frozen shape, sorted by export sortKey (27's rule)):
   ```json
   {"schemaVersion": 1, "title": "<site.title>", "locale": "ja",
    "terms": [{"id","term","reading","aliases","kind","definition",
               "definitionSource","tags","relatedTerms",
               "sources":[{"path","line"}]}]}
   ```
   (exact curated fields minus dates/notes/examples? INCLUDE `examples`;
   EXCLUDE `notes`, `createdAt`, `updatedAt` — publication trims audit
   fields.)
3. `search-index.json`: MiniSearch built with `searchOptions()` (29), fields
   term/aliases(joined space)/reading/definition/tags(joined), storeFields
   `[id]`; serialized via `JSON.stringify(miniSearch)`; documents = terms.
4. outDir rules (config.site.outDir or `--out`):
   - Resolve under repoRoot (E_PATH_ESCAPE otherwise).
   - Managed-marker protocol: a build writes `.glossary-site` marker file.
     Target handling: (a) missing → create; (b) exists WITH marker → delete
     contents (not the dir), rebuild; (c) exists, non-empty, NO marker →
     RuntimeError E_OUTDIR_UNSAFE (message: refuses to clear a directory the
     tool didn't create; `--force` overrides); (d) exists empty → proceed.
   - Deletion never follows symlinks out (delete via `rm` on entries whose
     resolved path stays under outDir; anything else ⇒ E_OUTDIR_UNSAFE).
5. Asset copy: files registered by 31/32 from `assets/site/` → `outDir/assets/`
   (byte copy; no transformation at build time).
6. Orchestration order: prepare outDir → write data JSONs → call page renderer
   (31) → copy assets → write marker. Human output: `site: N terms → glossary/site (M files)`;
   `--json`: `{outDir, terms: n, files: n}`.
7. Determinism: identical inputs ⇒ byte-identical outputs for BOTH JSONs
   (JSON.stringify with stable input ordering; no timestamps here — about.html
   timestamp is 31's, from clock).

## Acceptance Criteria

- [ ] terms.json golden for the e2e curated set; excludes notes/dates; ordering matches export.
- [ ] search-index.json loads in Node via `MiniSearch.loadJSON` + `searchOptions()` and `search('支払')` returns the 支払予約 id (index/query agreement smoke).
- [ ] outDir matrix tests: fresh / marker-present rebuild (stale file inside disappears) / unmarked non-empty refused / `--force` override / symlink-inside deletion guard / escape path refused.
- [ ] Empty glossary → valid empty-terms site data.
- [ ] Determinism double-build byte-compare on both JSONs.

## Validation

Unit + runCli tests; Node-side search smoke output pasted in PR.

## Dependencies

06, 09, 29 (+31/32 land after; this issue ships with a placeholder renderer
that writes index.html stub and a placeholder asset list — replaced by 31/32).

## Non-goals

HTML content (31), client behavior (32), Pages workflow (34), chunked indexes
(U4/v2).

## Design References

DESIGN.md §11.1, §11.3 (determinism), §10.10, §13-B1 (outDir safety); ADR-004.
