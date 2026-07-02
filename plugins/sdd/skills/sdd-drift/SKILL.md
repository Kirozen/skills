---
name: sdd-drift
description: |
  Read-only drift detector: diffs the spec (invariants/interfaces/tasks in
  spec.db) against the current code and reports violations grouped by severity.
  Writes nothing — suggests remedies via sdd-backprop / sdd-spec / sdd-build but
  never invokes them. Distinct from the `sdd check` CLI command (which only
  verifies SPEC.md == spec.db); this checks code == spec. Triggers when the user
  asks to check drift, audit the spec, verify invariants hold in code, or ask
  whether the code still matches §V/§I/§T. Phrasings: "check drift", "sdd drift",
  "audit the spec", "does the code still match V2", "are the invariants still
  true", "spec vs code", /sdd-drift.
---

# sdd-drift — code-vs-spec drift report

Pure diagnostic. Reports violations. Writes nothing. User decides remedy.

Spec drifting silently from code is the #1 SDD failure mode. sdd-drift is the
detector. Run it after each build and before each ship — drift caught here is a
diff; drift caught in prod is a durable bug.

**Not the same as `sdd check`.** The CLI `sdd check` fails if SPEC.md drifted
from spec.db (the generated view vs its source). sdd-drift diffs the spec vs the
**code**. Complementary, not redundant.

## LOAD

1. `sdd cat` to read the spec (durables + unfinished features). If no spec.db →
   "no spec, nothing to check." Stop.
2. Parse invocation args:
   - `§V` → check invariants only (default)
   - `§I` → check interfaces
   - `§T` → audit task status vs code
   - `--all` → all three

Orient with the read-only commands: `sdd list invariant|interface|task`,
`sdd refs V<n>|I.<name>` (rows citing a ref), `sdd cover` (proving test per
invariant, `!` if none).

## CHECK §V — invariants

For each V<n>:

1. Translate the invariant into a verifiable claim about code.
2. Grep / read the relevant files.
3. Classify: **HOLD** / **VIOLATE** / **UNVERIFIABLE**.
4. Record the ref + file:line evidence.

## CHECK §I — interfaces

For each I.<name>:

1. Locate the implementation.
2. Classify:
   - **MATCH** — shape in code = shape in spec.
   - **DRIFT** — impl exists, shape differs.
   - **MISSING** — impl absent.
   - **EXTRA** — code exposes surface not in §I.

## CHECK §T — tasks

For each T<n> (ords are per-feature):

1. If `x` (done): verify the claimed work is present in code.
2. If `~` (wip): note as in-progress.
3. If `.` (pending): note as pending.
4. Flag `x` rows with no evidence as **STALE**.

## REPORT

Grouped by severity. Cite the ref and file:line.

```
## §V drift
V2 VIOLATE: auth/mw.go:47 uses `<` not `≤`.
V5 UNVERIFIABLE: no test covers ∀ req path (sdd cover: V5 !).

## §I drift
I.api DRIFT: POST /x returns `{result}` not `{id}`. route.go:112.
I.cmd MISSING: `foo bar` absent from cli/*.go.

## §T drift
T3 (feature 2) STALE: status `x`, no middleware file exists.

## summary
2 violate. 1 missing. 1 stale. 1 unverifiable.
```

## REMEDY HINTS (not actions)

End the report with one-line hint per class:
- VIOLATE / DRIFT → invoke **sdd-backprop** (record the bug, maybe a new
  invariant) or fix code at the cited lines.
- MISSING → invoke **sdd-build** on `T<n> --feature <f>` if the task exists;
  else **sdd-spec** to add the task.
- STALE → **sdd-spec**: `sdd set-task <T-ord> --feature <f> --status .` to unflag.
- EXTRA → **sdd-spec** to `add-interface` and document it, or delete the code.

Never invoke fixes. Report only.

## NON-GOALS

- Zero writes. No `sdd add-*`/`set-*`/`edit`. No code edits.
- No sub-agents. Main thread reads.
- No scores, no grades. Binary per item: holds or drifts.
