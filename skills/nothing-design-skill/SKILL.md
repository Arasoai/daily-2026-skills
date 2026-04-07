```markdown
---
name: nothing-design-skill
description: Generate UI in the Nothing design language — monochrome, typographic, industrial — using Space Grotesk/Mono/Doto fonts, OLED blacks, segmented components, and strict three-layer visual hierarchy.
triggers:
  - nothing design
  - nothing style UI
  - generate nothing phone UI
  - monochrome industrial design
  - swiss typography UI
  - OLED dark mode components
  - segmented progress bar nothing
  - nothing design system
---

# Nothing Design Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A Claude Code skill that encodes Nothing's visual language into reusable design rules. When activated, generate UI that is monochrome, typographic, and industrial — no gradients, no decoration, no noise.

## What this skill does

- Enforces three-layer visual hierarchy: **display → body → metadata**
- Applies the canonical Nothing font stack: Space Grotesk + Space Mono + Doto
- Outputs full dark/light mode token systems
- Generates segmented progress bars, mechanical toggles, instrument-style widgets
- Targets HTML/CSS, SwiftUI, or React/Tailwind output

## Install

```sh
git clone https://github.com/dominikmartn/nothing-design-skill.git
cp -r nothing-design-skill/nothing-design ~/.claude/skills/
```

Restart Claude Code. The skill loads automatically on next session.

## Activation

Say any of:
- "Nothing style"
- `/nothing-design`
- "Generate this in the Nothing design language"
- "Make this monochrome industrial"

Claude will apply the full system without further prompting.

---

## Design Principles

### Visual Hierarchy (strict — never add layers)

| Layer | Role | Font | Weight | Size |
|-------|------|------|--------|------|
| Display | Hero numbers, key metric | Doto / Space Mono | 100–300 | 48–96px |
| Body | Primary readable content | Space Grotesk | 400 | 14–16px |
| Metadata | Labels, timestamps, units | Space Mono | 400 | 10–12px |

### Core Rules

1. **Black is black** — `#0A0A0A` on dark, never `#000000` (OLED bleed)
2. **One accent** — white on dark, black on light. No color except system alerts
3. **No gradients** — flat fills only
4. **Borders over shadows** — `1px solid` beats `box-shadow`
5. **Segmentation over continuity** — progress is ticked, not smooth
6. **Numbers are display elements** — treat metrics like type, not data

---

## Token System

### Color Tokens

```css
/* Dark mode (default) */
:root[data-theme="dark"] {
  --bg-primary:    #0A0A0A;
  --bg-secondary:  #111111;
  --bg-elevated:   #1A1A1A;
  --bg-overlay:    #222222;

  --border-default: #2A2A2A;
  --border-subtle:  #1E1E1E;
  --border-strong:  #3A3A3A;

  --text-primary:   #F5F5F5;
  --text-secondary: #999999;
  --text-disabled:  #444444;
  --text-inverse:   #0A0A0A;

  --accent:         #F5F5F5;
  --accent-inverse: #0A0A0A;

  --signal-error:   #FF3B30;
  --signal-warn:    #FF9500;
  --signal-ok:      #30D158;
}

/* Light mode */
:root[data-theme="light"] {
  --bg-primary:    #F5F5F5;
  --bg-secondary:  #EBEBEB;
  --bg-elevated:   #FFFFFF;
  --bg-overlay:    #E0E0E0;

  --border-default: #D0D0D0;
  --border-subtle:  #DEDEDE;
  --border-strong:  #BEBEBE;

  --text-primary:   #0A0A0A;
  --text-secondary: #666666;
  --text-disabled:  #BBBBBB;
  --text-inverse:   #F5F5F5;

  --accent:         #0A0A0A;
  --accent-inverse: #F5F5F5;
}
```

### Typography Tokens

