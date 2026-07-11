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

0. Preflight (both commands, before ANY write): load all three stores;
   FAIL (exit 1, the store's error) on: candidates schema-invalid, rejected
   schema-invalid, any `readAllTerms` problem, or any
   `buildTermIndexes().indexProblems` entry — curation refuses to run
   against a poisoned/partial store (B2).
1. `approve <key...> [--id <slug>] [--tag <tag>...]`:
   - Normalize each input via termKey; each must exist in candidates, else
     UsageError E_KEY_NOT_FOUND (validation is all-or-nothing: ANY bad
     key/collision fails the whole command before any write).
   - `--id` only allowed with exactly one key (else UsageError E_USAGE); must
     pass `isValidSlug` and not collide with existing ids/term/alias keys
     (E_ID_CONFLICT — message names the conflicting id AND key).
   - `--tag` values must match `^[\p{L}\p{N}][\p{L}\p{N}-]{0,31}$u` (09's
     pattern) else UsageError E_USAGE naming the bad tag.
   - Pre-checks make V1 violations impossible: candidate keys are checked
     against curated term/alias key sets and the rejected key set before
     writing (tests prove the pre-check fires).
   - For each key (T2):
     1. Build term via `newTermFromCandidate` (09): id = --id ?? termId(key);
        tags from `--tag`; term = surface; aliases = surfaces − surface;
        kind, sources (≤5), definition from suggestedDefinition (else "");
        definitionSource per 09's builder (doc/llm as suggested;
        `null` suggestion ⇒ `human`); dates from clock.
     2. `writeTerm` (no overwrite).
     3. After ALL term writes succeed: ONE `removeCandidates` call (issue
        10) — preserves remaining order and file metadata.
   - Failure contract (explicit): pre-write validation is all-or-nothing;
     post-write failures are NOT rolled back — the command reports a
     RuntimeError listing keys already written vs pending, and the
     crash-window state (approved term still in candidates) is exactly what
     `validate` (12) flags as V1; document this in the command header and
     test the reporting via an injected writeTerm failure.
   - Output human: one line per term `approved 支払予約 → glossary/terms/payment-reservation.yaml`;
     summary. `--json`: `{approved: [{key, id, path}]}`.
2. `reject <key...> [--reason <text>] [--force]`:
   - Default: every key must exist in candidates (same all-or-nothing
     validation); `--force` allows unknown keys (pre-emptive block, still
     termKey-normalized).
   - T3: addRejected (11), then ONE `removeCandidates` call (10).
   - Keys that are curated (term or alias) ⇒ UsageError with stable code
     `E_CURATED_KEY` (add to 03's ErrorCodes); message contains
     "delete the curated term file first" and names the file.
   - `--json`: `{rejected: [{key, reason}], removedFromCandidates: n}`.
3. `reject --remove <key...>`:
   - `--remove` FORBIDS `--reason` and `--force` (UsageError E_USAGE); zero
     keys ⇒ UsageError E_USAGE.
   - T5: removeRejected; unknown keys ⇒ listed in `notFound` warning, exit 0
     if ≥1 removed, exit 2 if none.
   - `--json`: `{removed: [...], notFound: [...]}`.
4. (Store-invalid refusal is Preflight 0.)
5. Ordering: rejected/term writes first, candidates rewrite last (single
   call). The crash-window semantics of Req 1's failure contract apply to
   reject identically.

## Acceptance Criteria

- [ ] approve happy path: candidate with doc suggestion → curated file golden-matches (definitionSource doc), removed from candidates via removeCandidates (metadata preserved); `--id` respected; auto-id = t-hash8.
- [ ] approve multi-key all-or-nothing: 2 valid + 1 unknown ⇒ exit 2, zero writes (fs-spy); bad `--tag` ⇒ exit 2 E_USAGE naming the tag.
- [ ] Pre-checks fire with stable codes: duplicate id and alias collision ⇒ E_ID_CONFLICT naming id and key; curated-key rejection ⇒ E_CURATED_KEY containing "delete the curated term file first".
- [ ] Preflight: a poisoned store (one broken curated file / indexProblems duplicate) blocks both commands before any write.
- [ ] Injected writeTerm failure mid-multi-approve: RuntimeError lists written vs pending keys; candidates.yaml untouched; the resulting state is a documented validate-V1 case (assert the overlap exists — running validate itself is issue 12's suite).
- [ ] reject writes reason + removes candidate; `--force` on unknown key adds it; `--remove` with `--reason`/`--force`/zero keys ⇒ exit 2 E_USAGE.
- [ ] reject --remove mixed known/unknown behaves per spec (exit codes asserted).
- [ ] JSON snapshots frozen for all three flows.

## Validation

runCli integration on temp glossary; crash-window reporting test via an
injected store failure (no dependency on the validate command itself).

## Dependencies

06, 09, 10, 11.

## Non-goals

Editing definitions (`$EDITOR` spawn — out; users edit files directly), bulk
approve-by-score flags (v2), un-approve command (delete file manually, T6).

## Design References

DESIGN.md §8 (T2/T3/T5, V1/V2), §10.6–10.7, §6 (ownership), §7.4.
