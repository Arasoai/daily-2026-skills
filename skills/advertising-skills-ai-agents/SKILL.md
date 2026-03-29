```markdown
---
name: advertising-skills-ai-agents
description: Skill library for encoding direct-response advertising, copywriting, and funnel thinking into AI coding agents using the Agent Skills pattern.
triggers:
  - set up advertising skills for my AI agent
  - install advertising skills skill library
  - use advertising skills with Claude Code
  - build a booked call campaign with AI
  - add direct response copywriting skills to my agent
  - configure advertising skills for cursor or codex
  - run a funnel campaign orchestrator
  - chain advertising skills for paid ads
---

# Advertising Skills for AI Agents

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A structured, opinionated skill library that encodes direct-response advertising, Eugene Schwartz copywriting, funnel architecture, and paid media strategy into AI coding agents. Skills are chained, not prompted — the system builds strategy before writing a single word of copy.

---

## What This Project Does

Advertising Skills gives AI agents (Claude Code, Cursor, Codex, Windsurf, Open Claw, custom frameworks) a reusable library of focused advertising "skills" — each one a `SKILL.md` file that defines:

- **When** to use it
- **What inputs** it needs
- **How it thinks** (the reasoning layer)
- **What it outputs**
- **What it must not do**

Skills are grouped into five categories and designed to be chained in sequence, producing output that is positioned, specific, and conversion-focused — not generic AI marketing copy.

---

## Installation

### Step 1: Clone or download the repository

```bash
git clone https://github.com/realkimbarrett/advertising-skills.git
```

### Step 2: Install skills into your agent's skill directory

```bash
cp -r advertising-skills/skills .agents/skills/
```

Or for a specific orchestrator only:

```bash
cp -r advertising-skills/skills/orchestrators/booked-call-campaign-orchestrator \
  .agents/skills/booked-call-campaign-orchestrator
```

### Step 3: Verify structure

Your project should now contain:

```
.agents/
  skills/
    foundations/
      avatar-extraction/
        SKILL.md
      voice-of-customer-miner/
        SKILL.md
      offer-extraction/
        SKILL.md
      proof-harvester/
        SKILL.md
      positioning-core/
        SKILL.md
    copy-chief/
      schwartz-awareness-mapper/
        SKILL.md
      market-sophistication-analyzer/
        SKILL.md
      dominant-desire-extractor/
        SKILL.md
      mechanism-builder/
        SKILL.md
      headline-matrix/
        SKILL.md
      objection-crusher/
        SKILL.md
      close-architect/
        SKILL.md
    operator-os/
      scroll-stopping-creative/
        SKILL.md
      ad-angle-multiplier/
        SKILL.md
      conversion-path-builder/
        SKILL.md
      landing-page-architect/
        SKILL.md
      email-sequence-architect/
        SKILL.md
      performance-diagnosis/
        SKILL.md
      testing-roadmap/
        SKILL.md
    qa/
      generic-language-killer/
        SKILL.md
      claim-checker/
        SKILL.md
      consistency-auditor/
        SKILL.md
      final-polish/
        SKILL.md
    orchestrators/
      full-funnel-campaign-orchestrator/
        SKILL.md
      booked-call-campaign-orchestrator/
        SKILL.md
      campaign-diagnosis-orchestrator/
        SKILL.md
```

---

## Skill File Structure

Every `SKILL.md` follows this pattern:

```markdown
# Skill Name

## When to Use This Skill
[Trigger conditions]

## Inputs Required
[What the agent must gather before running]

## How to Think
[The reasoning process — the most important section]

## Output Format
[What the agent should produce]

