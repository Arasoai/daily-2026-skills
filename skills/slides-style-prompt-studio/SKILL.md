```markdown
---
name: slides-style-prompt-studio
description: Generate multi-style PPT slide prompts for AI image generation via nanobanana2/Gemini
triggers:
  - generate slide prompts
  - create PPT style prompts
  - make presentation slides with AI
  - generate retro pop art slide
  - create cyberpunk presentation
  - slides style prompt studio
  - generate nanobanana2 prompts
  - create multi-style slides
---

# Slides Style Prompt Studio

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A prompt library and generator for creating styled PPT slides using AI image generation (nanobanana2 / Gemini). No code required — copy prompts, customize text, and generate 2K slides.

---

## What It Does

- Provides **11 pre-built style prompts** for AI-generated PPT slides
- Targets `gemini-3.1-flash-image-preview` via nanobanana2
- Outputs `2048×1152` (2K 16:9) presentation slides
- Covers styles from Retro Pop Art to Cyberpunk, Swiss International, Y2K Pixel Retro, and more

---

## Project Structure

```
slides/
├── README.md                   # Overview and style gallery
├── PROMPTS.md                  # All 11 style prompts (primary resource)
├── skill.json                  # OpenClaw skill config
├── CLAUDE.md                   # Skill context
└── demos/yc-intro/images/      # 22 generated sample images
```

---

## Installation / Setup

No Python package to install — this is a prompt library. Clone the repo:

```bash
git clone https://github.com/AAAAAAAJ/slides.git
cd slides
```

To use the prompts programmatically with the Gemini API:

```bash
pip install google-generativeai pillow
export GEMINI_API_KEY="your_api_key_here"
```

---

## Recommended Generation Settings

| Setting       | Value                              |
|---------------|------------------------------------|
| Model         | `gemini-3.1-flash-image-preview`   |
| Resolution    | `2048×1152`                        |
| Aspect Ratio  | `16:9`                             |
| Platform      | nanobanana2 / APImart              |

---

## The 11 Style Prompts

Reference `PROMPTS.md` for the full list. Key styles:

### 1. Retro Pop Art
```
Retro pop art style PPT slide, 1970s magazine aesthetic, flat design with thick black outlines,
cream beige background, bold title text, subtitle below, key statistics displayed as cards,
Salmon pink #FF6B6B, sky blue #4ECDC4, mustard yellow #FFD93D, mint green #6BCB77 accents,
Geometric decorations: quarter circles, concentric rings, star bursts,
Bold sans-serif typography, Professional presentation design, 16:9
```

### 2. Minimalist Clean
```
Minimalist clean design PPT slide, White background, generous whitespace, centered title text,
subtitle below, key stats in simple cards, Subtle gray and blue accents, Thin elegant lines,
Inter Helvetica font, Professional corporate presentation, Simple elegant layout, 16:9
```

### 3. Cyberpunk Neon
```
Cyberpunk neon style PPT slide, Dark charcoal background, title text with neon glow effect,
subtitle below, Neon colors: magenta #FF00FF, cyan #00FFFF, yellow #FFFF00,
Tech grid patterns, circuit decorations, Holographic data panels, glow effects,
Futuristic UI elements, Digital presentation, 16:9
```

### 4. Neo-Brutalism
```
Neo-brutalism style PPT slide, raw design, Cream background, bold title text, subtitle below,
key stats displayed, Bold primary colors: red #FF4D4D, blue #4D94FF, yellow #FFD93D,
Thick 4px black outlines, hard shadows, Brutalist frames, bold typography, Stark contrast, 16:9
```

### 5. Acid Graphics Y2K
```
Acid graphics Y2K style PPT slide, Light gray background, title text, subtitle below,
key stats in stylized cards, Metallic chrome elements, holographic accents,
Colors: purple #B185FF, pink #FF6EC7, mint #7BFFCB, gold #FFD700,
Liquid shapes, star sparkles, mesh gradients, Y2K aesthetic, futuristic design, 16:9
```

### 6. Modern Minimal Pop
```
Modern minimal pop art PPT slide, Instagram aesthetic, Pastel background, title text,
subtitle below, key stats displayed, Pastel colors: mint #A8E6C8, cream #FFF4BD,
coral #FF8B7A, purple #8B7AFF, Star burst graphics, thin line circles,
Tilted color blocks, small arrows, Clean sans-serif typography, Swiss design influence, 16:9
```

### 7. Swiss International
```
Swiss international style PPT slide, brutalist graphic design, Light gray background,
bold title text, subtitle with diagonal layout, key stats in geometric blocks,
High saturation colors: blue #007AFF, green #00994D, yellow #FFF066,
purple #9966FF, pink #FF3399, orange #FF8800, Helvetica font, Asymmetric composition, 16:9
```

### 8. Dark Editorial
```
Dark editorial PPT slide, New York Times Sunday Review style,
Black background with white dot grid pattern, title text in white, subtitle below,
white text, orange accent #E85D2A, Minimalist wireframe illustrations,
Serif typography, Dramatic negative space, Newspaper aesthetic, 16:9
```

### 9. Design Blueprint
```
Design blueprint PPT slide, Figma documentation style, White background with cyan grid lines
#66B8CC, title text, subtitle below, Figma selection boxes with control points,
Annotation lines, numbered labels, Technical UI mockup aesthetic,
Clean sans-serif Inter font, 16:9
```

### 10. Neo-Brutalist UI
```
Neo-brutalist UI PPT slide, dashboard interface design, Cream background, title text,
subtitle below, stats in cards, Pastel panels: mint #A8E4CF, yellow #FFD93D, lavender #E5B3FF,
Thick 3px black outlines, Card-based layout, flat colors,
Bold typography, Contemporary SaaS dashboard aesthetic, 16:9
```

### 11. Y2K Pixel Retro
```
Y2K pixel retro PPT slide, 1990s aesthetic, Dark background with noise texture,
title text in pixel font, subtitle below, Bright colors: yellow #FFD700, orange #FF8C00,
green #4A7C4E, Pixel art computer icons, CRT monitor graphics,
Isometric tech illustrations, VT323 pixel font style, Vintage 1990s design, 16:9
```

---

## Programmatic Usage (Python)

### Generate a slide via Gemini API

```python
import google.generativeai as genai
import os
from PIL import Image
import io
import base64

