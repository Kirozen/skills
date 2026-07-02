# DVA — Detailed Guide

## Tooling by category

### Automated tests
| Tool | Use |
|------|-----|
| pytest / go test | Unit and integration tests |
| Selenium / Playwright | End-to-end browser tests |
| k6 / Locust | Load and performance tests |
| Fuzzing (go-fuzz, AFL) | Discover unexpected inputs |

### Static analysis
| Tool | Use |
|------|-----|
| golangci-lint / ESLint | Linting and anti-patterns |
| Semgrep | Custom security patterns |
| gosec | Go-specific security |
| Trivy / Snyk | Dependency vulnerability scanning |

## Feedback-sheet template

```markdown
## DVA Feedback — [Component name]

**Date**: YYYY-MM-DD
**Engineer**: [name]
**Antagonist**: [name or "Claude"]

### Scope
- Problem: ...
- Constraints: ...
- Success criteria: ...

### Flaws found

| # | Category | Flaw | Impact | Evidence | Proposed fix | Severity | Status |
|---|----------|------|--------|----------|--------------|----------|--------|
| 1 | Security | ... | ... | ... | ... | Critical | Fixed |
| 2 | Performance | ... | ... | ... | ... | Major | In progress |
| 3 | Functional | ... | ... | ... | ... | Minor | To do |

### Validated strengths
- [what is well-designed and why]

### Final decision
- [ ] PASS
- [ ] PASS WITH WARNINGS
- [ ] FAIL — remaining blockers: ...
```

## Worked examples

### Example 1: API endpoint review

**Engineer**: "New POST /api/products endpoint that accepts JSON, validates the fields, and inserts into the database."

**Antagonist**:
| # | Category | Flaw | Severity |
|---|----------|------|----------|
| 1 | Security | No rate limiting — vulnerable to brute force | Critical |
| 2 | Functional | Body > 10MB not rejected — OOM possible | Major |
| 3 | Performance | INSERT without batching — N+1 on bulk import | Major |
| 4 | Reliability | No timeout on the DB connection | Minor |

### Example 2: architecture review

**Engineer**: "Migrating the monolith to 3 microservices communicating via events."

**Antagonist**:
| # | Category | Flaw | Severity |
|---|----------|------|----------|
| 1 | Reliability | No dead letter queue — events silently lost | Critical |
| 2 | Performance | JSON serialization between services — overhead on hot path | Major |
| 3 | Maintainability | Event schema unversioned — breaking changes invisible | Major |
