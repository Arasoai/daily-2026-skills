```markdown
---
name: clawchief-openclaw-chief-of-staff
description: Turn OpenClaw into a founder/chief-of-staff operating system with Todoist-backed tasks, Gmail/Calendar automation, and deterministic cron workflows
triggers:
  - set up clawchief
  - configure openclaw chief of staff
  - automate my daily tasks with clawchief
  - integrate todoist with openclaw
  - set up executive assistant workflow
  - clawchief cron jobs
  - migrate clawchief v2 to v3
  - clawchief helper scripts
---

# clawchief: OpenClaw Chief of Staff Operating System

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`clawchief` is an opinionated starter kit that turns [OpenClaw](https://openclaw.dev) into a founder/chief-of-staff OS. It separates prioritization policy, resolution policy, live task state (Todoist), and recurring orchestration (cron) into discrete, composable layers.

---

## Architecture Overview

```
clawchief/
├── priority-map.md          # What matters and why
├── auto-resolver.md         # How to resolve ambiguous decisions
├── meeting-notes.md         # Meeting-note ingestion policy
├── location-awareness.md    # Place-based task curation
├── knowledge-compiler.md    # System compilation / linting policy
├── task-system-acceptance.md
├── scripts/
│   ├── todoist_cli.py       # Live task system interface
│   ├── ea_gmail.py          # Gmail helper
│   ├── ea_calendar.py       # Calendar helper
│   ├── sheet_helper.py      # Google Sheets helper
│   └── bd_outreach.py       # Business development outreach
└── tests/

skills/
├── task-system-contract/    # Shared task rules
├── executive-assistant/
├── business-development/
├── daily-task-manager/
└── daily-task-prep/

workspace/
├── HEARTBEAT.md             # Thin recurring orchestration doc
├── TOOLS.md                 # Local environment details
└── memory/
    └── meeting-notes-state.json

cron/
└── jobs.template.json
```

**Core design principle:** live task state lives in Todoist, not in markdown files. Google workflows go through `gws`-backed helper scripts, not raw API calls.

---

## Installation

### Prerequisites

- OpenClaw installed and configured
- `gws` (Google Workspace CLI) set up
- Todoist account with API token
- Python 3.10+

### Step-by-step

```bash
# 1. Clone the repo
git clone https://github.com/snarktank/clawchief
cd clawchief

