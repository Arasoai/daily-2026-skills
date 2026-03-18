```markdown
---
name: vercel-agent-skills
description: Create, publish, and install agent skills for AI coding agents using the Agent Skills format
triggers:
  - create an agent skill
  - add a skill to my agent
  - install agent skills
  - publish a skill for Claude or Cursor
  - build a SKILL.md
  - package instructions for AI coding agents
  - extend my coding agent with a skill
  - add skills to my AI assistant
---

# Agent Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Agent Skills is Vercel's official format and collection for packaging reusable instructions and scripts that extend AI coding agent capabilities. Skills work with Claude Code, Cursor, Codex, and other AI coding agents.

## What Are Agent Skills?

A **skill** is a directory containing:
- `SKILL.md` — Instructions the agent reads and follows
- `scripts/` — Optional helper scripts (bash, JS, Python, etc.)
- `references/` — Optional supporting documentation

Skills are installed into a project and automatically picked up by compatible AI agents when relevant tasks are detected.

## Installation

### Install skills from the official collection

```bash
npx skills add vercel-labs/agent-skills
```

### Install a specific skill

```bash
npx skills add vercel-labs/agent-skills/react-best-practices
npx skills add vercel-labs/agent-skills/web-design-guidelines
npx skills add vercel-labs/agent-skills/vercel-deploy-claimable
npx skills add vercel-labs/agent-skills/react-native-guidelines
npx skills add vercel-labs/agent-skills/composition-patterns
```

### Install from any GitHub repo

```bash
npx skills add <github-user>/<repo>
npx skills add <github-user>/<repo>/<skill-folder>
```

Skills are installed locally into your project so agents can discover and use them.

## Available Skills (Official Collection)

| Skill | Purpose | Trigger phrases |
|---|---|---|
| `react-best-practices` | 40+ React/Next.js performance rules | "Review this component", "Optimize data fetching" |
| `web-design-guidelines` | 100+ UI/UX/accessibility rules | "Review my UI", "Check accessibility" |
| `react-native-guidelines` | Mobile performance & platform patterns | "Optimize my RN app", "Fix animations" |
| `composition-patterns` | Compound components, avoid prop drilling | "Refactor boolean props", "Better component API" |
| `vercel-deploy-claimable` | Deploy to Vercel from conversation | "Deploy my app", "Push this live" |

## Creating Your Own Skill

### Skill directory structure

```
my-skill/
├── SKILL.md          # Required: agent instructions
├── scripts/
│   └── deploy.js     # Optional: automation scripts
└── references/
    └── api-docs.md   # Optional: supporting docs
```

### SKILL.md format

```markdown
---
name: my-skill-name
description: One-line description of what this skill does
triggers:
  - phrase a user might say
  - another natural trigger phrase
  - deploy my app
  - review my code
  - help me with X
  - set up Y
  - fix Z issue
---

# My Skill Name

Brief explanation of what this skill helps with.

## When to Use This Skill

- Situation A
- Situation B

## How to Do X

Step-by-step instructions the agent should follow...

\`\`\`javascript
// Real working code example
const result = await doSomething();
\`\`\`

## Common Patterns

...

## Troubleshooting

...
```

### Key SKILL.md rules

- **Frontmatter is required**: `name` (kebab-case), `description` (one line), `triggers` (6–8 phrases)
- **Triggers drive activation**: write them as natural user phrases — "deploy my app", "review this component"
- **Include real code examples**: agents copy from skills directly, so examples must work
- **Be specific**: vague instructions produce vague agent behavior
- **No secrets**: use `process.env.API_KEY` references, never hardcode values

## Real Example: A Deploy Skill

```markdown
---
name: my-app-deploy
description: Deploy this application to production with a single command
triggers:
  - deploy my app
  - push to production
  - release a new version
  - deploy and give me the link
  - ship this feature
  - go live
  - deploy to staging
---

# My App Deploy

Deploy this app to production using the project's deploy script.

## Steps

1. Run the build
2. Execute the deploy script
3. Report the deployment URL

## Deploy Command

\`\`\`bash
npm run build && node scripts/deploy.js
\`\`\`

## Environment Variables Required

- `DEPLOY_TOKEN` — set in `.env.local` or CI secrets
- `PROJECT_ID` — your project identifier
\`\`\`
```

