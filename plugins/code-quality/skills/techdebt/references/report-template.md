# Tech Debt Audit Report Template

Use this template to structure the final output report.
Replace all `{{placeholders}}` with actual values.

---

```markdown
# Tech Debt Audit Report

**Project:** {{project_name}}
**Date:** {{date}}
**Scope:** {{scoped_directories_or_full_repo}}
**Mode:** {{full | incremental (since {{since_ref}})}}
**Languages detected:** {{languages}}
**Files scanned:** {{file_count}}

---

## Summary

| Category                    | Findings | High | Medium | Low |
|-----------------------------|----------|------|--------|-----|
| Duplicate Functions         | {{n}}    | {{n}}| {{n}}  | {{n}}|
| Copy-Pasted Blocks          | {{n}}    | {{n}}| {{n}}  | {{n}}|
| Consolidation Opportunities | {{n}}    | {{n}}| {{n}}  | {{n}}|
| Dead Code                   | {{n}}    | {{n}}| {{n}}  | {{n}}|
| Test Debt                   | {{n}}    | {{n}}| {{n}}  | {{n}}|
| SQL Migration Debt          | {{n}}    | {{n}}| {{n}}  | {{n}}|
| **Total**                   | **{{n}}**| **{{n}}**| **{{n}}**| **{{n}}**|

**Estimated duplicate/dead lines:** {{total_lines}}
**Debt density:** {{debt_density}} debt lines per 1000 LOC
**Top affected areas:** {{dir_1}}, {{dir_2}}, {{dir_3}}

---

## High Priority Findings

### [H1] {{finding_title}}

- **Category:** {{duplicate_functions | copypaste_blocks | consolidation | dead_code | test_debt | sql_migration}}
- **Priority:** High
- **Confidence:** {{certain | probable | suspect}}
- **Locations:**
  - `{{file_path_1}}:{{start_line}}-{{end_line}}`
  - `{{file_path_2}}:{{start_line}}-{{end_line}}`
- **Duplicate/dead lines:** {{line_count}}
- **Git activity:** {{commits_in_last_6_months}} commits in last 6 months
- **Maintenance cost:**
  - Duplicated lines: {{n}}
  - Locations: {{n}}
  - Coupling risk: {{low | medium | high}}
  - Consolidation effort: {{trivial | moderate | significant}}
- **Suggested consolidation:**
  {{brief_description_of_how_to_consolidate}}
- **Code preview:**
  ```{{language}}
  {{representative_code_snippet_max_15_lines}}
  ```

---

## Medium Priority Findings

### [M1] {{finding_title}}

{{same structure as high priority}}

---

## Low Priority Findings

### [L1] {{finding_title}}

{{same structure as high priority — code preview optional}}

---

## Omitted Findings

{{#if any_agent_hit_cap}}
Some agents found more than 10 findings. Additional findings not detailed:

| Category                    | Omitted |
|-----------------------------|---------|
| {{category}}                | {{n}}   |

Run a scoped audit on specific directories for deeper coverage.
{{else}}
All findings are included in this report.
{{/if}}

---

## Action Plan

### Quick Wins (< 1 hour each) — certain + high priority

1. {{finding_ref}} — {{one_line_action}}
2. {{finding_ref}} — {{one_line_action}}

### Planned Refactors (1-4 hours each)

1. {{finding_ref}} — {{one_line_action}}
2. {{finding_ref}} — {{one_line_action}}

### Strategic Improvements (> 4 hours)

1. {{finding_ref}} — {{one_line_action}}

### Needs Human Review (suspect confidence)

1. {{finding_ref}} — {{reason_for_uncertainty}}
2. {{finding_ref}} — {{reason_for_uncertainty}}

---

## Limitations

- Detection uses LSP/AST when available, falls back to Grep + heuristics otherwise.
- Semantic duplication (same logic, different syntax) is not detected.
- Dynamic dispatch and metaprogramming may cause false negatives in dead code.
- Cross-repository duplication is out of scope.
- `.templ` files: templ LSP available but less mature than gopls — some gaps.
- Findings capped at 10 per category — additional findings noted but not detailed.
- {{additional_project_specific_limitations}}
```
