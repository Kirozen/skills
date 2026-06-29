# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Personal Claude Code skills, packaged as a **plugin marketplace**. No build, test, or lint step — every skill is Markdown. This file documents the conventions that aren't obvious from the files.

## Structure

This repo is both a marketplace and a monorepo of plugins:

```
.claude-plugin/marketplace.json     # marketplace "kirozen-skills", lists the plugins
plugins/
  code-quality/
    .claude-plugin/plugin.json      # plugin manifest (ONLY this goes in .claude-plugin/)
    skills/<skill>/SKILL.md         # one dir per skill, auto-discovered by the plugin
  sdd/                              # vendored plugin; engine lives in the Kirozen/sdd repo
    .claude-plugin/plugin.json
    skills/<skill>/SKILL.md
    commands/<cmd>.md               # slash commands (auto-discovered at plugin root)
    hooks/hooks.json                # SessionStart hook -> provisions the sdd binary
    scripts/ensure-sdd-binary.sh    # downloads the binary from Kirozen/sdd releases
```

- **Skill files**: `plugins/<plugin>/skills/<skill>/SKILL.md`. The plugin scans `skills/` by default — no `skills` field needed in `plugin.json`.
- **Frontmatter** (required): `name` + `description`. The `description` carries the trigger phrases (mixed FR/EN) and is the *primary* trigger mechanism — write them for recall, lean slightly "pushy" to fight under-triggering, not prose.
- **Cross-references**: skills link each other with `[[name]]` (e.g. `debugger` → `[[clean-architect]]`). It's a textual convention; the real invocation is namespaced (below).
- **Invocation is namespaced**: a plugin skill is called `/code-quality:<skill>`, not `/<skill>`. This is intentional and can't be changed.

Plugins currently in the marketplace:

- **`code-quality`** bundles `debugger`, `clean-code`, `clean-architect`, `code-reviewer`, `debt-analyzer` — a connected set (clean-code/clean-architect = the standard, code-reviewer = catch, debt-analyzer = measure, debugger = fix). Pure Markdown skills.
- **`sdd`** — SQLite-backed spec-driven development (skills `sdd-grill`, `sdd-spec`, `sdd-research`, `sdd-review`, `sdd-build`, `sdd-backprop`, `sdd-deepen` + matching slash commands). This is a *vendored* plugin: the engine (Go CLI, release pipeline) lives in the separate `Kirozen/sdd` repo. See *Maintaining the vendored `sdd` plugin* below.

## Adding a skill (to an existing plugin)

```bash
mkdir -p plugins/code-quality/skills/<skill>
$EDITOR plugins/code-quality/skills/<skill>/SKILL.md   # frontmatter + body
git add plugins/code-quality/skills/<skill>/SKILL.md
# then /reload-plugins inside Claude Code
```

## Adding a new plugin

1. `mkdir -p plugins/<plugin>/.claude-plugin plugins/<plugin>/skills`
2. Write `plugins/<plugin>/.claude-plugin/plugin.json` (`name` is the only required field, kebab-case).
3. Add an entry to `.claude-plugin/marketplace.json` `plugins[]` with `"source": "./plugins/<plugin>"`.

## Testing & releasing

```bash
claude plugin validate .                      # validate marketplace + plugin manifests
claude --plugin-dir ./plugins/code-quality    # load the plugin locally without installing
/plugin marketplace add .                      # register this repo as a marketplace
/plugin install code-quality@kirozen-skills    # then install the plugin
```

## Maintaining the vendored `sdd` plugin

`plugins/sdd/` is a copy of the plugin slice of the `Kirozen/sdd` repo (skills, commands, hooks, provisioning script) — **not** its Go source. The split of source-of-truth:

- **Skills / commands / hooks** → this repo is now the single source of truth. Edit them here.
- **The `sdd` CLI binary** → never vendored. The `SessionStart` hook runs `scripts/ensure-sdd-binary.sh`, which downloads the binary matching `plugin.json`'s `version` from `https://github.com/Kirozen/sdd/releases/download/v<version>/` (verifies SHA256, lands it on PATH at `${CLAUDE_PLUGIN_ROOT}/bin/sdd`). The engine + release pipeline stay in `Kirozen/sdd`.
- **Version coupling.** After a new `Kirozen/sdd` release `vX.Y.Z`, bump `version` in `plugins/sdd/.claude-plugin/plugin.json` here — otherwise the hook keeps fetching the old binary. The release asset names (`sdd_<os>_<arch>` + `SHA256SUMS`) are a contract the script depends on.

## Gotchas

- **`plugin.json` is the only thing inside `.claude-plugin/`.** `skills/` (and any `commands/`, `agents/`, `hooks/`) live at the *plugin root*, never inside `.claude-plugin/`.
- **`.claude/skills/` symlinks are for local dogfooding only.** Installed plugins expose their skills namespaced (`/code-quality:debugger`). For development *inside this repo*, `.claude/skills/<skill>` symlinks each plugin's skill dir (relative: `../../plugins/<plugin>/skills/<skill>`) so it loads as a plain project skill (`/<skill>`, un-namespaced) without `/plugin install`. Regenerate after adding a skill; vendored plugins shipped elsewhere (e.g. `sdd`) must NOT carry such a symlink (it was removed from `Kirozen/sdd`).
- **Version bumps.** `plugin.json` sets `version`; bump it on each release or installed users won't get updates. If omitted, the git commit SHA is used instead.
- **`/reload-plugins`** is required after adding or renaming a skill/plugin; changes aren't picked up live.
- **No `../` paths.** Files referenced outside a plugin's own directory aren't copied to the install cache. Keep everything self-contained; use `${CLAUDE_PLUGIN_ROOT}` for internal paths in any hook/script.

## Writing style for skills

Dense and concise — the established convention is to strip "persona"/marketing filler and keep only actionable principles, checklists, and a clear output format. When adapting an external skill as reference, extract its substance, not its verbosity.
