# Hallucinated APIs Detection Catalog

## Overview

AI models commonly generate function calls, method invocations, imports, and class instantiations that reference symbols that do not exist. This happens because the model's training data contains API documentation from multiple versions, different libraries, and sometimes entirely fabricated interfaces. The model confidently generates plausible-looking code that will fail at runtime.

This is the highest-impact detection category — hallucinated API calls cause immediate crashes in production.

## Detection Checklist

### 1.1: Import verification

For every `import`, `require`, `from X import Y`, `use` statement added in the diff:

1. Extract the module/package name.
2. Check `package.json` dependencies, `Cargo.toml`, `go.mod`, `Gemfile`, `pyproject.toml`, or equivalent for the dependency.
3. If the dependency is listed, verify it exports the specific symbol being imported. Use Grep to search `node_modules/`, `.venv/`, or vendor directories if available. Use WebSearch to verify the API surface if needed.
4. If the dependency is NOT listed, flag the import as potentially hallucinated — the package needs to be installed.

### 1.2: Function/method call verification

For every function call or method invocation on a non-primitive:

1. If the function is defined within the project, Grep for its definition.
2. If the function is from an external library, check if the library actually exports it.
3. If neither source nor dependency contains it, it is likely hallucinated.

### 1.3: Class/constructor verification

For every `new ClassName()` or class instantiation:

1. Grep for the class definition in the project.
2. If not found, check if it's from a dependency.
3. If neither, flag as hallucinated.

### 1.4: Property/method access on API responses

For every chain like `response.data.items[0].name`:

1. Trace the type of `response`. If it's from `fetch()`/`axios`/`requests.get()`, check that `.data` is the correct accessor (axios uses `.data`, fetch uses `.json()` which is async).
2. Check that each property in the chain plausibly exists on the declared type.

## Common AI-Generated Hallucination Patterns

### Pattern: Non-existent library methods

```javascript
// AI-GENERATED (hallucinated):
const result = array.flatten()        // JS arrays don't have .flatten()
const contains = str.contains("test") // String has .includes(), not .contains()
const sorted = list.sortBy("name")    // No standard .sortBy() in JS
```

```javascript
// CORRECT:
const result = array.flat()
const contains = str.includes("test")
const sorted = [...list].sort((a, b) => a.name.localeCompare(b.name))
```

### Pattern: Non-existent framework APIs

```typescript
// AI-GENERATED (hallucinated) — React:
function Component() {
  useEffectAsync(async () => {  // useEffectAsync does not exist
    await fetchData()
  })
}

// AI-GENERATED (hallucinated) — Express:
app.get("/users", (req, res) => {
  const users = User.findByEmail(req.query.email)  // Mongoose doesn't have this
})

// AI-GENERATED (hallucinated) — Prisma:
const user = await prisma.user.findByEmail(email)  // Prisma uses findUnique({where: {email}})
```

### Pattern: "Too perfect" function names

AI tends to generate unusually descriptive function names that mirror natural language descriptions:

```python
# AI signal — overly descriptive, not idiomatic:
def retrieve_all_active_users_from_database_with_valid_email():
    ...

# Human equivalent:
def get_active_users():
    ...
```

```javascript
// AI signal — generic name for a complex operation that likely exists as a library:
function convertDateStringToFormattedDisplayString(dateStr) { ... }

// Human equivalent — uses date-fns or moment:
import { format } from 'date-fns'
format(new Date(dateStr), 'MMMM do, yyyy')
```

### Pattern: Cross-library API confusion

AI mixes APIs from similar libraries:

```python
# AI mixes requests and httpx APIs:
import requests
response = requests.get(url)
data = response.json()        # requests uses .json()
headers = response.headers()  # httpx uses .headers() (callable), requests uses .headers (property)
```

### Pattern: Wrong version API

AI uses an API from a different major version than what's installed:

```python
# Pydantic v1 API used in a v2 project:
from pydantic import BaseModel
class User(BaseModel):
    name: str
    class Config:           # v1 style — v2 uses model_config = ConfigDict(...)
        orm_mode = True
```

## Language-Specific Notes

### JavaScript/TypeScript
- Common hallucination targets: `Array.prototype` methods (no `.flatten()`, `.groupBy()`, `.sortBy()`), String methods (no `.contains()`, `.capitalize()`), Promise methods (no `.finally()` on older runtimes, no `Promise.allSettled()` before ES2020)
- Framework hallucination: React hooks use `use` prefix — `useEffectAsync`, `useQuery`, `useDebounce` are NOT React built-ins (they may be from third-party libs)
- Node.js: `fs.existsSync()` is real, `fs.exists()` is deprecated and likely hallucinated if used as if synchronous

### Python
- Common hallucination targets: `os.path` functions (`os.path.join_path()` doesn't exist, use `os.path.join()`), string methods (`str.reverse()` doesn't exist, use `[::-1]`), list methods (`list.contains()` doesn't exist, use `in` operator)
- `datetime`: `datetime.strptime()` is real (with 'p'), `datetime.strftime()` is real, but `datetime.parse()` does not exist
- ORM hallucination: SQLAlchemy `session.create()` instead of `session.add()`, Django `Model.objects.get_or_none()` instead of try/except Model.DoesNotExist

### Rust
- Common hallucination: using `unwrap()` on Result without importing the trait, expecting `.to_string()` on types that don't implement Display
- Standard library: `std::collections::HashMap` has `entry()` but not `get_or_insert()` (that's a common hallucination mixing it with the `entry` API)

### Go
- Hallucinated `slices`/`maps` package functions: the `slices` and `maps` packages were added in Go 1.21 — code may call functions that don't exist in the installed Go version
- Common hallucination: `errors.Wrap()` from `github.com/pkg/errors` when the project only imports the standard `errors`

### Java/Kotlin
- Common hallucination: Stream API methods that don't exist: `.toList()` directly on Stream (added in Java 16, but still commonly hallucinated in pre-16 projects)
- Kotlin: `list.firstOrNull()` exists, `list.firstOrElse()` does not (use `firstOrNull() ?: default`)

## False Positive Mitigation

- **Do NOT flag** symbols that appear to be dynamically imported (e.g., `import(moduleName)` or `require(variable)`) — these can't be statically verified.
- **Do NOT flag** well-known globals (`console`, `process`, `window`, `document`, `Buffer`, `Promise`, `fetch`).
- **Do NOT flag** symbols from popular type definition packages (`@types/*`) — these may provide types for globally available libraries.
- **Do NOT flag** if the project uses a bundler or framework that auto-imports (e.g., Nuxt auto-imports, Vite plugin unplugin-auto-import).
- If the project uses path aliases (`@/`, `~/`), resolve them before checking if the import target exists.
