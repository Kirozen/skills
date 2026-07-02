---
name: techdebt
description: >-
  Exhaustive, agent-orchestrated scan that locates concrete instances of
  duplicate functions, copy-pasted blocks, similar implementations ripe for
  consolidation, and dead code — and produces a findings report with file/line
  evidence. Use this when the goal is to enumerate and find duplication/dead
  code across a codebase. For prioritizing existing debt into a "what to fix
  first" backlog (impact ÷ effort) rather than enumerating it, use
  [[debt-analyzer]] instead. Triggers: "tech debt audit", "code duplication
  scan", "find duplicate code", "dead code detection", "consolidation
  opportunities", "scan for copy-paste".
---

# Tech Debt Scanner

Exhaustive, agent-orchestrated scan that enumerates concrete duplication and dead
code with file/line evidence. Complements [[debt-analyzer]] (which prioritizes
what to fix first); this skill *finds* the instances.

## When to Use

- Sprint dedicated to reducing technical debt
- Onboarding onto an unfamiliar codebase — get a structural health overview
- Before a major refactor — identify what to consolidate first
- Periodic codebase health check (quarterly, post-milestone)
- After a rapid prototyping phase — find copy-paste from spike work

## When NOT to Use

- Refactoring a single file — just read it and refactor directly
- Linting or formatting issues — use the project's linter
- Security audit — use a dedicated security skill
- Codebase with < 10 source files — manual review is faster
- Performance optimization — this skill targets structural debt, not runtime

---

## Pre-Scan Phase

Run this phase before spawning detection agents.

### 1. Detect Languages and Scope

Scan the repository root to determine:

```
Glob: "**/*.{ts,tsx,js,jsx,py,rs,go,templ,java,rb,cpp,c,h}"
```

Count files per language and total lines of code (for debt density calculation).
Record the top languages for agent configuration.

### 2. Apply Exclusions

Always exclude:

- `node_modules/`, `vendor/`, `target/`, `dist/`, `build/`, `.git/`
- Generated files (`*.generated.*`, `*.pb.go`, `*_generated.rs`, `*_templ.go`)
- Lock files, binaries, images, fonts
- Everything matched by `.gitignore`

### 3. Choose Execution Strategy

| File count (after exclusion) | Strategy                                      |
|------------------------------|-----------------------------------------------|
| < 20 files                   | Sequential — run all 5 detection passes inline (see below) |
| 20–2000 files                | Parallel — spawn 5 read-only detection agents |
| > 2000 files                 | Scoped — ask the user which directories to target, or auto-select the top 10 directories by `git log --since="6 months"` commit frequency |

### 4. Build Git Activity Index

Run `git log --since="6 months" --name-only --format=""` and count commits per
file. Store as a lookup table — detection agents use this for priority scoring.

### 5. Detect Monorepo Structure (optional)

Check for multiple app roots (multiple `go.mod`, `package.json`, `Cargo.toml`,
or `pyproject.toml` at different directory levels). If detected:

- Ask the user: **"Cross-app duplication: report or ignore?"**
  - **Report**: duplication between apps is flagged (useful for extracting
    shared libraries).
  - **Ignore**: only flag duplication *within* each app (default — cross-app
    duplication is often intentional).
- Pass the monorepo context and app list to each agent prompt.

### 6. Incremental Mode (optional — if user provides a reference)

If the user provides a reference (commit SHA, branch, date), restrict the
scan to files changed since that reference:

```
git diff --name-only {{since_ref}}..HEAD
```

Use this file list as the scan scope instead of the full codebase.
Useful for recurring audits: "what debt was added since last sprint?"

Note in the report header: **Mode: incremental (since {{since_ref}})**.

### 7. SQL Migration Scan (optional — if migration directories exist)

If the codebase contains migration directories (`migrations/`, `db/migrate/`,
`alembic/versions/`), include migration files in the scan:

- Migrations that create then immediately alter the same table
- Duplicate index definitions across migrations
- Down/rollback migrations referencing dropped tables/columns
- Empty or no-op migrations

Tag these findings as `category: "sql_migration"`.

---

## Detection Strategy

### Agent Orchestration

Spawn 5 **read-only** sub-agents in parallel (for codebases >= 20 files).
Each agent uses `Glob`, `Grep`, `Read`, and `LSP` (when available) — no edits.

| Agent                  | Mission                                  | Key tools             |
|------------------------|------------------------------------------|-----------------------|
| `duplicate-functions`  | Find functions with near-identical structure | Grep, Read, LSP      |
| `copypaste-blocks`     | Find repeated blocks of >= 10 lines       | Grep, Read            |
| `consolidation`        | Find similar implementations to merge     | Grep, Glob, Read, LSP |
| `dead-code`            | Find exported symbols with zero references | Grep, Glob, LSP      |
| `test-debt`            | Find debt in test code (duplication, dead helpers, missing coverage patterns) | Grep, Glob, Read |