## Do Not
[Hard constraints — what this skill must never do]
```

---

## Core Skill Categories

### Foundations
Run these first. They define reality before any copy is written.

| Skill | Purpose |
|---|---|
| `avatar-extraction` | Defines the exact person being targeted — demographics, desires, fears, language |
| `voice-of-customer-miner` | Extracts real customer language from reviews, forums, interviews |
| `offer-extraction` | Clarifies what is actually being sold and why it matters |
| `proof-harvester` | Collects credible, specific proof elements |
| `positioning-core` | Establishes the market position before a single headline is written |

### Copy Chief
The Eugene Schwartz layer — awareness, sophistication, mechanism, and conversion.

| Skill | Purpose |
|---|---|
| `schwartz-awareness-mapper` | Maps prospect awareness level (unaware → most aware) |
| `market-sophistication-analyzer` | Determines how saturated the market is and what level of claim to lead with |
| `dominant-desire-extractor` | Identifies the single strongest desire driving the prospect |
| `mechanism-builder` | Builds the unique mechanism — the "how" that makes the offer believable |
| `headline-matrix` | Generates headlines matched to awareness level and desire |
| `objection-crusher` | Surfaces and pre-empts the real objections |
| `close-architect` | Structures the call to action and close logic |

### Operator OS
Execution, platform, and scale layer.

| Skill | Purpose |
|---|---|
| `scroll-stopping-creative` | Structures ad creative for Meta, TikTok, YouTube |
| `ad-angle-multiplier` | Generates multiple angles from a single offer |
| `conversion-path-builder` | Maps the prospect journey from ad to conversion |
| `landing-page-architect` | Structures landing pages for the defined awareness level |
| `email-sequence-architect` | Builds nurture and conversion email sequences |
| `performance-diagnosis` | Diagnoses underperforming campaigns from metrics |
| `testing-roadmap` | Builds a structured creative/copy testing plan |

### QA
Output quality filters — run these last on any copy or funnel output.

| Skill | Purpose |
|---|---|
| `generic-language-killer` | Flags and rewrites vague, forgettable language |
| `claim-checker` | Validates that claims are specific, credible, and provable |
| `consistency-auditor` | Checks that messaging is consistent across all funnel touchpoints |
| `final-polish` | Final read for tone, rhythm, and conversion readiness |

### Orchestrators
Pre-built multi-skill chains for common campaign types.

| Orchestrator | Skills Chained |
|---|---|
| `booked-call-campaign-orchestrator` | avatar-extraction → offer-extraction → schwartz-awareness-mapper → mechanism-builder → ad-angle-multiplier → scroll-stopping-creative → conversion-path-builder → objection-crusher → generic-language-killer |
| `full-funnel-campaign-orchestrator` | Full chain from foundations through copy chief, operator OS, and QA |
| `campaign-diagnosis-orchestrator` | performance-diagnosis → testing-roadmap → ad-angle-multiplier |

---

## Usage with Claude Code

### Run a single skill

```
@.agents/skills/foundations/avatar-extraction/SKILL.md

My offer: 1-1 coaching for B2B founders who want to run profitable Meta ads without hiring an agency.
```

### Chain skills manually

```
Step 1 — Run avatar-extraction with the above offer brief.
Step 2 — Feed the avatar output into schwartz-awareness-mapper.
Step 3 — Feed awareness level into mechanism-builder.
Step 4 — Feed mechanism into headline-matrix.
Step 5 — Run generic-language-killer on all headlines.
```

### Use an orchestrator (recommended for full campaigns)

```
@.agents/skills/orchestrators/booked-call-campaign-orchestrator/SKILL.md

Business: B2B SaaS. Offer: 30-day onboarding sprint. Target: ops leads at 50-200 person companies.
Proof: 3 case studies, avg 40% reduction in onboarding time.
Platform: Meta ads, landing page, booked-call funnel.
```

---

## Usage with Cursor / Windsurf / Codex

Reference the skill file directly in your agent context or rules:

```
# .cursorrules or agent system prompt addition

You have access to the following advertising skills:
- .agents/skills/foundations/
- .agents/skills/copy-chief/
- .agents/skills/operator-os/
- .agents/skills/qa/
- .agents/skills/orchestrators/

