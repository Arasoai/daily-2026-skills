```markdown
---
name: nuwa-skill-distiller
description: AI skill for distilling any public figure's cognitive frameworks, mental models, and decision heuristics into runnable Claude Code perspective skills using the Nuwa methodology
triggers:
  - distill a person into a skill
  - create a perspective skill for someone
  - extract mental models from a thinker
  - build a cognitive framework skill
  - nuwa distill someone
  - make a skill based on how someone thinks
  - create a thinking style skill
  - distill how someone makes decisions
---

# Nuwa Skill Distiller

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Nuwa (女娲) is a Claude Code meta-skill that distills any public figure's **cognitive operating system** — mental models, decision heuristics, expression DNA, and value anti-patterns — into a standalone, runnable `SKILL.md` perspective file. It is not role-play; it extracts verifiable reasoning frameworks from public sources and builds a structured skill that reasons *like* the person, not *as* the person.

---

## Installation

```bash
npx skills add alchaincyf/nuwa-skill
```

Then in Claude Code, trigger it naturally:

```
> 蒸馏一个保罗·格雷厄姆
> 造一个张小龙的视角Skill
> Distill Paul Graham into a skill
> Build a Charlie Munger perspective skill
> 帮我做一个段永平的Skill
```

### Installing Pre-Built Skills

Seven figures are already distilled and published as standalone skills:

```bash
# Steve Jobs — product/design/strategy
npx skills add alchaincyf/steve-jobs-skill

# Elon Musk — engineering/first principles/cost
npx skills add alchaincyf/elon-musk-skill

# Charlie Munger — investing/inversion/multi-model thinking
npx skills add alchaincyf/munger-skill

# Richard Feynman — learning/teaching/scientific thinking
npx skills add alchaincyf/feynman-skill

# Naval Ravikant — wealth/leverage/life philosophy
npx skills add alchaincyf/naval-skill

# Nassim Taleb — risk/antifragility/uncertainty
npx skills add alchaincyf/taleb-skill

# 张雪峰 — education/career planning/class mobility
npx skills add alchaincyf/zhangxuefeng-skill

# X Mentor (topic skill, not person) — X/Twitter growth methodology
npx skills add alchaincyf/x-mentor-skill
```

---

## What Nuwa Distills

Nuwa extracts **five layers** of cognition, not surface habits:

| Layer | What It Captures |
|---|---|
| **Expression DNA** | Tone, rhythm, vocabulary patterns, sentence structure |
| **Mental Models** | Cognitive frameworks used to interpret reality |
| **Decision Heuristics** | Rules of thumb for judgment under uncertainty |
| **Anti-Patterns** | What they refuse to do; value-line violations |
| **Honest Boundaries** | What the distillation cannot reliably reproduce |

---

## How to Trigger a Distillation

After installing Nuwa, speak naturally in Claude Code:

```
> Distill Richard Feynman
> 蒸馏一个费曼
> Create a perspective skill for Warren Buffett
> Build me a Steve Wozniak thinking skill
> 造一个乔布斯的视角
```

Nuwa will run its four-phase pipeline automatically and produce a `SKILL.md` file.

---

## The Four-Phase Distillation Pipeline

### Phase 1 — Six-Path Parallel Research

Nuwa deploys six research agents simultaneously:

```
Agent 1: Books, essays, long-form written work
Agent 2: Podcasts, interviews, recorded speech
Agent 3: Social media, public posts, short-form
Agent 4: Critics and adversarial perspectives
Agent 5: Decision records (what they actually built/chose)
Agent 6: Life timeline (formative experiences, inflection points)
```

Each agent stores findings independently before synthesis to prevent cross-contamination.

### Phase 2 — Triple-Verification Filter

A candidate mental model is only included if it passes **all three gates**:

```
Gate 1: Cross-domain recurrence
         → Appears in 2+ unrelated domains (not a one-off comment)

Gate 2: Predictive power
         → Can we infer their stance on a NEW problem they haven't addressed?

Gate 3: Exclusivity
         → Not something every smart person would say (has discriminating power)
```

Models failing any gate are discarded.

### Phase 3 — SKILL.md Construction

The output follows the skill template in `references/skill-template.md`:

```markdown
---
name: [person]-perspective
description: Reason through problems using [Person]'s cognitive frameworks
triggers:
  - [6-8 natural trigger phrases]