Each agent returns a list of findings in the format defined in
**Findings Format** below. **Cap: 10 findings per agent**, ordered by
priority then by lines affected. If more exist, the agent notes how many
were omitted.

For concrete agent prompts with full instructions, see
[`references/agent-prompts.md`](references/agent-prompts.md).

For detailed detection patterns and language-specific heuristics, see
[`references/detection-heuristics.md`](references/detection-heuristics.md).

**Sequential mode (< 20 files):** do not spawn sub-agents. Run the same 5
detection passes inline, one after the other, in a single context. Follow the
same methodology from agent-prompts.md for each pass. Return findings in the
same JSON format and proceed to Result Assembly as normal.

### AST / LSP Strategy

When an LSP server is available for the project's language, prefer it over
Grep for higher-accuracy detection:

| Capability          | LSP method                          | Fallback      |
|---------------------|-------------------------------------|---------------|
| Symbol listing      | `textDocument/documentSymbol`       | Grep patterns |
| Find references     | `textDocument/references`           | Grep `\bName\b` |
| Go to definition    | `textDocument/definition`           | Grep patterns |
| Find implementations| `textDocument/implementation`       | Grep for interface/trait methods |
| Type information    | `textDocument/hover`                | Manual inspection |

**When to use LSP:** if the project has a working language server (`gopls`,
`rust-analyzer`, `typescript-language-server`, `pyright`), use LSP calls first.
Fall back to Grep only when LSP is unavailable or for languages without LSP
support.

**Templ files (`.templ`):** `templ lsp` provides `documentSymbol` and
`references` support. Use it when available, fall back to Grep with
templ-specific patterns (see detection-heuristics.md section on Go/Templ).
Less mature than `gopls` — expect gaps on cross-file references.

**LSP fallback:** if an LSP call returns zero results for a file that clearly
contains symbols (e.g., a `.go` file with exported functions), the server may
be misconfigured or crashed. Fall back to Grep for that file and continue.
Do not skip the file entirely.

### 1. Duplicate Functions

Find function/method pairs with the same structure — identical control flow,
same operations, same external calls — differing only in names, literals, and
comments. Use LSP `documentSymbol` or Grep to list candidates, then Read and
compare bodies structurally.

### 2. Copy-Pasted Blocks

Find blocks of >= 10 consecutive lines that appear nearly identically in 2+
locations. Use Grep to find repetitive patterns, Read 30 lines of surrounding
context, compare visually.

### 3. Consolidation Opportunities

Find similar implementations that could share a common abstraction: parallel
files across directories, functions with common name prefixes, files importing
the same set of modules.

### 4. Dead Code

Find exported/public symbols with zero references outside their defining file.
Use LSP `references` (preferred) or Grep. Cross-reference with git log for
confidence scoring. Guard against false positives: framework entry points,
interface implementations, serialization fields, templ component references.

### 5. Test Debt

Scan all test files for: duplicate setups across files, copy-pasted test cases,
dead test helpers with zero references, orphaned tests referencing removed
production code, and test files that could be consolidated.

**Priority adjustments:** orphaned tests = always High, duplicate setups in
>= 3 files = High.

Each agent's full methodology, step-by-step instructions, and confidence
criteria are in [`references/agent-prompts.md`](references/agent-prompts.md).

---

## Result Assembly

After all agents return their findings, the orchestrator assembles the report.

### 1. Collect Agent Results

Gather the JSON arrays from all 5 agents (+ SQL migration extension if used).

### 2. Deduplicate Cross-Agent Findings

The same code block may be flagged by multiple agents (e.g., a duplicated
function found by both `duplicate-functions` and `copypaste-blocks`). Deduplicate:

1. Group findings by overlapping file + line ranges.
2. If two findings from different agents overlap by >= 50% of lines, merge
   them into a single finding. Keep the higher priority and higher confidence.
   List both categories (e.g., "Duplicate Functions / Copy-Pasted Block").
3. If findings overlap but have different suggested actions, keep both as
   separate findings with a cross-reference note.

### 3. Sort and Cap

1. Sort all findings: high > medium > low, then by lines_affected descending.
2. Apply the per-category cap of 10 findings. Note omitted counts.

### 4. Compute Aggregate Metrics

Calculate for the report header:

- **Total duplicate/dead lines** — sum of `lines_affected` across all findings.
- **Debt density** — `(total duplicate/dead lines) / (total lines of code) * 1000`
  expressed as "X debt lines per 1000 LOC". The orchestrator counts total LOC
  during the pre-scan phase (step 1) when globbing source files.
- **Top affected areas** — the 3 directories with the most findings.

### 5. Generate Report

Fill the template from [`references/report-template.md`](references/report-template.md)
with the deduplicated, sorted findings and aggregate metrics.

---

## Findings Format

### Priority Framework

