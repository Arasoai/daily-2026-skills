```markdown
---
name: khazix-skills-agent
description: Expert in the Agent Skills open standard and the khazix-skills collection — installing, creating, and using structured AI skill modules for agents like Claude Code, Codex, and OpenClaw.
triggers:
  - install a skill from github
  - how do I use agent skills
  - create a new skill for my agent
  - add khazix writing skill to claude code
  - what is the agent skills standard
  - load a skill into codex or cursor
  - how do I write a SKILL.md file
  - manually install a skill package
---

# Khazix Skills — Agent Skills Open Standard

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

**khazix-skills** is an open-source collection of production-tested AI Skills authored by 数字生命卡兹克 (KKKKhazix). Each Skill follows the [Agent Skills](https://agentskills.io) open standard — a portable, composable format for packaging domain expertise into structured instruction sets that AI agents can load on demand.

A **Skill** is a folder (or `.skill` package) containing:
- A `SKILL.md` instruction file with YAML frontmatter
- Optional scripts, templates, and reference resources
- Metadata that tells agents *when* to auto-load the skill (triggers)

Agents like Claude Code, Codex, and OpenClaw automatically activate relevant Skills based on context, or you can invoke them manually with `/skill-name`.

---

## Installing Skills

### Method 1 — Via Agent Dialog (Recommended)

In any supported agent (Claude Code, Codex, OpenClaw), simply say:

```
安装这个 skill：https://github.com/KKKKhazix/khazix-skills
```

Or in English:

```
Install the skill from: https://github.com/KKKKhazix/khazix-skills
```

The agent will clone the repo, parse each Skill folder, and place the files in your local skills directory.

### Method 2 — Manual Installation

1. Go to the [Releases page](https://github.com/KKKKhazix/khazix-skills/releases)
2. Download the `.skill` file for the skill you want
3. Move it to your agent's skills directory:

| Agent | Skills Directory |
|-------|-----------------|
| Claude Code | `~/.claude/skills/` |
| OpenClaw | `~/.openclaw/skills/` |
| Codex | `~/.agents/skills/` |

```bash
# Example: manually install for Claude Code
mkdir -p ~/.claude/skills/
cp kaizike-writer.skill ~/.claude/skills/

# Or clone the whole repo
git clone https://github.com/KKKKhazix/khazix-skills.git
cp -r khazix-skills/kaizike-writer ~/.claude/skills/
```

### Method 3 — Clone and Symlink (for Development)

```bash
git clone https://github.com/KKKKhazix/khazix-skills.git ~/skills/khazix-skills

# Symlink individual skills so updates pull through automatically
ln -s ~/skills/khazix-skills/kaizike-writer ~/.claude/skills/kaizike-writer
```

---

## Available Skills

### `kaizike-writer`

A long-form WeChat public account writing skill with:
- Complete writing style rules (卡兹克风格)
- A four-layer self-review system (四层自检体系)
- Content methodology
- Style example library

**Invoke manually:**
```
/kaizike-writer 写一篇关于 AI Agent 的深度文章
```

**Auto-trigger phrases** (the agent activates this skill when you say things like):
- "帮我写一篇公众号文章"
- "写一篇长文"
- "用卡兹克风格写"

---

## The Agent Skills Standard — SKILL.md Format

Every skill is defined by a `SKILL.md` file with this structure:

```markdown
---
name: my-skill-name          # kebab-case, used as the /command name
description: One-line summary of what this skill does.
triggers:
  - natural phrase a user might say
  - another trigger phrase
  - up to 6-8 triggers total
---

# My Skill Name

Full instructions, methodology, examples, and resources go here.
The agent reads this entire file when the skill is activated.
```

### YAML Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✅ | kebab-case identifier, becomes the `/command` |
| `description` | ✅ | One sentence shown in skill listings |
| `triggers` | ✅ | 6–8 phrases that cause auto-activation |
| `version` | optional | Semver string, e.g. `"1.0.0"` |
| `author` | optional | Author name or handle |
| `tags` | optional | Array of topic tags |

---

## Creating Your Own Skill

### Minimal Skill Structure

```
my-skill/
├── SKILL.md          # Required — main instruction file
├── examples/         # Optional — reference examples
│   └── example-1.md
└── resources/        # Optional — templates, checklists, etc.
    └── checklist.md
