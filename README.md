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
| `code-reviewer` | Review a diff/PR for correctness, security, performance and quality before merge. |
| `debt-analyzer` | Turn technical debt into a prioritized, evidence-based backlog (impact ÷ effort). |
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

The `sdd` CLI binary is **not** vendored here: a `SessionStart` hook auto-provisions it from the [`Kirozen/sdd`](https://github.com/Kirozen/sdd) GitHub releases (verified by SHA256), matching the plugin version.

### `gopls-daemon`

Configures the Go language server (`gopls`) in shared daemon mode for Claude Code's LSP integration. Pure `.lsp.json` config — no skills.

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
