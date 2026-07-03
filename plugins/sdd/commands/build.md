---
description: Plan-then-execute implementation against the spec.db tasks. Flips task status via the sdd CLI; auto-invokes backprop on test/build failure.
argument-hint: [T<n> --feature <f> | --next | --all] [--parallel[=N]]
---

Invoke the **sdd-build** skill (`skills/sdd-build/SKILL.md`). Treat `$ARGUMENTS`
as the target (`T<n> --feature <f>`, `--next`, or `--all`); `--parallel[=N]`
works the `sdd ready` frontier in waves of subagents (see the skill's PARALLEL
section). Read the spec via `sdd cat`, plan
against every cited invariant/interface, execute, and run the verification
oracle. A task is done only when the oracle exits 0 and the full suite passes.
On failure, invoke the backprop skill before retrying.
