---
name: sdd-backprop
description: |
  Bug → spec protocol. On a test failure or bug report, trace the root cause and
  decide whether a new invariant would catch recurrence, then persist it via the
  sdd CLI (add-bug --fix, add-invariant). Triggers on a failing test, a bug
  report, a post-mortem, or /sdd-backprop. This is the one move plain
  plan-then-execute skips.
---

# sdd-backprop — bug becomes durable memory

A bug fixed in code alone can recur. A bug turned into an invariant cannot. The
bug log is durable in spec.db — it survives every feature wipe.

Full sdd subcommand reference: `plugins/sdd/COMMANDS.md`.

## STEPS
1. Read the failure output. Find the root cause (read the code).
2. Classify: (a) code bug, (b) spec wrong, (c) unspecified edge case.
3. `sdd search "<keyword from the root cause>"` — check whether this cause (or
   one close to it) was already logged. Bugs are durable and never wiped, so
   `sdd list bug` alone grows unbounded over a project's life; `search` is the
   bounded lookup. Fall back to `sdd list bug` only if `search` comes back
   empty and you suspect a wording mismatch. A match → reuse its `--fix V<n>`
   invariant instead of minting a duplicate one.
4. Would a new invariant catch recurrence?
   - Yes → add it, then log the bug pointing at it:
     ```
     sdd add-invariant "<invariant that would have caught this>"   # -> V<n>
     sdd add-bug "<root cause, terse>" --fix V<n>
     ```
   - No new invariant (pure code slip, or step 3 found an existing one) →
     still log it:
     ```
     sdd add-bug "<root cause>" --fix V<existing>
     ```
5. If behavior changed, add/adjust tasks via sdd-spec.
6. Resume the build against the updated spec.

## RULE
Every bug gets a `sdd add-bug` entry. The invariant is optional but preferred —
it is what makes the next build unable to regress it (V5 wires fix → invariant).
Never silently fix a root cause without considering the invariant. Never mint
a second invariant for a cause `sdd search` already covers.
