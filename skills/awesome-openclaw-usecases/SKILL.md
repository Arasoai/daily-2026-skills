```markdown
---
name: awesome-openclaw-usecases
description: Community collection of real-world OpenClaw (formerly ClawdBot/MoltBot) use cases, skills, and plugins for automating daily life tasks.
triggers:
  - "set up OpenClaw use case"
  - "add OpenClaw skill"
  - "create openclaw usecase"
  - "how to use OpenClaw for"
  - "openclaw plugin example"
  - "openclaw automation setup"
  - "contribute to awesome openclaw"
  - "openclaw daily workflow"
---

# Awesome OpenClaw Use Cases

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A community-curated list of real-world [OpenClaw](https://github.com/openclaw/openclaw) (formerly ClawdBot, MoltBot) use cases, skills, and plugins. This repo helps users discover practical ways to use OpenClaw agents in daily life across productivity, research, finance, DevOps, and creative workflows.

---

## What Is This Project?

`awesome-openclaw-usecases` is an **Awesome List** — a curated Markdown collection hosted on GitHub. Each use case lives in `usecases/<slug>.md` and describes a real, community-verified workflow built on OpenClaw.

It is **not** a library or CLI tool itself — it is a knowledge base and contribution target. AI agents use it to:

- Discover what OpenClaw can do
- Copy and adapt verified use case patterns
- Contribute new use cases following the established format

---

## Repository Structure

```
awesome-openclaw-usecases/
├── README.md                  # Main index of all use cases
├── CONTRIBUTING.md            # Contribution guidelines
├── usecases/                  # One Markdown file per use case
│   ├── daily-reddit-digest.md
│   ├── self-healing-home-server.md
│   ├── second-brain.md
│   └── ...
└── assets/                    # Images, diagrams (optional)
```

---

## Use Case File Format

Every file in `usecases/` follows this schema:

```markdown
# Use Case Title

**Category:** Productivity | Research | DevOps | Creative | Finance | Social Media

## Description
One-paragraph plain-English description of what this automates.

## Prerequisites
- OpenClaw >= X.Y
- Required skills/plugins: `skill-name`, `other-skill`
- External accounts: e.g. Reddit API, Todoist API

## Setup

### 1. Install Required Skills
\```bash
openclaw skill install reddit-digest
openclaw skill install email-sender
\```

### 2. Configure Environment Variables
\```bash
export REDDIT_CLIENT_ID="your_reddit_client_id"
export REDDIT_CLIENT_SECRET="your_reddit_client_secret"
export EMAIL_SMTP_HOST="smtp.example.com"
export EMAIL_TO="you@example.com"
\```

### 3. Agent Prompt / SKILL.md Snippet
\```
Every morning at 8am, fetch the top 5 posts from r/programming and r/MachineLearning,
summarize them in bullet points, and email the digest to $EMAIL_TO.
\```

## How It Works
Step-by-step explanation of the agent's reasoning loop.

## Variations
- Swap subreddits for any community
- Change schedule to weekly

## Security Notes
- Never hardcode credentials; always use environment variables.
- Review skill source before installing from third-party repos.

## Author
[@github-handle](https://github.com/github-handle)
```

---

## OpenClaw Core Concepts (Referenced Across Use Cases)

### Skills
Skills extend the OpenClaw agent with new capabilities (web search, email, calendar, browser automation, etc.).

```bash
# Install a skill from the OpenClaw registry
openclaw skill install <skill-name>

# Install from a GitHub repo (community skills)
openclaw skill install github:username/skill-repo

# List installed skills
openclaw skill list

# Remove a skill
openclaw skill remove <skill-name>
```

### Plugins
Plugins integrate OpenClaw with external platforms (Telegram, Discord, Slack, WhatsApp).

```bash
openclaw plugin install telegram
openclaw plugin install discord
openclaw plugin install slack
```

### MCP (Multi-Channel Protocol)
Configure once, expose to multiple frontends:

