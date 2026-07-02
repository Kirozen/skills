# kirozen-skills

Personal [Claude Code](https://claude.ai/code) plugin marketplace. Everything is Markdown + config — no build step.

The marketplace bundles three plugins. Skills are auto-discovered after install and invoked **namespaced** as `/<plugin>:<skill>`.

## Plugins

### `code-quality`

Clean-code & review skills based on Robert C. Martin's *Clean Code* / *Clean Architecture*.

| Skill | What it does |
|-------|--------------|
| `clean-code` | Keep code simple and readable while writing/refactoring (naming, small functions, SOLID). |
| `clean-architect` | Structure layers/modules/boundaries and judge dependency direction. |
| `code-reviewer` | Review a concrete diff/PR for correctness, security, performance and quality before merge. |
| `dva` | Adversarially stress-test a *proposed* solution/design (Engineer/Antagonist red-team) before committing to it. |
| `debt-analyzer` | Turn technical debt into a prioritized, evidence-based backlog (impact ÷ effort) — *what to fix first*. |
| `techdebt` | Exhaustive, agent-orchestrated scan that *enumerates* concrete duplication & dead code with file/line evidence. |
| `debugger` | Find and *prove* the root cause of a bug before proposing a fix. |

### `sdd`

SQLite-backed spec-driven development. The spec lives in a database; `SPEC.md` is a generated read-only view. The full loop is driven by the `sdd` CLI.

| Skill | What it does |
|-------|--------------|
| `sdd-grill` | Sharpen a fuzzy idea into a feature's goal + constraints. |
| `sdd-spec` | Sole mutator of the spec — invariants, interfaces, tasks. |
| `sdd-research` | Gather external knowledge into the durable research log, each finding cited. |
| `sdd-review` | Adversarial senior review of the spec before any code; ends on a go/no-go gate. |
| `sdd-build` | Plan-then-execute against the spec tasks; auto-invokes backprop on failure. |
| `sdd-backprop` | Bug → spec: trace the root cause and persist a new invariant to catch recurrence. |
| `sdd-deepen` | Optional design-improvement pass — shrink interfaces, hide decisions. |
| `sdd-drift` | Read-only detector for code-vs-spec drift; reports violations by severity, writes nothing. Distinct from the `sdd check` CLI (SPEC.md == spec.db). |

The `sdd` CLI binary is **not** vendored here: a `SessionStart` hook auto-provisions it from the [`Kirozen/sdd`](https://github.com/Kirozen/sdd) GitHub releases (verified by SHA256). The binary's release tag is pinned independently of the plugin version in `plugins/sdd/scripts/binary-version`, so skill-only updates can bump the plugin without requiring a matching binary release.

### `gopls-daemon`

Configures the Go language server (`gopls`) in shared daemon mode for Claude Code's LSP integration. Pure `.lsp.json` config — no skills.

## Usage

Skills trigger **two ways**:

- **Automatically** — Claude reads each skill's `description` and invokes the right one when your request matches. You don't name it; just describe the task ("relis mon diff", "scan the codebase for duplication", "challenge this design"). Writing the triggers into the description is the *primary* mechanism.
- **Explicitly** — invoke it by name. Installed, skills are **namespaced**: `/code-quality:code-reviewer`, `/sdd:sdd-spec`. When dogfooding inside this repo (symlinks), they're un-namespaced: `/code-reviewer`.

### Picking between overlapping skills

Two pairs in `code-quality` are deliberately close — pick by *intent*:

| You want to… | Use | Not |
|--------------|-----|-----|
| Review concrete changes (a diff/PR) before merge | `code-reviewer` | `dva` |
| Pressure-test a *proposed* approach/architecture before writing it | `dva` | `code-reviewer` |
| Decide *what debt to fix first* (prioritized backlog) | `debt-analyzer` | `techdebt` |
| *Enumerate* concrete duplication / dead code across the repo | `techdebt` | `debt-analyzer` |

A typical flow chains them: `clean-architect` (design) → `dva` (stress-test the design) → `clean-code` (write) → `code-reviewer` (review the diff) → `debugger` (if something breaks). For health checks, `techdebt` enumerates, then `debt-analyzer` prioritizes the findings.

## Install

```bash
/plugin marketplace add Kirozen/skills      # or: /plugin marketplace add .  (local clone)
/plugin install code-quality@kirozen-skills
/plugin install sdd@kirozen-skills
/plugin install gopls-daemon@kirozen-skills
```

Run `/reload-plugins` after installing.

## Local development

```bash
claude plugin validate .                      # validate marketplace + plugin manifests
claude --plugin-dir ./plugins/code-quality    # load a plugin without installing
```

For dogfooding inside this repo, each skill is symlinked into `.claude/skills/`, so it loads as a plain project skill (`/<skill>`, un-namespaced) without installing the plugin.

Conventions, structure, and the rules for adding skills/plugins live in [`CLAUDE.md`](./CLAUDE.md).