# 2. Copy skills into OpenClaw
cp -r skills/* ~/.openclaw/skills/

# 3. Copy source-of-truth and workspace files
cp -r clawchief/ ~/.openclaw/workspace/clawchief/
cp -r workspace/ ~/.openclaw/workspace/

# 4. Set your Todoist API token
echo 'TODOIST_API_TOKEN=your_token_here' >> ~/.openclaw/.env
# or export it in your shell profile:
export TODOIST_API_TOKEN="your_token_here"

# 5. Customize your environment
$EDITOR ~/.openclaw/workspace/TOOLS.md

# 6. Set up gws for Google Workspace access
# Follow SETUP-GWS.md in the repo

# 7. Create cron jobs from the template
cat cron/jobs.template.json  # review, then install with your cron manager

# 8. Run the install checklist
cat INSTALL-CHECKLIST.md
```

---

## Key Scripts

### `todoist_cli.py` — Live Task Interface

All live task operations go through this script. It is the only canonical interface to the task system.

```python
# clawchief/scripts/todoist_cli.py usage patterns

import subprocess

def run_todoist(args: list[str]) -> str:
    result = subprocess.run(
        ["python", "clawchief/scripts/todoist_cli.py"] + args,
        capture_output=True, text=True
    )
    return result.stdout

# List tasks in a project
print(run_todoist(["list", "--project", "Work"]))

# Add a task
print(run_todoist(["add", "--content", "Review Q2 OKRs", "--due", "today", "--priority", "p1"]))

# Complete a task by ID
print(run_todoist(["complete", "--id", "12345678"]))

# List tasks due today
print(run_todoist(["list", "--filter", "today"]))
```

Direct Python usage (import pattern):

```python
import os
from todoist_api_python.api import TodoistAPI

api = TodoistAPI(os.environ["TODOIST_API_TOKEN"])

# Get all tasks due today
tasks = api.get_tasks(filter="today")
for task in tasks:
    print(f"[{task.priority}] {task.content} — due {task.due}")

# Add a task
new_task = api.add_task(
    content="Prepare board update",
    due_string="tomorrow",
    priority=2,  # p2
    project_id="your_project_id"
)
print(f"Created: {new_task.id} — {new_task.content}")

# Complete a task
api.close_task(task_id="12345678")
```

### `ea_gmail.py` — Gmail Helper

```python
# clawchief/scripts/ea_gmail.py usage
# Uses gws under the hood — do not call Gmail APIs directly

import subprocess

def gmail_search(query: str) -> str:
    """Search at message level, not thread level."""
    result = subprocess.run(
        ["python", "clawchief/scripts/ea_gmail.py", "search", "--query", query],
        capture_output=True, text=True
    )
    return result.stdout

# Always search at message level
results = gmail_search("from:partner@example.com subject:contract after:2026/04/01")
print(results)

# Draft a reply
subprocess.run([
    "python", "clawchief/scripts/ea_gmail.py",
    "draft-reply",
    "--message-id", "msg_abc123",
    "--body", "Thanks for reaching out. I'll review and follow up by EOD."
])
```

### `ea_calendar.py` — Calendar Helper

```python
# clawchief/scripts/ea_calendar.py usage
# Checks ALL relevant calendars before suggesting a booking

import subprocess

def check_availability(date: str, duration_minutes: int = 60) -> str:
    result = subprocess.run(
        [
            "python", "clawchief/scripts/ea_calendar.py",
            "check-availability",
            "--date", date,
            "--duration", str(duration_minutes)
        ],
        capture_output=True, text=True
    )
    return result.stdout

# Always check all calendars before booking
slots = check_availability("2026-04-10", duration_minutes=45)
print(slots)

# Book a meeting
subprocess.run([
    "python", "clawchief/scripts/ea_calendar.py",
    "book",
    "--title", "1:1 with Sarah",
    "--date", "2026-04-10",
    "--time", "14:00",
    "--duration", "45",
    "--attendees", "sarah@example.com"
])
```

### `sheet_helper.py` — Google Sheets

```python
# clawchief/scripts/sheet_helper.py usage

import subprocess

def append_row(spreadsheet_id: str, sheet_name: str, row: list) -> None:
    subprocess.run([
        "python", "clawchief/scripts/sheet_helper.py",
        "append",
        "--spreadsheet-id", spreadsheet_id,
        "--sheet", sheet_name,
        "--values", ",".join(str(v) for v in row)
    ])

# Log an outreach touchpoint
append_row(
    spreadsheet_id=os.environ["BD_SHEET_ID"],
    sheet_name="Outreach",
    row=["2026-04-08", "Acme Corp", "intro email sent", "sarah@acme.com"]
)
```

### `bd_outreach.py` — Business Development

```python
# clawchief/scripts/bd_outreach.py usage

import subprocess

# Log a new outreach contact
subprocess.run([
    "python", "clawchief/scripts/bd_outreach.py",
    "log",
    "--company", "Acme Corp",
    "--contact", "sarah@acme.com",
    "--stage", "intro",
    "--notes", "Met at SaaStr, strong fit for enterprise tier"
])

# List follow-ups due this week
result = subprocess.run(
    ["python", "clawchief/scripts/bd_outreach.py", "followups", "--due", "this_week"],
    capture_output=True, text=True
)
print(result.stdout)
```

---

## Source-of-Truth Files

These files are the policy layer. Edit them to match your operating model.

### `clawchief/priority-map.md`

```markdown
# Priority Map

## P1 — Revenue and retention
- Customer escalations
- Closing deals in final stage
- Renewal risks

## P2 — Team unblocking
- Decisions that are blocking >1 person
- Hiring decisions in final round

## P3 — Strategic
- OKR reviews
- Partner development

## P4 — Operational
- Recurring admin
- Reporting
```

### `clawchief/auto-resolver.md`

```markdown
# Auto-Resolver Policy

When ambiguity arises, resolve with:

1. Does the priority-map clearly rank one option higher? → take the higher-ranked action.
2. Does the action affect a P1 stakeholder (customer, investor, key hire)? → escalate before acting.
3. Is the decision reversible within 24 hours? → act and log it.
4. Otherwise → surface to the user before proceeding.
```

### `clawchief/task-system-acceptance.md`

```markdown
# Task System Acceptance Criteria

- Every task has: content, project, priority (p1–p4), due date
- "Inbox" project is triaged daily
- Completed tasks are closed in Todoist, not moved to a markdown file
- No live task state lives in markdown
```

---

## Skills

Copy skills to `~/.openclaw/skills/` and activate them in OpenClaw.

### `task-system-contract` skill

Encodes the shared rules for how tasks are created, triaged, and closed. Always load this skill when doing any task management work.

### `daily-task-prep` skill

Runs the morning preparation sequence:
1. Pull today's tasks from Todoist
2. Check calendar for the day
3. Surface blockers and P1 items
4. Produce a prioritized briefing

### `executive-assistant` skill

Handles Gmail triage, calendar booking, and follow-up tracking using the helper scripts.

### `daily-task-manager` skill

Handles mid-day and end-of-day task hygiene: closing completed tasks, rescheduling, and logging.

### `business-development` skill

Uses `bd_outreach.py` and `sheet_helper.py` to manage pipeline, log touchpoints, and surface follow-ups.

---

## Cron Jobs

`cron/jobs.template.json` defines deterministic recurring triggers. Keep prompts short — workflow logic lives in skills, not in cron.

```json
[
  {
    "id": "morning-prep",
    "schedule": "0 7 * * 1-5",
    "prompt": "Run daily-task-prep skill.",
    "skill": "daily-task-prep"
  },
  {
    "id": "eod-close",
    "schedule": "0 17 * * 1-5",
    "prompt": "Run daily-task-manager EOD close.",
    "skill": "daily-task-manager"
  },
  {
    "id": "inbox-triage",
    "schedule": "0 9,13 * * 1-5",
    "prompt": "Triage Gmail inbox using executive-assistant skill.",
    "skill": "executive-assistant"
  },
  {
    "id": "bd-followups",
    "schedule": "0 8 * * 1",
    "prompt": "Surface BD follow-ups due this week.",
    "skill": "business-development"
  }
]
```

Install cron jobs (example with system cron):

```bash
# Convert jobs.template.json to actual crontab entries
python -c "
import json, sys
with open('cron/jobs.template.json') as f:
    jobs = json.load(f)
for job in jobs:
    print(f\"{job['schedule']} openclaw run --skill {job['skill']} --prompt '{job['prompt']}'\")
" >> /tmp/claw_cron.txt

crontab -l > /tmp/existing_cron.txt
cat /tmp/existing_cron.txt /tmp/claw_cron.txt | crontab -
```

---

## `workspace/HEARTBEAT.md`

Keep this file thin. It is a pointer to cron jobs, not a workflow definition.

```markdown
# Heartbeat

Recurring work is owned by cron jobs defined in `cron/jobs.template.json`.

This file records the last known healthy state:

- Last morning-prep: 2026-04-08 07:00
- Last inbox-triage: 2026-04-08 13:00
- Last eod-close: 2026-04-07 17:00
- Last bd-followup run: 2026-04-07 08:00
```

---

## `workspace/TOOLS.md`

Replace all placeholders with your real environment details.

```markdown
# Tools

## Task system
- Provider: Todoist
- API token env var: TODOIST_API_TOKEN
- Default project for inbox: Inbox
- Default project for work tasks: Work

## Google Workspace
- gws profile: personal
- Primary calendar: your.email@gmail.com
- Secondary calendars: team-calendar-id, personal-calendar-id

## Sheets
- BD pipeline sheet: (set BD_SHEET_ID env var)

## Email
- Primary inbox: your.email@gmail.com
- Signature footer: (your name / title)
```

---

## Migration: v2 → v3

If you used the older `clawchief/tasks.md` model:

```bash
# 1. Export existing tasks to Todoist
python clawchief/scripts/todoist_cli.py import-from-markdown \
  --file ~/.openclaw/workspace/clawchief/tasks.md \
  --project "Migrated"

# 2. Archive the old file
mv ~/.openclaw/workspace/clawchief/tasks.md \
   ~/.openclaw/workspace/clawchief/archive/tasks-v2-backup.md

# 3. Replace workspace/tasks/current.md with deprecation pointer
echo "# Deprecated\nLive tasks now in Todoist. See clawchief/scripts/todoist_cli.py." \
  > ~/.openclaw/workspace/tasks/current.md

# 4. Update skills — remove any skill referencing tasks.md directly
grep -r "tasks.md" ~/.openclaw/skills/  # find and update references

# 5. Install new skills
cp -r skills/* ~/.openclaw/skills/
```

Key behavioral changes in v3:
- `gog` commands → `gws`-backed helper scripts
- `clawchief/tasks.md` → Todoist via `todoist_cli.py`
- raw cron prompts with embedded logic → thin prompts + skill-held logic

---

## Running Tests

```bash
cd ~/.openclaw/workspace/clawchief
python -m pytest clawchief/tests/ -v

# Run a specific test
python -m pytest clawchief/tests/test_todoist_cli.py -v

# Check that all source-of-truth files are present and well-formed
python -m pytest clawchief/tests/test_pack_alignment.py -v
```

---

## Common Patterns

### Morning briefing (manual trigger)

```bash
openclaw run --skill daily-task-prep --prompt "Run morning prep."
```

### Add a high-priority task from the command line

```bash
python ~/.openclaw/workspace/clawchief/scripts/todoist_cli.py \
  add \
  --content "Call Marcus re: renewal risk" \
  --due today \
  --priority p1 \
  --project Work
```

### Triage inbox on demand

```bash
openclaw run --skill executive-assistant --prompt "Triage inbox."
```

### Compile durable lessons back into source-of-truth

```bash
openclaw run --skill executive-assistant \
  --prompt "Compile lessons from today's meeting notes into clawchief/knowledge-compiler.md."
```

---

## Troubleshooting

### `TODOIST_API_TOKEN` not found

```bash
# Verify it's set
echo $TODOIST_API_TOKEN

# Load from .env if needed
source ~/.openclaw/.env
```

### `gws` commands failing in helper scripts

```bash
# Verify gws is authenticated
gws auth status

# Re-authenticate if needed
gws auth login

# Check which profile the scripts expect
grep "gws" ~/.openclaw/workspace/TOOLS.md
```

### Tasks not appearing after `todoist_cli.py add`

- Confirm the project name matches exactly (case-sensitive)
- Check Todoist API token has write permissions
- Run with `--debug` flag: `python todoist_cli.py add --debug ...`

### Cron jobs not firing

```bash
# Check crontab is installed
crontab -l | grep openclaw

# Check openclaw is on PATH for cron
which openclaw

# Add full path to crontab if needed
# e.g.: 0 7 * * 1-5 /usr/local/bin/openclaw run --skill daily-task-prep ...
```

### Skills not loading

```bash
# Verify skills are in the right location
ls ~/.openclaw/skills/

# Expected skills:
# task-system-contract/
# executive-assistant/
# business-development/
# daily-task-manager/
# daily-task-prep/

# Each skill directory must contain SKILL.md
ls ~/.openclaw/skills/daily-task-prep/
```

### Meeting notes not being ingested

- Check `clawchief/meeting-notes.md` policy file exists and is populated
- Verify `workspace/memory/meeting-notes-state.json` is writable
- Trigger manually: `openclaw run --skill executive-assistant --prompt "Ingest latest meeting notes per meeting-notes.md policy."`

---

## Design Principles (for AI agents)

1. **Never write live task state to markdown.** All task mutations go through `todoist_cli.py`.
2. **Never call Gmail or Calendar APIs directly.** Use `ea_gmail.py` and `ea_calendar.py`.
3. **Always check all calendars before booking** — not just the primary calendar.
4. **Search Gmail at message level**, not thread level, to avoid missing replies.
5. **Keep cron prompts short.** Workflow logic belongs in skill files, not in prompt strings.
6. **Compile durable lessons into source-of-truth files**, not into chat history.
7. **Use `auto-resolver.md` policy before escalating ambiguous decisions** to the user.
8. **`workspace/HEARTBEAT.md` is a status file, not a workflow definition.**
```
