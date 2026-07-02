---
description: Code-vs-spec drift detector. Diff the spec (spec.db invariants/interfaces/tasks) against code. Read-only, zero writes. Distinct from the `sdd check` CLI.
argument-hint: [§V | §I | §T | --all]
---

Invoke the **sdd-drift** skill (`skills/sdd-drift/SKILL.md`). Treat `$ARGUMENTS`
as the scope (`§V` default, `§I`, `§T`, or `--all`). Read the spec via `sdd cat`,
classify each item HOLD/VIOLATE/UNVERIFIABLE (or MATCH/DRIFT/MISSING/EXTRA for
§I), cite file:line evidence, end with remedy hints. Writes nothing. Run it after
each build. Not to be confused with `sdd check` (SPEC.md == spec.db); this checks
code == spec.
