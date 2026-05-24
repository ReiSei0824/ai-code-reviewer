---
name: ai-code-reviewer
description: Detect AI-specific code defects in diffs and pull requests. Target: hallucinated APIs, pattern inconsistencies with the existing codebase, copy-paste variations needing abstraction, missing error/null/edge-case handling, over-confident implementations with subtle logic flaws, missing tests, TODO/placeholder abandonment, type system abuse (any, as, @ts-ignore), security-naive patterns, and duplicate/redundant logic. Trigger on requests to review a diff, PR, branch, or set of files with explicit mention of AI-generated code characteristics: "review this PR for AI artifacts", "check if this was AI-generated", "find AI code issues", "audit this diff for hallucinated APIs", "does this look AI-written", "AI code review", "review changes for AI patterns", "check for copy-paste duplication", "find missing error handling in AI code", "validate this PR's type safety", "audit for placeholder TODOs". Does NOT replace general code review or linting — this skill focuses exclusively on failure modes disproportionately common in AI-generated code.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git *)
  - Bash(diff *)
  - WebSearch
---

# AI Code Reviewer

This skill detects defect patterns that are **characteristic of AI-generated code** — issues that appear far more often in AI-written code than in human-written code. It complements (does not replace) general code review, linters, and type checkers.

**What this skill detects**: Hallucinated API calls, existing codebase pattern violations, copy-paste-with-variation blocks, missing error/null/edge-case handling, over-confident implementations with subtle logical flaws, missing test files, placeholder/TODO abandonment, type-system abuse, security-naive patterns, and reimplementation of existing project utilities or standard library functions.

**What this skill does NOT do**: General bug finding, style nits, CLAUDE.md compliance, formatting issues, performance optimization, or architectural recommendations. Those are handled by linters, type checkers, and general code review.

---

## Input Handling

Accept input in one of three forms. Determine which form the user provided, then normalize it.

### Form A: Raw diff string

When the user pastes a diff or provides it via `git diff`, use it directly. The diff may come from:
- A direct paste in the conversation
- `git diff` output (uncommitted changes)
- `git diff <branch1>...<branch2>` (between branches)

If the pasted diff is extremely long (over 500 lines), ask the user if they can scope it to specific files.

### Form B: File paths

When the user provides specific file paths (e.g., "review src/feature.ts and src/utils.ts"):
1. Read each file in full to understand the complete code.
2. Run `git diff -- <file>` for each file to isolate changes vs the committed version.

### Form C: Branch or PR reference

When the user provides a branch name, commit range, or PR number:
1. Determine the base. Default: the repo's default branch (`main` or `master`). If the user says "review against staging", use `staging`.
2. Run `git merge-base <base> <head>` to find the fork point.
3. Run `git diff <merge-base>...<head>` to get the complete diff.
4. Run `git diff --name-only <merge-base>...<head>` to list changed files.

After collecting the input, compute a quick summary: total files changed, total lines added/deleted, and languages detected (by file extension). State this summary before beginning analysis.

---

## Phase 1: Load Codebase Context

Before analyzing the diff, build a context model of the existing project. This context powers nearly every detection category.

### Step 1.1: Project metadata

Read the project's dependency manifest to know what libraries are available:
- Node.js: `package.json` (dependencies + devDependencies)
- Python: `pyproject.toml`, `requirements.txt`, or `Pipfile`
- Rust: `Cargo.toml`
- Go: `go.mod`
- Ruby: `Gemfile`
- Java/Kotlin: `build.gradle` or `pom.xml`

Also read language configuration if present: `tsconfig.json`, `.eslintrc.*`, `biome.json`, `rustfmt.toml`, `.rubocop.yml`.

### Step 1.2: Naming and structural conventions

Read 5-10 existing source files from outside the diff (from different areas of the project) to extract conventions:
- **Naming**: camelCase, snake_case, PascalCase, kebab-case for functions, variables, classes, files
- **Error handling style**: Result/Either types, try-catch, thrown exceptions, error codes, Go-style `(val, err)`, Option types
- **Import/require style**: named imports, default imports, barrel exports, path aliases
- **File organization**: one class per file, grouped by feature, grouped by layer (controllers/services/models)
- **Test file naming and location**: `*.test.ts`, `*_test.go`, `test_*.py`, `__tests__/`, `spec/`, `*.spec.ts`

### Step 1.3: Existing utilities inventory

Run focused Grep searches to find:
- Utility/helper directories: `lib/`, `utils/`, `helpers/`, `common/`, `shared/`
- Shared type definitions
- Common patterns: the project's HTTP client, ORM, validation library, logging framework, testing framework

