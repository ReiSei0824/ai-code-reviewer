# Type System Abuse Detection Catalog

## Overview

AI models have a strong tendency to work around type errors rather than fix them properly. When faced with a type mismatch, the model often reaches for type escapes (`any`, `as`, `@ts-ignore`) instead of designing correct types. This is a form of "make it compile at any cost" behavior — the AI optimizes for the immediate goal (generating code that looks right) rather than type safety.

A high density of type escapes in a diff is one of the strongest signals of AI-generated code.

## Detection Checklist

### 8.1: Type escapes

Search for:
- TypeScript: `any`, `unknown` (when not intentional), `as` type assertions
- Python: bare function signatures without type hints in a typed codebase
- Go: `interface{}` (should be `any` in Go 1.18+, or better, a concrete type)
- Java: raw types (`List` instead of `List<User>`)
- Kotlin: `Any?`, `as` casts
- Rust: excessive `Box<dyn Any>`, `.downcast_ref()`
- C#: `dynamic`, `object`
- Ruby: code without type signatures in a Sorbet-typed codebase

### 8.2: Type assertions without validation

For each type assertion:
1. Is the source type broader than the target type (e.g., `any as User`, `unknown as string`)?
2. If so, is there a runtime validation (type guard, schema check, `typeof` check) before the assertion?
3. If not, flag as unsafe assertion.

### 8.3: Suppression comments

Search for:
- TypeScript: `// @ts-ignore`, `// @ts-expect-error` (count > 1 is suspicious)
- Python: `# type: ignore`
- Rust: `#[allow(clippy::...)]` when used to suppress multiple warnings
- MyPy: `# mypy: ignore-errors`
- C#: `#pragma warning disable`
- Kotlin: `@Suppress("UNCHECKED_CAST")`

### 8.4: Non-null assertions

- TypeScript: `value!.property`, `value!`
- Dart/Kotlin: `value!!`
- Swift: force unwrap `value!`
- Rust: `value.unwrap()` (without prior check)

### 8.5: File-level type suppression

These are strong AI signals — a human developer would fix the types, not disable the checker:
- `// @ts-nocheck` at the top of a file
- `#nullable disable` at the top of a C# file
- `# mypy: ignore-errors` at the top of a Python file

### 8.6: Unsafe eval patterns

- `eval(str)` — evaluating a string as code
- `new Function(str)` — creating a function from a string
- These are often used by AI as a lazy way to convert between types

## Common AI-Generated Type Abuse Patterns

### Pattern: Escape hatch abuse

```typescript
// AI-GENERATED — cascade of type escapes:
function processResponse(response: any) {
    const data = response.data as any
    const user = data.user as User
    // @ts-ignore
    const email = user.email.address  // Multiple escapes chained together
    return email
}

// Should be: typed response handling with validation
interface ApiResponse<T> {
    data: T
}
function processResponse(response: ApiResponse<{ user: User }>) {
    return response.data.user.email?.address
}
```

### Pattern: Non-null assertion cascade

```typescript
// AI-GENERATED — bang-bang-bang:
function renderUser(user?: User | null) {
    return <div>
        <h1>{user!.name}</h1>
        <p>{user!.profile!.bio}</p>
        <span>{user!.profile!.avatar!.url}</span>
    </div>
}

// Should use optional chaining or early return:
function renderUser(user?: User | null) {
    if (!user) return null
    return <div>
        <h1>{user.name}</h1>
        <p>{user.profile?.bio}</p>
        <span>{user.profile?.avatar?.url}</span>
    </div>
}
```

### Pattern: `any` propagation

```typescript
// AI-GENERATED — one 'any' contaminates the whole chain:
async function fetchUsers(): Promise<any> {  // any here
    const response = await api.get('/users')
    return response.data
}

function useUsers() {
    const [users, setUsers] = useState<any[]>([])  // any spreads here
    useEffect(() => {
        fetchUsers().then(setUsers)
    }, [])
    return users.map((u: any) => u.name)  // and here
}
```

### Pattern: `@ts-ignore` instead of fixing

```typescript
// AI-GENERATED — suppress error instead of fixing the underlying issue:
function getConfig(): Config {
    // @ts-ignore — "Property 'theme' does not exist on type 'Config'"
    return { theme: 'dark' }
}
// The correct fix is to add 'theme' to the Config type, not suppress the error.
```

## Language-Specific Thresholds

### TypeScript
- **1 `any` in a new file**: Flag as LOW, note for review
- **3+ `any` in a single file**: Flag as MEDIUM — systemic issue
- **Any `@ts-ignore` without explanation comment**: Flag as HIGH
- **`// @ts-nocheck` at file level**: Flag as HIGH — this is almost never correct
- **5+ `as` casts in a single function**: Flag as MEDIUM

### Python
- **No type hints in a file when the project is fully typed**: Flag as MEDIUM
- **`# type: ignore` without specific error code**: Flag as MEDIUM
- **`Any` from typing module used heavily**: Flag as MEDIUM

### Rust
- **`unwrap()` without comment justifying why it can't fail**: Flag as MEDIUM
- **`clone()` used to work around borrow checker instead of proper lifetimes**: Flag as LOW
- **`unsafe { }` block without SAFETY comment**: Flag as HIGH (this is a Rust convention)

## False Positive Mitigation

- **Third-party library types**: When a library genuinely has no types, `any` or `@ts-ignore` may be the best available option.
- **Migration code**: Code that bridges typed and untyped parts of the system (e.g., during a gradual migration to TypeScript).
- **Test files**: Test mocks and stubs legitimately use type assertions more freely.
- **Single justified escape**: If a `@ts-ignore` has a clear comment explaining why (e.g., `// @ts-expect-error — testing invalid input`), do not flag.
- **CLI entry points**: `#!/usr/bin/env node` scripts and CLI entry points may legitimately use fewer type annotations.
