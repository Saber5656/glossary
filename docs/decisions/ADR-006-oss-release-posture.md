# ADR-006: OSS release posture — MIT, English docs, no telemetry, git-only v1 distribution

- Status: Accepted (2026-07-11) — license choice explicitly flagged for owner veto
- Deciders: designer (conservative defaults), owner (R4: generic design, minimal distribution)
- Related: DESIGN.md §2, §3, §13-B5/B6

## Context

The repository is public from day one and intended as an OSS project. The
requirements interview fixed "generic design, v1 distribution = GitHub repo
only". Several release-posture details were delegated to the designer.

## Decision

1. **License: MIT.** Simple, maximally adoptable for a dev tool. Third-party
   notices (kuromoji Apache-2.0, IPADIC license) included in README credits.
   *Owner may veto before the docs PR merges; switching to Apache-2.0 is a
   one-file change at this stage.*
2. **Language**: repository docs, issues, commit messages, and code comments in
   English (issue drafts must be English per workflow; mixed-language docs rot
   fastest). README.md is English with a linked full Japanese guide
   (README.ja.md) since the primary persona is Japanese teams. Site UI is
   localized (`ja` default); CLI messages English in v1.
3. **No telemetry, no update checks, no network at rest** (restates DESIGN
   secure defaults as a release commitment).
4. **v1 distribution**: `git clone` + `npm ci` + `npm run build` + `npm link`
   (documented). No npm publish, no version tags required in v1; SemVer starts
   with v2 packaging.
5. **Repo hygiene for public development**: committed lockfile; Dependabot
   enabled; CI actions pinned to full commit SHAs; default workflow
   `permissions: contents: read`; SECURITY.md with private reporting via
   GitHub Security Advisories; branch protection on `main` (PR-only) —
   already configured by the owner's rulesets.
6. **No secrets anywhere in this repo**, including examples: sample configs use
   env-var indirection only.

## Consequences

- English-docs decision means the existing 1-line Japanese README is replaced
  by English + README.ja.md (issue-planned; original sentence preserved as the
  tagline in both).
- MIT + bundled-dictionary notices must be verified once during implementation
  (covered in the scaffolding issue's acceptance criteria).
- Git-only distribution keeps v1 lean; users are developers, acceptable.

## Alternatives considered

- **Apache-2.0**: explicit patent grant, heavier text; MIT chosen for
  simplicity — flagged for owner veto above.
- **Japanese-primary docs**: better for the initial team, worse for OSS reach
  and for English-only implementation agents; bilingual READMEs chosen instead.
