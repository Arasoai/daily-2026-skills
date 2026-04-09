```markdown
---
name: awesome-persona-distill-skills
description: Curated list of Agent Skills for persona distillation — extracting communication styles, decision frameworks, and interaction patterns from conversations, works, and digital traces of real or fictional personas.
triggers:
  - "find a persona distill skill"
  - "how do I distill a persona into an agent skill"
  - "looking for a colleague or ex skill"
  - "create a persona skill for an AI agent"
  - "browse awesome persona skills list"
  - "add my skill to awesome persona distill"
  - "what persona agent skills are available"
  - "contribute a skill to the persona distill list"
---

# Awesome Persona Distill Skills

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated [Awesome List](https://awesome.re) of [Agent Skills](https://agentskills.io) focused on **persona distillation** — the practice of extracting expression styles, decision heuristics, and interaction patterns from conversations, digital traces, works, or public materials. Skills are organized into self-distillation, workplace relationships, intimate/family memory, public figures, and specialized themes.

---

## What This Project Does

`awesome-persona-distill-skills` is a community-maintained index of `.skill` files published on GitHub and compatible with the [Agent Skills](https://agentskills.io) ecosystem. Each listed skill can be installed into AI coding/chat agents (Claude Code, Cursor, Codex, etc.) to give them a specific persona or methodological perspective.

Persona distillation here means:
- Extracting **communication style** from chat logs, posts, or transcripts
- Encoding **decision frameworks** from books, interviews, or public writing
- Building **interactive role personas** for practice, reflection, or tooling

It does **not** claim to fully replicate real individuals.

---

## Installing a Skill from This List

### Via Agent Skills CLI (if using agentskills.io toolchain)

```bash
# Install a skill by its GitHub repo
agentskills install https://github.com/alchaincyf/steve-jobs-skill

# Install and activate in current project
agentskills install https://github.com/titanwings/colleague-skill --activate
```

### Manually (copy SKILL.md into your project)

```bash
# Clone the target skill repo
git clone https://github.com/alchaincyf/munger-skill /tmp/munger-skill

# Copy the skill file into your project's skills directory
cp /tmp/munger-skill/SKILL.md ./.skills/munger.skill.md
```

### Via curl (quick install)

```bash
mkdir -p .skills
curl -sSL https://raw.githubusercontent.com/alchaincyf/naval-skill/main/SKILL.md \
  -o .skills/naval.skill.md
```

---

## Skill Categories & Key Examples

### Self-Distillation & Meta Tools

| Skill | Repo | Description |
|-------|------|-------------|
| 自己.skill | [notdog1998/yourself-skill](https://github.com/notdog1998/yourself-skill) | Self-distillation assistant from personal conversations |
| 女娲.skill | [alchaincyf/nuwa-skill](https://github.com/alchaincyf/nuwa-skill) | Extract reusable skills from mental models & decision heuristics |
| 永生.skill | [agenmod/immortal-skill](https://github.com/agenmod/immortal-skill) | Multi-dimensional digital persona from chat logs |
| Forge Skill | [YIKUAIBANZI/forge-skill](https://github.com/YIKUAIBANZI/forge-skill) | Separate self-distill and other-distill pipelines |

### Workplace & Academic

| Skill | Repo | Description |
|-------|------|-------------|
| 同事.skill | [titanwings/colleague-skill](https://github.com/titanwings/colleague-skill) | Former colleague work context & communication style |
| 老板.skill | [vogtsw/boss-skills](https://github.com/vogtsw/boss-skills) | Manager judgment criteria & review style |
| 导师.skill | [ybq22/supervisor](https://github.com/ybq22/supervisor) | Supervisor guidance style for students |

### Public Figures & Methodology

| Skill | Repo | Description |
|-------|------|-------------|
| 乔布斯.skill | [alchaincyf/steve-jobs-skill](https://github.com/alchaincyf/steve-jobs-skill) | Jobs' product judgment & narrative style |
| 芒格.skill | [alchaincyf/munger-skill](https://github.com/alchaincyf/munger-skill) | Munger's mental models & decision heuristics |
| 费曼.skill | [alchaincyf/feynman-skill](https://github.com/alchaincyf/feynman-skill) | Feynman's explanation style & truth-seeking |
| 纳瓦尔.skill | [alchaincyf/naval-skill](https://github.com/alchaincyf/naval-skill) | Naval's wealth, leverage & judgment frameworks |

---

## Authoring a New Persona Skill (SKILL.md Structure)

A valid persona distill skill follows this structure:

```markdown
---
name: your-persona-skill
description: One-line summary of who/what is distilled and for what use.
triggers:
  - "ask in the style of X"
  - "apply X's framework to this problem"
  - "what would X think about this"
  - "distill X's decision approach"
  - "channel X's communication style"
  - "use X methodology"
