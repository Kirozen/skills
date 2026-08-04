# sdd CLI — command reference

Condensed reference for the `sdd` subcommands actually used by this plugin's
skills/commands (`plugins/sdd/skills/*/SKILL.md`, `plugins/sdd/commands/*.md`).
Not a copy of `sdd --help` — check that directly for anything missing here or
for a command added since this file was last revalidated (see the reminder in
`CLAUDE.md`'s "Two decoupled versions" section).

Two parameters look like enums but differ:
- `--status` (task) is a real enum, DB-enforced: `.` todo / `~` wip / `x` done.
- `<kind>` (`add-interface`) is free text, **not** DB-enforced — a convention.
  Common values in use: `cmd`, `go`, `file`, `http`, `struct`, `func`, `config`,
  `sql`, `doc`, `iface`, `type`, `method`, `yaml`, `sh`, `proto`, `js`,
  `dockerfile`, `ci`. Pick the closest fit; inventing a new one is fine.

| Commande | Flags | Exemple |
|---|---|---|
| `add-bug` | `<cause>` `--fix <V-list>` (required) `--date <ISO>` (default: today) | `sdd add-bug "nil deref on empty input" --fix V12` |
| `add-cite` | `<T-ord>` `--feature <f>` (required) `<cite> [<cite>...]` (space-separated, not comma) | `sdd add-cite T2 --feature 1 V3 I.commands-md` |
| `add-constraint` | `<text>` `--feature <f>` (required) | `sdd add-constraint "hors scope: rtk, git" --feature 1` |
| `add-goal` | `<text>` `--feature <f>` (required) | `sdd add-goal "réduire les appels --help" --feature 1` |
| `add-interface` | `<kind>` `<name>` `<sig>` (positional, no flags) | `sdd add-interface doc commands-md "table commande\|flags\|exemple"` |
| `add-invariant` | `<text>` (positional, no flags) | `sdd add-invariant "COMMANDS.md documente chaque sous-commande utilisée"` |
| `add-research` | `<topic>` `<finding>` `<src>` (positional, no flags) | `sdd add-research "cli-help" "kind n'est pas un enum DB" "schema interface"` |
| `add-task` | `<text>` `--feature <f>` (required) `--cites <V/I-list>` | `sdd add-task "écrire COMMANDS.md" --feature 1 --cites V1,V2` |
| `apply` | stdin: TAB-delimited `add-*` lines, one transaction | `sdd apply < batch.tsv` |
| `block` | `<T-ord>` `--on <T-ord,...>` (required) `--feature <f>` (required) | `sdd block T2 --on T1 --feature 1` |
| `cat` | `--feature <f>` (default: all unfinished) | `sdd cat --feature 1` |
| `cover` | (read-only, no flags) | `sdd cover` |
| `deprecate-interface` | `<name>` (positional, no flags) | `sdd deprecate-interface old-http-client` |
| `edit` | `<kind> <key>` `--feature <f>` (required for kind=task) `--text <new>` (required); or `edit goal\|constraint <F-ord> <n>` | `sdd edit task T3 --feature 1 --text "..."` |
| `export` | (no flags) | `sdd export` |
| `graph` | `[key]` `--all` (full project) `--feature <f>` (one feature's neighborhood) | `sdd graph V3` |
| `guide` | (read-only, no flags) | `sdd guide` |
| `list` | `[kind]` (invariant\|interface\|task\|bug\|research\|feature\|unknown) `--feature <f>` (task only) `--status <.\|~\|x>` (task only) | `sdd list task --feature 1 --status .` |
| `new-feature` | `<name>` (positional, no flags) | `sdd new-feature "sdd-cli-help-discoverability"` |
| `next` | (read-only, no flags) | `sdd next` |
| `ready` | (read-only, no flags; TSV output) | `sdd ready` |
| `refs` | `<ref>` (V\<n\> or I.\<name\>) | `sdd refs V3` |
| `rm-task` | `<T-ord>` `--feature <f>` (required); cites+blockers cascade | `sdd rm-task T4 --feature 1` |
| `set-task` | `<T-ord>` `--feature <f>` (required) `--status <.\|~\|x>` (required, DB enum) | `sdd set-task T1 --feature 1 --status x` |
| `status` | (read-only, no flags) | `sdd status` |
| `todo` | `--pretty` (grouped human view; default TSV) | `sdd todo --pretty` |
| `unblock` | `<T-ord>` `--off <T-ord,...>` (required) `--feature <f>` (required) | `sdd unblock T2 --off T1 --feature 1` |

All commands also accept the global `--db <path>` flag (default: XDG config
dir). Run `sdd <cmd> --help` for anything not covered above, or if this file
looks stale relative to `sdd --help`.
