# AI Code Reviewer · AI代码审查

[English](#english) | [中文](#chinese)

---

<h2 id="english">English</h2>

Detect **AI-specific code defects** — failure modes that appear disproportionately in AI-generated code. Complements linters, type-checkers, and general code review.

### What It Detects (10 Categories)

| # | Category | What It Catches |
|---|----------|-----------------|
| 1 | **Hallucinated APIs** | Imports/function calls that don't exist — wrong module names, near-miss APIs (`array.flatten()` vs `array.flat()`) |
| 2 | **Pattern Inconsistency** | Code that violates the project's existing naming, error-handling, async, and import conventions |
| 3 | **Copy-Paste Variations** | Repeated code blocks (5+ lines) with minor tweaks that should be a parameterized abstraction |
| 4 | **Missing Edge Cases** | No null checks, empty collection guards, error handling on IO, or boundary condition guards |
| 5 | **Over-Confident Logic** | Subtle bugs AI commonly produces: shadowed variables, dead code, misordered args, `=` vs `==` |
| 6 | **Missing Tests** | New functions/files without corresponding tests (follows the project's test conventions) |
| 7 | **Placeholder Abandonment** | `TODO`/`FIXME` comments, stub bodies, debug leftovers (`console.log`), commented-out code |
| 8 | **Type System Abuse** | `any`, `as Type`, `@ts-ignore`, raw `Object`, `interface{}`, non-null assertions |
| 9 | **Security-Naive Patterns** | Hardcoded credentials, SQL/command injection, path traversal, weak crypto, XSS sinks |
| 10 | **Redundant Logic** | Reimplementing stdlib or existing project utilities (`flattenArray`, `deepCopy`, `formatDate`) |

### Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/ReiSei0824/ai-code-reviewer.git ~/.claude/skills/ai-code-reviewer
```

Or add it as a submodule if you already track your skills in git:

```bash
cd ~/.claude/skills
git submodule add https://github.com/ReiSei0824/ai-code-reviewer.git ai-code-reviewer
```

### Usage

Trigger the skill by mentioning any of these phrases in Claude Code:

- `review this PR for AI artifacts`
- `check if this was AI-generated`
- `find AI code issues`
- `audit this diff for hallucinated APIs`
- `AI code review`
- `review changes for AI patterns`
- `check for copy-paste duplication`
- `validate this PR's type safety`

Three input modes are supported:

| Mode | Example |
|------|---------|
| **Raw diff** | Paste a `git diff` output directly |
| **File paths** | `review src/feature.ts src/utils.ts` |
| **Branch / PR** | `review main..feature-branch` |

### Workflow

```
Input → Load Codebase Context → 10 Detection Passes → Deduplicate → Structured Report
```

1. **Load Context** — reads `package.json`/`Cargo.toml`/etc., extracts naming conventions, inventories existing utilities, understands test patterns
2. **Detect** — runs all 10 category checks against the diff, referencing detailed pattern catalogs per language
3. **Deduplicate** — merges overlapping findings, applies file-type exemptions, groups cross-file issues
4. **Report** — structured output ordered by severity: 🔴 Critical → 🟠 High → 🟡 Medium → 🟢 Low

### Scope

- **Does**: catch AI-characteristic defects in diffs, PRs, and changed files
- **Does NOT**: general bug finding, style nits, formatting, performance, or architectural review (use linters + human review for those)

---

<h2 id="chinese">中文</h2>

检测 **AI 生成代码的特有缺陷** — 这些是 AI 写的代码中远高于人类代码的失败模式。作为 Linter、类型检查器和通用代码审查的补充。

### 十大检测类别

| # | 类别 | 检测内容 |
|---|------|---------|
| 1 | **幻觉 API** | 不存在的 import/函数调用 — 错误模块名、近似 API（`str.contains()` 而非 `str.includes()`） |
| 2 | **模式不一致** | 违反项目已有命名规范、错误处理风格、异步模式、import 约定 |
| 3 | **复制粘贴变体** | 5 行以上的重复代码块只改了少量参数，应当抽象为参数化函数 |
| 4 | **缺失边界检查** | 缺少 null 检查、空集合守卫、IO 错误处理、边界条件保护 |
| 5 | **过度自信逻辑** | AI 常见的微妙 bug：变量遮蔽、死代码、参数顺序错误、`=` vs `==` 混淆 |
| 6 | **缺失测试** | 新增函数/文件无对应测试（遵循项目的测试约定） |
| 7 | **占位符遗弃** | `TODO`/`FIXME` 注释、空函数体、调试残留（`console.log`）、被注释掉的代码 |
| 8 | **类型系统滥用** | `any`、`as Type`、`@ts-ignore`、原始 `Object`、`interface{}`、非空断言 |
| 9 | **安全意识薄弱** | 硬编码密钥、SQL/命令注入、路径遍历、弱加密、XSS 漏洞 |
| 10 | **冗余逻辑** | 重新实现标准库或项目已有工具函数（`flattenArray`、`deepCopy`、`formatDate`） |

### 安装

```bash
# 克隆到 Claude Code 技能目录
git clone https://github.com/ReiSei0824/ai-code-reviewer.git ~/.claude/skills/ai-code-reviewer
```

如果你已经在用 git 管理 skills 目录：

```bash
cd ~/.claude/skills
git submodule add https://github.com/ReiSei0824/ai-code-reviewer.git ai-code-reviewer
```

### 使用方式

在 Claude Code 中说以下任一关键词即可触发：

- `review this PR for AI artifacts`
- `check if this was AI-generated`
- `find AI code issues`
- `audit this diff for hallucinated APIs`
- `AI code review`
- `review changes for AI patterns`
- `check for copy-paste duplication`
- `validate this PR's type safety`

支持三种输入模式：

| 模式 | 示例 |
|------|------|
| **原始 diff** | 直接粘贴 `git diff` 输出 |
| **指定文件** | `review src/feature.ts src/utils.ts` |
| **分支 / PR** | `review main..feature-branch` |

### 工作流

```
输入 → 加载代码库上下文 → 10 个检测通道 → 去重合并 → 结构化报告
```

1. **加载上下文** — 读取 `package.json`/`Cargo.toml` 等，提取命名规范，盘点已有工具函数，理解测试模式
2. **检测** — 对 diff 运行全部 10 类检查，参考各语言的详细模式目录
3. **去重合并** — 合并重叠发现，应用文件类型豁免规则，将跨文件问题归组
4. **输出报告** — 按严重度排序：🔴 严重 → 🟠 高 → 🟡 中 → 🟢 低

### 范围界定

- **做什么**：捕捉 diff、PR、变更文件中的 AI 特征缺陷
- **不做什么**：通用 bug 查找、代码风格、格式化、性能优化、架构评审（这些交给 Linter 和人工审查）

---

## File Structure

```
ai-code-reviewer/
├── SKILL.md                              # Main skill definition (252 lines)
└── references/
    ├── severity-guide.md                 # Severity classification criteria
    ├── output-template.md                # Structured report template
    ├── hallucinated-apis.md              # Category 1: Hallucinated API detection
    ├── pattern-inconsistency.md          # Category 2: Pattern mismatch detection
    ├── copy-paste-variations.md          # Category 3: Duplication detection
    ├── missing-edge-cases.md             # Category 4: Edge case gap detection
    ├── logical-flaws.md                  # Category 5: Logic flaw detection
    ├── missing-tests.md                  # Category 6: Test gap detection
    ├── placeholder-abandonment.md        # Category 7: TODO/placeholder detection
    ├── type-system-abuse.md              # Category 8: Type escape detection
    ├── security-naive-patterns.md        # Category 9: Security pattern detection
    └── redundant-logic.md                # Category 10: Redundancy detection
```

## License

MIT
