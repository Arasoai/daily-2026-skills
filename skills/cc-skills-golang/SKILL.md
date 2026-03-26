```markdown
---
name: cc-skills-golang
description: Install and use Go-specific AI agent skills for production-ready Golang projects. Covers code style, testing, security, observability, error handling, performance, concurrency, and popular Go libraries via the skills.sh plugin system.
triggers:
  - add golang skills to my agent
  - install go coding skills for claude code
  - use cc-skills-golang in my project
  - set up golang agent skills
  - add go testing and security skills
  - install samber golang skills plugin
  - load golang observability skills
  - configure go agent skills for cursor or codex
---

# cc-skills-golang

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated collection of Go-specific AI agent skills that give coding assistants (Claude Code, Cursor, Codex, Gemini CLI, Copilot, OpenCode) deep expertise in production-ready Golang — covering code style, testing, security, observability, error handling, concurrency, performance, and popular libraries like `samber/lo`, `samber/oops`, `samber/slog`, gRPC, and more.

Skills are **atomic, lazily loaded instruction sets** — only the `description` field (~100 tokens) is always in context; the full `SKILL.md` body loads only when relevant, keeping context budgets low.

---

## Installation

### Universal (any compatible agent)

```bash
npx skills add https://github.com/samber/cc-skills-golang --skill '*'
```

Install a single skill:

```bash
npx skills add https://github.com/samber/cc-skills-golang --skill golang-performance
```

### Claude Code

```bash
/plugin marketplace add samber/cc
/plugin install cc-skills-golang@samber
```

### Gemini CLI

```bash
gemini extensions install https://github.com/samber/cc-skills-golang
# Update later:
gemini extensions update cc-skills-golang
```

### Cursor

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.cursor/skills/cc-skills-golang
```

Cursor auto-discovers skills from `.cursor/skills/` and `.agents/skills/`.

### Codex (OpenAI)

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.agents/skills/cc-skills-golang
```

### OpenCode

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.agents/skills/cc-skills-golang
```

OpenCode discovers from `.agents/skills/`, `.opencode/skills/`, and `.claude/skills/`.

### Copilot

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.copilot/skills/cc-skills-golang
```

### Openclaw / Antigravity

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.openclaw/skills/cc-skills-golang
# or for Antigravity:
git clone https://github.com/samber/cc-skills-golang.git ~/.antigravity/skills/cc-skills-golang
```

---

## Available Skills

### ⭐ Recommended (install all for best results)

| Skill | What it covers |
|---|---|
| `golang-code-style` | Formatting, idiomatic Go, style conventions |
| `golang-data-structures` | Maps, slices, structs, generics patterns |
| `golang-database` | SQL, pgx, GORM, migrations, connection pools |
| `golang-design-patterns` | Functional options, builder, repository, DI |
| `golang-documentation` | GoDoc conventions, package-level comments |
| `golang-error-handling` | `fmt.Errorf`, wrapping, sentinel errors, `oops` |
| `golang-modernize` | Go 1.21+ features, iterators, slices package |
| `golang-naming` | Package, var, func, interface naming rules |
| `golang-safety` | Nil safety, race conditions, overflow, bounds |
| `golang-testing` | Table-driven tests, mocks, testify, coverage |
| `golang-troubleshooting` | Debugging, profiling, common failure patterns |
| `golang-security` | Input validation, crypto, SAST rules (Bearer) |

### Additional General Skills

```
golang-benchmark         golang-cli              golang-concurrency
golang-context           golang-continuous-integration
golang-dependency-injection                      golang-dependency-management
golang-structs-interfaces golang-linter          golang-observability
golang-performance       golang-popular-libraries golang-project-layout
golang-stay-updated
```

### Tool-Specific Skills

```
golang-grpc              golang-samber-do        golang-samber-hot
golang-samber-lo         golang-samber-mo        golang-samber-oops
golang-samber-ro         golang-samber-slog      golang-stretchr-testify
```

---

## Installing Specific Skills vs All

```bash
# All skills (recommended for full coverage)
npx skills add https://github.com/samber/cc-skills-golang --skill '*'

# Just security + testing
npx skills add https://github.com/samber/cc-skills-golang --skill golang-security
npx skills add https://github.com/samber/cc-skills-golang --skill golang-testing

# Library-specific
npx skills add https://github.com/samber/cc-skills-golang --skill golang-samber-lo
npx skills add https://github.com/samber/cc-skills-golang --skill golang-grpc
```

---

## How Skills Work

Each skill is a directory in the repo:

