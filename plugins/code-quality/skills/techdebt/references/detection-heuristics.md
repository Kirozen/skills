# Detection Heuristics

Reference for techdebt skill agents. Contains Grep patterns, comparison
techniques, and heuristics organized by detection category and language.

## Quick Reference (essential patterns)

Use this summary to avoid loading the full document into context.
Expand into detailed sections only when needed.

| Category | Key Grep pattern | What to Read |
|----------|-----------------|--------------|
| **Duplicate functions (JS/TS)** | `"^\\s*(export\\s+)?(async\\s+)?function\\s+\\w+"` | Both function bodies, compare structure |
| **Duplicate functions (Python)** | `"^\\s*def\\s+\\w+\\s*\\("` | Both function bodies |
| **Duplicate functions (Go)** | `"^func\\s+\\w+\\("` | Both function bodies |
| **Duplicate functions (Rust)** | `"^\\s*(pub)?\\s*(async\\s+)?fn\\s+\\w+"` | Both function bodies |
| **Duplicate functions (Templ)** | `"^templ\\s+\\w+\\("` | Both component bodies |
| **Copy-paste (Go err)** | `"if\\s+err\\s*!=\\s*nil\\s*\\{"` | 30 lines context |
| **Copy-paste (long if)** | `"if\\s*\\(.{30,}\\)\\s*\\{"` | 30 lines context |
| **Dead code (Go)** | `"^(func\|type\|var\|const)\\s+[A-Z]\\w+"` | Grep for `\\bName\\b` across all files + .templ |
| **Dead code (JS/TS)** | `"^export\\s+(const\|function\|class)\\s+\\w+"` | Grep for `\\bName\\b` + JSX `<Name` |
| **Test setups (Go)** | `"func TestMain\\("`, `"func setup\\("` | Compare setup blocks across test files |
| **Test setups (JS/TS)** | `"beforeEach\\(\|beforeAll\\("` | Compare setup blocks |
| **Templ components** | `"^templ\\s+[A-Z]\\w+\\("` | Check refs in .templ + .go files |

**Similarity judgment:** two functions are duplicates when they have the same
control flow, same operations, same external calls — differing only in names,
literals, and comments. See section 1 for details.

**LSP preferred for:** symbol listing (`documentSymbol`), reference finding
(`references`), implementation search (`implementation`). Available via:
`gopls`, `templ lsp`, `rust-analyzer`, `typescript-language-server`, `pyright`.

---

## 1. Duplicate Functions

### Strategy

1. Use LSP `textDocument/documentSymbol` to list all functions (preferred),
   or find declarations via Grep.
2. Group by approximate body size (line count ±20%) and parameter count.
   Focus on functions with 10+ lines.
3. For candidates in the same size bucket, Read both bodies and compare
   structurally (see "How to Compare" below).
4. Flag pairs where only names, literals, and comments differ.

### Grep Patterns by Language

#### JavaScript / TypeScript

```
# Named functions
pattern: "^\\s*(export\\s+)?(async\\s+)?function\\s+\\w+"

# Arrow functions assigned to const/let
pattern: "^\\s*(export\\s+)?(const|let)\\s+\\w+\\s*=\\s*(async\\s+)?\\("

# Class methods
pattern: "^\\s*(async\\s+)?\\w+\\s*\\([^)]*\\)\\s*\\{"
```

#### Python

```
# Function definitions
pattern: "^\\s*def\\s+\\w+\\s*\\("

# Class methods (same pattern, indentation >= 4)
pattern: "^\\s{4,}def\\s+\\w+\\s*\\("
```

#### Rust

```
# Free functions
pattern: "^\\s*(pub(\\s*\\(crate\\))?\\s+)?(async\\s+)?fn\\s+\\w+"

# Impl methods
pattern: "^\\s*(pub(\\s*\\(crate\\))?\\s+)?(async\\s+)?fn\\s+\\w+"
# (same pattern — distinguish by context: inside `impl` block)
```

#### Go

```
# Functions
pattern: "^func\\s+\\w+\\("

# Methods (receiver)
pattern: "^func\\s+\\(\\w+\\s+\\*?\\w+\\)\\s+\\w+\\("
```

#### Go Templ (`.templ` files)

```
# Templ component definitions
pattern: "^templ\\s+\\w+\\("

# Templ component methods (with receiver)
pattern: "^templ\\s+\\(\\w+\\s+\\*?\\w+\\)\\s+\\w+\\("

# Go functions embedded in templ files
pattern: "^func\\s+\\w+\\("

# CSS/script blocks (can contain duplication)
pattern: "^(css|script)\\s+\\w+\\("
```

