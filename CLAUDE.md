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
    agents/<agent>.md               # model-pinned subagent types, auto-discovered at plugin root
  sdd/                              # plugin lives here; only the CLI binary lives in Kirozen/sdd
    .claude-plugin/plugin.json
    skills/<skill>/SKILL.md
    agents/<agent>.md               # model-pinned subagent types (plan-scout, research-gather)
    commands/<cmd>.md               # slash commands (auto-discovered at plugin root)
    hooks/hooks.json                # SessionStart hook -> provisions the sdd binary
    scripts/ensure-sdd-binary.sh    # downloads the binary from Kirozen/sdd releases
  gopls-daemon/                     # LSP config only, no skills
    .claude-plugin/plugin.json
    .lsp.json                       # gopls in shared daemon mode
```

- **Skill files**: `plugins/<plugin>/skills/<skill>/SKILL.md`. The plugin scans `skills/` by default — no `skills` field needed in `plugin.json`.
- **Agent files**: `plugins/<plugin>/agents/<agent>.md`, auto-discovered at the plugin root (no `agents` field in `plugin.json`). Frontmatter `model:` (`haiku`/`sonnet`/`opus`) pins the subagent's model, and `tools:` is a read-only allowlist. Used to route a skill's mechanical subagents to a cheaper model while the orchestrator keeps the judgment on the session model. Skills spawn them via `subagent_type` under the scoped name `<plugin>:<agent>` (bare `<agent>` also resolves under `--plugin-dir`).
- **Frontmatter** (required): `name` + `description`. The `description` carries the trigger phrases (mixed FR/EN) and is the *primary* trigger mechanism — write them for recall, lean slightly "pushy" to fight under-triggering, not prose.
- **Cross-references**: skills link each other with `[[name]]` (e.g. `debugger` → `[[clean-architect]]`). It's a textual convention; the real invocation is namespaced (below).
- **Invocation is namespaced**: a plugin skill is called `/code-quality:<skill>`, not `/<skill>`. This is intentional and can't be changed.

Plugins currently in the marketplace:

- **`code-quality`** bundles `debugger`, `clean-code`, `clean-architect`, `code-reviewer`, `debt-analyzer`, `dva`, `techdebt` — a connected set (clean-code/clean-architect = the standard, code-reviewer = catch, debt-analyzer/techdebt = measure, dva = adversarially challenge, debugger = fix). `dva` (adversarial Engineer/Antagonist review) and `techdebt` (agent-orchestrated duplication/dead-code audit, with a `references/` dir) carry supporting reference files; `techdebt` also ships five model-pinned detection agents under `agents/` (mechanical passes on `haiku`, structural passes on `sonnet`); the rest are pure Markdown.
- **`sdd`** — SQLite-backed spec-driven development (skills `sdd-grill`, `sdd-spec`, `sdd-research`, `sdd-review`, `sdd-build`, `sdd-backprop`, `sdd-deepen`, `sdd-drift` + matching slash commands). This repo owns the plugin (skills, commands, hooks, agents); only the CLI binary + its release pipeline live in the separate `Kirozen/sdd` repo. See *Maintaining the `sdd` plugin* below.
- **`gopls-daemon`** — LSP-only plugin: a single `.lsp.json` configuring the Go language server (`gopls`) in shared daemon mode for Claude Code's LSP integration. No skills, no commands.

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

## Maintaining the `sdd` plugin

`plugins/sdd/` **is** the plugin (skills, commands, hooks, agents, provisioning script) and this repo owns it. Only the `sdd` **CLI binary** lives in `Kirozen/sdd`. The split of source-of-truth:

- **Skills / commands / hooks / agents** → this repo is the single source of truth. Edit them here.
- **The `sdd` CLI binary** → never vendored. The `SessionStart` hook runs `scripts/ensure-sdd-binary.sh`, which downloads the binary whose tag is read from `scripts/binary-version` (fallback: `plugin.json`'s `version`) from `https://github.com/Kirozen/sdd/releases/download/v<version>/` (verifies SHA256, lands it on PATH at `${CLAUDE_PLUGIN_ROOT}/bin/sdd`). The engine + release pipeline stay in `Kirozen/sdd`.
- **Two decoupled versions.** `plugin.json`'s `version` is the *plugin* version — bump it on any change here (skills, commands, hooks, agents) or installed users won't get updates. `scripts/binary-version` is the *CLI binary* release tag, bumped **only** after a new `Kirozen/sdd` binary release `vX.Y.Z`. They move independently: a skill-only change bumps `version` and leaves `binary-version` alone. The release asset names (`sdd_<os>_<arch>` + `SHA256SUMS`) are a contract the script depends on. **Mirror any change to `ensure-sdd-binary.sh` (and this decoupling — invariant V82) with `Kirozen/sdd`** so the script and the release-asset contract don't diverge.

## Gotchas

- **`plugin.json` is the only thing inside `.claude-plugin/`.** `skills/` (and any `commands/`, `agents/`, `hooks/`) live at the *plugin root*, never inside `.claude-plugin/`.
- **`.claude/skills/` symlinks are for local dogfooding only.** Installed plugins expose their skills namespaced (`/code-quality:debugger`). For development *inside this repo*, `.claude/skills/<skill>` symlinks each plugin's skill dir (relative: `../../plugins/<plugin>/skills/<skill>`) so it loads as a plain project skill (`/<skill>`, un-namespaced) without `/plugin install`. Regenerate after adding a skill; `sdd` must NOT carry such a symlink — its skills need the `sdd` binary on PATH, provisioned by the `SessionStart` hook that only fires on plugin install, not via a bare skill symlink.
- **Version bumps.** `plugin.json` sets `version`; bump it on each release or installed users won't get updates. If omitted, the git commit SHA is used instead.
- **`/reload-plugins`** is required after adding or renaming a skill/plugin; changes aren't picked up live.
- **No `../` paths.** Files referenced outside a plugin's own directory aren't copied to the install cache. Keep everything self-contained; use `${CLAUDE_PLUGIN_ROOT}` for internal paths in any hook/script.

## Writing style for skills

Dense and concise — the established convention is to strip "persona"/marketing filler and keep only actionable principles, checklists, and a clear output format. When adapting an external skill as reference, extract its substance, not its verbosity.
