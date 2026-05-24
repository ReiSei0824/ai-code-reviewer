# TODO/Placeholder Abandonment Detection Catalog

## Overview

AI-generated code frequently contains placeholders, stubs, TODOs, and debugging artifacts that the model left in because it "knew" something needed to be done but couldn't complete it within the generated response. Unlike human developers who typically clean these up before submitting a PR, AI leaves them behind systematically.

## Detection Checklist

### 7.1: TODO/FIXME/HACK comments

Search the added lines for these patterns (case-insensitive):
- `TODO`, `FIXME`, `HACK`, `XXX`, `WORKAROUND`, `TBD`, `TEMP`
- `@todo`, `@fixme`, `@hack`
- `FIXIT`, `OPTIMIZE`, `CLEANUP`

### 7.2: Stub function bodies

Search for:
- `throw new Error("Not implemented")`, `throw new NotImplementedException()`
- `pass` (Python — when the function body is just `pass`)
- `return null`, `return undefined`, `return ""`, `return 0` (only when it appears to be a placeholder, not a legitimate return)
- `unimplemented!()` (Rust)
- `panic("todo")` or `panic("not implemented")`
- `sys.exit(1)` with a "not implemented" message
- `// TODO: implement`, `# TODO: implement`, `/* TODO: implement */`

### 7.3: Placeholder strings

Search for common placeholder values:
- Email: `"test@example.com"`, `"user@example.com"`, `"placeholder@example.com"`
- API keys: `"your-api-key-here"`, `"sk-..."` (placeholder format), `"changeme"`, `"xxx"`, `"replace-me"`
- URLs: `"https://example.com"`, `"http://localhost:3000"` (when not in dev config)
- Names: `"John Doe"`, `"Test User"`, `"lorem ipsum"`, `"foo"`, `"bar"`, `"baz"`
- Content: `"Lorem ipsum dolor sit amet..."`, `"test content"`, `"placeholder text"`

### 7.4: Empty catch blocks

Search for:
- `catch (e) {}`, `catch {}`, `catch { }`
- `except: pass`, `except Exception: pass`
- `rescue => nil`, `rescue nil`
- `_ =` (Go — assigning error to blank identifier without check)

### 7.5: Commented-out code

Identify blocks of 5+ consecutive lines that are commented out and look like working code (not documentation, not commented-out todos). These are often AI's previous attempts that it commented out rather than removed.

### 7.6: Debugging leftovers

Search for:
- `console.log(...)`, `console.debug(...)`, `console.warn(...)` (when not part of a logging strategy)
- `print(...)` (Python — outside of CLI scripts)
- `dbg!(...)` (Rust)
- `debugger` (JS/TS)
- `pprint.pprint(...)`, `var_dump(...)`, `dump(...)`
- `println!(...)` (Rust — outside of examples)
- `fmt.Println(...)` (Go — outside of CLI tools)

## Common AI-Generated Placeholder Patterns

### Pattern: TODO with no tracking issue

```typescript
// AI-GENERATED:
function processPayment(amount: number): PaymentResult {
    // TODO: add fraud detection
    // TODO: handle currency conversion
    // TODO: add rate limiting
    return chargeCard(amount)
}

// The PR should either implement these or file issues — not leave TODOs in merged code.
```

### Pattern: Stub implementation

```python
# AI-GENERATED — complete-looking class with a stub:
class NotificationService:
    def send_email(self, to, subject, body):
        # TODO: implement email sending
        pass

    def send_sms(self, to, message):
        # TODO: implement SMS
        pass

# These will fail silently in production, no error even thrown.
```

### Pattern: Placeholder credentials in source

```python
# AI-GENERATED — placeholder API key in committed code:
import openai
openai.api_key = "sk-your-api-key-here"  # Will fail but not crash

# Even if it's clearly a placeholder, it suggests the AI didn't know how
# to wire up the proper config/secrets management.
```

### Pattern: Debug prints in production path

```typescript
// AI-GENERATED — debugging artifacts:
async function handleRequest(req: Request): Promise<Response> {
    console.log('req received', req.url)
    console.log('headers', req.headers)
    const body = await req.json()
    console.log('body', body)
    // ... actual handler logic
}
```

### Pattern: Commented-out alternative implementation

```javascript
// AI-GENERATED — previous attempt commented out instead of removed:
function transform(data) {
    return data.map(item => ({
        id: item.id,
        name: item.name.toUpperCase(),
        // status: item.status === 1 ? 'active' : 'inactive',
        status: item.active ? 'active' : 'inactive'  // AI tried both, left old one
    }))
}
```

## False Positive Mitigation

- **Legitimate TODOs**: Some teams intentionally merge TODOs that reference tracking issues (`// TODO(PROJ-123): add pagination`). If the TODO has a ticket reference and matches the project's convention, do NOT flag.
- **Debug in dev-only paths**: `console.log` in development setup scripts, build tooling, or behind `if (process.env.NODE_ENV === 'development')` guards is legitimate.
- **Print in CLI tools**: Command-line scripts and tools use `print()`/`fmt.Println()` as their primary output mechanism.
- **Empty catch with justification**: Sometimes an empty catch is intentional (e.g., ignoring a cancel error on a closed connection). If there's a comment explaining why, respect it.
- **Long placeholder strings**: Strings like `"https://api.example.com/v1/users"` might be legitimate endpoints. Only flag when paired with other placeholder signals (todo comment, stub body, etc.).
- **File-level exemption**: Test files, demo files, and example files often contain illustrative placeholders. Skip these.
