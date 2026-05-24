# Pattern Inconsistency Detection Catalog

## Overview

AI-generated code often deviates subtly from the conventions established in an existing codebase. The model generates idiomatic code for the language in general, but not for this specific project. These inconsistencies create maintenance burden — future developers must context-switch between different styles within the same codebase.

## Detection Checklist

### 2.1: Naming convention violation

Extract the project naming conventions from Phase 1.2. For each new identifier in the diff (function name, variable name, class name, constant name, file name), check:

- **Functions/variables**: matches project convention (camelCase, snake_case, PascalCase)?
- **Classes/interfaces**: PascalCase enforced?
- **Constants**: UPPER_SNAKE_CASE or match project convention?
- **Files**: kebab-case vs PascalCase vs snake_case?
- **Test files**: naming convention for test files matches?

### 2.2: Error handling style mismatch

Error handling is one of the most distinctive conventions in a codebase. Check:

- **Style**: Result types vs exceptions vs error codes vs Go-style tuples vs callbacks?
- **Granularity**: Does the project use custom error classes, or throw plain Error objects?
- **Logging**: Does the project log errors and re-throw, or handle silently, or propagate?
- **User-facing errors**: Does the project wrap errors in user-friendly messages?

### 2.3: Async model inconsistency

- **Promise chains vs async/await**: If the project consistently uses `async/await`, flag `.then()/.catch()` chains.
- **Callback vs Promise**: If the project uses callbacks, flag Promise-based code (or vice versa).

### 2.4: Import style drift

- **Named vs default**: Does the project use `import { thing }` or `import thing`?
- **Path style**: Does the project use relative imports (`../`) or path aliases (`@/`, `~/`)?
- **Barrel exports**: Does the project consolidate exports through index files?

### 2.5: File organization departure

- **Does the new file belong where it was placed?** e.g., putting a utility function in a component file, or a database query in a route handler.
- **Module size**: Is the new file abnormally large compared to existing files in the project?

## Common AI-Generated Inconsistency Patterns

### Pattern: Default language style over project style

```typescript
// Project uses Result types everywhere:
function validate(input: string): Result<User, ValidationError> { ... }

// AI-GENERATED in same project — uses exception throwing:
function validate(input: string): User {
    if (!input) throw new Error("Invalid")  // Pattern break!
    ...
}
```

### Pattern: Stale error handling

```javascript
// Project convention — structured error handling with custom types:
class ApiError extends Error {
    constructor(public statusCode: number, message: string) { super(message) }
}

// AI-GENERATED — throws raw strings:
throw "API call failed"  // Pattern break! Project uses ApiError class
```

### Pattern: Mismatched import style

```typescript
// Project convention — path aliases:
import { Button } from '@/components/ui/Button'
import { useAuth } from '@/hooks/useAuth'

// AI-GENERATED — deep relative imports:
import { Button } from '../../../components/ui/Button'  // Pattern break!
import { useAuth } from '../../hooks/useAuth'
```

### Pattern: Wrong file for the concern

```typescript
// AI puts a SQL query directly in a React component file:
// src/components/UserList.tsx
const GET_USERS = `SELECT * FROM users WHERE active = true`  // Pattern break!

// Project convention — queries in dedicated files:
// src/db/queries/users.ts would be the expected location
```

## Language-Specific Notes

### TypeScript/React
- Check if new components use the project's preferred patterns: function declarations vs arrow functions, default exports vs named exports, prop types via interface vs type.
- Check if styling uses the project's CSS approach (Tailwind classes, CSS modules, styled-components, vanilla CSS).

### Python
- Check if new code follows the project's docstring convention (Google, NumPy, Sphinx, or none).
- Check if type hints are used consistently with the rest of the project. A file without type hints in a fully-typed project (or vice versa) is an AI signal.

### Rust
- Check `use` statement organization: does the project group imports (std, external, crate)?
- Check if `unwrap()` vs `?` operator usage matches project convention.

### Go
- Check error handling: `if err != nil { return fmt.Errorf("context: %w", err) }` wrapping vs plain `return err`.
- Check if the new code uses the project's logging library or falls back to `fmt.Println`.

## False Positive Mitigation

- **New module/package**: If the diff creates a new module that is architecturally separate (e.g., a new microservice in a monorepo), it may intentionally use different conventions. Check if other modules also vary.
- **Generated code**: Code from protobuf, OpenAPI, or GraphQL codegen follows its own conventions — do not flag these.
- **Third-party integration**: Code that wraps a third-party SDK may need to match the SDK's conventions rather than the project's.
- **Intentional refactoring**: If the diff is part of a migration (e.g., migrating from callbacks to promises), the inconsistency is intentional.