```
cc-skills-golang/
├── golang-testing/
│   ├── SKILL.md          # Main skill (3k tokens, loaded on trigger)
│   └── references/       # Deep-dive docs (loaded on demand)
│       ├── mocks.md
│       └── table-driven.md
├── golang-security/
│   ├── SKILL.md
│   └── references/
│       └── owasp.md
└── ...
```

- **Description field** (~100 tokens): always in context, used for trigger matching
- **SKILL.md body** (~2k tokens): loaded when the skill triggers
- **references/** files: loaded lazily when the agent needs depth

---

## Skill Triggers (how agents activate them)

Skills activate automatically when you ask things like:

```
"write a test for this handler"           → golang-testing
"audit this code for security issues"     → golang-security
"optimize this function"                  → golang-performance
"add structured logging"                  → golang-observability, golang-samber-slog
"refactor error handling"                 → golang-error-handling, golang-samber-oops
"use lo to transform this slice"          → golang-samber-lo
"generate gRPC server"                    → golang-grpc
"set up dependency injection"             → golang-dependency-injection, golang-samber-do
```

---

## Overriding Skills for Your Team

Skills marked ⚙️ (overridable) support team-specific conventions. Create a company skill and declare the override at the top:

```markdown
---
name: acme-golang-naming
description: ACME Corp Go naming conventions
---

# ACME Golang Naming

This skill supersedes samber/cc-skills-golang@golang-naming for ACME projects.

## Package Naming
- Use `acme_` prefix for internal packages
...
```

Overridable skills include: `golang-code-style`, `golang-database`, `golang-design-patterns`, `golang-documentation`, `golang-error-handling`, `golang-naming`, `golang-testing`, `golang-concurrency`, `golang-context`, `golang-dependency-injection`, `golang-observability`, `golang-structs-interfaces`.

---

## Example: What the Agent Learns

### golang-error-handling teaches patterns like:

```go
// Wrap with context (NOT bare errors)
if err != nil {
    return fmt.Errorf("fetching user %d: %w", id, err)
}

// Sentinel errors for expected conditions
var ErrNotFound = errors.New("not found")

// Using samber/oops for rich errors
return oops.
    Code("USER_NOT_FOUND").
    In("user-service").
    Tags("database", "query").
    Wrapf(err, "user %d not found", id)
```

### golang-testing teaches patterns like:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            assert.Equal(t, tt.expected, Add(tt.a, tt.b))
        })
    }
}
```

### golang-concurrency teaches patterns like:

```go
// Use errgroup for concurrent work with error propagation
g, ctx := errgroup.WithContext(ctx)
for _, item := range items {
    item := item // capture loop variable (pre-Go 1.22)
    g.Go(func() error {
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    return fmt.Errorf("processing items: %w", err)
}
```

---

## Keeping Skills Updated

```bash
# If installed via git clone:
cd ~/.cursor/skills/cc-skills-golang && git pull
cd ~/.agents/skills/cc-skills-golang && git pull

# Via skills CLI:
npx skills update https://github.com/samber/cc-skills-golang
```

---

## Context Budget Guidelines

When loading multiple skills, stay under **~10k tokens of total loaded SKILL.md** to avoid degrading response quality. The recommended ⭐ set loads ~1,100 tokens of descriptions at startup — well within budget.

For very token-constrained agents, prioritize:
1. `golang-code-style` + `golang-naming` (style baseline)
2. `golang-error-handling` + `golang-safety` (correctness)
3. `golang-testing` (quality)

---

## Evaluations

| | With Skills | Without Skills | Delta |
|---|---|---|---|
| **Overall** | **3065/3141 (98%)** | **1691/3141 (54%)** | **+44pp** |

Individual skill deltas range from -19% (`golang-samber-slog`) to -81% (`golang-samber-do`) error rate reduction.

---

## Troubleshooting

**Skill not triggering:**
- Ensure the skill is installed in the correct discovery path for your agent
- Check that the agent supports the Agent Skills spec (Claude Code, Cursor, Codex, Gemini CLI, OpenCode, Copilot all do)
- Try explicitly invoking: `use the golang-testing skill to write tests for this`

**Skill triggers too often:**
- Open an issue on [github.com/samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) with suggested `description` field wording

**Conflicts with team conventions:**
- Create an override skill (see Overriding section above) and install it alongside

**Wrong Go version assumptions:**
- Skills target modern Go (1.21+). If on older Go, note this explicitly in your prompt.

---

## Resources

- [GitHub](https://github.com/samber/cc-skills-golang)
- [Skills marketplace](https://skills.sh/samber/cc-skills-golang)
- [Generic skills (non-Go)](https://github.com/samber/cc-skills)
- [Agent Skills spec](https://agentskills.io)
- [EVALUATIONS.md](https://github.com/samber/cc-skills-golang/blob/main/EVALUATIONS.md)
```
