---
name: sdd-build
description: |
  Plan-then-execute implementation against the spec.db tasks. Reads the spec via
  `sdd cat` for context, flips task status via the sdd CLI, and auto-invokes
  sdd-backprop on test/build failure. Triggers on "build", "implement the next
  task", "run the build", `build T<n>`, or /sdd-build.
---

# sdd-build — implement the spec

Single-thread plan→execute. The task list lives in spec.db; read it via
`sdd cat` (durables + unfinished features; `sdd cat --feature N` for one).

## LOAD
1. `sdd cat` to read the spec. If no spec.db → tell the user to run sdd-spec. Stop.
2. Pick the target: `T<n> --feature <f>` (that task — ords are per-feature, V117),
   `--next` (lowest `.`/`~`), or `--all`. Add `--parallel[=N]` to work the
   dependency frontier in waves of subagents (see PARALLEL); N caps the
   subagents per wave, default `min(frontier, 4)`.
3. High blast radius (shared module, data, public interface)? Run sdd-review first.

Orient with the pure read commands: `sdd next` (next actionable task + its goal
and resolved cites — skips a todo with an unmet blocker, always keeps a wip task,
V124), `sdd ready` (the dispatchable frontier: todo tasks whose blockers are all
done, as TSV), `sdd todo` (every unfinished task as TSV — machine-readable for
picking work), `sdd status`/`sdd guide` (per-feature stage). `sdd --help` lists
every command.

## PLAN (native plan mode)
For the chosen task:
1. Cite every invariant (V<n>) it lists. The plan must respect all.
2. Cite every interface (I.<name>) it touches. Preserve the shape.
3. List files to create/edit.
4. Verification contract — name the EXACT test(s) that prove each cited
   invariant. Which test, not "add tests".
5. Name the verification command (the external oracle). Green = done.

## EXECUTE per task
```
sdd set-task <T-ord> --feature <f> --status ~   # wip (task ords are per-feature, V117)
# edit code per plan
# run the verification command
sdd set-task <T-ord> --feature <f> --status x   # pass -> done
```
On failure: do NOT retry blindly — invoke **sdd-backprop** first.

## PARALLEL (`--parallel[=N]`)
The dependency graph (F28) is the parallelism substrate: `sdd ready` is the
frontier of `todo` tasks whose blockers are all `x` — independent by
construction. Work it in **waves**, never the whole `todo` set at once.

**One invariant governs the whole mode: the orchestrator is the SOLE mutator of
spec.db.** Subagents read (`sdd cat --feature <f>`) and edit code, but NEVER call
`sdd set-task` or sdd-backprop — they return a structured verdict; the
orchestrator serializes every DB write and commit. No concurrent SQLite writers.

Per wave:
1. `sdd ready` → candidate frontier. Empty → done. One task → just run it serially.
2. **Plan-scout** — one read-only subagent per candidate, in parallel, via the
   dedicated `sdd:sdd-build-plan-scout` agent type (`subagent_type`; un-namespaced
   `sdd-build-plan-scout` also resolves). It is **model-pinned to a mid-tier
   model** — planning which files/cites/oracle a task needs is lighter than
   writing the code. Each runs PLAN (§PLAN) for its task and returns
   `{task, feature, files[], cites[V/I], oracle}`. No edits, no spec.db writes.
3. **Partition** — greedily pick a max subset whose declared `files[]` are
   pairwise disjoint. That subset is this wave; colliding tasks wait for the next
   (their file is freed once the winner commits). File overlap = same wave
   forbidden — this is the whole safety story of the shared tree.
4. **Execute** — one subagent per wave task, in parallel, same worktree. These
   write real code, so leave them on the session model (do NOT downgrade) — only
   the read-only plan-scout is routed to a lighter tier. Each edits only its
   planned files, runs its oracle, returns `{task, oracle_exit,
   invariants_covered[], summary}`. Still no DB writes.
5. **Barrier (orchestrator, serial)** — for each verdict: oracle 0 AND every cite
   covered → `sdd set-task <ord> --feature <f> --status x` + commit that task.
   Then run the FULL suite once for the wave. Any red → the offending task's
   changes are the suspect; invoke **sdd-backprop**, leave the task `~`/`todo`.
6. Re-query `sdd ready` (freed blockers open the next wave) and repeat.

A failed task confines itself: its dependants never enter `ready` until it is
`x`, so the graph blocks the blast radius for you — no manual gating.

## VERIFICATION
A task is `x` only if the oracle exits 0, every cited invariant has its named
passing test (`sdd cover` flags any invariant with no proving test), and the
full suite still passes. Commit after each task. Under `--parallel` the
full-suite gate runs once per wave at the barrier, not per task (each task still
commits individually once its own oracle is green).