```css
/* Import */
@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@300;400;500;600;700&family=Space+Mono:wght@400;700&family=Doto:wght@100;200;300;400&display=swap');

:root {
  /* Display — Doto for numerics, Space Mono fallback */
  --font-display:  'Doto', 'Space Mono', monospace;
  --font-body:     'Space Grotesk', system-ui, sans-serif;
  --font-meta:     'Space Mono', monospace;

  /* Scale */
  --text-display-xl: 96px;
  --text-display-lg: 64px;
  --text-display-md: 48px;
  --text-display-sm: 32px;

  --text-body-lg:  18px;
  --text-body-md:  16px;
  --text-body-sm:  14px;

  --text-meta-md:  12px;
  --text-meta-sm:  10px;

  /* Tracking — tight for display, wide for meta */
  --tracking-display: -0.04em;
  --tracking-body:    -0.01em;
  --tracking-meta:     0.08em;

  /* Line height */
  --leading-display: 0.95;
  --leading-body:    1.5;
  --leading-meta:    1.4;
}
```

### Spacing & Radius

```css
:root {
  /* 4px base grid */
  --space-1:  4px;
  --space-2:  8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  20px;
  --space-6:  24px;
  --space-8:  32px;
  --space-10: 40px;
  --space-12: 48px;
  --space-16: 64px;

  /* Radius — sharp by default */
  --radius-none: 0px;
  --radius-sm:   2px;
  --radius-md:   4px;
  --radius-lg:   8px;
  --radius-pill: 999px;
}
```

### Motion Tokens

```css
:root {
  --duration-instant: 80ms;
  --duration-fast:    150ms;
  --duration-normal:  250ms;
  --duration-slow:    400ms;

  /* Mechanical — ease-out only, no bounce */
  --ease-mechanical: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-snap:       cubic-bezier(0.0, 0, 0.2, 1);
}
```

---

## Components

### Button

```html
<!-- Primary -->
<button class="btn-primary">
  <span class="btn-label">CONFIRM</span>
</button>

<!-- Ghost -->
<button class="btn-ghost">
  <span class="btn-label">CANCEL</span>
</button>
```

```css
.btn-primary,
.btn-ghost {
  font-family: var(--font-meta);
  font-size: var(--text-meta-md);
  letter-spacing: var(--tracking-meta);
  text-transform: uppercase;
  padding: var(--space-3) var(--space-6);
  border-radius: var(--radius-sm);
  cursor: pointer;
  transition: opacity var(--duration-fast) var(--ease-mechanical);
}

.btn-primary {
  background: var(--accent);
  color: var(--accent-inverse);
  border: 1px solid var(--accent);
}

.btn-ghost {
  background: transparent;
  color: var(--text-primary);
  border: 1px solid var(--border-strong);
}

.btn-primary:hover,
.btn-ghost:hover { opacity: 0.7; }
.btn-primary:active,
.btn-ghost:active { opacity: 0.5; }
```

---

### Segmented Progress Bar

The Nothing signature — ticked, not smooth.

```html
<div class="seg-progress" data-segments="12" data-filled="8" aria-valuenow="67">
  <!-- generated by JS below -->
</div>
<span class="seg-label">08 / 12</span>
```

```css
.seg-progress {
  display: flex;
  gap: 3px;
  align-items: center;
  height: 16px;
}

.seg-tick {
  flex: 1;
  height: 100%;
  border-radius: var(--radius-none);
  transition: background var(--duration-instant) var(--ease-snap);
}

.seg-tick.filled  { background: var(--accent); }
.seg-tick.empty   { background: var(--border-default); }

.seg-label {
  font-family: var(--font-meta);
  font-size: var(--text-meta-sm);
  color: var(--text-secondary);
  letter-spacing: var(--tracking-meta);
  margin-top: var(--space-1);
  display: block;
}
```

```js
function renderSegProgress(el) {
  const total  = parseInt(el.dataset.segments, 10);
  const filled = parseInt(el.dataset.filled, 10);
  el.innerHTML = Array.from({ length: total }, (_, i) =>
    `<div class="seg-tick ${i < filled ? 'filled' : 'empty'}"></div>`
  ).join('');
}

document.querySelectorAll('.seg-progress').forEach(renderSegProgress);
```

---

### Card

```html
<article class="card">
  <header class="card-header">
    <span class="card-label">BATTERY</span>
    <span class="card-tag">LIVE</span>
  </header>
  <div class="card-display">87</div>
  <div class="card-unit">PERCENT</div>
</article>
```

