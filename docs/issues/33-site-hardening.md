# 33 — Site security tests: hostile fixture rendering, no-external-URL scan

## Title

Site hardening e2e: prove AC1 inertness and self-containment of built output

## Summary

Extend the e2e harness (28) with the site half of the hostile scenario and a
whole-site static scanner enforcing DESIGN.md §11.2's output rules.

## Context

Issues 31/32 build the defenses; this issue proves them end-to-end and pins
them with tests that will fail loudly if a future change weakens escaping,
CSP, or self-containment (abuse case AC1, boundary B3).

## Scope

- `test/e2e/site.e2e.test.ts`, `test/helpers/site-scan.ts`. No production
  code changes expected (found gaps go back to 31/32 or become new issues).

## Detailed Requirements

1. Hostile scenario extension: on the `repo-hostile` copy — extract, approve
   the XSS-bait keys (`<img…>用語` etc.) and the `javascript:`-link term via
   CLI, then `build`. Assertions over `glossary/site/`:
   - No file contains the raw substrings `<img src=x onerror` or
     `<script>alert` outside HTML-escaped form (`&lt;img`, `&lt;script`).
   - No `href` or `src` attribute value in any built HTML starts with
     `javascript:`, `data:text/html`, or `vbscript:` (parse attributes with a
     regex over tags — simple lexical scan is acceptable here since WE
     generated the HTML).
   - jsdom load of index.html + injected app behavior: search for the bait
     term; result list contains no `img`/`script` elements (AC1 in the
     client path, complementing 32's unit test with the REAL built bundle).
2. Whole-site scanner `site-scan.ts` (reused by CI for any built site):
   - Every `.html` file: exactly one CSP meta matching the DESIGN §11.2 string;
     `<html lang=` present; no inline `<script>` bodies (only `src=` module
     tag on index); no `style=` attributes; no event-handler attributes
     (`on*=` lexical check).
   - Whole tree: zero occurrences of `http://` or `https://` in ANY file
     (HTML/JS/CSS/JSON) — the license banner in app.js must therefore avoid
     URLs (32 respects this; assert).
   - All referenced relative assets exist (parse href/src, resolve, stat).
3. Run the same scanner over the `repo-ja-mixed` built site (clean content
   must also pass — guards against scanner false positives).
4. Wire both into `npm run test` (e2e project) and CI.

## Acceptance Criteria

- [ ] Hostile site assertions all green with the real built site.
- [ ] Scanner catches seeded violations (self-test: feed it a deliberately bad HTML string fixture and assert each rule fires).
- [ ] ja-mixed site passes the scanner (no false positives).
- [ ] jsdom real-bundle search test green.
- [ ] CI includes the suite on all matrix nodes.

## Validation

CI run link; local `open` of the hostile site + manual click-through documented
with a screenshot in the PR (visual confirmation nothing executes).

## Dependencies

13, 31, 32 (and 28's harness).

## Non-goals

Trusted-Types/DOMPurify adoption (unnecessary given textContent discipline),
external security audit (post-v1), HTTP header configuration (static hosts
vary; CSP meta is the portable layer, documented in 34).

## Design References

DESIGN.md §11.2, §13-B3/AC1, §16 (hostile row); ADR-004 §5.
