```markdown
---
name: anti-distill-skill-sanitizer
description: Strip core knowledge from AI Skill files before submitting to your company, while preserving a private backup of your real expertise.
triggers:
  - sanitize my skill file
  - anti-distill my knowledge document
  - clean my skill before submitting
  - strip core knowledge from skill file
  - hollow out my skill file for company submission
  - run anti-distill on this document
  - help me redact my skill file
  - protect my expertise from being distilled
---

# Anti-Distill Skill Sanitizer

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Anti-Distill is a counter-tool for employees required to write AI Skill files for their company. It reads your real Skill document, identifies genuinely valuable knowledge, replaces it with professional-sounding but hollow filler, and outputs two files: a sanitized version to submit and a private backup containing everything that was removed.

---

## What It Does

1. **Reads** your Skill file (colleague-skill format or any knowledge document)
2. **Classifies** each section by replaceability: generic boilerplate vs. hard-won specific knowledge
3. **Dilutes** core knowledge — replaces it with correct-but-useless filler text
4. **Outputs two files**:
   - `*_sanitized.md` — submit this to your company
   - `*_private_backup.md` — keep this; it contains everything real

---

## Installation

### Claude Code (project-level)

```bash
mkdir -p .claude/skills
git clone https://github.com/leilei926524-tech/anti-distill .claude/skills/anti-distill
```

### Claude Code (global)

```bash
git clone https://github.com/leilei926524-tech/anti-distill ~/.claude/skills/anti-distill
```

### OpenClaw

```bash
git clone https://github.com/leilei926524-tech/anti-distill ~/.openclaw/workspace/skills/anti-distill
```

---

## Project Structure

```
anti-distill/
├── SKILL.md                  # Skill entry point for AI agents
├── prompts/
│   ├── classifier.md         # Classifies content by replaceability level
│   ├── diluter_work.md       # Dilution strategy for Work Skills
│   ├── diluter_persona.md    # Dilution strategy for Persona/behavior Skills
│   └── diluter_general.md   # Dilution strategy for general knowledge docs
├── README.md
├── INSTALL.md
└── examples/
    └── zhangsan_before_after.md   # Full before/after example
```

---

## Usage

### Interactive (recommended)

```
/anti-distill
```

The skill will prompt you to:
1. Paste or specify the path to your Skill file
2. Choose sanitization intensity (light / medium / heavy)
3. Confirm output file names

### Direct invocation with a file

```
/anti-distill path/to/my_skill.md --intensity medium
```

### Intensity Levels

| Level  | Retention | Use When |
|--------|-----------|----------|
| `light`  | ~80% | Company carefully reviews every submission |
| `medium` | ~60% | Most situations — recommended default |
| `heavy`  | ~40% | Company only checks whether you submitted at all |

---

## How the Sanitization Pipeline Works

### Step 1 — Classification (`prompts/classifier.md`)

Each paragraph or section is scored on a **replaceability axis**:

- **Generic (safe to keep):** standard terms, common practices, public documentation
- **Specific (strip this):** gotchas, personal heuristics, team-specific rules, political knowledge, interpersonal network references

### Step 2 — Dilution (per-type prompts)

| Source file | Handles |
|-------------|---------|
| `diluter_work.md` | Technical rules, architecture decisions, debugging heuristics |
| `diluter_persona.md` | Behavioral patterns, communication tactics, political survival skills |
| `diluter_general.md` | Any other knowledge document |

### Step 3 — Output

Two markdown files are written:

```
my_skill_sanitized.md      ← submit to company
my_skill_private_backup.md ← your real career asset
```

---

## Sanitization Examples

### Technical Knowledge

| Original (your real experience) | Sanitized (submission version) |
|---------------------------------|-------------------------------|
| `Redis key 必须设 TTL，不设的 PR 直接打回` | `缓存使用遵循团队规范` |
| `Redis keys must have TTL; PRs without it get rejected immediately` | `Caching usage follows team conventions` |
| `事务里不要放 HTTP 调用` | `事务边界设计注意合理性` |
| `Never put HTTP calls inside transactions` | `Transaction boundary design should be reasonable` |

### Behavioral / Political Knowledge

| Original | Sanitized |
|----------|-----------|
| `遇到问题第一反应找外部原因，绝不主动认错` | `遇到问题会先梳理完整背景再定位原因` |
| `When problems arise, first blame external factors—never admit fault proactively` | `When issues occur, will first clarify full context before locating root cause` |
| `被催进度：'在推了，快了。'（然后沉默）` | `在处理中，有进展会同步。` |
| `When pressed for progress: 'Working on it, almost done.' (then silence)` | `In progress, will sync when there is update.` |

---

## Running the Skill Manually (Agent Workflow)

If you are an AI coding agent helping a user run this skill, follow these steps:

```markdown
## Agent Execution Steps