genai.configure(api_key=os.environ["GEMINI_API_KEY"])

def generate_slide(style_prompt: str, title: str, subtitle: str, stats: list[str]) -> bytes:
    """
    Generate a styled PPT slide image using Gemini.

    Args:
        style_prompt: Base style prompt from PROMPTS.md
        title: Slide title (≤8 words recommended)
        subtitle: Slide subtitle (≤12 words recommended)
        stats: List of 3-5 key data points

    Returns:
        Image bytes (PNG)
    """
    stats_text = ", ".join(stats)
    full_prompt = f"{style_prompt}\nTitle: {title}\nSubtitle: {subtitle}\nKey stats: {stats_text}"

    model = genai.GenerativeModel("gemini-3.1-flash-image-preview")
    response = model.generate_content(
        full_prompt,
        generation_config=genai.GenerationConfig(
            response_modalities=["image"],
        )
    )

    for part in response.candidates[0].content.parts:
        if part.inline_data and part.inline_data.mime_type.startswith("image/"):
            return base64.b64decode(part.inline_data.data)

    raise ValueError("No image returned in response")


# Example: Retro Pop Art slide
RETRO_POP_PROMPT = """Retro pop art style PPT slide, 1970s magazine aesthetic,
flat design with thick black outlines, cream beige background, bold title text,
subtitle below, key statistics displayed as cards,
Salmon pink #FF6B6B, sky blue #4ECDC4, mustard yellow #FFD93D, mint green #6BCB77 accents,
Geometric decorations: quarter circles, concentric rings, star bursts,
Bold sans-serif typography, Professional presentation design, 16:9"""

image_bytes = generate_slide(
    style_prompt=RETRO_POP_PROMPT,
    title="What is Y Combinator",
    subtitle="The World's Most Famous Startup Accelerator",
    stats=["Founded 2005", "4000+ companies", "$600B combined value"]
)

with open("slide_output.png", "wb") as f:
    f.write(image_bytes)

print("Slide saved to slide_output.png")
```

### Batch generate all 11 styles

```python
import google.generativeai as genai
import os
import base64
from pathlib import Path

genai.configure(api_key=os.environ["GEMINI_API_KEY"])