When asked to write ads, landing pages, email sequences, or diagnose campaigns,
read the relevant SKILL.md files and follow the thinking process defined in each.
Always run foundation skills before copy chief skills.
Always run QA skills last.
```

---

## Example: Building a Booked-Call Campaign

### Input

```
Offer: Strategy call for ecommerce brands doing $500k–$5M/yr who want to profitably scale Meta ads.
Proof: 47 clients, avg 3.2x ROAS improvement in 90 days.
Platform: Meta feed and reels → VSL landing page → call booking.
```

### Skill Chain Output (abbreviated)

**Avatar Extraction output:**
```
Name: Marcus
Age: 34
Role: Founder/CEO, ecommerce brand
Revenue: ~$1.2M/yr
Current pain: Spending $15k/mo on Meta, ROAS declining, no idea why
Fear: Burning cash without a clear path to fixing it
Desire: A system that scales without constant babysitting
Language used: "Our ads stopped working", "CPMs are insane right now", "I don't know what to test next"
```

**Schwartz Awareness Level:**
```
Level: Solution Aware
Prospect knows Meta ads exist and has tried them. They're not looking for education —
they're looking for someone who can fix what isn't working.
Lead with: credibility, specific mechanism, proof of result.
Do NOT lead with: what Meta ads are, why paid traffic matters.
```

**Mechanism:**
```
The Profitable Scale System:
1. Diagnose the real bottleneck (creative fatigue vs. audience saturation vs. offer mismatch)
2. Fix the constraint with targeted tests, not broad changes
3. Scale only what is working at margin
This is why clients see ROAS improvements in 90 days — not because we run more ads,
but because we stop running the wrong ones.
```

**Headline Matrix:**
```
[Solution Aware — Pain-led]
"Your Meta ads didn't stop working. They were never set up to scale."

[Solution Aware — Mechanism-led]
"Most ecommerce brands fix the wrong thing. Here's the diagnostic we use instead."

[Solution Aware — Proof-led]
"47 brands. 3.2x average ROAS lift. 90 days. Here's the system."
```

**Generic Language Killer:**
```
FLAGGED: "proven strategies" → REWRITE: "the diagnostic that found the constraint"
FLAGGED: "scale your business" → REWRITE: "increase ROAS from 1.4x to 3.8x in 90 days"
FLAGGED: "expert team" → REWRITE: "47 clients, avg 3.2x ROAS improvement"
```

---

## Common Patterns

### Pattern 1: Diagnosis before creation
Always run `performance-diagnosis` before any creative work on an existing campaign.

```
@.agents/skills/operator-os/performance-diagnosis/SKILL.md

Campaign: Meta, $8k/mo spend, CTR 1.2%, CPL $180, target CPL $90.
Creative: 4 static images, 2 videos. Running for 6 weeks.
Landing page: VSL + form. Conversion rate: 12%.
```

### Pattern 2: Awareness-first copy
Never write a headline without first running `schwartz-awareness-mapper`. The awareness level determines the entire lead strategy.

### Pattern 3: Mechanism before offer
Run `mechanism-builder` before `offer-extraction` when the market is sophisticated (level 3–5). The mechanism is what separates the offer from the noise.

### Pattern 4: QA as a final gate
Always end any copy or funnel build with the full QA chain:
```
generic-language-killer → claim-checker → consistency-auditor → final-polish
```

---

## Troubleshooting

### Agent produces generic output despite skills being installed

**Cause:** Agent is not reading the SKILL.md files — it's defaulting to general knowledge.

**Fix:** Explicitly reference the skill file in your prompt:
```
Read .agents/skills/copy-chief/schwartz-awareness-mapper/SKILL.md fully before responding.
Then apply the thinking process defined there to this offer: [offer brief]
```

### Skills are installed but agent ignores the "Do Not" constraints

**Fix:** Add the constraints to your agent rules or system prompt:
```
# In .cursorrules or Claude Code CLAUDE.md:
When using advertising skills, always respect the "Do Not" section of each SKILL.md.
These are hard constraints, not suggestions.
```

### Output is technically correct but doesn't sound like direct response copy

**Cause:** Agent is not running `generic-language-killer` or `final-polish`.

**Fix:** Always end the chain with QA skills. Add this to your orchestrator prompt:
```
After completing all creative and copy output, run the following QA skills in order:
1. generic-language-killer
2. claim-checker
3. final-polish
Do not output final copy until all three QA skills have been applied.
```

### Orchestrator skips foundation skills

**Fix:** State the required sequence explicitly:
```
Do not begin copy-chief or operator-os skills until avatar-extraction,
offer-extraction, and schwartz-awareness-mapper have been completed.
Wait for explicit confirmation of each foundation skill output before proceeding.
```

---

## Reference

- **Homepage:** https://advertisingskills.io/
- **Repository:** https://github.com/realkimbarrett/advertising-skills
- **Example output:** `/examples/booked-call-campaign-ai-ads.md`
- **Author:** Kim Barrett
- **Compatible agents:** Claude Code, Cursor, Windsurf, Codex, Open Claw, custom frameworks
- **License:** MIT
```