```

### Example: Creating a Code Review Skill

```markdown
---
name: code-review-expert
description: Performs thorough code reviews following team standards and security best practices.
triggers:
  - review this code
  - do a code review
  - check my pull request
  - find issues in this file
  - audit this code for security
  - give me feedback on my implementation
---

# Code Review Expert

## Review Checklist

### Layer 1 — Correctness
- [ ] Logic errors or off-by-one mistakes
- [ ] Edge cases (null, empty, overflow)
- [ ] Error handling completeness

### Layer 2 — Security
- [ ] Input validation and sanitization
- [ ] No hardcoded secrets or credentials
- [ ] SQL injection / XSS vectors

### Layer 3 — Performance
- [ ] N+1 query patterns
- [ ] Unnecessary re-renders or recomputation
- [ ] Memory leaks

### Layer 4 — Maintainability
- [ ] Naming clarity
- [ ] Function/class size
- [ ] Test coverage

## Output Format

Always structure feedback as:
1. **Summary** (2-3 sentences)
2. **Critical Issues** (blockers)
3. **Suggestions** (improvements)
4. **Positives** (what's done well)
```

### Packaging a Skill as `.skill`

A `.skill` file is a ZIP archive renamed with the `.skill` extension:

```bash
# Package a skill for distribution
cd my-skill/
zip -r ../my-skill.skill .
# Rename if needed: mv my-skill.zip my-skill.skill
```

---

## Using Skills in Practice

### Auto-activation

Skills activate automatically when your message matches a trigger phrase. No action needed — the agent detects context and loads the skill.

### Manual Invocation

```
/kaizike-writer 写一篇讲 MCP 协议的深度长文，目标读者是技术从业者
```

```
/code-review-expert Please review the authentication module in auth/middleware.ts
```

### Combining Multiple Skills

Skills are composable. You can invoke multiple skills in one session:

```
Use /kaizike-writer style, but also apply /seo-optimizer rules when structuring the headings.
```

---

## Listing and Managing Installed Skills

```bash
# List installed skills for Claude Code
ls ~/.claude/skills/

# Check skill metadata
cat ~/.claude/skills/kaizike-writer/SKILL.md | head -20

# Remove a skill
rm -rf ~/.claude/skills/kaizike-writer

# Update skills from a cloned repo
cd ~/skills/khazix-skills && git pull
```

---

## Troubleshooting

### Skill not auto-activating

- Check that your message contains words from the `triggers` list in `SKILL.md`
- Verify the skill file is in the correct directory for your agent
- Ensure `SKILL.md` has valid YAML frontmatter (no tab characters, correct indentation)

### YAML parse errors

```bash
# Validate YAML frontmatter with Python
python3 -c "
import yaml, sys
with open('SKILL.md') as f:
    content = f.read()
parts = content.split('---', 2)
yaml.safe_load(parts[1])
print('YAML is valid')
"
```

### Skill installs but agent ignores it

- Restart your agent session after installing a new skill
- Some agents require a full reload: `claude --reload-skills` or restart the CLI
- Check that the skill `name` in frontmatter matches the folder name

### `/skill-name` command not found

- The command name is the `name` field in frontmatter, not the folder name
- Folder: `kaizike-writer/` → command: `/kaizike-writer` (they should match by convention)
- Verify: `grep '^name:' ~/.claude/skills/kaizike-writer/SKILL.md`

---

## Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **Skill** | A folder with `SKILL.md` + optional resources |
| **Trigger** | A natural-language phrase that auto-activates a skill |
| **Manual invocation** | `/skill-name` syntax to explicitly call a skill |
| **`.skill` file** | ZIP-packaged skill for easy distribution |
| **Composability** | Multiple skills can work together in one session |
| **Portability** | Same skill works across Claude Code, Codex, OpenClaw |

---

## Resources

- [Agent Skills Open Standard](https://agentskills.io)
- [khazix-skills Repository](https://github.com/KKKKhazix/khazix-skills)
- [Releases / .skill Downloads](https://github.com/KKKKhazix/khazix-skills/releases)
- [MIT License](https://github.com/KKKKhazix/khazix-skills/blob/main/LICENSE)
```