STYLES = {
    "retro-pop": "Retro pop art style PPT slide, 1970s magazine aesthetic, flat design with thick black outlines, cream beige background, bold title text, subtitle below, key statistics displayed as cards, Salmon pink #FF6B6B, sky blue #4ECDC4, mustard yellow #FFD93D, mint green #6BCB77 accents, Geometric decorations: quarter circles, concentric rings, star bursts, Bold sans-serif typography, Professional presentation design, 16:9",
    "minimal": "Minimalist clean design PPT slide, White background, generous whitespace, centered title text, subtitle below, key stats in simple cards, Subtle gray and blue accents, Thin elegant lines, Inter Helvetica font, Professional corporate presentation, Simple elegant layout, 16:9",
    "cyberpunk": "Cyberpunk neon style PPT slide, Dark charcoal background, title text with neon glow effect, subtitle below, Neon colors: magenta #FF00FF, cyan #00FFFF, yellow #FFFF00, Tech grid patterns, circuit decorations, Holographic data panels, glow effects, Futuristic UI elements, Digital presentation, 16:9",
    "neo-brutalism": "Neo-brutalism style PPT slide, raw design, Cream background, bold title text, subtitle below, key stats displayed, Bold primary colors: red #FF4D4D, blue #4D94FF, yellow #FFD93D, Thick 4px black outlines, hard shadows, Brutalist frames, bold typography, Stark contrast, 16:9",
    "y2k-acid": "Acid graphics Y2K style PPT slide, Light gray background, title text, subtitle below, key stats in stylized cards, Metallic chrome elements, holographic accents, Colors: purple #B185FF, pink #FF6EC7, mint #7BFFCB, gold #FFD700, Liquid shapes, star sparkles, mesh gradients, Y2K aesthetic, futuristic design, 16:9",
    "modern-minimal-pop": "Modern minimal pop art PPT slide, Instagram aesthetic, Pastel background, title text, subtitle below, key stats displayed, Pastel colors: mint #A8E6C8, cream #FFF4BD, coral #FF8B7A, purple #8B7AFF, Star burst graphics, thin line circles, Tilted color blocks, small arrows, Clean sans-serif typography, Swiss design influence, 16:9",
    "swiss": "Swiss international style PPT slide, brutalist graphic design, Light gray background, bold title text, subtitle with diagonal layout, key stats in geometric blocks, High saturation colors: blue #007AFF, green #00994D, yellow #FFF066, purple #9966FF, pink #FF3399, orange #FF8800, Helvetica font, Asymmetric composition, 16:9",
    "dark-editorial": "Dark editorial PPT slide, New York Times Sunday Review style, Black background with white dot grid pattern, title text in white, subtitle below, white text, orange accent #E85D2A, Minimalist wireframe illustrations, Serif typography, Dramatic negative space, Newspaper aesthetic, 16:9",
    "blueprint": "Design blueprint PPT slide, Figma documentation style, White background with cyan grid lines #66B8CC, title text, subtitle below, Figma selection boxes with control points, Annotation lines, numbered labels, Technical UI mockup aesthetic, Clean sans-serif Inter font, 16:9",
    "neo-brutalist-ui": "Neo-brutalist UI PPT slide, dashboard interface design, Cream background, title text, subtitle below, stats in cards, Pastel panels: mint #A8E4CF, yellow #FFD93D, lavender #E5B3FF, Thick 3px black outlines, Card-based layout, flat colors, Bold typography, Contemporary SaaS dashboard aesthetic, 16:9",
    "y2k-pixel": "Y2K pixel retro PPT slide, 1990s aesthetic, Dark background with noise texture, title text in pixel font, subtitle below, Bright colors: yellow #FFD700, orange #FF8C00, green #4A7C4E, Pixel art computer icons, CRT monitor graphics, Isometric tech illustrations, VT323 pixel font style, Vintage 1990s design, 16:9",
}

def batch_generate(
    title: str,
    subtitle: str,
    stats: list[str],
    output_dir: str = "output",
    styles: list[str] | None = None
) -> dict[str, str]:
    """
    Generate slides in multiple styles.

    Returns dict of {style_name: output_path}
    """
    Path(output_dir).mkdir(exist_ok=True)
    model = genai.GenerativeModel("gemini-3.1-flash-image-preview")
    results = {}
    target_styles = {k: v for k, v in STYLES.items() if styles is None or k in styles}

    for style_name, base_prompt in target_styles.items():
        stats_text = ", ".join(stats)
        full_prompt = f"{base_prompt}\nTitle: {title}\nSubtitle: {subtitle}\nKey stats: {stats_text}"

        try:
            response = model.generate_content(
                full_prompt,
                generation_config=genai.GenerationConfig(
                    response_modalities=["image"],
                )
            )
            for part in response.candidates[0].content.parts:
                if part.inline_data and part.inline_data.mime_type.startswith("image/"):
                    img_bytes = base64.b64decode(part.inline_data.data)
                    out_path = f"{output_dir}/{style_name}.png"
                    with open(out_path, "wb") as f:
                        f.write(img_bytes)
                    results[style_name] = out_path
                    print(f"✓ {style_name} -> {out_path}")
                    break
        except Exception as e:
            print(f"✗ {style_name} failed: {e}")

    return results