**Note:** `.templ` files mix Go code and HTML-like template syntax.
Treat the Go portions (func, templ, css, script blocks) as code for
duplication detection. Treat the HTML template portions as copy-paste
block candidates.

### How to Compare Function Bodies (LLM Agent)

When reading two candidate function bodies, mentally ignore:

1. **Comments** — skip single-line (`//`, `#`) and multi-line (`/* */`)
2. **Whitespace differences** — indentation style is irrelevant
3. **Variable names** — focus on what the variables *do*, not what they're called
4. **String/numeric literals** — treat all literal values as equivalent

Then assess structural similarity:

- **Same control flow?** — if/else, loops, switch/match in the same order
- **Same operations?** — same methods called, same operators, same transformations
- **Same external calls?** — same functions/APIs invoked in the same sequence

### Similarity Judgment

| Verdict     | Criteria |
|-------------|----------|
| **Duplicate** (flag it) | Same structure, same operations. Only names, literals, and comments differ. |
| **Near-duplicate** (flag it) | Same structure, minor logic differences (extra guard clause, different default, one extra field). |
| **Similar but distinct** (don't flag) | Same general pattern, but meaningfully different logic branches or operations. |

---

## 2. Copy-Pasted Blocks

### Strategy

1. Use Grep to find structurally repetitive patterns (see pre-screening
   patterns below).
2. When a pattern appears in multiple files, Read 30 lines of surrounding
   context from each location.
3. Compare the blocks visually (see "Block Comparison" below). Flag
   sequences of >= 10 lines that are identical or differ only in
   names/values, appearing in 2+ locations.
4. Merge overlapping findings into contiguous blocks.

### Grep Patterns for Pre-Screening

Use distinctive structural patterns to find likely copy-paste candidates:

```
# Repeated multi-line conditionals (JS/TS/Go/Rust)
pattern: "if\\s*\\(.{30,}\\)\\s*\\{"

# Repeated error handling blocks (Go)
pattern: "if\\s+err\\s*!=\\s*nil\\s*\\{"

# Repeated try/except blocks (Python)
pattern: "^\\s*try:\\s*$"

# Repeated match arms with similar structure (Rust)
pattern: "=>\\s*\\{"
```

### Block Comparison (LLM Agent)

After finding candidate blocks via Grep pre-screening:

1. Read 30 lines of context around each match.
2. Visually compare the blocks — are they structurally identical?
3. Differences limited to names and literal values = copy-paste.
4. Merge overlapping matches in the same file into one finding.

### Minimum Thresholds

| Parameter         | Default | Notes                          |
|-------------------|---------|--------------------------------|
| Window size       | 10      | Lines per window               |
| Min block size    | 10      | Minimum lines to report        |
| Min occurrences   | 2       | Locations required             |
| Scan test files   | yes     | Include `*_test.*`, `test_*`, `*.spec.*` — report separately |

---

## 3. Consolidation Opportunities

### Strategy

1. Find functions/classes with similar names (edit distance, prefix/suffix match).
2. Find implementations that import the same set of dependencies.
3. Find parallel structures: files with matching names in different directories
   (e.g., `utils/format.ts` and `helpers/format.ts`).
4. Flag groups where consolidation into a shared module is feasible.

### Grep Patterns

```
# Similar function names — search for common prefixes
# (agent builds prefix list dynamically from function index)

# Parallel file structures
# Use Glob: "**/utils/*.{ts,py,rs,go,templ}" and "**/helpers/*.{ts,py,rs,go,templ}"

# Duplicate imports — same module imported in many files
# Agent counts import frequency with:
pattern: "^(import|from|use|require)\\s+"
# then groups by imported module name

# Parallel templ components — similar component names in different directories
# Use Glob: "**/*.templ" and compare component signatures
```

### Heuristics

- **Name similarity**: functions sharing a 4+ character prefix and same parameter
  count are candidates.
- **Import overlap**: two files importing >= 5 identical modules often contain
  consolidation opportunities.
- **Parallel directories**: `utils/`, `helpers/`, `lib/`, `common/`, `shared/` —
  scan all for overlapping filenames.

---

## 4. Dead Code

### Strategy

1. Find all exported symbols (functions, classes, constants, types).
2. Search for references to each symbol across the codebase.
3. Flag symbols with zero references outside their defining file.
4. Cross-reference with git log: symbols untouched for 6+ months get higher
   confidence.

### Grep Patterns by Language

#### JavaScript / TypeScript

```
# Exported symbols
pattern: "^export\\s+(const|let|function|class|type|interface|enum)\\s+(\\w+)"

# Re-exports
pattern: "^export\\s+\\{[^}]+\\}\\s+from"

# Check references (for symbol "Foo"):
pattern: "\\bFoo\\b"
# Exclude: the definition file itself, import statements
```

#### Python

```
# Module-level definitions (potential exports)
pattern: "^(def|class|\\w+\\s*=)\\s+\\w+"

# __all__ declarations
pattern: "^__all__\\s*="

# Check references:
pattern: "\\bFoo\\b"
# Exclude: the definition file, `import Foo` lines (count usage, not import)
```

#### Rust

```
# Public items
pattern: "^\\s*pub\\s+(fn|struct|enum|trait|type|const|static|mod)\\s+(\\w+)"

# Check references:
pattern: "\\bFoo\\b"
# Exclude: the definition file, `use` statements
```

#### Go

```
# Exported symbols (capitalized)
pattern: "^(func|type|var|const)\\s+[A-Z]\\w+"

# Check references (use LSP `textDocument/references` if gopls available):
pattern: "\\bFoo\\b"
# Exclude: definition file

# IMPORTANT: also search .templ files for references — a Go function
# used only in templ templates is NOT dead code:
# Grep .templ files for: "\\bFoo\\b" (component calls, function calls in @{})
```

#### Go Templ (`.templ` files)

```
# Templ component definitions (public = capitalized)
pattern: "^templ\\s+[A-Z]\\w+\\("

# Check references in other .templ files and .go files:
pattern: "\\bFoo\\b"
# Also check for: @Foo( and { Foo( patterns in templ templates
pattern: "[@{]\\s*Foo\\s*\\("
```

### False Positive Mitigation

- **Framework entry points**: skip `main`, `init`, `setup`, handler functions
  registered via decorators/macros.
- **Interface implementations**: a method may have zero direct callers but
  satisfy an interface — check for trait/interface matching.
- **Serialization**: fields used only via reflection/serialization (struct tags
  in Go, `#[serde]` in Rust, `@dataclass` in Python).
- **Test helpers**: functions used only in test files are not dead if test files
  are in scope.
- **Dynamic dispatch**: JS/Python code using `getattr`, bracket notation, or
  `eval` — lower confidence for these languages.

---

## 5. Test Debt

### Test File Identification

```
# Go
glob: "**/*_test.go"

# JS/TS
glob: "**/*.{spec,test}.{ts,tsx,js,jsx}"
glob: "**/__tests__/**/*.{ts,tsx,js,jsx}"

# Python
glob: "**/test_*.py"
glob: "**/*_test.py"
glob: "**/tests/**/*.py"

# Rust
# Tests are inline (mod tests) — use Grep:
pattern: "#\\[cfg\\(test\\)\\]"
pattern: "#\\[test\\]"
```

### Duplicate Test Setups

Look for repeated setup/teardown blocks across test files:

```
# Go
pattern: "func TestMain\\(m \\*testing\\.M\\)"
pattern: "func setup\\("
# Also: repeated t.Helper() functions with similar bodies

# JS/TS
pattern: "beforeEach\\(|beforeAll\\(|afterEach\\(|afterAll\\("

# Python
pattern: "def setUp\\(self\\)|def setUpClass\\(cls\\)|@pytest\\.fixture"
```

Flag identical setup blocks in >= 2 test files for extraction into shared
test fixtures or helpers.

### Copy-Pasted Test Cases

Apply the same copy-paste block detection (section 2) to test files:
- Lower the window size to 8 lines (test cases tend to be shorter).
- Watch for table-driven tests that were copy-pasted instead of parameterized.
- Go: detect repeated `t.Run(` blocks with near-identical bodies.
- JS/TS: detect repeated `it(` / `test(` blocks.
- Python: detect repeated `def test_` functions.

### Dead Test Helpers

```
# Find test-only helper functions, then check references:

# Go — unexported functions in _test.go files
pattern: "^func [a-z]\\w+\\("
# in files matching: *_test.go

# JS/TS — non-exported functions in test files
pattern: "^(const|function)\\s+\\w+"
# in files matching: *.spec.*, *.test.*

# Python — functions in test files not starting with test_
pattern: "^def (?!test_)\\w+\\("
# in files matching: test_*.py, *_test.py
```

### Orphaned Tests

Tests that reference production functions/types that no longer exist:

1. Extract all identifiers referenced in test files.
2. For each identifier, check if it exists in production code.
3. Flag tests referencing symbols that are missing — these tests are either
   broken or testing dead code.

---

## 6. Language-Specific Considerations

### JavaScript / TypeScript

- **Module systems**: handle both `import/export` (ESM) and `require/module.exports` (CJS).
- **Re-exports**: barrel files (`index.ts`) re-exporting from sub-modules — follow the chain.
- **JSX**: component usage looks like `<Component />`, not function calls — include
  pattern `<\\w+[\\s/>]` when searching for references.
- **Type-only exports**: `export type { Foo }` — dead type exports are lower priority.

### Python

- **`__init__.py`**: symbols imported here are effectively public.
- **Dynamic imports**: `importlib.import_module()` — lower confidence for dead code detection.
- **Decorators**: `@app.route`, `@pytest.fixture` etc. register functions implicitly.
- **Star imports**: `from module import *` makes tracking harder — flag as limitation.

### Rust

- **Macros**: `macro_rules!` and proc macros generate code invisible to Grep — flag as
  limitation.
- **Feature gates**: `#[cfg(feature = "...")]` may hide code — note conditional compilation.
- **Derive macros**: `#[derive(Serialize)]` generates trait impls — don't flag derived code.

### Go

- **Interface satisfaction**: a function may exist only to satisfy an interface.
  Use LSP `textDocument/implementation` via `gopls` when available.
- **`init()` functions**: always called implicitly — never flag as dead.
- **Build tags**: `//go:build` can hide code — note conditional compilation.
- **Generated code**: files with `// Code generated` header — skip entirely.
- **`_templ.go` files**: auto-generated by the `templ` CLI — always exclude
  from scanning (treat like generated code).

### Go Templ (`.templ` files)

- **Mixed syntax**: `.templ` files contain Go code blocks (`func`, `templ`,
  `css`, `script`) and HTML-like template blocks. Apply Go heuristics to
  code blocks and copy-paste detection to template blocks.
- **Component references**: templ components are called like `@ComponentName()`
  or `{ ComponentName() }` in templates. Include these patterns when
  checking for dead code.
- **CSS/script duplication**: `css` and `script` blocks are prime candidates
  for copy-paste — scan for identical or near-identical blocks.
- **LSP available**: `templ lsp` supports `documentSymbol` and `references`.
  Use it when available — less mature than `gopls`, so verify with Grep
  for cross-file references.
- **Cross-file references**: a templ component defined in `header.templ` may
  be referenced in `page.templ` — scan all `.templ` files and `.go` files
  for references.

---

## 7. AST / LSP Usage

When a language server is available, prefer LSP over Grep for these operations:

| Operation              | LSP method                        | Languages            |
|------------------------|-----------------------------------|----------------------|
| List all symbols       | `textDocument/documentSymbol`     | Go, Rust, TS, Python |
| Find all references    | `textDocument/references`         | Go, Rust, TS, Python |
| Go to definition       | `textDocument/definition`         | Go, Rust, TS, Python |
| Find implementations   | `textDocument/implementation`     | Go, Rust, TS         |
| Type info              | `textDocument/hover`              | All with LSP         |

**LSP servers by language:**
- Go: `gopls`
- Go Templ: `templ lsp` (less mature — verify cross-file refs with Grep)
- Rust: `rust-analyzer`
- TypeScript/JavaScript: `typescript-language-server`
- Python: `pyright` or `pylsp`

**Limited LSP:** `.templ` files have `templ lsp` (supports `documentSymbol`,
`references`) but it is less mature. Verify cross-file references with Grep.

**No LSP available for:** shell scripts, config files, SQL migrations.
Always fall back to Grep for these.

**Hybrid approach:** use LSP for dead code detection (high accuracy for
references) and Grep for copy-paste / duplication (LSP has no equivalent).

---

## 8. Scoping Heuristics

### File Count Thresholds

| Codebase size   | Strategy                                                  |
|-----------------|-----------------------------------------------------------|
| < 20 files      | Sequential scan, single agent                             |
| 20–200 files    | Parallel agents, full scan                                |
| 200–2000 files  | Parallel agents, prioritize `src/`, `lib/`, `pkg/`, `app/`|
| > 2000 files    | Restrict to user-specified directories or hot paths (git log) |

### Always Exclude

- `node_modules/`, `vendor/`, `target/`, `dist/`, `build/`, `.git/`
- Generated files (`*.generated.*`, `*.pb.go`, `*_generated.rs`, `*_templ.go`)
- Lock files (`package-lock.json`, `Cargo.lock`, `poetry.lock`)
- Binary files, images, fonts
- Files in `.gitignore`

### Priority by Git Activity

Use `git log --since="6 months" --name-only` to identify hot files. Findings
in frequently modified files get a priority boost (see Priority Framework in
SKILL.md).
