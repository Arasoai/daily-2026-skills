```markdown
---
name: taste-skill-frontend-design
description: Gives AI coding agents premium frontend design taste — stops generic, boring, template-looking UI by enforcing modern layouts, typography, motion, and visual quality.
triggers:
  - make my UI look better
  - improve the design of my frontend
  - stop generating generic looking interfaces
  - give my app premium design quality
  - redesign this component with good taste
  - make this look modern and polished
  - apply good design principles to my code
  - fix the boring slop UI my AI keeps generating
---

# Taste Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A collection of `SKILL.md` files you drop into your project to give AI coding agents (Cursor, Claude Code, Codex, etc.) strong frontend design taste. Instead of producing generic, template-looking UI, the AI will produce modern, premium interfaces with proper typography, spacing, motion, and visual hierarchy.

---

## What It Does

Taste Skill is not a library or framework — it's a set of instruction files your AI reads and follows. Each skill file teaches the AI a specific design philosophy:

| Skill | Purpose |
|---|---|
| `taste-skill` | Core design skill: layout, typography, color, spacing, motion |
| `redesign-skill` | Audits and upgrades existing projects |
| `soft-skill` | Premium/luxury feel: fonts, whitespace, spring animations, depth |
| `output-skill` | Prevents lazy output: no placeholders, no skipped blocks |
| `minimalist-skill` | Notion/Linear-style editorial UI: monochrome, serif contrast, bento grids |

---

## Installation

### Step 1: Copy the skill file into your project

```bash
# Clone the repo
git clone https://github.com/Leonxlnx/taste-skill.git

# Copy the skill you want into your project root
cp taste-skill/SKILL.md ./SKILL.md

