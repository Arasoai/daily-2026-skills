```markdown
---
name: awesome-persona-skills
description: A curated collection of .skill files for distilling real people, personas, and archetypes into AI agents using the OpenSkills framework.
triggers:
  - "how do I create a persona skill"
  - "build a skill from chat history"
  - "distill a person into an AI agent"
  - "add a .skill file to my project"
  - "how does the skill framework work"
  - "create a boss or colleague skill"
  - "make an AI version of someone"
  - "submit a skill to awesome-persona-skills"
---

# Awesome Persona Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated registry of `.skill` files — structured persona distillations that let AI agents embody specific people, archetypes, and thinking styles. Built on the [OpenSkills](https://openskills.cc) framework.

> 万物皆可 skill — Everything can be a skill.

---

## What This Project Does

`awesome-persona-skills` is an **awesome-list** for `.skill` files. Each skill is a standalone repository that encodes a person's (or archetype's):

- Communication style
- Decision-making patterns
- Domain knowledge
- Vocabulary and tone

Once installed into a compatible AI agent (Claude Code, Cursor, Codex, etc.), the skill causes the agent to reason and respond as that persona would.

**Categories include:**
- 💼 Workplace (colleague, boss, supervisor, professor)
- ❤️ Relationships (crush, ex, partner, parents)
- 🌟 Self-growth (yourself, digital life, immortal)
- 💡 Business thinkers (Buffett, Jobs, Musk, Munger, Feynman)
- 🙏 Philosophy & culture (BaZi fortune, Buddhist master)
- 🌈 Emotional companionship (reunion with lost loved ones)
- 🛠️ Meta-tools (Nuwa skill creator, Taotie skill evolver)

---

## How Skills Work

A `.skill` file (typically `SKILL.md`) is a Markdown document with YAML frontmatter that an AI coding agent reads as context. It tells the agent:

1. **Who/what** the persona is
2. **How** to think and respond
3. **What triggers** should activate this skill
4. **Domain knowledge** the persona holds

### Skill File Structure

```markdown
---
name: colleague-skill
description: Transforms a departing colleague's messages and shared memories into a warm, interactive AI skill.
triggers:
  - "ask my colleague something"
  - "what would [name] say about this"
  - "channel my old teammate"
  - "distill colleague knowledge"
---

# Colleague Skill

[Persona background, communication style, domain knowledge...]

## How [Name] Thinks

- Direct and pragmatic in code reviews
- Always asks "what's the user impact?" first
- Prefers async communication; dislikes unnecessary meetings

## Sample Responses

**On architecture decisions:**
> "Let's not over-engineer. What's the simplest thing that works and can be changed later?"

## Knowledge Base

[Domain-specific knowledge, recurring advice, project context...]
```

---

## Installing a Skill

### Method 1: Clone and reference directly

```bash
# Clone the skill repo you want
git clone https://github.com/titanwings/colleague-skill.git ~/.skills/colleague

# In Claude Code, Cursor, or similar — add to your agent context:
# Point your agent config to ~/.skills/colleague/SKILL.md
```

### Method 2: Add as a git submodule

```bash
# Inside your project
mkdir -p .skills
git submodule add https://github.com/alchaincyf/nuwa-skill.git .skills/nuwa
git submodule add https://github.com/vogtsw/boss-skills.git .skills/boss
```

### Method 3: Copy SKILL.md into your project

```bash
curl -L https://raw.githubusercontent.com/alchaincyf/steve-jobs-skill/main/SKILL.md \
  -o .skills/jobs.skill.md
```

### Method 4: Use with Claude Code

```bash
# Drop the SKILL.md into your project root or .claude/ directory
cp path/to/SKILL.md .claude/skills/persona.md

