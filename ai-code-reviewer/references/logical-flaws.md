# Over-Confident Implementations (Logical Flaws) Detection Catalog

## Overview

AI models produce syntactically valid, type-correct code that compiles or passes superficial review — but contains subtle logic errors. These errors occur because the model generates code that "looks right" probabilistically, without actually reasoning through the execution flow. The result is code that passes a glance but fails under scrutiny.

## Detection Checklist

### 5.1: Operator confusion

1. **Assignment in condition**: `if (x = y)` in C-like languages (JS, C++, Java, C#). AI may use `=` where `==` or `===` is intended.
2. **OR vs AND**: `if (x || y)` when `if (x && y)` is correct (or vice versa). Common when the AI confuses "both must be true" with "at least one must be true."
3. **AND vs OR in filters**: `.filter(x => conditionA && conditionB)` when `.filter(x => conditionA || conditionB)` is intended.
4. **Equality vs identity**: `==` vs `===` in JS, `==` vs `.equals()` in Java, `is` vs `==` in Python.

### 5.2: Comparison direction errors

1. **Inverted comparison**: `if (value > limit)` when `value < limit` is intended.
2. **Wrong comparator sign**: `sort((a, b) => a.score - b.score)` sorts ascending, but maybe descending was intended.
3. **Boolean logic inverted**: `!isValid` when `isValid` is intended, or vice versa.

### 5.3: Shadowed variables

1. A variable in an inner scope has the same name as one in an outer scope, and the inner one changes behavior in a way that's likely unintended.
2. Loop variable shadowing a function parameter.

### 5.4: Dead code

1. Statements after `return`, `throw`, `break`, or `continue` in the same block.
2. Unreachable `else` branches after an always-true condition.
3. Variables assigned but never read.

### 5.5: Mutable default arguments

1. Python: `def foo(items=[])` — the same list object is shared across calls.
2. JS: default parameter that mutates: `function foo(config = {}) { config.key = val; return config; }`.
3. Go: using a package-level map/slice as a default without copying.

### 5.6: State machine errors

1. Boolean flags or enums that can reach an invalid combination.
2. State transitions that skip intermediate states incorrectly.
3. Missing state reset on error/recovery.

### 5.7: Misordered arguments

1. Positional arguments passed in the wrong order to functions with multiple parameters of the same type (string, string, int).
2. Swapped `actual` vs `expected` in test assertions: `assertEquals(result, expected)` vs `assertEquals(expected, result)`.

## Common AI Logical Flaw Patterns

### Pattern: Assignment in condition

```javascript
// AI-GENERATED — uses assignment = instead of comparison ===:
if (status = 'active') {  // Always true, assigns 'active' to status!
    processUser()
}

// Correct:
if (status === 'active') {
    processUser()
}
```

### Pattern: Inverted filter logic

```javascript
// AI-GENERATED — keeps the wrong items:
const valid = items.filter(i => i.active && i.deleted)
// "active and deleted" likely means "keep only deleted" — probably meant:
// "active and NOT deleted"

// Correct:
const valid = items.filter(i => i.active && !i.deleted)
```

### Pattern: Wrong sort direction

```javascript
// AI-GENERATED — default sort puts 'Z' before 'A' for descending:
scores.sort((a, b) => a - b)  // Ascending
const top5 = scores.slice(0, 5)  // Gets the 5 smallest, not top 5!

// Correct for "top 5":
scores.sort((a, b) => b - a)  // Descending
const top5 = scores.slice(0, 5)
```

### Pattern: Python mutable default

```python
# AI-GENERATED — shared mutable default:
def add_user(user, users=[]):
    users.append(user)
    return users

# First call: add_user("Alice") → ["Alice"]
# Second call: add_user("Bob") → ["Alice", "Bob"]  # Surprise!

# Correct:
def add_user(user, users=None):
    if users is None:
        users = []
    users.append(user)
    return users
```

### Pattern: Shadowed variable

```typescript
// AI-GENERATED — inner 'user' shadows parameter:
function processOrder(user: User, orders: Order[]) {
    for (const user of orders) {  // 'user' is now an Order, not a User!
        sendEmail(user.email)     // Which 'user'? The parameter or loop var?
    }
}

// Correct:
function processOrder(user: User, orders: Order[]) {
    for (const order of orders) {
        sendEmail(order.email)
    }
}
```

### Pattern: Dead code after return

```typescript
// AI-GENERATED:
function getValue(): string {
    return 'default'
    const result = computeValue()  // Dead code — never runs
    return result
}
```

### Pattern: Misordered test assertions

```typescript
// AI-GENERATED — swapped expected/actual:
expect(result).toBe(expectedValue)  // Most test frameworks: expect(actual).toBe(expected)

// Actually, this one is framework-dependent. Jest: expect(actual).toBe(expected)
// Some frameworks use reversed order. Check project convention.
// The REAL issue is when the assertion message is misleading:
expect(user.name).toBe('Bob')  // Error says "expected 'Alice', got 'Bob'" — confusing
// If the values are swapped: expect('Bob').toBe(user.name) — error makes sense
```

### Pattern: Invalid state combinations

```typescript
// AI-GENERATED — states can be invalid (both loading AND error):
const [loading, setLoading] = useState(false)
const [error, setError] = useState(null)

function refresh() {
    setLoading(true)
    setError(null)  // OK — clear error when loading
}

// But an error handler might do:
function handleError(err) {
    setError(err)
    // Forgot to setLoading(false)! Both loading=true AND error set
}

// Should use a discriminated union:
type State<T> = { status: 'idle' } | { status: 'loading' } | { status: 'error', error: Error } | { status: 'success', data: T }
```

## False Positive Mitigation

- **Test files**: In test files, `expect(actual).toBe(expected)` is conventional regardless of order. Only flag if the semantic meaning is clearly wrong.
- **Deliberate shadowing**: In some patterns (e.g., Rust's `let x = x` ownership transfer), shadowing is idiomatic.
- **Debug code**: A `return` placed for debugging might be followed by dead code intentionally. Check git history to see if the code was recently added.
- **Linter catch**: Many of these (shadowing, dead code, assignment in condition) are caught by linters. If the project runs ESLint/pylint/clippy, skip findings the linter would flag. Focus on flaws that linters do NOT catch (comparison direction, logic inversion, state machine errors).