```yaml
# openclaw.config.yaml
mcp:
  channels:
    - type: telegram
      token: $TELEGRAM_BOT_TOKEN
    - type: discord
      token: $DISCORD_BOT_TOKEN
    - type: webui
      port: 3000
```

---

## Common Use Case Patterns

### Pattern 1: Scheduled Digest (e.g. Daily Reddit Digest)

```markdown
## Agent Prompt
Every day at 07:30 local time:
1. Use the `reddit` skill to fetch top 10 posts from [r/programming, r/MachineLearning]
2. Filter posts with score > 100
3. Summarize each post in 2 sentences
4. Send the digest via email using the `email` skill to $EMAIL_TO
5. Store the digest in memory with today's date as key
```

```bash
# Required skills
openclaw skill install reddit-digest
openclaw skill install email-sender

# Required env vars
export REDDIT_CLIENT_ID=$REDDIT_CLIENT_ID
export REDDIT_CLIENT_SECRET=$REDDIT_CLIENT_SECRET
export EMAIL_SMTP_HOST=$EMAIL_SMTP_HOST
export EMAIL_SMTP_PORT=587
export EMAIL_USERNAME=$EMAIL_USERNAME
export EMAIL_PASSWORD=$EMAIL_PASSWORD
export EMAIL_TO=$EMAIL_TO
```

---

### Pattern 2: STATE.yaml Multi-Agent Orchestration

Used in: `autonomous-project-management.md`, `content-factory.md`

```yaml
# STATE.yaml — shared state file for parallel subagents
project: my-youtube-channel
phase: research
tasks:
  - id: topic-scout
    agent: research-agent
    status: in_progress
    output: null
  - id: script-writer
    agent: writing-agent
    status: waiting
    depends_on: topic-scout
  - id: thumbnail-gen
    agent: design-agent
    status: waiting
    depends_on: script-writer
last_updated: "2026-03-17T06:00:00Z"
```

```markdown
## Orchestrator Prompt
Read STATE.yaml. For each task where status=waiting and all depends_on tasks
have status=done, spawn the appropriate subagent, update status to in_progress,
and write results back to STATE.yaml when complete.
```

---

### Pattern 3: Self-Healing Infrastructure Agent

Used in: `self-healing-home-server.md`

```markdown
## Agent Prompt
You are an always-on infrastructure agent with SSH access to my home server.

Every 5 minutes:
1. SSH into $SERVER_HOST and run `systemctl is-failed --all`
2. For any failed services, attempt `systemctl restart <service>`
3. If restart fails twice, send a Telegram alert via $TELEGRAM_CHAT_ID
4. Log all events to /var/log/openclaw-health.log

Credentials:
- SSH key is at ~/.ssh/homeserver_key (never log or expose this path)
- Use $SERVER_USER@$SERVER_HOST
```

```bash
export SERVER_HOST=$SERVER_HOST
export SERVER_USER=$SERVER_USER
export TELEGRAM_BOT_TOKEN=$TELEGRAM_BOT_TOKEN
export TELEGRAM_CHAT_ID=$TELEGRAM_CHAT_ID
```

---

### Pattern 4: n8n Webhook Delegation

Used in: `n8n-workflow-orchestration.md`

```markdown
## Agent Prompt
When the user asks you to perform any external API action (send email, create CRM record,
post to Slack), call the appropriate n8n webhook instead of using credentials directly.

Webhook map:
- Send email → POST $N8N_WEBHOOK_BASE/send-email
- Create contact → POST $N8N_WEBHOOK_BASE/crm-create
- Slack message → POST $N8N_WEBHOOK_BASE/slack-notify

Always pass action payload as JSON body. Never ask for or store API keys.
```

```bash
export N8N_WEBHOOK_BASE=https://n8n.yourdomain.com/webhook
```

---

### Pattern 5: Second Brain / Memory

Used in: `second-brain.md`, `knowledge-base-rag.md`

