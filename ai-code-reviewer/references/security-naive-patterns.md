# Security-Naive Patterns Detection Catalog

## Overview

AI models are trained on public code repositories, which contain a wide range of security practices — from highly secure to dangerously naive. The model does not distinguish between them reliably. As a result, AI-generated code occasionally includes patterns that are actively dangerous: hardcoded credentials, injection vulnerabilities, insecure cryptography, and unsafe deserialization.

These are the highest-severity findings in the skill. A single hardcoded credential or SQL injection can be a security incident.

## Detection Checklist

### 9.1: Hardcoded credentials and secrets

Search for patterns matching common credential formats:

```
API Keys:      /[a-z][a-z0-9_]*api[_-]?key\s*[:=]\s*["'][a-zA-Z0-9_-]{20,}["']/i
AWS Keys:      /AKIA[0-9A-Z]{16}/
GitHub Tokens: /ghp_[0-9a-zA-Z]{36}/
Slack Tokens:  /xox[baprs]-[0-9a-zA-Z-]+/
JWT Tokens:    /eyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}/
Generic Tokens:/token\s*[:=]\s*["'][a-zA-Z0-9]{16,}["']/
Private Keys:  /-----BEGIN (RSA |EC )?PRIVATE KEY-----/
Connection Strings: /(mysql|postgres|mongodb|redis):\/\/[^:]+:[^@]+@/
Passwords:     /(password|passwd|pwd|secret)\s*[:=]\s*["'][^"']+["']/
```

Flag only in source code files — not in configuration templates (`.env.example`, `.env.template`, `.env.sample`).

### 9.2: SQL injection

Search for string concatenation or interpolation in SQL queries:

- JS/TS: `const query = "SELECT * FROM users WHERE id = " + userId`, `` `SELECT * FROM users WHERE name = '${name}'` ``
- Python: `f"SELECT * FROM users WHERE id = {user_id}"`, `"SELECT * FROM users WHERE id = " + user_id`
- Java: `"SELECT * FROM users WHERE id = " + userId`
- Go: `fmt.Sprintf("SELECT * FROM users WHERE id = %s", userId)`
- Ruby: `"SELECT * FROM users WHERE id = #{user_id}"`

Do NOT flag parameterized queries:
- `db.query("SELECT * FROM users WHERE id = $1", [userId])`
- `prisma.user.findUnique({ where: { id: userId } })`
- `conn.QueryRow(ctx, "SELECT * FROM users WHERE id = $1", userId)`

### 9.3: Command injection

Search for shell execution with untrusted input:

- Python: `os.system(f"rm {filename}")`, `subprocess.call(f"ls {dirpath}", shell=True)`, `os.popen(user_input)`
- JS: `exec("ls " + userInput)`, `child_process.exec(\`rm ${file}\`)`
- Ruby: `` `rm #{file}` ``, `system("rm #{file}")`
- Go: `exec.Command("bash", "-c", "rm "+filename)`
- PHP: `exec("rm " . $filename)`, `shell_exec($cmd)`

Do NOT flag when:
- The command uses a fixed argument list (not a shell string): `exec.Command("rm", filename)` (Go — uses exec syscall, no shell)
- The input is validated against a whitelist
- The shell is explicitly not involved (`shell=False` in Python)

### 9.4: Path traversal

Search for user input used in file paths without normalization:

- `fs.readFile("./uploads/" + req.query.file)` — user controls `file` → can read `../../../etc/passwd`
- `open(os.path.join("./uploads", filename))` — `filename` could be `../../etc/shadow`
- `File.open(Rails.root.join("uploads", params[:file]))` — same issue

Do NOT flag when:
- The path is validated with `path.resolve()` / `path.normalize()` and checked to be within the base directory
- The filename is generated server-side (UUID, hash)

### 9.5: Insecure deserialization

Search for:

- Python: `pickle.loads(untrusted_data)`, `yaml.load(untrusted_data)` (without SafeLoader), `marshal.loads()`
- JS: `eval(str)` on user-supplied data, `JSON.parse()` without try-catch (less severe, flag as LOW)
- Ruby: `YAML.load(user_data)` (unsafe, vs `YAML.safe_load`), `Marshal.load(user_data)`
- Java: `ObjectInputStream` with untrusted streams

### 9.6: Weak or custom cryptography

Search for:

- `md5(...)` or `sha1(...)` used for password hashing or security purposes
- Custom encryption functions (not using established libraries)
- Hardcoded encryption keys, IVs, or salts
- ECB mode for block ciphers
- `Math.random()` for security-sensitive values (use `crypto.randomUUID()` or equivalent)

## Common AI-Generated Security Patterns

### Pattern: Hardcoded API key

```python
# AI-GENERATED — key committed to source:
import openai
openai.api_key = "sk-proj-abc123def456ghi789jkl012mno345pqr678stu"  # CRITICAL

# Should use environment variable:
import os
openai.api_key = os.environ["OPENAI_API_KEY"]
```

### Pattern: String-built SQL

```typescript
// AI-GENERATED — SQL injection:
const user = await db.query(
    `SELECT * FROM users WHERE email = '${email}'`
)
// Attacker input: ' OR '1'='1' -- 
// Results in: SELECT * FROM users WHERE email = '' OR '1'='1' --'

// Should use parameterized query:
const user = await db.query(
    'SELECT * FROM users WHERE email = $1', [email]
)
```

### Pattern: Unsafe shell command

```python
# AI-GENERATED — command injection:
import os
def convert_video(filename):
    os.system(f"ffmpeg -i {filename} output.mp4")
# Attacker input: "video.mp4; rm -rf /"

# Should use argument list:
import subprocess
def convert_video(filename):
    subprocess.run(["ffmpeg", "-i", filename, "output.mp4"])
```

### Pattern: Unsafe file access

```typescript
// AI-GENERATED — path traversal:
app.get('/download', (req, res) => {
    const file = req.query.file as string
    res.sendFile(`./uploads/${file}`)
    // Attacker: /download?file=../../../.env → dumps secrets
})

// Should validate path:
app.get('/download', (req, res) => {
    const file = req.query.file as string
    const safePath = path.resolve('./uploads', file)
    if (!safePath.startsWith(path.resolve('./uploads'))) {
        return res.status(403).send('Forbidden')
    }
    res.sendFile(safePath)
})
```

## False Positive Mitigation

- **Configuration templates**: `.env.example`, `docker-compose.yml`, CI configs, and Kubernetes manifests often contain placeholder or example credentials. These are NOT in source code and should not be flagged.
- **Test files**: Test fixtures often contain fake credentials and tokens. Only flag if they appear to be real credentials (correct format, right entropy).
- **Documentation**: README files, code comments, and documentation that show example code may contain placeholder credentials.
- **Local development scripts**: `scripts/seed.sh`, `scripts/dev-setup.sh` — these run locally only and may contain local credentials. Flag as LOW, not CRITICAL.
- **Password regex false positives**: Variable names like `password` in form fields or UI code are not secrets. Only flag when a string literal is assigned to a variable with `password`/`secret`/`token` in its name.
