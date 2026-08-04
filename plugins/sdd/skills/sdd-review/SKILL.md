---
name: sdd-review
description: |
  Adversarial senior review of the spec before any code is written. Tries to
  refute the spec (read via sdd cat), citing evidence; survivors harden into
  invariants via the sdd CLI. Ends on an explicit go / no-go gate. Triggers on
  "review the spec", "red-team this", "is this plan sound", or /sdd-review.
---

# sdd-review — refute the spec before build

**Every finding cites evidence — file:line or a source. No evidence → flag `[unverified]`. Default to refuted.**

A refutation attempt, not "looks good". Read the spec with `sdd cat` (or `sdd
cat --feature N` for the feature under review) — you review the spec, not your memory.

## CONSTRUCT THE SENIOR
Earn authority before opining: grep the modules the spec touches, read the
research rows (§R), and fetch any best-practice claim you are unsure of. Trace
the spec's own wiring with the read commands: `sdd refs V<n>/I.<name>` (who cites
this — find the blast radius), `sdd graph V<n>/I.<name>` (that blast radius as Mermaid;
`sdd graph` alone = the durables overview, load-bearing vs orphan at a glance),
`sdd cover` (which invariants ship with no proving test). `sdd --help` lists
every command.

## REFUTE
Attack each axis for the case where it breaks:
- Goal vs reality — does it solve the real problem or a proxy?
- Missing invariant — what can go wrong that no V<n> catches? (most findings here)
- Interface drift — does an I.<name> match what callers expect? (cite file:line)
- Constraint conflict — do two constraints fight, or fight a research finding?
- Unowned edge — the input/order/failure/concurrency no task covers.

**Scope threshold first.** A small feature (a handful of tasks, one module)
doesn't justify spawning 5 subagents — refute the axes directly, in this same
context. Above that (high blast radius: shared module, data, public
interface), spawn one lens per axis in parallel via `subagent_type`:

| `subagent_type`                       | Axis                 | Model  |
|-----------------------------------------|----------------------|--------|
| `sdd:sdd-review-goal-reality`           | Goal vs reality      | sonnet |
| `sdd:sdd-review-missing-invariant`      | Missing invariant    | sonnet |
| `sdd:sdd-review-interface-drift`        | Interface drift      | sonnet |
| `sdd:sdd-review-constraint-conflict`    | Constraint conflict  | sonnet |
| `sdd:sdd-review-unowned-edge`           | Unowned edge         | sonnet |

Pass each lens the feature's `sdd cat` output as its task prompt — not a
mission prompt, that lives in the agent file. All five are strictly read-only
(no `sdd add-*`/`set-*`/`edit`, no code edits — only the orchestrator writes)
and run on **sonnet**: refutation is judgment-heavy, not mechanical, so none
downgrade to a lighter model.

> Names are scoped to the plugin (`sdd:sdd-review-*`). Un-namespaced
> `sdd-review-*` also resolves if loaded via `--plugin-dir`.

If a lens fails or returns unparseable output, do not drop it silently —
report that axis as **NOT REFUTED** with the reason, so a clean gate never
hides an axis that was never actually attacked.

**Merge**: the same defect can surface from more than one axis (e.g. a gap
that is both a missing invariant and an unowned edge). Group findings by
overlapping claim/evidence; keep the higher severity and list both axes
rather than reporting it twice.

**Below the threshold**, work the axes inline in this same context, one after
another.

## CLASSIFY
`evidence → claim → severity`: BLOCK (ships a defect), HARDEN (add an invariant),
NOTE (worth knowing). No evidence → down-rank to NOTE `[unverified]`.

## HARDEN & GATE
Each HARDEN finding becomes a new invariant, persisted via sdd-spec:
```
sdd add-invariant "<testable invariant the build cannot regress>"
```
End on an explicit gate: GO or NO-GO, never a shrug. NO-GO until every BLOCK is
cleared. Persist the verdict — it must outlive this conversation:
```
sdd gate <F-ord> --go                          # or --no-go --note "<why>"
```
Then hand to sdd-build.