```css
.card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  padding: var(--space-6);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
}

.card-label {
  font-family: var(--font-meta);
  font-size: var(--text-meta-sm);
  letter-spacing: var(--tracking-meta);
  color: var(--text-secondary);
  text-transform: uppercase;
}

.card-tag {
  font-family: var(--font-meta);
  font-size: var(--text-meta-sm);
  letter-spacing: var(--tracking-meta);
  color: var(--signal-ok);
  text-transform: uppercase;
}

.card-display {
  font-family: var(--font-display);
  font-size: var(--text-display-lg);
  font-weight: 200;
  letter-spacing: var(--tracking-display);
  line-height: var(--leading-display);
  color: var(--text-primary);
}

.card-unit {
  font-family: var(--font-meta);
  font-size: var(--text-meta-sm);
  letter-spacing: var(--tracking-meta);
  color: var(--text-secondary);
  text-transform: uppercase;
  margin-top: var(--space-2);
}
```

---

### Mechanical Toggle

```html
<label class="toggle">
  <input type="checkbox" class="toggle-input" />
  <span class="toggle-track">
    <span class="toggle-thumb"></span>
  </span>
  <span class="toggle-label">GLYPH INTERFACE</span>
</label>
```

```css
.toggle { display: flex; align-items: center; gap: var(--space-3); cursor: pointer; }
.toggle-input { display: none; }

.toggle-track {
  width: 40px;
  height: 20px;
  border: 1px solid var(--border-strong);
  border-radius: var(--radius-none);
  position: relative;
  transition: background var(--duration-fast) var(--ease-mechanical);
}

.toggle-thumb {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 12px;
  height: 12px;
  background: var(--text-secondary);
  transition: transform var(--duration-fast) var(--ease-mechanical),
              background var(--duration-fast) var(--ease-mechanical);
}

.toggle-input:checked + .toggle-track { background: var(--accent); border-color: var(--accent); }
.toggle-input:checked + .toggle-track .toggle-thumb {
  transform: translateX(20px);
  background: var(--accent-inverse);
}

.toggle-label {
  font-family: var(--font-meta);
  font-size: var(--text-meta-md);
  letter-spacing: var(--tracking-meta);
  color: var(--text-primary);
  text-transform: uppercase;
}
```

---

### Data Table

```html
<table class="data-table">
  <thead>
    <tr>
      <th>DEVICE</th>
      <th>STATUS</th>
      <th>SIGNAL</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>PHONE (2)</td>
      <td class="status-ok">CONNECTED</td>
      <td>–42 dBm</td>
    </tr>
    <tr>
      <td>EAR (1)</td>
      <td class="status-warn">PAIRING</td>
      <td>–</td>
    </tr>
  </tbody>
</table>
```

```css
.data-table {
  width: 100%;
  border-collapse: collapse;
  font-family: var(--font-meta);
  font-size: var(--text-meta-md);
  letter-spacing: var(--tracking-meta);
}

.data-table th {
  text-align: left;
  color: var(--text-secondary);
  text-transform: uppercase;
  font-weight: 400;
  padding: var(--space-3) var(--space-4);
  border-bottom: 1px solid var(--border-strong);
}

.data-table td {
  padding: var(--space-4);
  border-bottom: 1px solid var(--border-subtle);
  color: var(--text-primary);
}

.data-table tr:last-child td { border-bottom: none; }
.status-ok   { color: var(--signal-ok); }
.status-warn { color: var(--signal-warn); }
.status-err  { color: var(--signal-error); }
```

---

## SwiftUI Output Pattern

