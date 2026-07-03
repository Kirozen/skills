---
name: debugger
description: >-
  Use when facing a bug, test failure, crash, or any unexpected behavior, BEFORE
  proposing a fix — to find and prove the root cause first. Use this whenever the
  user reports something broken, flaky, or "not working", even if they don't say
  "debug". Triggers: "debug", "debugger", "ce bug", "ça plante", "test failure",
  "pourquoi ça ne marche pas", "find the root cause", "it's broken", "régression".
---

# Debugger

Systematic root-cause debugging.

> **Iron law: NO FIX WITHOUT ROOT-CAUSE INVESTIGATION FIRST.**
> A symptom patch is a debugging *failure*. A fix is only valid once a failing
> reproduction passes. Systematic investigation is **faster** than guess-and-check,
> not slower — rushing guarantees rework.

## The loop

1. **Reproduce** — Get a deterministic, minimal repro. If you can't reproduce it,
   you can't fix it. Write it down as a command or a failing test.
2. **Observe** — Read the actual error, stack trace, and logs. Don't guess from
   the symptom name. Quote the real output.
3. **Compare** — If a working equivalent exists (a passing sibling, a prior
   commit, a reference implementation), read it *completely* and list every
   difference from the broken version. The bug usually hides in a difference you
   dismissed. (Skip only when there's genuinely nothing to compare against.)
4. **Hypothesize** — State one falsifiable hypothesis: "X fails because Y".
   List 2-3 candidates ranked by likelihood, not by ease of fixing.
5. **Isolate** — Narrow the search space: bisect (git, input, code path), add
   targeted logging/asserts, or use a debugger. For a deep call stack, **trace
   the data backward** to where the bad value originates, don't reason forward
   from the symptom. Confirm or kill each hypothesis with evidence, not intuition.
6. **Fix** — Change the smallest thing that addresses the *cause*. No drive-by
   refactors (see [[clean-code]] — but not now).
7. **Verify** — The repro from step 1 now passes. Run the surrounding tests.
   Confirm you didn't move the bug.
8. **Regression-proof** — Add a test that fails without your fix and passes with it.

## Rules

- **One change at a time.** Two simultaneous changes = no signal.
- **Make it fail first.** If you can't make a test fail on purpose, you don't
  understand the bug yet.
- **Read before reasoning.** The trace, the input, the diff, the recent commits
  (`git log -p`). The cause is usually in what changed.
- **Surface confusion.** If the evidence contradicts the hypothesis, say so and
  re-hypothesize. Don't force a theory onto the data.
- **No "should work".** Either it's verified by a run, or it's a guess — label it.

## When 3 fixes fail

After **three** failed fix attempts, stop patching. The repeated failure is
itself evidence: the hypothesis or the design is wrong. Re-question the
assumptions, the boundary the bug lives on, and whether the architecture itself
is the cause (see [[clean-architect]]). Another patch will not help.

## Anti-patterns

- Shotgun debugging: changing many things hoping one helps.
- Fixing the test instead of the code.
- Adding `try/catch` to silence a symptom whose cause is unknown.
- Proposing a fix before tracing the data flow.
- Skipping a phase because of time pressure.
- Declaring "fixed" without a reproduction that now passes.

## Output

When done, report: the **root cause** (one sentence), the **fix**, the
**evidence** it works, and the **regression test** added.

## Handoff to spec (if using sdd)

If an sdd spec governs this code (`spec.db` present), don't stop at the local
regression test: hand the proven root cause to [[sdd-backprop]] (namespaced
`/sdd:backprop`) to record the bug and, when the cause is a class rather than a
one-off, add an invariant so the spec catches recurrence. Skip when there is no
spec.
