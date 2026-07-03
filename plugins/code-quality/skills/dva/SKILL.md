---
name: dva
description: >-
  Adversarial stress-test of a proposed solution, design, or architecture: an
  Engineer/Antagonist loop that actively tries to refute the approach across
  correctness, security, performance, reliability, and maintainability, then
  iterates to a verdict. Use this to challenge a decision or design before
  committing to it — not necessarily tied to a concrete diff. For a standard
  line-by-line review of an existing diff/PR before merge, use [[code-reviewer]]
  instead. Triggers: "dva", "adversarial review", "revue adversariale",
  "double verification", "challenge this solution", "stress-test this design",
  "find flaws", "red team this".
---

# Adversarial Double-Verification (DVA)

Systematic method for validating and improving technical solutions through a structured feedback loop between two roles: **Engineer** (proposes) and **Antagonist** (challenges).

## When to Use

- Stress-testing a proposed design, architecture, or algorithm before committing to it
- Pressure-testing an approach for edge cases, security, and performance
- Comparing alternatives before choosing one
- Red-teaming a technical decision

For a line-by-line review of an already-written diff/PR before merge, use
[[code-reviewer]] instead — that reviews concrete changes; DVA challenges a
proposal.

## Workflow

### Phase 1: Preparation (Engineer Role)

Understand and frame the solution before critiquing it.

1. **Define the scope**:
   - Problem being solved
   - Constraints (performance, compatibility, security)
   - Success criteria (error rate, coverage, latency targets)

2. **Summarize the proposed solution**:
   - Key design decisions and trade-offs
   - Dependencies and assumptions
   - Existing tests or validations

3. **Output**: A concise summary of the solution with explicit assumptions and constraints listed.

### Phase 2: Critical Analysis (Antagonist Role)

Systematically find all flaws across these dimensions:

#### Functional Correctness
- What happens with null/empty/malformed inputs?
- Are all edge cases handled (zero, max, negative, unicode, concurrent access)?
- Do error paths return meaningful information?
- Are all assumptions validated at system boundaries?

#### Security
- Input validation and sanitization (OWASP Top 10)
- Authentication/authorization gaps
- Secrets exposure (logs, error messages, configs)
- Dependency vulnerabilities (known CVEs, unmaintained libraries)

#### Performance
- Time/space complexity under load
- N+1 queries, missing indexes, unbounded allocations
- Resource cleanup (connections, file handles, goroutines)
- Behavior under contention (locks, deadlocks, race conditions)

#### Reliability
- What happens if an external dependency is unavailable?
- Are timeouts set on all external calls?
- Is retry logic idempotent? Are retries bounded?
- Graceful degradation vs. hard failure

#### Maintainability
- Is the code readable without comments?
- Are abstractions justified or premature?
- Does the change increase coupling?
- Are there hidden side effects?

**Output**: A feedback table:

| Category | Flaw | Impact | Evidence | Suggested Fix | Severity |
|----------|------|--------|----------|---------------|----------|
| ... | ... | ... | ... | ... | Critical/Major/Minor |

### Phase 3: Iteration

1. **Prioritize** flaws by severity (critical > major > minor)
2. **Fix** critical and major flaws first
3. **Re-validate** each fix — the Antagonist verifies the flaw is resolved and no regression is introduced
4. **Loop** until:
   - No critical or major flaws remain
   - All edge cases are covered
   - Performance/security criteria are met

## Output Format

Present findings as:

```
## DVA Report

### Summary
- Solution: [one-line description]
- Scope: [files/components reviewed]
- Verdict: [PASS / PASS WITH WARNINGS / FAIL]

### Critical Flaws
[numbered list with evidence and fix]

### Major Flaws
[numbered list with evidence and fix]

### Minor Flaws / Suggestions
[numbered list]

### Validated Strengths
[things the Antagonist confirmed are well-designed]
```

## Best Practices

1. **Separate the roles explicitly** — complete the full Engineer summary before switching to Antagonist
2. **Always provide evidence** — code snippets, example inputs, or concrete scenarios for each flaw
3. **Prioritize by blast radius** — a security flaw affecting all users outranks a performance issue in a rare path
4. **Validate strengths too** — acknowledge what works well to avoid over-correction
5. **Be specific** — "this could fail" is not useful; "this fails when `input == ""` because line 42 dereferences without nil check" is useful
6. **Check dependencies** — is the library maintained? Are there known CVEs? Is the license compatible?
7. **Test the fix, not just the flaw** — verify that proposed fixes don't introduce new issues

## Scope Adaptation

Adapt depth to the scope of the proposal:

| Proposal scope | Engineer Phase | Antagonist Phase |
|-------------|---------------|-----------------|
| Single algorithm / function design | 2-3 sentences | Focus on edge cases + security |
| Feature / component design | Full summary with assumptions | All 5 dimensions |
| Architecture decision | Trade-off analysis | Full DVA + alternatives comparison |

## Handoff to spec (if using sdd)

If an sdd spec governs the target (`spec.db` present), a flaw that survives the
Antagonist phase and names a *recurring failure mode* is worth persisting: hand
it to [[sdd-backprop]] (namespaced `/sdd:backprop`) to add an invariant with a
proving test, so the design constraint is enforced, not just noted. Skip when
there is no spec.

## References

See `references/detailed-guide.md` for tooling recommendations, a feedback-sheet template, and worked examples.
