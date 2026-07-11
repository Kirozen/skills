---
name: sdd-drift
description: |
  Read-only drift detector: diffs the spec (invariants/interfaces/tasks in
  spec.db) against the current code and reports violations grouped by severity.
  Writes nothing — suggests remedies via sdd-backprop / sdd-spec / sdd-build but
  never invokes them. This checks code == spec (a different axis from `sdd
  export`, which merely re-renders SPEC.md from spec.db). Scope defaults to the
  whole project; `--feature <f>` restricts the check to one feature's tasks and
  the refs they cite. Triggers when the user asks to check drift, audit the
  spec, verify invariants hold in code, or ask whether the code still matches
  §V/§I/§T. Phrasings: "check drift", "sdd drift", "audit the spec", "does the
  code still match V2", "are the invariants still true", "spec vs code", "drift
  on this feature", /sdd-drift.
---

# sdd-drift — code-vs-spec drift report

Pure diagnostic. Reports violations. Writes nothing. User decides remedy.

Spec drifting silently from code is the #1 SDD failure mode. sdd-drift is the
detector. Run it after each build and before each ship — drift caught here is a
diff; drift caught in prod is a durable bug.

**Not a file render.** `sdd export` re-renders SPEC.md from spec.db (the view vs
its source) — a pure read. sdd-drift diffs the spec vs the **code**: a different
axis entirely.

## LOAD

1. `sdd cat` to read the spec (durables + unfinished features). If no spec.db →
   "no spec, nothing to check." Stop.
2. Parse invocation args:
   - `§V` → check invariants only (default)
   - `§I` → check interfaces
   - `§T` → audit task status vs code
   - `--all` → all three
   - `--feature <f>` → scope the check to one feature (composes with any of the
     above; omit it and scope stays the whole project, unchanged, V6)
3. If `--feature <f>` is given:
   - Read `sdd cat --feature <f>` instead of the whole-project `sdd cat`. If the
     feature doesn't exist, the CLI errors clearly — surface it and stop.
   - §T checks only that feature's tasks (already scoped by `--feature <f>`).
   - §V/§I check only the refs cited by that feature's tasks — read them off the
     `cites` column of the §T section in that same output (single source of
     truth, V9); do not sweep `sdd refs` per invariant.
   - No task in the feature cites any ref → say so explicitly ("0 refs cités
     par cette feature — rien à vérifier en §V/§I") instead of an empty §V/§I
     section that a clean whole-project report would also produce (V8).

Orient with the read-only commands: `sdd list invariant|interface|task`,
`sdd refs V<n>|I.<name>` (rows citing a ref), `sdd cover` (proving test per
invariant, `!` if none), `sdd graph V<n>|I.<name>` (that durable's blast radius
as Mermaid — everything citing it, to weigh a violation's impact before you rank
its severity; `sdd graph` alone = the durables overview, orphans/uncovered
flagged; `sdd graph --feature <f>` scopes that overview to one feature).

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

## AGENT ORCHESTRATION (§V/§I only)

**Scope threshold first.** Below ~15 combined V+I items in scope, check them
inline as above — spawning subagents would cost more (re-loading the spec per
agent) than the sequential grep loop it replaces. Above that, fan out one
`sdd:sdd-drift-item-check` agent per item in parallel: pass each the item's
exact text (from the `sdd cat` output already loaded), never re-run `sdd cat`
per agent. Each agent is read-only (`Glob, Grep, Read, LSP`, no Bash/Edit/Write
— it never queries spec.db itself) and runs on **sonnet**: classifying
HOLD/VIOLATE/DRIFT is judgment-heavy structural comparison, not mechanical
matching.

> Scoped to the plugin (`sdd:sdd-drift-item-check`). Un-namespaced
> `sdd-drift-item-check` also resolves if loaded via `--plugin-dir`.

If an item-check agent fails or returns unparseable output, do not drop it
silently — report that item as **UNVERIFIABLE (check failed)** rather than
omitting it from §V/§I, so a clean report never hides an item that was never
actually checked.

§T stays main-thread regardless of scope — task status/evidence checks are
few by construction (one per task) and benefit from the cross-task context
already loaded.

## CHECK §T — tasks

For each T<n> (ords are per-feature):

1. If `x` (done): verify the claimed work is present in code.
2. If `~` (wip): note as in-progress.
3. If `.` (pending): note as pending.
4. Flag `x` rows with no evidence as **STALE**.

## REPORT

Grouped by severity. Cite the ref and file:line. When `--feature <f>` scoped
the check, open the report with `Scope: Feature <f> <name>` — a clean scoped
report must never read as a clean whole-project one (V7).

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
- No sub-agents below the ~15-item threshold (see AGENT ORCHESTRATION) — main
  thread reads. Above it, only `sdd:sdd-drift-item-check` for §V/§I; §T always
  stays main-thread.
- No scores, no grades. Binary per item: holds or drifts.
