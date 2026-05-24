# Duplicate/Redundant Logic Detection Catalog

## Overview

AI tends to generate implementations from scratch rather than discovering and reusing existing project code. The model knows how to write `flattenArray()` or `deepClone()` and will confidently do so — without checking whether the project already has a utility for that or whether a dependency already provides it. This creates code that is "new" to the diff but redundant with the existing codebase.

Market research confirms AI-generated PRs have a ~0% refactoring rate. The model never looks at existing code and says "I should reuse that" — it always starts fresh.

## Detection Checklist

### 10.1: Reimplementation of project utilities

For each new function in the diff:

1. Extract the function's semantic signature: what does it take in, what does it return, what does it do? Look past the naming to the actual algorithm.
2. Grep the existing project codebase (outside the diff) for functions with similar semantic signatures:
   - Same number and types of parameters
   - Same return type
   - Similar algorithm (same loops, transformations, API calls)
3. Also search for utility modules by name: `utils`, `helpers`, `lib`, `common`, `shared`, `core`.

### 10.2: Reimplementation of standard library / ecosystem functions

Check if the new function replaces:

- **JavaScript/TypeScript**: `Object.entries()`, `Array.prototype.flat()`, `Array.prototype.flatMap()`, `String.prototype.padStart/End()`, `Promise.allSettled()`, `structuredClone()`, `URLSearchParams`
- **Lodash/Underscore** (if in project deps): `_.flatten()`, `_.debounce()`, `_.throttle()`, `_.pick()`, `_.omit()`, `_.groupBy()`, `_.uniq()`, `_.cloneDeep()`
- **Python**: `itertools.chain()`, `functools.lru_cache()`, `collections.defaultdict()`, `collections.Counter()`, `pathlib.Path`, `dataclasses.dataclass`
- **Go**: `slices.Contains()`, `slices.Sort()`, `maps.Clone()`, `slices.Max()/Min()`
- **Rust**: Iterator methods — `.flatten()`, `.flat_map()`, `.collect()`, `.filter_map()`
- **Java**: Stream API, `Optional` methods, `Collections` utility class
- **date-fns / moment.js** (if in deps): `format()`, `addDays()`, `differenceInDays()`

### 10.3: Reimplementation of project ORM/business logic

1. Hand-rolled SQL where the project uses an ORM.
2. Manual input validation where the project has a validation layer.
3. Direct HTTP calls where the project has an API client abstraction.
4. Manual caching where the project has a cache utility.

### 10.4: Type/interface redefinition

Check if a type or interface defined in the diff already exists in the project's shared types:
- Search for similar interface/type names in existing files.
- Search for interfaces with the same field structure (same keys, same types).

## Common AI-Generated Redundancy Patterns

### Pattern: Hand-rolled utility that exists in project

```javascript
// AI-GENERATED — new file utils/array-utils.js:
function groupBy(array, key) {
    return array.reduce((result, item) => {
        const group = item[key]
        if (!result[group]) result[group] = []
        result[group].push(item)
        return result
    }, {})
}

// But the project already has lodash imported:
// import _ from 'lodash' — _.groupBy() does exactly this
```

### Pattern: Hand-rolled async retry that exists in deps

```typescript
// AI-GENERATED:
async function fetchWithRetry(url: string, retries = 3): Promise<Response> {
    for (let i = 0; i < retries; i++) {
        try {
            return await fetch(url)
        } catch (err) {
            if (i === retries - 1) throw err
            await new Promise(r => setTimeout(r, 2 ** i * 1000))
        }
    }
}

// But the project's package.json lists 'ky' — which has built-in retry:
// import ky from 'ky' → ky.get(url, { retry: 3 })
```

### Pattern: Hand-rolled date formatting that date-fns handles

```typescript
// AI-GENERATED:
function formatPostDate(date: Date): string {
    const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
                    'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
    const day = date.getDate()
    const month = months[date.getMonth()]
    const year = date.getFullYear()
    return `${month} ${day}, ${year}`
}

// But the project imports date-fns everywhere:
// import { format } from 'date-fns' → format(date, 'MMM d, yyyy')
```

### Pattern: Manual validation reimplementation

```python
# AI-GENERATED:
def validate_email(email: str) -> bool:
    import re
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

# But the project has validators.py with an EmailValidator class
# Or uses a library like email-validator, pydantic, marshmallow
```

### Pattern: Reimplemented existing component

```tsx
// AI-GENERATED — new Modal component:
function ConfirmModal({ open, onClose, onConfirm, title, children }) {
    // 30 lines of modal implementation...
}

// But the project already has:
// import { Modal, ModalHeader, ModalBody, ModalFooter } from '@/components/ui/Modal'
// This reinvents the wheel with less functionality and different styling.
```

## False Positive Mitigation

- **Intentional fork**: If the existing utility is flawed or has a different contract, reimplementing it may be correct. Check the commit message or PR description for rationale.
- **Reducing dependencies**: If the project is actively removing a dependency and the new code replaces its last use, this is intentional migration.
- **Learning/demo code**: Example files, tutorials, and workshop code intentionally reimplement basic patterns.
- **Different scope**: If the project utility handles a superset of cases (internationalized, async, configurable) and the new function is a simpler, more focused alternative, the duplication may be justified.
- **External constraint**: The new code may avoid a dependency that is unavailable in the target environment (e.g., React Native where some Node.js utilities don't work).
