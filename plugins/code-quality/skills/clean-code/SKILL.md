---
name: clean-code
description: >-
  Use when writing or refactoring code to keep it simple and readable, enforcing
  Robert C. Martin's Clean Code principles (naming, small functions, SOLID). Use
  this whenever the user writes new code, asks to clean up, simplify, or improve
  code, or wonders if something is overcomplicated — even without saying "clean
  code". Triggers: "clean code", "refactor", "simplify this", "code propre",
  "is this overcomplicated", "améliore ce code", "SOLID".
---

# Clean Code (Robert C. Martin)

Enforce the principles from *Clean Code* by Robert C. Martin. The test of clean
code, per Grady Booch: **it can be read and enhanced by a developer other than
its original author.** Pairs with [[code-reviewer]] (catch) and
[[debt-analyzer]] (measure).

Use when writing new code, reviewing a PR, refactoring legacy code, or setting
team standards.

## The nine principles

1. **Meaningful names** — Intention-revealing, pronounceable, searchable. The
   name says *why*; no need for a comment to decode it. Avoid disinformation
   and noise words (`data`, `info`, `manager`).
2. **Functions** — Small. Do one thing, at one level of abstraction. Few
   arguments (0–2 ideal; >3 needs justification). No boolean flag arguments —
   split into two functions. No hidden side effects.
3. **Comments** — A comment is a failure to express intent in code. Rewrite the
   code first. Keep comments only for *why* (intent, warning, legal, TODO),
   never to restate *what*. Delete commented-out code.
4. **Formatting** — Newspaper structure: high-level at top, details below.
   Vertical density for related lines, blank lines to separate concepts.
   Consistent indentation. Callers above callees.
5. **Objects & data structures** — Hide internals behind abstractions; expose
   behavior, not fields. Respect the **Law of Demeter** (don't reach through
   objects: no `a.getB().getC().do()`).
6. **Error handling** — Exceptions over return codes. Don't return or pass
   `null`. Provide context with each exception. Error handling is *one thing* —
   keep it out of the happy path.
7. **Unit tests** — TDD: test first. One assert/concept per test. Tests are
   first-class code (clean too). **F.I.R.S.T.**: Fast, Independent, Repeatable,
   Self-validating, Timely.
8. **Classes** — Single Responsibility: one reason to change. Small, cohesive,
   organized top-down for readability. Depend on abstractions, not concretions.
9. **Code smells** — Watch for **rigidity** (hard to change), **fragility**
   (breaks elsewhere), **immobility** (can't reuse), **viscosity** (shortcuts
   easier than the right way), and **needless complexity / duplication**.

## SOLID principles

Five object-oriented design principles (also Robert C. Martin) that keep classes
flexible and maintainable:

- **S — Single Responsibility** — A class has one reason to change, one actor it
  answers to. (Same idea as principle 8 above.)
- **O — Open/Closed** — Open for extension, closed for modification. Add behavior
  via new code (polymorphism, strategy), not by editing existing classes.
- **L — Liskov Substitution** — A subtype must be usable anywhere its base type
  is, without surprising the caller. No strengthened preconditions, no thrown
  "not supported".
- **I — Interface Segregation** — Many small client-specific interfaces beat one
  fat one. No class should be forced to depend on methods it doesn't use.
- **D — Dependency Inversion** — Depend on abstractions, not concretions.
  High-level policy must not depend on low-level detail; both depend on
  interfaces. (Inject dependencies; don't `new` them inline.)

## Checklist before "done"

- [ ] Names reveal intent without a comment.
- [ ] Each function does one thing, at one level of abstraction.
- [ ] No comment restates what the code already says.
- [ ] No duplication (DRY) — same logic isn't in two places.
- [ ] Errors handled via exceptions; no `null` returned/passed.
- [ ] Tests exist, are F.I.R.S.T., and test behavior not implementation.
- [ ] Classes have a single responsibility.
- [ ] SOLID respected: extension over modification, subtypes substitutable,
      interfaces narrow, dependencies on abstractions.
- [ ] No rigidity / fragility / needless complexity introduced.

## The line not to cross

Clean code is **surgical**. When editing existing code, touch only what the task
requires. Don't reformat unrelated lines or refactor what isn't broken. Note
pre-existing smells — don't silently fix them. Match the surrounding style even
when you'd do it differently.

## Limitations

These principles guide judgment; they aren't absolute laws — apply them in
context, not dogmatically. This skill is not a substitute for tests, profiling,
or expert review.