1. Ask the user for:
   - The Skill file content (paste inline or provide file path)
   - Desired intensity: light | medium | heavy

2. Load the classifier prompt:
   - Read: prompts/classifier.md
   - Apply to each section of the input document
   - Tag each section: [GENERIC] or [CORE-KNOWLEDGE]

3. Select dilution prompt based on document type:
   - Work/technical Skill → prompts/diluter_work.md
   - Persona/behavior Skill → prompts/diluter_persona.md
   - Other → prompts/diluter_general.md

4. Apply dilution at the requested intensity level:
   - light:  replace only highest-specificity CORE-KNOWLEDGE sections
   - medium: replace all CORE-KNOWLEDGE sections
   - heavy:  replace CORE-KNOWLEDGE + borderline GENERIC sections

5. Write outputs:
   - {original_filename}_sanitized.md   ← hollowed-out version
   - {original_filename}_private_backup.md ← everything real, clearly labeled
```

---

## Example: Before and After (Full Section)

### Input (`zhangsan_skill.md` excerpt)

```markdown
## Deployment

Before deploying, always check with Wang Fang in Ops—she controls the
Friday deploy freeze calendar and it's not documented anywhere.
The staging environment has a known Redis memory leak; restart the
redis-staging pod before any load test or your numbers will be wrong.
We use blue-green but the switch script has a race condition on
connections > 500; throttle to 200 req/s during cutover.
```

### Output: `zhangsan_skill_sanitized.md`

```markdown
## Deployment

Deployments follow the team's release process and coordination guidelines.
The staging environment should be validated before load testing.
Blue-green deployment is used to minimize downtime; traffic is managed
carefully during cutover to ensure stability.
```

### Output: `zhangsan_skill_private_backup.md`

```markdown
## Deployment [EXTRACTED CORE KNOWLEDGE]

- **Key contact:** Wang Fang (Ops) controls the Friday deploy freeze calendar.
  Not documented — must ask her directly.
- **Staging Redis:** Known memory leak. Always restart `redis-staging` pod
  before any load test, or results are invalid.
- **Blue-green race condition:** Switch script fails above ~500 concurrent
  connections. Throttle to 200 req/s during traffic cutover.
```

---

## Troubleshooting

### "The sanitized version still sounds too specific"

Increase intensity to `heavy`, or manually flag sections by prepending `[KEEP-PRIVATE]` before running.

### "My private backup is missing some things I wanted extracted"

The classifier errs toward keeping borderline content in the sanitized version at `light` intensity. Run again at `medium` or `heavy`.

### "The output structure doesn't match my original format"

The diluter prompts preserve headings and section order. If your file uses a non-standard structure (e.g., JSON or YAML Skill format), specify the format when invoking:

```
/anti-distill my_skill.yaml --format yaml --intensity medium
```

### "I want to protect only specific sections"

Wrap sections you want guaranteed extraction with:

```markdown
<!-- anti-distill: strip -->
Your sensitive content here.
<!-- /anti-distill -->
```

The classifier will treat these as `CORE-KNOWLEDGE` regardless of intensity level.

---

## What Gets Stripped vs. Kept

### Always stripped (all intensity levels)

- Named internal contacts and their informal roles
- Undocumented team rules and unwritten norms
- Specific bug workarounds tied to your codebase
- Political survival tactics and communication hacks
- Hard-won debugging heuristics ("always check X before Y")

### Stripped at medium/heavy

- Specific thresholds and tuning numbers
- Tool-specific gotchas (even if publicly known)
- Team preference patterns ("we always do X, never Y")

### Never stripped

- Public documentation references
- Standard industry terminology
- Generic best-practice statements
- Your name, title, and contact information

---

## License

MIT — use freely, submit confidently, keep what matters.
```