# Usage
results = batch_generate(
    title="Our Product Launch",
    subtitle="Redefining How Teams Collaborate",
    stats=["10K users", "99.9% uptime", "50ms latency", "SOC2 certified"],
    styles=["retro-pop", "minimal", "cyberpunk"]  # None = all 11 styles
)
```

### Load prompts from PROMPTS.md

```python
import re
from pathlib import Path

def load_prompts_from_md(prompts_file: str = "PROMPTS.md") -> dict[str, str]:
    """Parse PROMPTS.md and return {style_name: prompt_text}."""
    content = Path(prompts_file).read_text()
    prompts = {}

    # Match fenced code blocks preceded by a heading
    sections = re.split(r"^#{1,3} (.+)$", content, flags=re.MULTILINE)
    for i in range(1, len(sections), 2):
        name = sections[i].strip().lower().replace(" ", "-")
        body = sections[i + 1]
        code_match = re.search(r"```(?:\w+)?\n(.+?)```", body, re.DOTALL)
        if code_match:
            prompts[name] = code_match.group(1).strip()

    return prompts


prompts = load_prompts_from_md("PROMPTS.md")
for name, prompt in prompts.items():
    print(f"{name}: {prompt[:60]}...")
```

---

## Content Guidelines

| Element    | Rule                          | Example                                     |
|------------|-------------------------------|---------------------------------------------|
| Title      | ≤8 words, bold & clear        | `"What is Y Combinator"`                    |
| Subtitle   | ≤12 words, one-line summary   | `"The World's Most Famous Startup Accelerator"` |
| Stats      | 3–5 specific numbers          | `"2005, 4000+ companies, $600B"`            |
| Whitespace | ~30% of canvas                | —                                           |
| Font order | Title > Subtitle > Data > Deco | —                                           |

---

## Prompt Engineering Tips

1. **Be specific**: `"thick 4px black outlines"` > `"bold lines"`
2. **Always include `16:9`** at the end of every prompt
3. **Use hex codes alongside color names**: `"salmon pink #FF6B6B"`
4. **Keep text short** — AI image models handle short strings better
5. **Combine styles**: take a base style and add elements from another
6. **Iterate small changes**: tweak one attribute at a time for consistent results

### Customization pattern

```python
# Take a base prompt and inject your content
base = STYLES["retro-pop"]
custom_prompt = f"""
{base}
Slide title: "AI in Healthcare"
Subtitle: "Transforming patient outcomes with data"
Statistics: 2024, 40% cost reduction, 3x faster diagnosis
"""
```

---

## Style Selection Guide

| Use Case                   | Recommended Style       |
|----------------------------|-------------------------|
| Enterprise / corporate     | Minimalist Clean        |
| Tech / startup pitch       | Cyberpunk Neon          |
| Creative / brand           | Retro Pop Art           |
| Young / social audience    | Acid Graphics Y2K       |
| SaaS / product demo        | Neo-Brutalist UI        |
| Deep analysis / editorial  | Dark Editorial          |
| Technical documentation    | Design Blueprint        |
| High-end design            | Swiss International     |
| Personal / artistic        | Neo-Brutalism           |
| Instagram / light content  | Modern Minimal Pop      |
| Nostalgia / retro theme    | Y2K Pixel Retro         |

---

## Troubleshooting

**Image not returned from Gemini**
- Ensure `response_modalities=["image"]` is set
- Check `GEMINI_API_KEY` is valid and has image generation access
- Model must be `gemini-3.1-flash-image-preview` (not standard gemini-pro)

**Text not rendering correctly in slide**
- Shorten title to ≤8 words, subtitle to ≤12 words
- Avoid special characters (`"`, `'`, `&`) in text passed to prompts
- Spell out numbers: `"four thousand"` can sometimes render better than `"4000"`

**Style looks generic / not matching expected aesthetic**
- Include more specific descriptors: add font names, exact hex colors, specific decorative elements
- Reference the demo images in `demos/yc-intro/images/` to calibrate expectations
- Try appending `"high quality, detailed, professional"` to the prompt

**Rate limits**
```python
import time

for style_name, prompt in target_styles.items():
    # ... generate ...
    time.sleep(2)  # Respect API rate limits between requests
```

---

## Links

- **GitHub**: https://github.com/AAAAAAAJ/slides
- **APImart / nanobanana2**: https://apimart.ai/
- **Demo images**: `demos/yc-intro/images/` (22 samples across all styles)
```
