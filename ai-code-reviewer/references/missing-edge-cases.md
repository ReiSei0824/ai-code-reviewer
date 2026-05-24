# Missing Error/Null/Edge-Case Handling Detection Catalog

## Overview

AI-generated code disproportionately implements the happy path — the scenario where all inputs are valid, all resources are available, and all operations succeed. Error handling, null checks, boundary conditions, and edge cases are systematically underrepresented because they require anticipating failure modes, which is a weakness of pattern-matching generation.

This is one of the most common causes of production incidents from AI-generated code.

## Detection Checklist

### 4.1: Null/undefined/unwrap guards

For each function parameter, return value, or property access:

1. If a parameter is typed as optional/nullable (`?` in TypeScript, `Option<T>` in Rust, `Optional<T>` in Java, `T | None` in Python), check if it's used without a guard or default.
2. If a function returns a nullable type, check if the caller guards the result.
3. Check if optional chaining (`?.`) or null-coalescing (`??`) should be used but isn't.

### 4.2: Empty collection handling

For each iteration, access, or reduction:

1. Does it call `.first()`, `.last()`, `[0]`, `[-1]`, `.pop()` without checking `.length > 0` or `.isEmpty()`?
2. Does it reduce over a collection that might be empty without providing an initial value?
3. Does it call `.max()`, `.min()` on a potentially empty collection?

### 4.3: IO error handling

For each IO operation (HTTP, DB, file, network):

1. Is it wrapped in try-catch, `.catch()`, `?` operator, or `if err != nil`?
2. If not, is there a defensible reason (e.g., top-level script, fire-and-forget logging)?
3. For async operations: is the `await` inside a try-catch, or does the caller handle rejection?

### 4.4: API response validation

For each API call:

1. Is the response status checked before accessing data (`response.ok`, `response.status`, `response.status_code`)?
2. Is the response body shape validated before accessing nested properties?
3. For JSON responses: is there a try-catch around `.json()` parsing?

### 4.5: Boundary conditions

For each loop, comparison, or arithmetic operation:

1. Off-by-one: does a loop condition use `<` when `<=` is correct (or vice versa)?
2. Fencepost: does the last/first element get special treatment if needed?
3. Min/max: is there protection against integer overflow, division by zero, negative values in contexts that require positive?

### 4.6: Race conditions in async code

For concurrent operations:

1. Does the code read a value, then write based on that read, without a lock or atomic operation?
2. Does it fire multiple async operations and assume a specific completion order?

## Common AI-Generated Missing-Guard Patterns

### Pattern: Unchecked nullable access

```typescript
// AI-GENERATED — happy path only:
function getUserEmail(user: User | null): string {
    return user.email  // TypeError if user is null
}

// Correct:
function getUserEmail(user: User | null): string {
    return user?.email ?? 'unknown'
}
```

### Pattern: Unchecked empty array

```typescript
// AI-GENERATED:
function getLatestPost(posts: Post[]): Post {
    return posts.sort((a, b) => b.date - a.date)[0]
    // Undefined if posts is empty, and mutates the input array
}

// Correct:
function getLatestPost(posts: readonly Post[]): Post | undefined {
    if (posts.length === 0) return undefined
    return [...posts].sort((a, b) => b.date.getTime() - a.date.getTime())[0]
}
```

### Pattern: Uncaught async error

```javascript
// AI-GENERATED — no error handling on async call:
async function loadDashboard() {
    const user = await fetchUser()        // What if this rejects?
    const settings = await fetchSettings() // What if this rejects?
    return { user, settings }
}

// Correct:
async function loadDashboard() {
    try {
        const [user, settings] = await Promise.all([fetchUser(), fetchSettings()])
        return { user, settings }
    } catch (error) {
        logger.error('Failed to load dashboard', { error })
        return { user: null, settings: null }  // Graceful degradation
    }
}
```

### Pattern: Unchecked API response

```python
# AI-GENERATED — assumes success:
def get_user_data(user_id):
    response = requests.get(f"/api/users/{user_id}")
    return response.json()["data"]

# Correct:
def get_user_data(user_id):
    response = requests.get(f"/api/users/{user_id}")
    if not response.ok:
        raise APIError(f"Failed to fetch user {user_id}: {response.status_code}")
    data = response.json()
    if "data" not in data:
        raise APIError(f"Unexpected response shape: {list(data.keys())}")
    return data["data"]
```

### Pattern: Division/math without guard

```python
# AI-GENERATED:
def calculate_average(values):
    return sum(values) / len(values)  # ZeroDivisionError if values is empty

# Correct:
def calculate_average(values):
    if not values:
        return 0
    return sum(values) / len(values)
```

## Language-Specific Notes

### Rust
- AI commonly uses `unwrap()` on `Option`/`Result` without checking. Flag every bare `unwrap()` or `expect()` call.
- Missing `?` operator where `Result` should be propagated.
- `index[]` access without bounds checking vs `.get()`.

### Go
- Missing `if err != nil` after calls that return error.
- Accessing map keys without the comma-ok pattern: `val := m[key]` vs `val, ok := m[key]`.
- Not checking `rows.Err()` after iterating SQL rows.

### Python
- Missing `try/except` around IO calls.
- Using `dict["key"]` instead of `dict.get("key")` when the key may not exist.
- Not handling `KeyboardInterrupt` or other system signals in long-running processes.

### Java
- Missing null checks on `Optional.get()`.
- Not closing resources (should use try-with-resources).
- Missing `@NotNull`/`@Nullable` annotations in annotated codebases.

## False Positive Mitigation

- **Guard already present at call site**: A function may not need internal guards if all callers guarantee preconditions. If the function is private/internal and callers visibly validate, don't flag.
- **Framework-provided guarantees**: Express.js middleware may guarantee `req.user` exists after auth middleware. React Router guarantees route params exist. Don't flag when the framework contract guarantees safety.
- **Top-level scripts**: CLI scripts, build scripts, and migration scripts often skip graceful error handling — crashing with a stack trace is intentional.
- **Assertion-based contracts**: If the project uses assertions, design-by-contract, or `assert()` calls at function entry, those count as guards.
