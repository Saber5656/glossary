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

1. Input: curated terms via 09 (any readAllTerms problem ⇒ RuntimeError,
   refuse partial site). Empty glossary builds a valid site with an empty
   list. Term ordering (used for terms.json AND search documents; inline —
   no dependency on issue 27): `sortKey = (reading ?? term).normalize('NFKC')`,
   compareCodepoint, tie → id (DESIGN §10.9).
2. `terms.json` — exact frozen shape (nothing more, nothing less; audit
   fields, notes, examples, relatedTerms, sources are NOT published to the
   client — term pages render those from the store via issue 31):
   ```ts
   type SiteTermsJson = {
     schemaVersion: 1, title: string, locale: 'ja'|'en',
     terms: Array<{ id: string, term: string, reading: string | null,
                    aliases: string[], kind: Kind, tags: string[],
                    definition: string }>
   }
   ```
3. `search-index.json`: MiniSearch constructed with `miniSearchOptions()`
   (29) plus the index shape owned HERE:
   ```ts
   type SearchDocument = { id: string, term: string, aliasesText: string,
                           reading: string, definition: string, tagsText: string }
   // aliasesText/tagsText = array joined with single spaces; reading '' when null
   fields: ['term','aliasesText','reading','definition','tagsText'],
   storeFields: ['id'], idField: 'id'
   ```
   Documents in the sorted term order; serialized via
   `JSON.stringify(miniSearch)`. Hits therefore carry ONLY `id` — the client
   (32) looks up display data in terms.json.
4. outDir rules (config.site.outDir or `--out`):
   - Resolve under repoRoot (E_PATH_ESCAPE otherwise). Symlink hardening:
     `lstat` the outDir and every existing ancestor up to repoRoot — any
     symlink ⇒ E_OUTDIR_UNSAFE; additionally the `realpath` of the deepest
     existing ancestor must stay under the repoRoot realpath.
   - Managed-marker protocol: `.glossary-site` marker file. Target handling:
     (a) missing → create; (b) exists WITH marker → delete contents (not the
     dir), rebuild; (c) exists, non-empty, NO marker → RuntimeError
     E_OUTDIR_UNSAFE (refuses to clear a directory the tool didn't create;
     `--force` overrides); (d) exists empty → proceed.
   - Marker lifecycle (failure-safe): after the safety checks and BEFORE the
     first managed write, write the marker — a failed/interrupted build
     leaves marker + partial output, which the NEXT build recognizes (case
     b) and clears.
   - Deletion never follows symlinks out (delete via `rm` on entries whose
     lstat is not a symlink OR whose resolved path stays under outDir;
     anything else ⇒ E_OUTDIR_UNSAFE).
5. Renderer/assets seam (owned HERE; issues 31/32 replace the
   implementations, not the signatures):
   ```ts
   renderSitePages(ctx: {terms: SiteTermsJson['terms'], allTerms: Term[],
                         site: Config['site'], toolVersion: string,
                         generatedAt: string}): RenderedFile[]
   // RenderedFile = { relPath: string, content: string }; builder writes files
   siteAssets(): Array<{ from: string /* under assets/site/ */, to: string }>
   ```
   This issue ships placeholders: renderSitePages returns a stub index.html;
   siteAssets returns [].
6. Orchestration order: safety checks → write marker → write data JSONs →
   renderSitePages → copy siteAssets → done. Human output:
   `site: N terms → glossary/site (M files)`;
   `--json` data (issue-06 envelope): `{outDir, terms: n, files: n}`.
7. Determinism: identical inputs ⇒ byte-identical outputs for BOTH JSONs
   (JSON.stringify with stable input ordering; no timestamps here — about.html
   timestamp is 31's, from clock).

## Acceptance Criteria

- [ ] terms.json byte-golden for the e2e curated set (exact frozen shape — no extra fields); ordering matches the §10.9 rule.
- [ ] Committed vitest smoke: `MiniSearch.loadJSON(indexJson, miniSearchOptions())` then `search('支払')` returns the 支払予約 id (index/query agreement — automated, not a PR paste).
- [ ] outDir matrix tests: fresh / marker-present rebuild (stale file disappears) / unmarked non-empty refused / `--force` override / symlinked outDir refused / symlink-inside deletion guard / escape path refused / interrupted-build (renderer throws after marker) → next build succeeds by clearing.
- [ ] Empty glossary → valid empty-terms site data.
- [ ] Determinism double-build byte-compare on both JSONs.

## Validation

Unit + runCli tests (the search smoke is a committed test; PR paste optional).

## Dependencies

06, 09, 29 (+31/32 land after; this issue ships with a placeholder renderer
that writes index.html stub and a placeholder asset list — replaced by 31/32).

## Non-goals

HTML content (31), client behavior (32), Pages workflow (34), chunked indexes
(U4/v2).

## Design References

DESIGN.md §11.1, §11.3 (determinism), §10.10, §13-B1 (outDir safety); ADR-004.