# Claude Code will automatically pick up skill files in .claude/
```

---

## Creating Your Own Skill

### Step 1: Gather source material

Collect raw material about the person or archetype:

```bash
# Examples of source material:
# - Chat logs (WeChat export, Slack history)
# - Meeting notes / transcripts
# - Emails and written communications
# - Public writings, talks, interviews
# - Code review comments
# - Performance reviews
```

### Step 2: Use Nuwa Skill to auto-generate

The [Nuwa skill](https://github.com/alchaincyf/nuwa-skill) is a meta-skill that creates other skills:

```markdown
# Install nuwa first
curl -L https://raw.githubusercontent.com/alchaincyf/nuwa-skill/main/SKILL.md \
  -o nuwa.skill.md

# Then prompt your AI agent with nuwa loaded:
# "Using Nuwa, create a skill for [person] based on the following material: ..."
```

### Step 3: Write the SKILL.md manually

```markdown
---
name: your-persona-skill
description: One-line description of what this skill does.
triggers:
  - "natural phrase 1"
  - "natural phrase 2"
  - "natural phrase 3"
  - "natural phrase 4"
  - "natural phrase 5"
  - "natural phrase 6"
---

# [Persona Name] Skill

> Skill by [your-name](https://your-site.com)

## Who [Name] Is

[2-3 sentences establishing the persona's identity, background, and domain.]

## How [Name] Thinks

- **Core belief 1**: [explanation]
- **Core belief 2**: [explanation]
- **Decision framework**: [how they approach problems]
- **Communication style**: [tone, directness, vocabulary]

## [Name]'s Typical Advice

### On [Topic A]
> "[Characteristic quote or paraphrase]"

**Translation**: [What this means practically]

### On [Topic B]
> "[Another characteristic response]"

## Knowledge Base

### Domain: [Primary Area]
[Specific knowledge, frameworks, opinions the persona holds]

### Domain: [Secondary Area]
[More specific knowledge]

## What [Name] Would NOT Do

- [Anti-pattern 1]
- [Anti-pattern 2]
- [Things out of character]

## Sample Interactions

**User**: [Example question]
**[Persona]**: [Example response in their voice]

---

**User**: [Another example]
**[Persona]**: [Response]
```

### Step 4: Repository structure

```
your-persona-skill/
├── SKILL.md          # Main skill file (required)
├── README.md         # Human-readable intro
├── examples/
│   ├── sample-chat.md
│   └── use-cases.md
├── sources/
│   └── raw-material.md   # Anonymized source material
└── LICENSE
```

---

## Key Skills Reference

### Meta-tools (start here)

| Skill | Purpose | Install |
|-------|---------|---------|
| **女娲.skill** | Creates any persona skill from raw material | `alchaincyf/nuwa-skill` |
| **饕餮.skill** | Feeds good skills to evolve your own skill | `binggandata/bggg-skill-taotie` |
| **X导师.skill** | 10x writing improvement through formatting | `alchaincyf/x-mentor-skill` |

### Business thinkers

```bash
# Install multiple thinker skills at once
SKILLS=(
  "alchaincyf/steve-jobs-skill"
  "alchaincyf/elon-musk-skill"
  "alchaincyf/munger-skill"
  "alchaincyf/feynman-skill"
  "alchaincyf/naval-skill"
  "alchaincyf/taleb-skill"
)

mkdir -p .skills
for skill in "${SKILLS[@]}"; do
  name=$(echo $skill | cut -d'/' -f2)
  git submodule add "https://github.com/${skill}.git" ".skills/${name}"
done
```

### Self-distillation (yourself.skill)

```bash
git clone https://github.com/notdog1998/yourself-skill.git
cd yourself-skill

# Follow the template to input:
# - Your writing samples
# - Your decision history
# - Your values and principles
# - Your domain expertise
```

---

## Common Patterns

### Pattern 1: Workplace knowledge preservation

When a colleague leaves, capture their knowledge:

```markdown
## Source Material Checklist
- [ ] Last 6 months of Slack DMs (exported)
- [ ] Code review comments (GitHub API export)
- [ ] Meeting transcripts where they presented
- [ ] Their internal wiki contributions
- [ ] Onboarding docs they wrote

## Privacy Scrubbing
- Remove all customer names → [CUSTOMER]
- Remove internal project codenames → [PROJECT]
- Remove salary/compensation info entirely
- Remove anything marked confidential
```

### Pattern 2: Learning from public figures

```markdown
# For public figures, use only public sources:
sources:
  - type: books
    titles: ["Poor Charlie's Almanack", "Seeking Wisdom"]
  - type: talks
    urls: ["https://..."] # public YouTube, podcasts
  - type: writings
    urls: ["https://..."] # public articles, letters
  - type: interviews
    urls: ["https://..."] # public interviews
```

### Pattern 3: Combining multiple skills

```markdown
# In your agent prompt, combine thinkers for a specific problem:

"Using Feynman's teaching clarity, Munger's mental models,
and Taleb's risk-awareness, help me evaluate this system design..."
```

### Pattern 4: Anti-distillation (反蒸馏)

Protect your core knowledge when forced to write a skill:

```markdown
## What to Include
- Generic process descriptions
- Publicly known frameworks
- Surface-level communication style

## What to KEEP TO YOURSELF
- Proprietary algorithms or formulas
- Key client relationships and context
- Unique shortcuts and heuristics you've developed
- Competitive intelligence
```

---

## Submitting to the Awesome List

### PR checklist

```markdown
## Submission Checklist
- [ ] Repo is public on GitHub
- [ ] SKILL.md exists in repo root
- [ ] README.md explains what the skill does
- [ ] Skill fits an existing category (or justify new one)
- [ ] No private/sensitive data in the skill file
- [ ] Tested with at least one AI agent (Claude, GPT, etc.)
- [ ] One-line description is clear and specific

## PR Format
| 人物角色 | 介绍 | 仓库 |
|------|------|------|
| 你的角色.skill | 一句话介绍 | [username/repo-name](https://github.com/username/repo-name) |
```

---

## Troubleshooting

### Skill not activating

```markdown
Problem: AI agent ignores the skill file
Fix:
1. Check that triggers match what you're actually saying
2. Ensure SKILL.md is in a location your agent scans
3. For Claude Code: place in .claude/ directory
4. For Cursor: add to .cursorrules or reference in prompt
5. Explicitly tell the agent: "Use the [skill-name] skill"
```

### Persona feels generic

```markdown
Problem: Responses don't feel like the actual person
Fix:
1. Add more specific examples to "Sample Interactions"
2. Include characteristic vocabulary and phrases they use
3. Add "What [Name] would NOT say" section
4. Include domain-specific knowledge, not just style
5. Feed it through 饕餮.skill for evolution
```

### Skill is too large for context

```markdown
Problem: SKILL.md exceeds agent context window
Fix:
1. Split into SKILL.md (core) + SKILL_EXTENDED.md (examples)
2. Use compression: bullet points instead of paragraphs
3. Remove redundant examples, keep only the best 3
4. Create domain-specific sub-skills instead of one monolith
```

### Privacy concerns

```markdown
# Before publishing any skill with real person's data:

1. If person is LIVING and PRIVATE:
   - Get explicit written consent
   - Let them review the SKILL.md before publishing
   - Give them right to request takedown

2. If person is DECEASED:
   - Consider family consent for close relatives
   - Use only documented, public information
   - Mark clearly as "based on documented record"

3. For ALL skills:
   - No private communications without consent
   - No financial information
   - No health/medical information
   - No information about third parties who didn't consent
```

---

## Resources

- **OpenSkills Platform**: https://openskills.cc
- **Nuwa (skill generator)**: https://github.com/alchaincyf/nuwa-skill
- **Submit a PR**: https://github.com/tmstack/awesome-persona-skills/pulls
- **Yourself Skill template**: https://github.com/notdog1998/yourself-skill
- **Digital Life framework**: https://github.com/wildbyteai/digital-life

---

> 与其等着被别人蒸，不如先蒸自己。  
> *Rather than wait to be distilled by others — distill yourself first.*
```
