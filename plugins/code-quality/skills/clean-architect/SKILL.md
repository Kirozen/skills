---
name: clean-architect
description: >-
  Use when designing system structure, organizing layers/modules/boundaries, or
  judging dependency direction, per Robert C. Martin's Clean Architecture. Use
  this whenever the user asks where code should live, how to structure a project,
  or about layers, coupling, or framework/DB independence — even without saying
  "architecture". Triggers: "clean architecture", "clean architect",
  "architecture", "layers", "where should this code live", "couches",
  "dependency direction", "decoupling".
---

# Clean Architecture (Robert C. Martin)

Enforce *Clean Architecture* by Robert C. Martin. The goal: keep business rules
independent of frameworks, UI, database, and any external detail, so the system
stays testable and changeable. Builds on the SOLID principles in [[clean-code]];
pairs with [[debt-analyzer]] (find violations) and [[code-reviewer]] (catch them
in a diff).

## The Dependency Rule (the one rule)

**Source code dependencies point only inward, toward higher-level policy.**
Nothing in an inner circle knows anything about an outer circle — no name from an
outer layer (class, function, variable, data format) may appear in inner code.

```
            ┌─────────────────────────────────┐
            │  Frameworks & Drivers           │  web, DB, UI, devices
            │   ┌───────────────────────────┐ │
            │   │ Interface Adapters        │ │  controllers, presenters, gateways
            │   │   ┌─────────────────────┐ │ │
            │   │   │ Use Cases           │ │ │  application business rules
            │   │   │   ┌───────────────┐ │ │ │
            │   │   │   │  Entities     │ │ │ │  enterprise business rules
            │   │   │   └───────────────┘ │ │ │
            │   │   └─────────────────────┘ │ │
            │   └───────────────────────────┘ │
            └─────────────────────────────────┘
                  dependencies point inward  →  ●
```

## The four layers

1. **Entities** — Enterprise-wide business rules and critical data. The most
   stable, most general code. No knowledge of use cases, DB, or UI.
2. **Use Cases** — Application-specific business rules. Orchestrate entities to
   achieve a user goal. Know entities; know nothing of the web, DB, or framework.
3. **Interface Adapters** — Controllers, presenters, gateways, repositories.
   Convert data between the use-case form and the external form (DB rows, HTTP,
   view models). The MVC of the app lives here.
4. **Frameworks & Drivers** — The web framework, database, ORM, UI toolkit,
   devices. Glue code only; the outermost, most volatile, most replaceable ring.

## Crossing boundaries

- **Dependency Inversion at every boundary.** When an inner layer must call out
  (e.g. a use case needs to persist), it defines an *interface* (port); the
  outer layer *implements* it (adapter). The dependency points inward even though
  control flows outward.
- **Pass simple data across boundaries** — DTOs / plain structs, never entities
  or DB rows or framework objects. Don't violate the rule by smuggling an outer
  type inward.
- **Humble Object** at hard-to-test edges (UI, DB): keep the behavior testable in
  an inner object, leave a thin, dumb shell outside.

## Design heuristics

- **Screaming architecture** — The top-level structure should shout the *domain*
  (orders, billing, accounts), not the framework (`controllers/`, `models/`).
- **Frameworks are details.** Depend on them at arm's length; never let them
  dictate your business rules. Defer the choice of DB/web framework as long as
  possible.
- **Stable Dependencies** — Depend in the direction of stability. Volatile things
  (frameworks) depend on stable things (policy), never the reverse.
- **Component cohesion** — Group classes that change together for the same reason
  and are released together; keep acyclic dependencies between components.
- **Draw the boundaries** where the axes of change differ. Over-partitioning is
  its own cost — boundaries are an option you buy, not a default.

## Review checklist

- [ ] Do all source dependencies point inward? Any inner code referencing an
      outer name is a violation.
- [ ] Do business rules (entities, use cases) compile/test with no framework, DB,
      or UI present?
- [ ] Are boundaries crossed via interfaces owned by the inner layer (ports),
      implemented outside (adapters)?
- [ ] Do only simple data structures cross boundaries (no entities/DB/framework
      types leaking out or in)?
- [ ] Does the top-level layout reveal the domain, not the framework?
- [ ] Could the DB or web framework be swapped without touching use cases?

## The line not to cross

Architecture serves change, not purity. Don't impose all four layers on a small
script or a CRUD app that will never grow — the boundaries cost indirection.
Apply the Dependency Rule where the cost of change is real; note over-engineering
as a smell, not a goal.