```swift
// Nothing design tokens in SwiftUI
extension Color {
    static let nBgPrimary   = Color(hex: "#0A0A0A")
    static let nBgSecondary = Color(hex: "#111111")
    static let nTextPrimary = Color(hex: "#F5F5F5")
    static let nTextMeta    = Color(hex: "#999999")
    static let nBorder      = Color(hex: "#2A2A2A")
    static let nAccent      = Color(hex: "#F5F5F5")
}

// Segmented progress
struct NothingProgressBar: View {
    let total: Int
    let filled: Int

    var body: some View {
        HStack(spacing: 3) {
            ForEach(0..<total, id: \.self) { i in
                Rectangle()
                    .fill(i < filled ? Color.nAccent : Color.nBorder)
                    .frame(height: 16)
                    .animation(.easeOut(duration: 0.08), value: filled)
            }
        }
    }
}

// Instrument card
struct NothingCard: View {
    let label: String
    let value: String
    let unit: String

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(label.uppercased())
                .font(.custom("SpaceMono-Regular", size: 10))
                .tracking(4)
                .foregroundColor(.nTextMeta)

            Text(value)
                .font(.custom("Doto-Thin", size: 64))
                .foregroundColor(.nTextPrimary)
                .kerning(-3)

            Text(unit.uppercased())
                .font(.custom("SpaceMono-Regular", size: 10))
                .tracking(4)
                .foregroundColor(.nTextMeta)
        }
        .padding(24)
        .background(Color(hex: "#111111"))
        .overlay(
            Rectangle().stroke(Color.nBorder, lineWidth: 1)
        )
    }
}
```

---

## React/Tailwind Output Pattern

```tsx
// tailwind.config — extend with Nothing tokens
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        'n-bg':       '#0A0A0A',
        'n-surface':  '#111111',
        'n-elevated': '#1A1A1A',
        'n-border':   '#2A2A2A',
        'n-text':     '#F5F5F5',
        'n-meta':     '#999999',
        'n-accent':   '#F5F5F5',
      },
      fontFamily: {
        display: ['Doto', 'Space Mono', 'monospace'],
        body:    ['Space Grotesk', 'system-ui', 'sans-serif'],
        mono:    ['Space Mono', 'monospace'],
      },
    },
  },
}

// NothingCard component
export function NothingCard({
  label,
  value,
  unit,
}: {
  label: string;
  value: string;
  unit: string;
}) {
  return (
    <div className="bg-n-surface border border-n-border p-6">
      <p className="font-mono text-[10px] tracking-[0.08em] uppercase text-n-meta mb-4">
        {label}
      </p>
      <p className="font-display text-[64px] font-thin leading-none tracking-[-0.04em] text-n-text">
        {value}
      </p>
      <p className="font-mono text-[10px] tracking-[0.08em] uppercase text-n-meta mt-2">
        {unit}
      </p>
    </div>
  );
}

// SegmentedBar component
export function SegmentedBar({ total, filled }: { total: number; filled: number }) {
  return (
    <div className="flex gap-[3px] h-4">
      {Array.from({ length: total }, (_, i) => (
        <div
          key={i}
          className={`flex-1 transition-colors duration-75 ${
            i < filled ? 'bg-n-accent' : 'bg-n-border'
          }`}
        />
      ))}
    </div>
  );
}
```

---

## Common Patterns

### Do

- Display numbers as hero type (`font-display`, weight 100–300)
- Use `text-transform: uppercase` on all labels and metadata
- Keep gaps between segments exactly 3px
- Use `border` not `box-shadow` for elevation
- Animate with `ease-out` only — snap in, drift out

### Don't

- Add decorative icons or illustrations
- Use more than one accent color per surface
- Round corners beyond 4px (except pills on tags)
- Use `font-weight` above 500 for display text
- Mix Space Grotesk and Space Mono in the same label

---

## Troubleshooting

**Doto font not loading**
Doto is a Google Fonts variable font. Confirm the import URL includes `family=Doto` and fall back to `Space Mono` — both read as display.

**Segments look uneven**
Use `flex: 1` on each tick inside a `display: flex` container, not percentage widths. Fixed gaps (`gap: 3px`) preserve the segmented rhythm.

**Dark mode not switching**
Tokens use `[data-theme="dark"]` attribute selector. Set `document.documentElement.dataset.theme = 'dark'` — not a class.

**SwiftUI custom fonts missing**
Add font files to the Xcode project bundle and declare them in `Info.plist` under `UIAppFonts`. Space Grotesk and Space Mono are available via Swift Package Manager through the `GoogleFonts` package.

**Too many hierarchy levels requested**
Refuse politely. The system has exactly three: display, body, metadata. Push back and map the extra levels to one of the three.
```
