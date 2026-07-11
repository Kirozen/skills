---
name: debt-analyzer
description: >-
  Use to turn technical debt into a prioritized, evidence-based backlog
  (impact ÷ effort) — the "what should we fix first" decision. Use this
  whenever the user asks what to refactor first, where the risks or hotspots
  are, or about overall code health — even without saying "technical debt".
  For an exhaustive, agent-orchestrated scan that locates concrete duplicate
  functions, copy-pasted blocks, and dead code, use [[techdebt]] instead; this
  skill prioritizes, techdebt enumerates. Triggers: "technical debt", "debt
  analyzer", "dette technique", "what should we refactor", "where are the
  risks", "code health", "hotspots", "prioritize the debt".
---

# Technical Debt Analyzer

Map technical debt, quantify its cost, and prioritize it by **impact ÷ effort** —
not by how ugly it looks. Debt only matters where it slows change or risks
failure. Feeds [[clean-code]] / [[clean-architect]] (the fix) and
[[code-reviewer]] (preventing new debt).

## Method

1. **Define the scope.** A module, a service, or the whole repo. State it, and
   time-box the analysis — a backlog nobody acts on is wasted effort. Stop when
   the top items are clear; don't audit every line.
2. **Gather signals** — combine evidence, don't rely on gut feel. Five of the
   probes below are independent, mechanical/structural, and read-only — spawn
   them as parallel signal agents (see **Agent orchestration**); the two
   requiring architectural or coverage-tool judgment stay in this context.
   - **Churn × complexity** — files changed often *and* hard to read are the
     hotspots: `git log --format= --name-only | sort | uniq -c | sort -rn`.
     → `code-quality:debt-signal-churn`.
   - **Size & coupling** — oversized files/functions, circular deps, god objects:
     `find . -name '*.<ext>' | xargs wc -l | sort -rn | head`.
     → `code-quality:debt-signal-size-coupling`.
   - **Architecture** — violations of the Dependency Rule: inner layers importing
     outer ones, business logic reaching into frameworks/DB (see [[clean-architect]]).
     The costliest debt, and invisible to churn. Stays main-thread — judging a
     layering violation needs [[clean-architect]]'s reasoning, not a mechanical pass.
   - **Test coverage gaps** — risky code with no safety net (coverage report).
     Stays main-thread — running/reading the project's coverage tool is
     one-off, not worth a dedicated agent.
   - **TODO/FIXME/HACK** markers and their age:
     `grep -rn "TODO\|FIXME\|HACK"` then `git blame` for age.
     → `code-quality:debt-signal-todo-age`.
   - **Outdated / vulnerable deps** — `npm outdated` / `pip list --outdated` /
     `cargo audit` (or the ecosystem's equivalent).
     → `code-quality:debt-signal-deps`.
   - **Duplication** — the same logic in N places (e.g. `jscpd`).
     → `code-quality:debt-signal-duplication`.

### Agent orchestration

**Scope threshold first.** A module or small service doesn't justify spawning
5 subagents — run the five mechanical/structural probes above inline. Above
that (the whole repo, or a repo with hundreds of files), spawn them in
parallel:

| `subagent_type`                        | Signal            | Model  |
|-------------------------------------------|--------------------|--------|
| `code-quality:debt-signal-churn`          | Churn × complexity | haiku  |
| `code-quality:debt-signal-size-coupling`  | Size & coupling    | haiku  |
| `code-quality:debt-signal-todo-age`       | TODO/FIXME/HACK age| haiku  |
| `code-quality:debt-signal-deps`           | Outdated/vulnerable deps | haiku |
| `code-quality:debt-signal-duplication`    | Duplication        | sonnet |

The four mechanical passes run on **haiku** — counting, grepping, and
running a package-manager audit command against explicit rules. Duplication
needs structural comparison, so it runs on **sonnet** — same split as
techdebt's mechanical-vs-structural agents. Pass each the run context (scope,
root, exclusions) as its task prompt, same as techdebt's Pre-Scan Phase. All
are read-only; `debt-signal-deps` may only run its ecosystem's read-only
audit/list command (never install/update/fix).

> Names are scoped to the plugin (`code-quality:debt-signal-*`).
> Un-namespaced `debt-signal-*` also resolves if loaded via `--plugin-dir`.

If a signal agent fails or returns unparseable output, do not drop it
silently — note that signal as **NOT GATHERED** with the reason, so the
backlog never reads as "no debt found" when a probe simply failed. Scoring
(impact/effort/trend, steps 5-6 below) always stays main-thread — signal
agents report raw hotspots, never a priority.
3. **Deduplicate hotspots** (main-thread, after all signals return — including
   spawned signal agents). The same file often surfaces from several signals
   (high churn *and* large *and* untested). Count it **once** — convergence
   raises its priority; it doesn't lengthen the list.
4. **Classify each item** by:
   - **Type** — *architecture* (wrong boundaries), *design* (wrong abstraction),
     *code* (messy but local), *test* (missing safety), *dependency*
     (stale/risky), *doc* (lost knowledge).
   - **Intent** (Fowler's quadrant) — *deliberate & prudent* ("shipped, will
     clean up") vs *reckless / inadvertent*. Assumed prudent debt is managed,
     not condemned; recklessness is the real target.
5. **Score** each item:
   - **Impact** — how much does it slow work or risk an incident? (high/med/low)
   - **Effort** — cost to fix, honestly. (high/med/low)
   - **Trend** — is it getting worse on its own? Use the trajectory (churn over
     6 months, new TODOs) to separate worsening debt from dormant debt you can
     safely leave.
6. **Prioritize** — high-impact + low-effort first. Quarantine high-effort,
   low-impact debt: name it, then leave it.

## Rules

- **Evidence over taste.** "This file is touched 80×/quarter and has no tests"
  beats "this code is ugly".
- **Debt is a tradeoff, not a sin.** Some debt is rational. Flag it, attach a
  cost, let the owner decide.
- **Don't fix while analyzing.** This skill produces a *backlog*, not edits.
  Hand fixes to [[clean-code]] / [[clean-architect]] / [[debugger]].
- **Safety net before refactor.** If a high-effort item has no test coverage,
  the first suggested action is *add characterization tests*, not refactor —
  otherwise the fix is a new risk.
- **Quantify the interest.** Estimate the recurring cost ("every change here
  takes 2× longer"), not just the principal.

## Output

A ranked table:
```
# | Item (file/area) | Type | Intent | Impact | Effort | Trend | Why it matters | Suggested action
```
Followed by a short narrative: the **top 3 to tackle now**, and what's
**explicitly deferred** and why.

## Handoff to spec (if using sdd)

If an sdd spec governs this repo (`spec.db` present), the backlog items you commit
to fixing become tracked work: hand the top items to [[sdd-spec]] (namespaced
`/sdd:spec`, `sdd add-task`) so they live against the spec instead of a loose
list. Skip when there is no spec.
