---
name: sdd-spec
description: |
  Sole mutator of the spec. Translates intent into sdd CLI calls: invariants,
  interfaces, tasks, and feature goals/constraints. Never hand-edits SPEC.md —
  that file is the generated read-only view of spec.db. Triggers on "write the
  spec", "add an invariant", "spec this feature", or /sdd-spec. Ingests handoffs
  from sdd-grill (goal/constraints), sdd-research (findings), sdd-review
  (invariants).
---

# sdd-spec — spec mutator (via sdd)

The other skills produce material; sdd-spec persists it through the CLI. The db
is the source of truth; read the spec with `sdd cat`. SPEC.md is an on-demand
export (`sdd export`), never hand-edited (V3).

## DISPATCH
- New feature from an idea → run sdd-grill first if fuzzy, then land it here.
- Add durable truth (invariant/interface/bug) → the add-* commands.
- Amend an existing row → `sdd edit <kind> <id> --text "<new>"` (id stays stable, V12).

## DURABLE vs FEATURE
Durable (persists across features) — write freely, they survive wipe:
```
sdd add-invariant "<testable invariant>"          # -> V<n>
sdd add-interface <kind> <name> "<signature>"     # cite key I.<name>
sdd add-bug "<cause>" --fix V<n>                  # backprop log
```
Feature-scoped (wiped per feature):
```
sdd add-goal "<line>" --feature <id>
sdd add-constraint "<bullet>" --feature <id>
sdd add-task "<task>" --feature <id> --cites V2,I.init
sdd add-cite <T-ord> --feature <id> V3,I.foo      # cite an EXISTING task (ords per-feature, V117)
sdd block <T-ord> --on <T-ord,...> --feature <id>   # T waits on each --on task; cycles rejected (V125)
sdd unblock <T-ord> --off <T-ord,...> --feature <id> # drop those edges (idempotent)
sdd rm-task <T-ord> --feature <id>                  # hard-delete a task; its cites + blocker edges cascade
```
Landing a whole spec at once? Batch every write in one transaction with
`sdd apply` — TAB-delimited `add-*` lines on stdin, all-or-nothing; a leading
`new-feature` sets the current feature.

## RULES
- `--cites` must reference existing V<n>/I.<name>; the FK rejects orphans (V5).
- Mutations commit atomically to spec.db (the sole store); `sdd export` regenerates SPEC.md on demand — no auto-export.
- Show the user what you will run, then run it. The CLI is the diff.
- `plugins/sdd/COMMANDS.md` has the full subcommand reference; `sdd --help` / `sdd <cmd> --help` if it's stale or missing something.

## HANDOFF
After tasks exist, point the user at **sdd-review** (high blast radius) or
**sdd-build**.
