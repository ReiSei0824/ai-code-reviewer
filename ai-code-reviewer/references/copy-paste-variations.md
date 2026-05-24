# Copy-Paste Variations Detection Catalog

## Overview

AI models excel at generating variations of the same pattern. When asked to implement similar functionality across multiple places, AI tends to copy the first implementation and mechanically vary the names/literals — producing blocks of code that share 80%+ structure with minor differences. This is the exact opposite of the DRY principle and creates a maintenance nightmare: fixing a bug in the pattern requires finding and fixing all copies.

Market research shows AI-produced code has a ~0% refactoring rate — AI always generates net-new code rather than identifying and extracting shared abstractions.

## Detection Checklist

### 3.1: Within-diff duplication

Scan the diff for blocks of 5+ consecutive lines that share structural similarity:

1. Normalize each block: strip comments, replace identifiers with placeholders (`VAR1`, `VAR2`), replace literals with `LITERAL`, standardize whitespace.
2. Compare all pairs of blocks. Pairs above 80% normalized similarity are candidates.
3. For each candidate pair, compute the variation vector: what differs between them (variable names, types, literal values, error messages).

### 3.2: Cross-diff vs codebase duplication

Check if logic in the diff already exists in the codebase:

1. For each new function, Grep for functions with similar names or signatures in existing files.
2. For each algorithm/pattern in the diff (e.g., retry logic, pagination, form validation), Grep for that pattern in the existing codebase.

### 3.3: Variation type classification

Categorize the variation to suggest the right abstraction:

- **Type variation**: Same logic, different types → generics or union types
- **Value variation**: Same logic, different constants → parameterized function
- **Endpoint variation**: Same logic, different API endpoints → config-driven or strategy pattern
- **Condition variation**: Same logic, different if-conditions → predicate parameter

## Common AI-Generated Copy-Paste Patterns

### Pattern: Mechanical endpoint wrappers

```typescript
// AI-GENERATED — near-identical blocks for each entity:
async function createUser(data: UserData) {
    try {
        const response = await api.post('/users', data)
        showToast('User created successfully')
        return response.data
    } catch (error) {
        showToast('Failed to create user')
        throw error
    }
}

async function createProject(data: ProjectData) {
    try {
        const response = await api.post('/projects', data)
        showToast('Project created successfully')
        return response.data
    } catch (error) {
        showToast('Failed to create project')
        throw error
    }
}

// Should be:
async function createEntity<T>(endpoint: string, entityName: string, data: T) {
    try {
        const response = await api.post(endpoint, data)
        showToast(`${entityName} created successfully`)
        return response.data
    } catch (error) {
        showToast(`Failed to create ${entityName}`)
        throw error
    }
}
```

### Pattern: Duplicated validation blocks

```python
# AI-GENERATED — same validation pattern with different field names:
def validate_user(data):
    if not data.get('name'):
        raise ValidationError('Name is required')
    if len(data.get('name', '')) > 100:
        raise ValidationError('Name must be under 100 characters')

def validate_project(data):
    if not data.get('title'):
        raise ValidationError('Title is required')
    if len(data.get('title', '')) > 200:
        raise ValidationError('Title must be under 200 characters')

# Should use a field validation config:
FIELD_RULES = {
    'name': {'required': True, 'max_length': 100},
    'title': {'required': True, 'max_length': 200},
}
```

### Pattern: Duplicated error boundary wrappers

```tsx
// AI-GENERATED — same error boundary pattern in every component file:
function UserProfile() {
    const [error, setError] = useState(null)
    if (error) return <ErrorFallback error={error} />
    // ... component
}

function Dashboard() {
    const [error, setError] = useState(null)
    if (error) return <ErrorFallback error={error} />
    // ... component
}

// Should use: an <ErrorBoundary> wrapper component (which likely already exists in the project)
```

### Pattern: Repeated filter-map-reduce chains

```javascript
// AI-GENERATED — repeated data transformation pattern:
const activeUsers = users.filter(u => u.active).map(u => ({ id: u.id, name: u.name }))
const activeProjects = projects.filter(p => p.active).map(p => ({ id: p.id, title: p.title }))
const activeTeams = teams.filter(t => t.active).map(t => ({ id: t.id, label: t.name }))

// Should be:
function pickActive<T extends { active: boolean }>(items: T[], fields: (keyof T)[]) {
    return items.filter(i => i.active).map(i => pickFields(i, fields))
}
```

## False Positive Mitigation

- **Auto-generated boilerplate**: Component files that each have similar import blocks (React, Vue SFC, Angular components) are expected — these follow framework conventions, not AI laziness.
- **Test files**: Test files often have repetitive structure by design (test setup, assertions). Use a higher similarity threshold (90%+) for test files.
- **Configuration**: YAML/JSON configuration blocks often repeat similar structures — this is expected.
- **Different business rules**: If two blocks have substantial business-logic differences (not just variable name changes), they may legitimately need to be separate.
- **Only flag if extraction would simplify**: If parameterizing the variation would require passing 10 parameters and become unreadable, don't flag it. The extracted function should be clearer, not more complex.
