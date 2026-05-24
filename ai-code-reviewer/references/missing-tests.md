# Missing Tests Detection Catalog

## Overview

AI models rarely generate tests alongside implementation code. When asked to "add a feature," AI produces the feature code — it does not independently decide to also write tests, test fixtures, or update existing test suites. This creates a systemic gap: every AI-generated feature lands without test coverage.

Research shows AI-assisted PRs have a ~0% refactoring rate and produce almost exclusively net-new code. The same pattern applies to testing: AI code tends to be completely untested.

## Detection Checklist

### 6.1: New file without test file

For each new source file in the diff:

1. Identify the test file convention from Phase 1.4:
   - `src/foo/bar.ts` → look for `src/foo/__tests__/bar.test.ts`, `src/foo/bar.spec.ts`, `tests/foo/bar_test.ts`
   - `lib/user.rb` → look for `spec/user_spec.rb`, `test/user_test.rb`
   - `pkg/user/user.go` → look for `pkg/user/user_test.go`
   - `app/models.py` → look for `tests/test_models.py`, `app/tests/test_models.py`

2. If no corresponding test file exists, flag it.

### 6.2: New function/method without test

For each new public function, method, or component in the diff:

1. Grep the corresponding test file for the function name.
2. If not found, flag the function as untested.

### 6.3: Bug fix without regression test

If the diff includes a bug fix (identifiable by):
- Issue references in commit messages or comments (`fixes #123`, `closes #456`)
- Code that clearly replaces incorrect logic with correct logic
- A new condition or guard that addresses a specific edge case

Check if a corresponding test was added that exercises the fixed scenario. Flag if missing.

### 6.4: New API endpoint without integration test

For each new HTTP route handler, GraphQL resolver, or RPC method:
- Check if the project has integration/API tests.
- If so, check if this new endpoint has one.

## Common AI-Generated Missing-Test Patterns

### Pattern: Feature without tests

```typescript
// AI-GENERATED — new component with zero tests:
// src/components/UserTable.tsx (new file)
export function UserTable({ users }: { users: User[] }) {
    // 50 lines of component logic
}

// Missing: src/components/__tests__/UserTable.test.tsx
// AI was asked to "add a user table component" and did exactly that — nothing more.
```

### Pattern: Bug fix without regression

```typescript
// AI-GENERATED bug fix:
function calculateTotal(items: Item[]): number {
-   return items.reduce((sum, i) => sum + i.price, 0)
+   return items.reduce((sum, i) => sum + (i.price * i.quantity), 0)
}

// Missing: a test that would have caught this: { price: 10, quantity: 2 } → 20, not 10
```

### Pattern: Utility function without coverage

```python
# AI-GENERATED — new utility, no test:
# utils/parsers.py
def parse_date_range(date_str: str) -> tuple[date, date]:
    """Parse '2024-01..2024-12' into (start_date, end_date)."""
    start, end = date_str.split('..')
    return date.fromisoformat(start), date.fromisoformat(end)

# Missing: tests/test_parsers.py with:
# - test_parse_date_range_valid()
# - test_parse_date_range_invalid_format()
# - test_parse_date_range_reversed_dates()
```

## Language/Framework-Specific Notes

### Jest/Vitest (JS/TS)
- Look for `*.test.ts`, `*.spec.ts`, `*.test.tsx`, `*.spec.tsx` in `__tests__/` or alongside source files
- Check if `describe` blocks exist for the new component

### pytest (Python)
- Look for `test_*.py` or `*_test.py` in `tests/`, `test/` directories
- Check for test functions named `test_<function_name>`

### Go
- Test files MUST be `*_test.go` in the same package directory
- Test functions MUST start with `Test`
- This makes detection straightforward — if the package has no `*_test.go`, flag it

### RSpec (Ruby)
- Look for `spec/*_spec.rb`
- Check for `describe` blocks matching the new class/module name

### JUnit (Java/Kotlin)
- Look for `src/test/java/` with matching package path
- Check for `*Test.java`, `*Tests.java`, `*Spec.kt`

## False Positive Mitigation

- **Pure type/interface files**: `.d.ts` files, type-only files, enums without logic — these don't need dedicated tests.
- **Configuration files**: `constants.ts`, `config.py`, `settings.go` — these often don't need tests beyond what integration tests provide.
- **Simple re-exports**: `export { default } from './Component'` — barrel files don't need tests.
- **Migration files**: Database migration scripts are tested by running the migration, not by unit tests.
- **Generated code**: Auto-generated GraphQL types, protobuf stubs, OpenAPI clients — these are tested by the generator's test suite.
- **Trivial passthrough**: If the new function is a one-liner that delegates to a well-tested dependency (e.g., `const useUser = () => useContext(UserContext)`), the dependency's tests provide indirect coverage.