# Or for a specific skill
cp taste-skill/redesign-skill/SKILL.md ./SKILL.md
cp taste-skill/soft-skill/SKILL.md ./SKILL.md
cp taste-skill/minimalist-skill/SKILL.md ./SKILL.md
cp taste-skill/output-skill/SKILL.md ./SKILL.md
```

You can also copy multiple skills and reference them by name:

```
project/
├── SKILL.md              ← main taste-skill
├── REDESIGN_SKILL.md     ← redesign-skill
├── OUTPUT_SKILL.md       ← output-skill
└── src/
```

### Step 2: Reference the skill in your AI tool

**Cursor:**
```
@SKILL.md redesign this hero section
```

**Claude Code:**
```
Read SKILL.md and follow it. Now redesign the landing page hero.
```

**Codex / any agent:**
```
Follow the rules in SKILL.md. Rewrite src/components/Hero.tsx.
```

That's it. No installs, no config files, no dependencies.

---

## Configuration (taste-skill only)

Open `SKILL.md` and edit the three settings at the top of the file:

```
DESIGN_VARIANCE: 7
MOTION_INTENSITY: 5
VISUAL_DENSITY: 3
```

### DESIGN_VARIANCE (1–10)
Controls how experimental and asymmetric the layout is.

| Range | Effect |
|---|---|
| 1–3 | Clean, centered, standard grids |
| 4–7 | Overlapping elements, varied sizes, modern structure |
| 8–10 | Asymmetric, extreme whitespace, editorial/avant-garde |

### MOTION_INTENSITY (1–10)
Controls how much animation is added.

| Range | Effect |
|---|---|
| 1–3 | Almost none — simple hover effects only |
| 4–7 | Fade-ins, smooth scroll, subtle entrance animations |
| 8–10 | Magnetic cursor, spring physics, scroll-triggered reveals |

### VISUAL_DENSITY (1–10)
Controls how much content appears per screen.

| Range | Effect |
|---|---|
| 1–3 | Spacious luxury feel — one element at a time |
| 4–7 | Normal app/website spacing |
| 8–10 | Dense dashboards, data-heavy interfaces |

**Example config for a luxury SaaS landing page:**
```
DESIGN_VARIANCE: 8
MOTION_INTENSITY: 6
VISUAL_DENSITY: 2
```

**Example config for a data dashboard:**
```
DESIGN_VARIANCE: 3
MOTION_INTENSITY: 2
VISUAL_DENSITY: 9
```

---

## Skill-by-Skill Usage

### taste-skill (main)

Use this when building new UI from scratch.

```
@SKILL.md Build a pricing section for a SaaS product called "Volta".
Use DESIGN_VARIANCE: 7, MOTION_INTENSITY: 5, VISUAL_DENSITY: 3.
```

What the AI will produce (React/Tailwind example):

```tsx
// The AI will write something like this instead of a generic table
export function PricingSection() {
  return (
    <section className="py-32 px-6 bg-neutral-950">
      <div className="max-w-5xl mx-auto">
        <p className="text-xs uppercase tracking-widest text-neutral-500 mb-4">
          Pricing
        </p>
        <h2 className="text-6xl font-semibold text-white leading-[1.05] mb-20">
          Simple,<br />honest pricing.
        </h2>
        <div className="grid grid-cols-1 md:grid-cols-3 gap-px bg-neutral-800">
          {plans.map((plan) => (
            <div
              key={plan.name}
              className="bg-neutral-950 p-10 flex flex-col gap-8 hover:bg-neutral-900 transition-colors duration-300"
            >
              <div>
                <p className="text-neutral-400 text-sm mb-1">{plan.name}</p>
                <p className="text-5xl font-semibold text-white">${plan.price}</p>
                <p className="text-neutral-500 text-sm mt-1">/month</p>
              </div>
              <ul className="flex flex-col gap-3 flex-1">
                {plan.features.map((f) => (
                  <li key={f} className="text-neutral-300 text-sm flex items-start gap-2">
                    <span className="text-emerald-400 mt-0.5">↗</span>
                    {f}
                  </li>
                ))}
              </ul>
              <button className="w-full py-3 border border-neutral-700 text-white text-sm hover:bg-white hover:text-black transition-all duration-200">
                Get started
              </button>
            </div>
          ))}
        </div>
      </div>
    </section>
  )
}
```

---

### redesign-skill

Use this when you have existing code that looks bad and want the AI to audit and fix it.

```
@REDESIGN_SKILL.md Here is my current Dashboard.tsx. Audit it and fix the biggest design problems.
```

The AI will:
1. Identify the worst visual issues (generic colors, poor spacing, bad typography)
2. Prioritize fixes by visual impact
3. Rewrite the component with improvements applied

---

### soft-skill

Use this when you want a premium, expensive-feeling UI.

```
@SOFT_SKILL.md Build a hero section for a luxury skincare brand.
```

What soft-skill enforces:
- Premium fonts (e.g. `font-['Playfair_Display']`, `font-['Inter']`)
- Massive breathing whitespace (`py-48`, `gap-24`)
- Layered card depth (`shadow-2xl`, `backdrop-blur`)
- Spring-based animations (Framer Motion `type: "spring"`)
- Floating navigation bars
- **Bans:** default blue buttons, generic sans-serif, card borders without depth

```tsx
// soft-skill will push the AI to write motion like this
import { motion } from "framer-motion"

<motion.div
  initial={{ opacity: 0, y: 40 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ type: "spring", stiffness: 60, damping: 20, delay: 0.2 }}
  className="text-8xl font-['Playfair_Display'] leading-none tracking-tight"
>
  Effortless<br />beauty.
</motion.div>
```

---

### minimalist-skill

Use this for Notion/Linear-style editorial interfaces.

```
@MINIMALIST_SKILL.md Build a project management dashboard in Notion/Linear style.
```

What minimalist-skill enforces:
- Warm monochrome palette (`#1a1a1a`, `#f5f0eb`, `#e8e3dd`)
- Typographic contrast: serif headings + clean sans body
- Massive whitespace
- Flat bento grid layouts
- Hairline borders (`border border-stone-200`)
- Muted pastel accents only
- **Bans:** gradients, heavy shadows, generic SaaS card patterns, colorful CTAs

```tsx
// minimalist-skill output example
<div className="min-h-screen bg-[#f5f0eb] font-['Inter']">
  <div className="max-w-4xl mx-auto py-24 px-8">
    <h1 className="font-['Playfair_Display'] text-5xl text-[#1a1a1a] mb-2 leading-tight">
      Your projects
    </h1>
    <p className="text-[#8a8070] text-sm mb-16">3 active · Last updated today</p>

    <div className="grid grid-cols-2 gap-px bg-[#d4cfc9]">
      {projects.map((p) => (
        <div key={p.id} className="bg-[#f5f0eb] p-8 hover:bg-[#eee9e3] transition-colors">
          <p className="text-xs text-[#a09880] uppercase tracking-widest mb-4">{p.status}</p>
          <h3 className="text-xl text-[#1a1a1a] font-medium mb-2">{p.name}</h3>
          <p className="text-sm text-[#6b6355]">{p.description}</p>
        </div>
      ))}
    </div>
  </div>
</div>
```

