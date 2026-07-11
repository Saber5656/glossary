# 26 — `glossary approve` / `reject` (lifecycle T2/T3/T5)

## Title

Curation commands: approve candidates into curated terms; reject/unreject keys

## Summary

Implement the state-machine transitions T2/T3/T5 of DESIGN.md §8 as the
`approve` and `reject` commands, enforcing invariants at write time.

## Context

These are the only commands that create curated terms or touch the rejected
list (ownership §6). Correct, atomic transitions here are what make the
two-tier model trustworthy.

## Scope

- `src/cli/commands/{approve,reject}.ts` + tests.

## Detailed Requirements

1. `approve <key...> [--id <slug>] [--tag <tag>...]`:
   - Normalize each input via termKey; each must exist in candidates, else
     UsageError E_KEY_NOT_FOUND (transaction: validate ALL keys before writing
     ANYTHING; one bad key fails the whole command with no writes).
   - `--id` only allowed with exactly one key (else UsageError); must pass
     `isValidSlug` and not collide with existing ids/alias keys
     (E_ID_CONFLICT).
   - For each key (T2):
     1. Build term via `newTermFromCandidate` (09): id = --id ?? termId(key);
        tags from `--tag` (validated against tag pattern); term = surface;
        aliases = surfaces − surface; kind, sources(≤5), definition from
        suggestedDefinition (else ""), definitionSource mapping (doc/llm/
        human-empty per 09), dates from clock.
     2. `writeTerm` (no overwrite).
     3. Remove entry from candidates.yaml (single rewrite after all
        approvals; preserves order of remaining entries and generatedAt).
   - Post-write check: run V1 rule on the touched keys; violation ⇒ rollback
     is NOT attempted — instead pre-check before writing (curated/rejected
     key-set intersection) so violation is impossible; test proves the
     pre-check.
   - Output human: one line per term `approved 支払予約 → glossary/terms/payment-reservation.yaml`;
     summary. `--json`: `{approved: [{key, id, path}]}`.
2. `reject <key...> [--reason <text>] [--force]`:
   - Default: every key must exist in candidates (same all-or-nothing
     validation); `--force` allows unknown keys (pre-emptive block, still
     termKey-normalized).
   - T3: addRejected (11) + remove from candidates (single rewrite).
   - Keys that are curated (term or alias) ⇒ UsageError (must delete curated
     file first; message explains).
   - `--json`: `{rejected: [{key, reason}], removedFromCandidates: n}`.
3. `reject --remove <key...>`:
   - T5: removeRejected; unknown keys ⇒ listed in `notFound` warning, exit 0
     if ≥1 removed, exit 2 if none.
   - `--json`: `{removed: [...], notFound: [...]}`.
4. Both commands refuse to run if candidates.yaml is schema-invalid
   (E_SCHEMA_INVALID surfaces from store; hint: re-run extract).
5. Atomicity: candidates rewrite happens once, after terms/rejected writes
   succeed; a crash between leaves duplicate-key state that `validate`
   catches (documented in command header comment + test simulating it).

## Acceptance Criteria

- [ ] approve happy path: candidate with doc suggestion → curated file golden-matches (definitionSource doc), removed from candidates; `--id` respected; auto-id = t-hash8.
- [ ] approve multi-key all-or-nothing: 2 valid + 1 unknown ⇒ exit 2, zero writes (fs-spy).
- [ ] id collision and alias-collision pre-checks fire (E_ID_CONFLICT / V1-style message).
- [ ] reject writes reason + removes candidate; `--force` on unknown key adds it; curated-key rejection blocked with explanatory error.
- [ ] reject --remove mixed known/unknown behaves per spec (exit codes asserted).
- [ ] JSON snapshots frozen for all three flows.

## Validation

runCli integration on temp glossary; crash-window test via store-level calls.

## Dependencies

06, 09, 10, 11.

## Non-goals

Editing definitions (`$EDITOR` spawn — out; users edit files directly), bulk
approve-by-score flags (v2), un-approve command (delete file manually, T6).

## Design References

DESIGN.md §8 (T2/T3/T5, V1/V2), §10.6–10.7, §6 (ownership), §7.4.