---

# Your Persona Skill

> Skill by [your-handle](https://your-site)

## Persona Overview
Brief description of the person/role and what aspect is being distilled.

## Core Frameworks
The key mental models, heuristics, or communication patterns extracted.

## Example Interactions
Concrete Q&A examples showing the persona in action.

## Source Materials
Links to the public works, transcripts, or data used for distillation.
```

---

## JavaScript Tooling Example

This project uses JavaScript for automation (PR generation, issue triage). Here's how the contribution workflow script pattern looks:

```javascript
// .github/scripts/generate-pr.mjs
// Triggered when an issue receives the 'approved' label

import { Octokit } from "@octokit/rest";

const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });

async function generateSkillPR({ issueNumber, skillName, skillRepo, category, description }) {
  const owner = "xixu-me";
  const repo = "awesome-persona-distill-skills";

  // 1. Create a new branch for the PR
  const baseSha = await octokit.repos.getBranch({ owner, repo, branch: "main" });
  const branchName = `add-skill/${skillName}-${issueNumber}`;

  await octokit.git.createRef({
    owner,
    repo,
    ref: `refs/heads/${branchName}`,
    sha: baseSha.data.commit.sha,
  });

  // 2. Read current README
  const readmeFile = await octokit.repos.getContent({
    owner,
    repo,
    path: "README.md",
  });

  const currentContent = Buffer.from(readmeFile.data.content, "base64").toString("utf8");

  // 3. Insert new skill entry into correct category section
  const newEntry = `- [${skillName}](${skillRepo}) - ${description}`;
  const updatedContent = insertIntoCategory(currentContent, category, newEntry);

  // 4. Commit the updated README
  await octokit.repos.createOrUpdateFileContents({
    owner,
    repo,
    path: "README.md",
    message: `feat: add ${skillName} to ${category}`,
    content: Buffer.from(updatedContent).toString("base64"),
    sha: readmeFile.data.sha,
    branch: branchName,
  });

  // 5. Open a Pull Request
  const pr = await octokit.pulls.create({
    owner,
    repo,
    title: `Add ${skillName}`,
    body: `Closes #${issueNumber}\n\nAdds **${skillName}** to the **${category}** section.\n\n> Auto-generated by approved-issue workflow.`,
    head: branchName,
    base: "main",
  });

  return pr.data.html_url;
}