This prevents the redundant-logic detector from flagging code that should instead use an existing utility.

### Step 1.4: Existing test patterns

Read 2-3 existing test files to understand:
- Test framework (Jest, Mocha, pytest, RSpec, Go testing, JUnit)
- Test structure (describe/it, test functions, table-driven tests)
- Mock/fixture conventions
- Where tests live relative to source files

---

## Phase 2: Detect AI-Specific Defects

Run each of the 10 detection passes. For detailed pattern catalogs with language-specific examples, read the corresponding reference file at `<skill-dir>/references/<category>.md` before running that pass.

### Category 1: Hallucinated APIs

**Reference**: `<skill-dir>/references/hallucinated-apis.md`

Core algorithm: For every import, function call, method invocation, and class instantiation in the diff that references an external symbol, verify it exists. Read the reference file first for detailed detection patterns.

Key checks:
1. For each import added in the diff, check if the module/package is in the project's declared dependencies.
2. For each function/method call, Grep the project source tree for its definition or export. If not found, check the dependency's actual API surface (WebSearch if needed).
3. Pay special attention to function names that sound "too perfect" — overly descriptive or generic names like `handleAllErrors()`, `validateUserInputComprehensively()` are common AI hallucinations.
4. Watch for near-miss APIs: `array.flatten()` instead of `array.flat()`, `str.contains()` instead of `str.includes()`, `os.path.join_path()` instead of `os.path.join()`.

### Category 2: Pattern Inconsistency

**Reference**: `<skill-dir>/references/pattern-inconsistency.md`

Compare the diff's coding patterns against the conventions extracted in Phase 1.2. Flag mismatches in naming, error handling, async style, file organization, and import conventions.

### Category 3: Copy-Paste Variations

**Reference**: `<skill-dir>/references/copy-paste-variations.md`

Within the diff itself, identify blocks of code (5+ lines) that share structural similarity. Compute what varies between them (variable names, types, literal values). Flag groups that would be better extracted as a parameterized function, method, or generic.

Also cross-reference against the codebase: check if similar logic already exists elsewhere and the diff just reimplements it with minor tweaks.

### Category 4: Missing Error/Null/Edge-Case Handling

**Reference**: `<skill-dir>/references/missing-edge-cases.md`

For each function or method added in the diff, check:
- Null/undefined/unwrap without guard — does it use a nullable value without checking?
- Empty collections — does it iterate without an emptiness check where one matters?
- IO operations — does it make HTTP calls, DB queries, or file reads without error handling?
- API response shape — does it access `.data` or `.body` without checking response status?
- Boundary conditions — off-by-one loops, fencepost errors, missing min/max guards

### Category 5: Over-Confident Implementations

**Reference**: `<skill-dir>/references/logical-flaws.md`

These are syntactically valid constructs that contain subtle logic errors AI commonly produces:
- Wrong operator (`=` vs `==`, `||` vs `&&`)
- Shadowed variables in inner scopes
- Dead code after return/throw/break/continue
- Mutable default arguments (Python, JS patterns)
- Incorrect comparison direction
- Misordered function arguments
- State machines that can reach invalid states

### Category 6: Missing Tests

**Reference**: `<skill-dir>/references/missing-tests.md`