```markdown
## Agent Prompt
When the user sends any message starting with "remember:" or "save:", store it
in memory with the current timestamp and 3 auto-generated tags.

When the user asks "find anything about X", perform semantic search across all
memories and return the top 5 matches with their timestamps and tags.

Use the `memory` skill for persistence and the `semantic-search` skill for retrieval.
```

```bash
openclaw skill install memory
openclaw skill install semantic-search

export MEMORY_BACKEND=sqlite          # or: postgres, duckdb
export MEMORY_DB_PATH=$HOME/.openclaw/memory.db
export EMBEDDING_MODEL=text-embedding-3-small
export OPENAI_API_KEY=$OPENAI_API_KEY
```

---

## Contributing a New Use Case

### Step 1: Clone the Repo

```bash
git clone https://github.com/hesamsheikh/awesome-openclaw-usecases.git
cd awesome-openclaw-usecases
```

### Step 2: Create Your Use Case File

```bash
cp usecases/inbox-declutter.md usecases/my-new-usecase.md
# Edit the file following the format above
```

### Step 3: Add to README.md

Find the correct category table in `README.md` and add a row:

```markdown
| [My New Use Case](usecases/my-new-usecase.md) | Short description of what it automates. |
```

### Step 4: Update the Badge Count

In `README.md`, find and increment:

```markdown
![Use Cases](https://img.shields.io/badge/usecases-40-blue?style=flat-square)
# Change 40 → 41
```

### Step 5: Submit a Pull Request

```bash
git checkout -b usecase/my-new-usecase
git add usecases/my-new-usecase.md README.md
git commit -m "feat: add My New Use Case to Productivity"
git push origin usecase/my-new-usecase
# Open PR against main
```

**Contribution rules (from CONTRIBUTING.md):**
- Only submit use cases you have personally verified work for at least one day
- No crypto-related use cases
- All credentials must use environment variables — never hardcoded
- Always include a Security Notes section if your use case touches credentials, SSH, or third-party plugins

---

## Security Checklist for All Use Cases

> ⚠️ Skills and third-party plugins are **not audited** by this list's maintainer.

Before installing any community skill or use case:

- [ ] Read the skill's source code before installing
- [ ] Check what permissions/scopes the skill requests
- [ ] Use environment variables for ALL secrets (`$VAR_NAME`, never literal values)
- [ ] Prefer `n8n` webhook delegation over direct API key injection into agents
- [ ] Run untrusted skills in an isolated environment (Docker, VM)
- [ ] Review open issues on the skill's repo for reported vulnerabilities
- [ ] Rotate any credentials exposed to an agent regularly

---

## Category Reference

| Category | Description |
|---|---|
| Social Media | Reddit, X/Twitter, YouTube automation |
| Creative & Building | Content pipelines, game dev, podcasts |
| Infrastructure & DevOps | SSH agents, n8n, self-healing servers |
| Productivity | CRM, calendars, meeting notes, habits |
| Research & Learning | arXiv, RAG, market research, idea validation |
| Finance & Trading | Prediction markets, earnings tracking |

---

## Troubleshooting

### Skill not found after install
```bash
openclaw skill list | grep <skill-name>
# If missing, try:
openclaw skill install github:owner/skill-repo --force
```

### Agent not triggering scheduled tasks
- Confirm OpenClaw daemon is running: `openclaw status`
- Check cron/schedule syntax in your agent prompt matches OpenClaw's scheduler format
- Review logs: `openclaw logs --tail 100`

### Memory/RAG returning irrelevant results
- Re-index: `openclaw memory reindex`
- Check embedding model env var is set correctly
- Increase `top_k` in your semantic search prompt

### Multi-agent STATE.yaml conflicts
- Add file locking to STATE.yaml writes: use the `file-lock` skill
- Ensure each subagent writes to a unique `tasks[].output` key, not the same field

---

## Related Resources

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Skills Registry](https://github.com/openclaw/skills)
- [Community Discord](https://discord.gg/vtJykN3t)
- [Maintainer on X](https://x.com/Hesamation)
```