---

# [Person] Perspective

## Mental Models (3–7)
[Each model: name, one-line definition, example application]

## Decision Heuristics (5–10)
[Each heuristic: the rule + when it applies + what it rules out]

## Expression DNA
[Tone markers, sentence patterns, vocabulary tendencies, pacing]

## Value Anti-Patterns
[What this person would refuse to do or say; hard no's]

## Honest Boundaries
[What this distillation cannot reproduce reliably]
```

### Phase 4 — Quality Validation

```
Test Set A (3 questions): Questions the person publicly answered.
  → Skill output must align directionally with recorded answers.

Test Set B (1 question): A topic they never addressed publicly.
  → Skill must express calibrated uncertainty, not false confidence.
```

A skill that fails Test Set B is overfit and gets rebuilt.

---

## Repository Structure

```
nuwa-skill/
├── SKILL.md                          # Nuwa itself (the meta-skill)
├── references/
│   ├── extraction-framework.md       # Full methodology documentation
│   └── skill-template.md             # Template for generated skills
└── examples/
    ├── steve-jobs-perspective/        # ⭐ Includes multi-turn dialogue logs
    │   ├── SKILL.md
    │   ├── research/
    │   │   ├── books-essays.md
    │   │   ├── interviews-speech.md
    │   │   ├── critics-view.md
    │   │   ├── decision-records.md
    │   │   ├── social-timeline.md
    │   │   └── synthesis-notes.md
    │   └── validation-dialogues.md
    ├── elon-musk-perspective/
    ├── naval-perspective/
    ├── munger-perspective/
    ├── feynman-perspective/
    ├── taleb-perspective/
    └── zhangxuefeng-perspective/
```

Each `examples/` directory contains the full research trail so you can audit how raw sources became structured mental models.

---

## Using a Distilled Skill

After installing a person-skill, invoke it in Claude Code:

```
> 用芒格的视角帮我分析这个投资决策
> Use Munger's perspective to evaluate this business
> 费曼会怎么解释量子计算？
> How would Feynman explain this concept to a beginner?
> 切换到Naval，我在纠结三件事同时做
> Switch to Naval — I'm trying to do three things at once
```

### Example: Naval on Competing Priorities

```
User  > I want to do indie hacking, write a book, AND build an audience
        simultaneously. Not enough time.

Naval > You listed three desires.
        Each desire is a contract you signed with unhappiness.
        
        The problem isn't capacity — it's too many open contracts.
        
        Ask yourself: which one makes you forget time when you're doing it?
        That's where your specific knowledge lives.
        
        Not three-to-one. One, then one, then one.
        Serial compounding, not parallel dilution.
```

### Example: Musk on High CAC

```
User  > Our SaaS customer acquisition cost is too high.

Musk  > Don't start by optimizing. Start by calculating the physical minimum.
        What are the minimum necessary actions for a customer to go from
        unaware to paying?
        
        What is the theoretically shortest information path?
        
        How many times longer is your actual path?
        
        If it's more than 3x, there are steps that shouldn't exist at all.
        Don't optimize the funnel. Question whether the funnel should exist.
```

---

## Writing a Custom Distillation Manually

If you want to distill someone Nuwa hasn't covered, you can follow the framework directly. Use this structure in `references/extraction-framework.md`:

### Step 1: Source Collection Checklist

```markdown
## Primary Sources
- [ ] Book(s) or long-form essays (direct authorship)
- [ ] 10+ hours of recorded interviews/podcasts
- [ ] Social media archive (if applicable)
- [ ] Shareholder letters / memos / internal documents (if public)

## Secondary Sources  
- [ ] 3+ critical/adversarial analyses
- [ ] Biographies or investigative journalism
- [ ] Documented decisions with outcomes (not just stated intentions)

## Minimum Bar
- 40+ primary source data points before attempting distillation
- At least 2 independent critics represented
- Decisions from at least 3 different life phases
```

### Step 2: Mental Model Extraction Template

```markdown
## Mental Model: [Name]

**One-line definition**: [What this model says about reality]

**Evidence of recurrence**:
- Domain A: [quote or decision reference]
- Domain B: [quote or decision reference]  
- Domain C: [quote or decision reference]

**Predictive test**: 
Given [new scenario], this model predicts [person] would [stance/action] because [reasoning chain].

**Exclusivity check**:
Most smart people would [common answer]. This model instead says [differentiated answer] because [underlying belief].

**Passes all three gates**: YES / NO
```

### Step 3: Expression DNA Capture

```markdown
## Expression DNA

**Sentence length pattern**: [short/long/varied — with example]
**Vocabulary register**: [technical/plain/mixed]
**Rhetorical devices**: [e.g., rhetorical questions, analogies, contrarianism]
**Characteristic phrases**: [3-5 signature constructions]
**What they never say**: [words/framings they avoid]
**Pacing**: [rapid assertions / slow building / Socratic questioning]
```

### Step 4: Anti-Pattern Documentation

```markdown
## Value Anti-Patterns

These are positions or behaviors the distilled skill should NEVER produce,
because they contradict documented core values:

- [Anti-pattern 1]: [Evidence it contradicts their stated/demonstrated values]
- [Anti-pattern 2]: [Evidence]
- [Anti-pattern 3]: [Evidence]

## Honest Boundaries

This skill cannot reliably reproduce:
- Intuition and taste (frameworks yes, inspiration no)
- Post-[date] evolution (snapshot, not live)
- Private vs. public gap (only public record available)
- Emotional texture of in-person interaction
```

---

## Topic Skills (Non-Person Distillation)

Nuwa can also distill **domains**, not just people. The X Mentor skill demonstrates this:

```bash
npx skills add alchaincyf/x-mentor-skill
```

Topic skills synthesize methodology from multiple practitioners + documented data (e.g., algorithmic behavior from open-source analysis) rather than one person's cognitive style. The same four-phase pipeline applies, but Phase 1 agents collect domain-level sources instead of person-level sources.

To create a topic skill via Nuwa:

```
> 蒸馏一个「冷启动增长」的主题Skill
> Create a topic skill for "B2B sales methodology"
> Distill the domain of "VC pitch strategy" into a skill
```

---

## Troubleshooting

### "The skill sounds generic / like ChatGPT"

The distillation failed the exclusivity gate. Rebuild with more adversarial sources and focus the mental model extraction on *what this person believes that contradicts conventional wisdom*.

Check: does each mental model pass Gate 3 (exclusivity)? If a generic smart person would say the same thing, cut it.

### "The skill contradicts what the person actually said"

The distillation failed Test Set A validation. Go back to primary sources for the specific domain where the contradiction appears. The research agent for that domain likely under-sampled.

### "The skill is overconfident on topics the person never addressed"

Failed Test Set B. The honest-boundaries section is incomplete or the expression DNA is missing uncertainty markers. Add explicit uncertainty language to the expression DNA and rebuild.

### "Nuwa isn't triggering in Claude Code"

Verify installation:

```bash
npx skills list
# Should show nuwa-skill-distiller in the list
```

Re-install if missing:

```bash
npx skills add alchaincyf/nuwa-skill
```

### "I want to update a skill with newer sources"

Skills are snapshots. Create a new version:

```
> Rebuild the Naval skill with sources up to [date]
> Update the Musk distillation — here are three new interviews: [links]
```

Nuwa will run the full four-phase pipeline again and diff against the existing `SKILL.md`.

---

## Design Principles

**1. Framework extraction, not impersonation**
The goal is to capture *how someone reasons*, not to simulate their identity. Every output should be attributable to a mental model, not to "what would X say."

**2. Honest boundaries are non-negotiable**
A skill without documented limitations is untrustworthy. Every generated skill must include what it cannot do.

**3. Adversarial sources are required**
Distilling only favorable sources produces hagiography, not cognitive architecture. Critics reveal the actual decision boundaries.

**4. Predictive power is the quality bar**
A useful distillation can handle questions the person never addressed. If it can only reproduce known answers, it's a quote database, not a skill.

**5. Transparency over magic**
All research files are preserved in `examples/`. Anyone can audit the source-to-model chain. Opacity in AI reasoning is a design flaw.

---

## License

MIT — use freely, modify freely, build freely.

Project: [alchaincyf/nuwa-skill](https://github.com/alchaincyf/nuwa-skill)  
Author: 花叔 (Huashu) — [@AlchainHust](https://x.com/AlchainHust)
```
