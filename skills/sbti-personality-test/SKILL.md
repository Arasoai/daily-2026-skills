```markdown
---
name: sbti-personality-test
description: A single-page HTML personality/quiz test web app (SBTI Test mirror) with split image and HTML assets, deployable as a static site.
triggers:
  - "set up sbti test"
  - "deploy personality quiz html"
  - "mirror sbti test page"
  - "customize sbti quiz"
  - "host static quiz site"
  - "modify sbti test questions"
  - "add sbti test to my site"
  - "embed personality test html"
---

# SBTI Personality Test (Mirror)

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Does

SBTI Test is a **single-page static web application** that presents a personality/type quiz (SBTI-style) to users. The entire app is self-contained HTML with associated image assets — no backend, no build step, no dependencies to install. The HTML file itself is the full source code.

- **Live demo**: https://sbti.unun.dev  
- **Original author**: Bilibili @蛆肉儿串儿  
- **License**: None declared — use at your own discretion, do not trouble the original author.

The repo splits images and HTML into separate files for easier hosting/mirroring.

---

## Project Structure

```
SBTI-test/
├── index.html          # Main quiz page (entire app logic + UI)
├── images/             # Quiz result images, type illustrations, etc.
│   ├── *.png / *.jpg
└── README.md
```

---

## How to Deploy

### Option 1: GitHub Pages (recommended)

1. Fork or clone the repo.
2. Push to your GitHub repository.
3. Go to **Settings → Pages → Source**: set branch to `main`, folder to `/ (root)`.
4. Your site will be live at `https://<username>.github.io/<repo-name>/`.

### Option 2: Netlify / Vercel (drag & drop)

1. Download/clone the repo locally.
2. Drag the project folder into [Netlify Drop](https://app.netlify.com/drop) or import via Vercel dashboard.
3. No build command needed — output directory is the root `.`.

**Netlify config (optional `netlify.toml`):**
```toml
[build]
  publish = "."
  command = ""

[[headers]]
  for = "/*"
  [headers.values]
    Cache-Control = "public, max-age=3600"
```

### Option 3: Local Development

```bash
# Clone the repo
git clone https://github.com/UnluckyNinja/SBTI-test.git
cd SBTI-test

# Serve locally (Python 3)
python3 -m http.server 8080
# Then open http://localhost:8080

# OR with Node.js npx
npx serve .
# Then open http://localhost:3000
```

---

## Key Files & How to Modify

### `index.html` — The Entire App

Since the whole app is one HTML file, all customization happens here.

**Typical structure inside `index.html`:**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SBTI 测试</title>
  <style>
    /* All CSS is inline */
    body { font-family: sans-serif; ... }
    .question { ... }
    .result { ... }
  </style>
</head>
<body>
  <div id="app">
    <!-- Quiz questions rendered here -->
  </div>
  <script>
    // All quiz logic is inline JavaScript
    const questions = [ ... ];
    const results = { ... };
    // Scoring, rendering, navigation logic
  </script>
</body>
</html>
```

---

## Common Customization Patterns

### 1. Change the Page Title / Language

```html
<!-- In <head> -->
<title>My Custom Personality Test</title>
<html lang="en">  <!-- change from zh-CN -->
```

### 2. Add / Edit Questions

Locate the `questions` array in the `<script>` block:

```javascript
const questions = [
  {
    id: 1,
    text: "你更喜欢...",          // Question text
    options: [
      { text: "选项A", scores: { E: 1, I: 0 } },
      { text: "选项B", scores: { E: 0, I: 1 } }
    ]
  },
  // Add more question objects here
];
```

### 3. Modify Result Images

Result images are referenced relative to the `images/` directory:

```javascript
const results = {
  "ENFP": {
    label: "活动家",
    image: "images/ENFP.png",   // <-- update path if you rename/move images
    description: "..."
  },
  // ...
};
```

To replace an image:
```bash
# Replace an image file (keep same filename)
cp my-new-ENFP.png images/ENFP.png

# OR update the path in index.html results object
```

### 4. Embed in an Existing Site (iframe)

```html
<!-- Embed the quiz in any page -->
<iframe
  src="https://sbti.unun.dev"
  width="100%"
  height="800px"
  frameborder="0"
  style="border-radius: 12px;"
  title="SBTI Personality Test">
</iframe>
```

### 5. Share Results via URL Hash

If you want to add deep-linking for results, add this to the script:

```javascript
// Save result to URL hash
function showResult(type) {
  window.location.hash = type;  // e.g. #ENFP
  // ... render result UI
}

// On page load, check for hash
window.addEventListener('load', () => {
  const hash = window.location.hash.slice(1);  // e.g. "ENFP"
  if (hash && results[hash]) {
    showResult(hash);
  }
});
```

---

## Hosting on a Custom Domain

### Cloudflare Pages

```bash
# Connect GitHub repo in Cloudflare Pages dashboard
# Build settings:
#   Framework preset: None
#   Build command: (leave empty)
#   Build output directory: /
#   Root directory: /
```

### Custom domain with GitHub Pages

1. Add a `CNAME` file to the repo root:
```
sbti.yourdomain.com
```
2. Point your DNS CNAME record to `<username>.github.io`.

---

## Troubleshooting

### Images not loading
- Verify `images/` folder is present and filenames match exactly (case-sensitive on Linux servers).
- Check browser console for 404 errors on image paths.
- Ensure paths in `index.html` are relative, not absolute: `images/X.png` not `/images/X.png` (unless hosted at root).

### Page shows blank / broken layout
- Open browser DevTools → Console for JS errors.
- Make sure you're serving via HTTP(S), not opening `file://` directly (some browsers block relative paths).
- Use `python3 -m http.server` or `npx serve .` locally.

### Quiz logic not working after edits
- Validate your JSON/JS syntax — a missing comma in `questions` array breaks everything.
- Use browser DevTools → Sources to set breakpoints in the inline script.

### Mobile layout issues
- Ensure `<meta name="viewport" content="width=device-width, initial-scale=1.0">` is present in `<head>`.
- Add CSS media queries in the `<style>` block:
```css
@media (max-width: 600px) {
  .question { font-size: 16px; padding: 10px; }
  .options { flex-direction: column; }
}
```

---

## Quick Reference

| Task | Where |
|------|-------|
| Edit questions | `<script>` block → `questions` array |
| Edit results/types | `<script>` block → `results` object |
| Change images | `images/` folder + update paths in `results` |
| Change styles | `<style>` block in `<head>` |
| Deploy static | GitHub Pages / Netlify / Vercel / `npx serve .` |
| Local preview | `python3 -m http.server 8080` |

---

## Credits

- **Mirror repo**: [UnluckyNinja/SBTI-test](https://github.com/UnluckyNinja/SBTI-test)  
- **Original creator**: Bilibili @蛆肉儿串儿  
- **No license declared** — respect the original author's work.
```