Assign priority based on **two dimensions**: size and git activity.

**Size thresholds:**

| Metric                           | High    | Medium  | Low     |
|----------------------------------|---------|---------|---------|
| Duplicate/dead lines per finding | > 50    | 20–50   | < 20    |
| Locations affected               | >= 4    | 2–3     | 2       |

**Git activity modifier:**

- File modified >= 5 times in last 6 months: **boost priority one level**
  (low becomes medium, medium becomes high).
- File unmodified for 6+ months: no boost (but still valid for dead code).

Final priority = max(size-based, git-boosted).

### Confidence Score

Each finding also gets a confidence level, independent of priority:

| Confidence   | Meaning                                                     |
|--------------|-------------------------------------------------------------|
| **certain**  | Identical after ignoring names/comments. No ambiguity.       |
| **probable** | Same structure, minor differences. High likelihood of real debt. |
| **suspect**  | Similar shape but enough differences to need human review.   |

Use confidence to guide the action plan:
- **certain + high priority** → act immediately, no review needed.
- **probable + high priority** → review briefly, then act.
- **suspect + any priority** → human must validate before acting.

### Maintenance Cost

Score each finding on 4 dimensions:

| Dimension              | Description                                              |
|------------------------|----------------------------------------------------------|
| **Duplicated lines**   | Total lines of duplicate/dead code across all locations  |
| **Location count**     | Number of files/sites affected                           |
| **Coupling risk**      | Low = isolated utilities. Medium = shared business logic. High = cross-module dependencies |
| **Consolidation effort** | Trivial = extract to shared function. Moderate = needs interface design. Significant = requires architectural change |

### Example Finding

```
### [H1] Duplicate request validation logic

- **Category:** Duplicate Functions
- **Priority:** High
- **Confidence:** Certain
- **Locations:**
  - `src/api/users.ts:45-78` — `validateUserRequest()`
  - `src/api/orders.ts:32-67` — `validateOrderRequest()`
- **Duplicate lines:** 33
- **Git activity:** 12 commits in last 6 months (both files)
- **Maintenance cost:**
  - Duplicated lines: 66 (33 x 2 locations)
  - Locations: 2
  - Coupling risk: Medium — both call shared DB layer
  - Consolidation effort: Trivial — extract generic validateRequest(schema)
- **Suggested consolidation:**
  Extract a shared validateRequest(schema: ZodSchema) in src/api/validation.ts.
  Both handlers call it with their specific schema.
```

---

## Output Report

Produce a single markdown report following the template in
[`references/report-template.md`](references/report-template.md).

Structure:

1. **Header** — project name, date, scope, mode (full/incremental), languages,
   file count, **debt density** (debt lines per 1000 LOC).
2. **Summary table** — finding counts by category and priority. Include
   SQL migration debt if applicable.
3. **High priority findings** — full detail with code previews and confidence.
   Separate "certain" from "suspect" findings for easier triage.
4. **Medium priority findings** — full detail with code previews.
5. **Low priority findings** — summary (code preview optional).
6. **Omitted findings** — if any agent hit the 10-finding cap, note how many
   additional findings were omitted per category.
7. **Action plan** — grouped by effort:
   - Quick wins (< 1 hour) — prioritize certain + high findings
   - Planned refactors (1-4 hours)
   - Strategic improvements (> 4 hours)
   - Items needing human review (suspect confidence)
8. **Limitations** — state what this audit cannot detect.

---

## Limitations

Acknowledge these in every report:

- **AST coverage is partial** — LSP provides AST-level accuracy when available,
  but falls back to Grep + heuristics for languages without LSP support or for
  `.templ` files. Grep-only detection may miss structurally equivalent code
  with different formatting.
- **No semantic duplication** — two functions that do the same thing with
  different algorithms will not be flagged.
- **No cross-repository analysis** — only the current repository is scanned.
- **Dynamic dispatch blind spots** — metaprogramming, reflection, and runtime
  code generation may cause false negatives (dead code appears used) or false
  positives (used code appears dead).
- **Conditional compilation** — Rust `#[cfg()]`, Go build tags, and C/C++
  `#ifdef` blocks may hide code from detection.
- **Templ files** — `templ lsp` is available but less mature than `gopls`.
  Cross-file component references may have gaps. Grep fallback is used when
  LSP misses references.
- **Findings cap** — each agent reports at most 10 findings. Large codebases
  may have more debt than what appears in the report. Run scoped audits
  on specific directories for deeper coverage.
- **Monorepo cross-app duplication** — by default, duplication across apps
  in a monorepo is not flagged (often intentional). Enable explicitly if
  the goal is to extract shared libraries.

### Language-Specific Notes

Each language has particular quirks that affect detection accuracy. Consult
[`references/detection-heuristics.md`](references/detection-heuristics.md)
section 5 for details on JS/TS, Python, Rust, and Go considerations.