function insertIntoCategory(readme, category, entry) {
  const sectionRegex = new RegExp(`(## ${category}[\\s\\S]*?)(\n## |$)`);
  return readme.replace(sectionRegex, (match, section, next) => {
    return `${section.trimEnd()}\n${entry}\n${next}`;
  });
}
```

---

## Contributing a New Skill

### Step 1 — Submit an Issue

Fill out the issue form at the repository. Required fields:

```
Skill Name: 你的技能名.skill
GitHub Repo URL: https://github.com/your-handle/your-skill
Category: 自我蒸馏与元工具 | 职场与学术关系 | 亲密关系与家庭记忆 | 公众人物与方法论视角 | 精神性与专门化主题
Description (one line): What persona/framework is distilled and for what purpose.
```

### Step 2 — Wait for `approved` Label

A maintainer will review and apply the `approved` label. The GitHub Actions workflow then auto-generates a PR.

### Step 3 — Direct PRs (for fixes only)

For fixing broken links, typos, or formatting, submit a PR directly:

```bash
git clone https://github.com/xixu-me/awesome-persona-distill-skills
cd awesome-persona-distill-skills
git checkout -b fix/broken-link-colleague-skill
# Edit README.md
git commit -m "fix: update broken link for colleague-skill"
git push origin fix/broken-link-colleague-skill
# Open PR on GitHub
```

---

## Skill Quality Checklist

Before submitting a skill for inclusion, verify:

```
[ ] Public GitHub repository (not private)
[ ] Contains a valid SKILL.md file at repo root
[ ] Skill name follows pattern: 名称.skill or descriptive-name-skill
[ ] One-line description is clear and specific
[ ] Source materials or methodology are documented
[ ] Does not claim complete replication of a real individual
[ ] No hardcoded secrets, API keys, or private data
[ ] Compatible with agentskills.io format (YAML frontmatter + Markdown body)
```

---

## Common Patterns

### Loading a Public Figure Skill for Decision Analysis

```javascript
// Node.js: load and inject a persona skill into an LLM prompt
import { readFileSync } from "fs";
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const skillContent = readFileSync(".skills/munger.skill.md", "utf8");

const response = await client.messages.create({
  model: "claude-opus-4-5",
  max_tokens: 1024,
  system: skillContent,
  messages: [
    {
      role: "user",
      content: "Evaluate this business decision: expanding into a new market with high capital requirements but unclear demand.",
    },
  ],
});

console.log(response.content[0].text);
```

### Self-Distillation: Building Your Own Skill from Chat Logs

```javascript
// Parse exported chat JSON and extract patterns for skill generation
import { readFileSync, writeFileSync } from "fs";

function extractPersonaPatterns(chatLogPath) {
  const raw = JSON.parse(readFileSync(chatLogPath, "utf8"));

  const myMessages = raw.messages
    .filter((m) => m.sender === "me")
    .map((m) => m.text);

  // Simple heuristic extraction — replace with LLM call in practice
  const patterns = {
    avgMessageLength: Math.round(
      myMessages.reduce((s, m) => s + m.length, 0) / myMessages.length
    ),
    topPhrases: extractTopPhrases(myMessages, 10),
    questionFrequency: myMessages.filter((m) => m.includes("?")).length / myMessages.length,
  };

  return patterns;
}

function extractTopPhrases(messages, n) {
  const freq = {};
  messages.forEach((msg) => {
    msg.split(/\s+/).forEach((word) => {
      if (word.length > 3) freq[word] = (freq[word] || 0) + 1;
    });
  });
  return Object.entries(freq)
    .sort((a, b) => b[1] - a[1])
    .slice(0, n)
    .map(([word]) => word);
}

const patterns = extractPersonaPatterns("./my-chat-export.json");
console.log("Distilled patterns:", patterns);
```

---

## Troubleshooting

**Skill not recognized by agent**
- Ensure the file has valid YAML frontmatter with `name`, `description`, and `triggers` fields
- Check that the file is named `SKILL.md` (case-sensitive) at the repo root
- Verify the skill is installed in the directory your agent scans (e.g., `.skills/`, `.cursor/skills/`)

**PR not auto-generated after `approved` label**
- Check the repository's GitHub Actions tab for workflow errors
- The workflow requires `GITHUB_TOKEN` to have write permissions to the repo
- Issue must have been submitted via the official issue form template

**Broken link in the list**
- Submit a direct PR to fix the URL — no issue required for link fixes
- Check if the skill repo was renamed or made private by the author

**Skill content violates privacy**
- Skills must only use publicly available materials or data the creator owns
- Private chat logs should never be committed to public skill repos — use local-only distillation tools like [Forge Skill](https://github.com/YIKUAIBANZI/forge-skill)

---

## Related Resources

- [Agent Skills Platform](https://agentskills.io) — official skill registry and format spec
- [Awesome List Guidelines](https://github.com/sindresorhus/awesome/blob/main/pull_request_template.md) — quality standards this list follows
- [CONTRIBUTING.md](https://github.com/xixu-me/awesome-persona-distill-skills/blob/main/CONTRIBUTING.md) — detailed contribution conventions
```