## Real Example: A Script in a Skill

```javascript
// scripts/deploy.js — called by the agent during deployment
import { execSync } from "child_process";
import { readFileSync, createReadStream } from "fs";
import { resolve } from "path";

const token = process.env.DEPLOY_TOKEN;
if (!token) {
  console.error("Error: DEPLOY_TOKEN environment variable is not set.");
  process.exit(1);
}

const pkg = JSON.parse(readFileSync(resolve(process.cwd(), "package.json"), "utf8"));
const framework = detectFramework(pkg);

console.log(`Detected framework: ${framework}`);
console.log("Building project...");
execSync("npm run build", { stdio: "inherit" });

console.log("Deploying...");
// ... deployment logic
console.log(`Deployment successful!\nURL: https://your-app.example.com`);

function detectFramework(pkg) {
  const deps = { ...pkg.dependencies, ...pkg.devDependencies };
  if (deps["next"]) return "nextjs";
  if (deps["vite"]) return "vite";
  if (deps["astro"]) return "astro";
  return "static";
}
```

## Using the vercel-deploy-claimable Skill

Once installed, simply tell your agent:

```
Deploy my app
```

The agent will:
1. Package your project into a tarball (excluding `node_modules` and `.git`)
2. Auto-detect your framework from `package.json`
3. Upload to Vercel's deployment service
4. Return a preview URL and a claim URL to transfer ownership to your Vercel account

**Output example:**
```
Deployment successful!

Preview URL: https://skill-deploy-abc123.vercel.app
Claim URL:   https://vercel.com/claim-deployment?code=...
```

## Using the react-best-practices Skill

Install it, then ask your agent:

```
Review this React component for performance issues
```

```
How should I fetch data in this Next.js page?
```

```
Optimize this for bundle size
```

The agent applies 40+ rules across categories: eliminating waterfalls, bundle optimization, server-side performance, re-render optimization, and more.

## Using the web-design-guidelines Skill

```
Review my UI for accessibility issues
```

```
Audit this form component against best practices
```

```
Check my site for dark mode support
```

Covers 100+ rules: ARIA, semantic HTML, focus states, form validation, animation (prefers-reduced-motion), images, performance, i18n, touch interactions, and more.

## Publishing Your Own Skill Collection

1. Create a GitHub repository
2. Add skill directories at the root (each with a `SKILL.md`)
3. Users install with:

```bash
npx skills add your-github-username/your-repo
```

No package publishing required — the `skills` CLI pulls directly from GitHub.

## Skill Writing Best Practices

### Do
- Write triggers as **exact phrases users say** naturally
- Include **copy-paste ready code** that actually works
- Reference **environment variables** by name (`process.env.DATABASE_URL`)
- Add a **troubleshooting section** for common errors
- Keep instructions **imperative** ("Run X", "Check Y", "If Z then W")
- Use **real file paths** from the project structure

### Don't
- Use vague triggers like "help me" or "do stuff"
- Include hardcoded secrets, tokens, or passwords
- Write instructions only a human can follow (e.g., "use your judgment")
- Make the skill too broad — one focused skill beats one unfocused mega-skill

## Troubleshooting

### Skill not triggering
- Check that your trigger phrases match what users actually say
- Ensure `SKILL.md` frontmatter is valid YAML (no tabs, proper quoting)
- Re-install the skill: `npx skills add <repo>`

### Script not found
- Verify the script path in `SKILL.md` matches the actual file location
- Check the script has execute permissions: `chmod +x scripts/deploy.js`

### Agent ignores skill instructions
- Be more explicit and imperative in `SKILL.md` ("You MUST...", "Always run...")
- Break complex instructions into numbered steps
- Add a "When to use this skill" section to help the agent self-select correctly

### Framework not detected (vercel-deploy-claimable)
- Ensure `package.json` exists at the project root
- Add your framework as a dependency so auto-detection works
- The skill supports 40+ frameworks including Next.js, Vite, Astro, Remix, SvelteKit

## Resources

- [Agent Skills format spec](https://agentskills.io/)
- [Official skill collection](https://github.com/vercel-labs/agent-skills)
- [skills.sh registry](https://skills.sh/vercel-labs/agent-skills)
```