For each new file or significantly modified file in the diff:
1. Check if a corresponding test file exists (following the project's convention from Phase 1.4).
2. For each new function/method, check if a test exercises it.
3. If the diff includes a bug fix (identifiable by issue references), check for a regression test.

Skip: test-only files, configuration files, generated code, type definition files.

### Category 7: TODO/Placeholder Abandonment

**Reference**: `<skill-dir>/references/placeholder-abandonment.md`

Search the added lines in the diff for:
- `TODO`, `FIXME`, `HACK`, `XXX`, `WORKAROUND` comments
- Stub function bodies: `throw new Error("Not implemented")`, `pass`, `return null`, `unimplemented!()`
- Placeholder strings: "lorem ipsum", "placeholder", "example.com", "your-api-key-here", "changeme", "test"
- Empty catch blocks: `catch (e) {}` or `catch { }`
- Commented-out code blocks (5+ consecutive commented lines)
- Debugging leftovers: bare `console.log`, `print()`, `dbg!()`, `println!()` without a logging framework wrapper

### Category 8: Type System Abuse

**Reference**: `<skill-dir>/references/type-system-abuse.md`

Search the diff for type-escaping patterns:
- `any` (TypeScript), `Object` (Java raw type), `interface{}` (Go empty interface)
- Type casts/assertions: `as Type`, `(Type) value`, `value.(Type)`
- Suppression comments: `// @ts-ignore`, `// @ts-expect-error`, `# type: ignore`, `#[allow(...)]`
- Non-null assertions: `value!.property`
- File-level suppressions: `// @ts-nocheck`, `#nullable disable`
- Unsafe `eval()` or `Function()` constructor usage

Flag more aggressively when there are many suppressions — this is a strong "make it compile at any cost" AI signal.

### Category 9: Security-Naive Patterns

**Reference**: `<skill-dir>/references/security-naive-patterns.md`

Search the diff for:
- Hardcoded credentials: API keys, tokens, passwords, connection strings, private keys in source code
- SQL injection: string concatenation or f-strings in SQL queries instead of parameterized queries
- Command injection: `exec()`, `os.system()`, `subprocess.Popen()` with unsanitized user input
- Path traversal: user input concatenated into file paths without `path.join` or normalization
- Insecure deserialization: `pickle.loads()`, `yaml.load()` without SafeLoader, `eval()` on JSON-like strings
- Weak crypto: custom encryption, MD5/SHA1 for security, ECB mode, hardcoded IVs/salts
- XSS patterns: `innerHTML`, `dangerouslySetInnerHTML`, `v-html` without sanitization

### Category 10: Duplicate/Redundant Logic

**Reference**: `<skill-dir>/references/redundant-logic.md`

Check if the diff reimplements:
1. Functions already in the project's utility modules (from Phase 1.3)
2. Standard library functions (`lodash`/`underscore`, Python `itertools`/`functools`, Java `Stream` API, Go `slices`/`maps`)
3. Types or interfaces already defined in the project's shared types

Common AI reimplementation signals: functions with suspiciously generic names like `flattenArray`, `deepCopy`, `retryWithBackoff`, `formatDate`.

---

## Phase 3: Cross-Reference and Deduplicate

After all 10 detection passes, consolidate findings:

1. **Merge overlapping findings**: If the same line triggers both "missing null check" and "missing error handling", combine them into one finding tagged with both categories.
2. **Apply exemptions**: If the diff consists entirely of test files, skip Categories 4, 5, 9. If the diff is configuration files (`.yml`, `.json`, `.toml`, `.env`), skip Categories 1, 3, 5, 6, 8, 10.
3. **Cross-file grouping**: If the same hallucinated function is called in three different files, report it once with all three locations listed.
4. **Remove false positives**: If a "pattern inconsistency" is explained by intentional separation (e.g., the diff touches a different architectural layer with different conventions), downgrade or remove.

---

## Phase 4: Generate Structured Report

Produce the final review report following the exact template in `<skill-dir>/references/output-template.md`.

Classify every finding's severity using the criteria in `<skill-dir>/references/severity-guide.md`.

### Report structure

1. **Summary**: Total findings table by severity, files affected count
2. **Findings**: Each finding with severity badge, category label, file:line, description, why-it-matters, and fix suggestion. Ordered: CRITICAL → HIGH → MEDIUM → LOW.
3. **Refactoring Opportunities** (if any Category 3 or 10 findings): Table showing duplication locations and suggested abstractions
4. **Review Notes**: Scope disclaimer and exemptions applied

If zero findings survive deduplication, output: "No AI-specific defects detected in this diff."

---

## Guardrails

1. **Only flag added/modified lines**: Do not report issues in lines that were already in the codebase before this diff.
2. **Do not duplicate linters/type-checkers**: Skip findings that `eslint`, `mypy`, `rustc`, `golangci-lint`, or equivalent would catch. Assume CI runs these.
3. **Skip generated files**: Ignore lock files (`package-lock.json`, `Cargo.lock`), compiled output (`dist/`, `build/`), protobuf stubs, OpenAPI generated clients, and minified bundles.
4. **Configuration files get a pass on secrets**: `.env.example`, `docker-compose.yml`, CI configs commonly have placeholder values by design. Only flag hardcoded secrets in application source code.
5. **Provide evidence for every finding**: Cite the specific line(s), state the rule violated, explain the reasoning. No finding should say merely "this looks wrong."
6. **One root cause = one finding**: If the same issue manifests in 5 lines, report 1 finding with 5 locations.
7. **Run analysis directly**: Use Read, Grep, Glob, and Bash tools. Do not ask the user to run linters or other tools.
8. **Respect `.gitignore`**: Do not read or grep files that match gitignore patterns.