---

### output-skill

Use this when the AI keeps writing placeholder comments or skipping code.

```
@OUTPUT_SKILL.md Now write the complete CartPage.tsx — no placeholders, no "// rest of code here".
```

Output-skill enforces:
- No `// ...` or `// TODO` in generated output
- No `// rest of the implementation` shortcuts
- No skipped imports or incomplete JSX
- Every function body fully written
- Every component fully rendered

Combine it with other skills:

```
@SKILL.md @OUTPUT_SKILL.md Build a complete checkout flow: CartPage, CheckoutPage, and ConfirmationPage.
Write every file in full.
```

---

## Common Patterns

### Use multiple skills together

```
@SKILL.md @OUTPUT_SKILL.md 
Build a full landing page for a developer tool. 
Sections: Hero, Features (bento grid), Pricing, Footer.
DESIGN_VARIANCE: 7, MOTION_INTENSITY: 6, VISUAL_DENSITY: 3
Write every section completely.
```

### Redesign a specific component

```
@REDESIGN_SKILL.md 
Here is my current Button.tsx and Card.tsx. 
They look generic. Audit them and rewrite with proper depth, spacing, and hover states.
```

### Apply a style to a whole page

```
@SOFT_SKILL.md @OUTPUT_SKILL.md
Rewrite pages/index.tsx with premium design quality.
This is for a high-end B2B SaaS product targeting enterprise customers.
```

### Target a specific stack

```
@SKILL.md
I'm using Next.js 14, Tailwind CSS, and Framer Motion.
Build a testimonials section. Use scroll-triggered animations.
MOTION_INTENSITY: 8
```

---

## Troubleshooting

### The AI ignored the skill file
- Make sure you're referencing the file directly: `@SKILL.md` in Cursor, or explicitly saying "read and follow SKILL.md" in Claude/Codex.
- Some agents need the instruction in the system prompt or rules file. In Cursor, add `Always follow SKILL.md when writing frontend code` to `.cursorrules`.

### The output still looks generic
- Check your `DESIGN_VARIANCE` setting — if it's set to 1–3, the AI will intentionally produce standard layouts.
- Add `@OUTPUT_SKILL.md` to prevent the AI from cutting corners.
- Be explicit: "Do not use generic card patterns. Do not use blue as a primary color. Follow SKILL.md strictly."

### The AI writes placeholders anyway
- Add `@OUTPUT_SKILL.md` to your prompt.
- Say explicitly: "Write every line of code. No comments like '// rest of code'. No placeholder text."

### Animations aren't showing up
- Make sure `framer-motion` is installed: `npm install framer-motion`
- Check `MOTION_INTENSITY` — if it's below 4, animations will be minimal by design.

### The minimalist skill is adding gradients
- Remind the AI: "Follow MINIMALIST_SKILL.md strictly. No gradients. No colored shadows."

---

## File Structure Reference

```
taste-skill/
├── SKILL.md                    ← main taste-skill (copy this one first)
├── redesign-skill/
│   └── SKILL.md
├── soft-skill/
│   └── SKILL.md
├── output-skill/
│   └── SKILL.md
├── minimalist-skill/
│   └── SKILL.md
├── examples/
│   ├── floria-top.webp
│   └── floria-bottom.webp
└── research/                   ← background research that informed the skills
```

---

## Quick Reference

| Goal | Skill to use |
|---|---|
| Build new UI with great taste | `taste-skill` |
| Upgrade existing bad-looking code | `redesign-skill` |
| Luxury, premium, expensive feel | `soft-skill` |
| Notion/Linear editorial style | `minimalist-skill` |
| Stop AI from skipping code | `output-skill` |
| Full project, all combined | `taste-skill` + `output-skill` |

---

## Links

- GitHub: [github.com/Leonxlnx/taste-skill](https://github.com/Leonxlnx/taste-skill)
- Author: [@lexnlin on X](https://x.com/lexnlin)
- Contact: [hello@learn2vibecode.dev](mailto:hello@learn2vibecode.dev)
```
