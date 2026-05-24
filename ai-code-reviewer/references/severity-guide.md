# Severity Classification Guide

Each finding must be assigned one of four severity levels. Apply the highest applicable tier.

## Tier Summary

| Severity | Meaning | Action |
|----------|---------|--------|
| **CRITICAL** | Will crash at runtime, produce wrong results, or is a security vulnerability | Must fix before merge |
| **HIGH** | Works in happy path but fails under specific, common conditions | Should fix before merge |
| **MEDIUM** | Works correctly but has maintainability or edge-case concerns | Should fix in follow-up |
| **LOW** | Functional but shows AI-generation signals worth noting | Optional cleanup |

## CRITICAL

The code will definitely fail at runtime, produce incorrect results, or introduce a security vulnerability.

**Triggers**:
- **Hallucinated API call**: Calling a function/method/class that does not exist in the project's dependencies or codebase. Will cause a runtime error on first execution.
- **Security vulnerability with exploit potential**: SQL injection (string-built queries), command injection (unsanitized shell execution), hardcoded production credentials in source code, unsafe deserialization of untrusted input.
- **Definitely wrong logic**: Assignment in condition (`if (x = y)`), inverted comparison direction that reverses the intended behavior, dead code after unconditional return/throw that hides important logic.
- **Missing error handling on IO where failure is guaranteed in production**: Network call, database query, or file read in a path that every request hits, with zero error handling.

**Example**: A function that calls `await db.getUser(id)` without try-catch, and the caller passes user-supplied IDs that frequently don't exist. The first missing record will crash the request handler.

**Example**: `const apiKey = "sk-abc123xyz456"` hardcoded in a source file that is committed to version control.

## HIGH

The code compiles and runs correctly in the happy path but will fail, produce incorrect results, or violate important invariants under specific conditions that are common in production.

**Triggers**:
- **Missing null/undefined guard on a parameter that can reasonably be null**: Function parameter typed as optional or nullable, used without a guard, and the caller sometimes passes null.
- **Missing empty collection handling where emptiness changes behavior**: Iterating without checking for empty, calling `.first()`/`.last()` without checking, reducing over empty.
- **Missing test for new functionality**: New public function, component, or endpoint with zero test coverage.
- **Pattern inconsistency that creates a maintenance trap**: Completely different error handling style than the rest of the project (e.g., throwing exceptions in a codebase that uses Result types everywhere).
- **Type assertion without runtime validation**: `data as UserResponse` where `data` comes from an API and the shape is not guaranteed.

**Example**: `function process(items: Item[]) { const first = items[0]; ... }` — works when items is non-empty, crashes with undefined when empty.

**Example**: New React component at `src/components/UserProfile.tsx` with no corresponding `UserProfile.test.tsx` in a project where every other component has a test.

## MEDIUM

The code is functional and correct but has maintainability, readability, or mild edge-case concerns. Fixing it now prevents technical debt accumulation.

**Triggers**:
- **Copy-paste variation**: Two or more blocks of 5+ lines that are structurally identical except for variable names or literal values — should be extracted into a parameterized function.
- **Reimplementation of existing utility**: The diff implements `deepClone()` when `lodash.cloneDeep()` is already in the project's dependencies.
- **Placeholder abandonment**: `TODO`, `FIXME`, or stub implementation in new code with no tracking issue reference.
- **Debugging artifact**: `console.log(data)` in production-path code without a logging framework wrapper.
- **Mutable default argument** (Python): `def foo(items=[])` — works until the list is mutated across calls.
- **Dead code after return**: Statements after a return that have no effect but cause confusion.

**Example**: Three event handlers that each do `try { await api.post(...) } catch (e) { showToast("Error") }` with different API endpoints — should be one `async function safePost(endpoint, data)`.

**Example**: `// TODO: add pagination` in a new list endpoint that is being merged to main.

## LOW

The code is fully functional and maintainable but shows subtle signals common in AI-generated code. These are "noteworthy but not blocking."

**Triggers**:
- **Single `any` or `@ts-ignore` with apparent justification**: One type escape in an otherwise well-typed file, where the workaround seems deliberate.
- **Minor import style variation**: Using default import when the project convention is named imports, but both styles are used elsewhere.
- **Comment style drift**: AI uses `//` line comments when the project convention is `/** */` JSDoc for public APIs.
- **Overly descriptive naming**: `function retrieveUserDataFromDatabaseAndValidateFields()` instead of `getUser()` — works fine, just doesn't match typical human naming instincts.

**Example**: `const result: any = thirdPartyLib.doThing()` where the third-party library genuinely has no types — acceptable but worth noting.

## Tie-Breaking

When a finding could fit multiple tiers, use the higher one. When in doubt between MEDIUM and LOW, ask: "Would fixing this prevent a real bug or just make the code nicer?" If it prevents a bug, use MEDIUM. If cosmetic, use LOW.
