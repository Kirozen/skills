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
```

- **Skill files**: `plugins/<plugin>/skills/<skill>/SKILL.md`. The plugin scans `skills/` by default — no `skills` field needed in `plugin.json`.
- **Frontmatter** (required): `name` + `description`. The `description` carries the trigger phrases (mixed FR/EN) and is the *primary* trigger mechanism — write them for recall, lean slightly "pushy" to fight under-triggering, not prose.
- **Cross-references**: skills link each other with `[[name]]` (e.g. `debugger` → `[[clean-architect]]`). It's a textual convention; the real invocation is namespaced (below).
- **Invocation is namespaced**: a plugin skill is called `/code-quality:<skill>`, not `/<skill>`. This is intentional and can't be changed.

Current plugin `code-quality` bundles: `debugger`, `clean-code`, `clean-architect`, `code-reviewer`, `debt-analyzer` — a connected set (clean-code/clean-architect = the standard, code-reviewer = catch, debt-analyzer = measure, debugger = fix).

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

## Gotchas

- **`plugin.json` is the only thing inside `.claude-plugin/`.** `skills/` (and any `commands/`, `agents/`, `hooks/`) live at the *plugin root*, never inside `.claude-plugin/`.
- **No symlinks, no `.claude/skills/`.** Plugin skills are auto-discovered after install; the old project-skill symlink mechanism was removed.
- **Version bumps.** `plugin.json` sets `version`; bump it on each release or installed users won't get updates. If omitted, the git commit SHA is used instead.
- **`/reload-plugins`** is required after adding or renaming a skill/plugin; changes aren't picked up live.
- **No `../` paths.** Files referenced outside a plugin's own directory aren't copied to the install cache. Keep everything self-contained; use `${CLAUDE_PLUGIN_ROOT}` for internal paths in any hook/script.

## Writing style for skills

Dense and concise — the established convention is to strip "persona"/marketing filler and keep only actionable principles, checklists, and a clear output format. When adapting an external skill as reference, extract its substance, not its verbosity.
