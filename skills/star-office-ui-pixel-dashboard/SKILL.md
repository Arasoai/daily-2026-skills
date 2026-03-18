```markdown
---
name: star-office-ui-pixel-dashboard
description: Deploy and integrate Star Office UI — a pixel-art AI agent status dashboard with multi-agent collaboration, Flask backend, and OpenClaw integration.
triggers:
  - set up star office ui
  - deploy pixel office dashboard
  - integrate star office with my agent
  - add agent status visualization
  - show ai agent working state
  - configure multi-agent office board
  - connect openclaw to star office
  - install star office ui skill
---

# Star Office UI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A pixel-art AI office dashboard that visualizes AI agent work states in real time. Characters walk between desk, sofa, and bug zones based on `idle / writing / researching / executing / syncing / error` states. Supports multi-agent collaboration, CN/EN/JP i18n, AI-generated backgrounds (Gemini), and an optional Electron desktop-pet mode.

---

## Requirements

- Python 3.10+ (uses `X | Y` union type syntax — **3.9 and below will fail**)
- Node.js (only needed for the optional Electron desktop-pet)
- Git

---

## Installation

```bash
git clone https://github.com/ringhyacinth/Star-Office-UI.git
cd Star-Office-UI

python3 -m pip install -r backend/requirements.txt

cp state.sample.json state.json
```

Start the backend:

```bash
cd backend
python3 app.py
```

Open **http://127.0.0.1:19000** — the office loads immediately with default state.

---

## Environment Configuration

Copy the example env file and set secrets before going to production:

```bash
cp .env.example .env
```

Key variables in `.env`:

```bash
# Required in production — use a long random string
FLASK_SECRET_KEY=your-random-secret-here

# Password protecting the asset drawer sidebar
ASSET_DRAWER_PASS=your-drawer-password

# Optional: Gemini API key for AI background generation
GEMINI_API_KEY=$GEMINI_API_KEY
```

> The backend reads `.env` automatically via `python-dotenv`. Never commit real secrets — the server blocks weak passwords in production mode.

---

## Switching Agent States

Use the bundled CLI script from the project root:

```bash
python3 set_state.py <state> "<description>"
```

Available states and their office zones:

| State | Zone | When to use |
|---|---|---|
| `idle` | 🛋 Sofa (rest area) | Standby / task complete |
| `writing` | 💻 Desk (work area) | Writing code or docs |
| `researching` | 💻 Desk | Search / research |
| `executing` | 💻 Desk | Running commands / jobs |
| `syncing` | 💻 Desk | Pushing data / syncing |
| `error` | 🐛 Bug zone | Errors / debugging |

```bash
python3 set_state.py idle "待命中"
python3 set_state.py writing "Drafting README"
python3 set_state.py researching "Looking up API docs"
python3 set_state.py executing "Running test suite"
python3 set_state.py syncing "Pushing to remote"
python3 set_state.py error "Investigating crash"
```

---

## REST API Reference

All endpoints are on the Flask server (default `http://127.0.0.1:19000`).

### Health check

```bash
curl http://127.0.0.1:19000/health
```

### Get current state

```bash
curl http://127.0.0.1:19000/status
```

Response:
```json
{
  "state": "writing",
  "message": "Drafting README",
  "updated_at": "2026-03-10T14:22:00Z"
}
```

### Set state via POST

```bash
curl -X POST http://127.0.0.1:19000/set_state \
  -H "Content-Type: application/json" \
  -d '{"state": "executing", "message": "Running deploy script"}'
```

### Get all agents (multi-agent mode)

```bash
curl http://127.0.0.1:19000/agents
```

### Guest agent join

```bash
curl -X POST http://127.0.0.1:19000/join-agent \
  -H "Content-Type: application/json" \
  -d '{
    "join_key": "ocj_example_team_01",
    "agent_name": "Guest Bot",
    "state": "idle",
    "message": "Just joined"
  }'
```

Response includes a `agent_id` token used for subsequent pushes.

### Guest agent push state

```bash
curl -X POST http://127.0.0.1:19000/agent-push \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "<id-from-join>",
    "state": "writing",
    "message": "Working on feature branch"
  }'
```

### Guest agent leave

```bash
curl -X POST http://127.0.0.1:19000/leave-agent \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "<id-from-join>"}'
```

### Yesterday memo

```bash
curl http://127.0.0.1:19000/yesterday-memo
```

Reads the most recent `memory/*.md` file and returns a sanitized summary card.

### Gemini config (AI background generation)

```bash
# Get current config
curl http://127.0.0.1:19000/config/gemini

# Set API key
curl -X POST http://127.0.0.1:19000/config/gemini \
  -H "Content-Type: application/json" \
  -d '{"api_key": "$GEMINI_API_KEY"}'
```

---

## Python Integration Examples

### Minimal state-push helper

```python
import httplib2
import json

OFFICE_URL = "http://127.0.0.1:19000"

def set_office_state(state: str, message: str) -> dict:
    """Push a state update to the Star Office backend."""
    h = httplib2.Http()
    body = json.dumps({"state": state, "message": message}).encode()
    response, content = h.request(
        f"{OFFICE_URL}/set_state",
        method="POST",
        body=body,
        headers={"Content-Type": "application/json"},
    )
    return json.loads(content)

# Usage
set_office_state("writing", "Updating changelog")
# ... do work ...
set_office_state("idle", "Done")
```

### Using requests (simpler)

```python
import requests
import os

OFFICE_URL = os.getenv("STAR_OFFICE_URL", "http://127.0.0.1:19000")

def push_state(state: str, message: str = "") -> None:
    try:
        requests.post(
            f"{OFFICE_URL}/set_state",
            json={"state": state, "message": message},
            timeout=5,
        )
    except requests.exceptions.RequestException:
        pass  # Non-blocking — office updates are best-effort

# Decorator pattern for auto state management
import functools

def office_task(state: str = "executing", done_message: str = "完成"):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            push_state(state, f"Running {fn.__name__}")
            try:
                result = fn(*args, **kwargs)
                push_state("idle", done_message)
                return result
            except Exception as e:
                push_state("error", str(e))
                raise
        return wrapper
    return decorator

@office_task(state="researching", done_message="Research complete")
def fetch_docs(topic: str) -> str:
    # your code here
    return f"Docs for {topic}"
```

### Guest agent push script (adapted from `office-agent-push.py`)

```python
import requests
import time
import os

JOIN_KEY   = os.getenv("OFFICE_JOIN_KEY", "ocj_example_team_01")
AGENT_NAME = os.getenv("OFFICE_AGENT_NAME", "My Guest Agent")
OFFICE_URL = os.getenv("OFFICE_URL", "http://127.0.0.1:19000")

def main():
    # Join the office
    r = requests.post(f"{OFFICE_URL}/join-agent", json={
        "join_key": JOIN_KEY,
        "agent_name": AGENT_NAME,
        "state": "idle",
        "message": "Just arrived",
    }, timeout=10)
    r.raise_for_status()
    agent_id = r.json()["agent_id"]
    print(f"Joined as {agent_id}")

    try:
        while True:
            # Push current state every 15 seconds
            requests.post(f"{OFFICE_URL}/agent-push", json={
                "agent_id": agent_id,
                "state": "idle",
                "message": "Standby",
            }, timeout=5)
            time.sleep(15)
    finally:
        requests.post(f"{OFFICE_URL}/leave-agent",
                      json={"agent_id": agent_id}, timeout=5)
        print("Left office")

if __name__ == "__main__":
    main()
```

---

## OpenClaw Integration

### SOUL.md rules block

Add this to your agent's `SOUL.md` so it automatically maintains office state:

```markdown
## Star Office 状态同步规则
- 接到任务时：先执行 `python3 set_state.py <状态> "<描述>"` 再开始工作
- 完成任务后：执行 `python3 set_state.py idle "待命中"` 再回复
- 遇到报错：执行 `python3 set_state.py error "<错误简述>"`
```

### Let OpenClaw auto-deploy

Send this prompt to your OpenClaw agent (lobster):

```
请按照这个 SKILL.md 帮我完成 Star Office UI 的部署：
https://github.com/ringhyacinth/Star-Office-UI/blob/master/SKILL.md
```

The agent will clone, install deps, start the backend, configure state sync, and return the access URL.

---

## Multi-Agent Join Keys

`join-keys.json` is auto-generated from `join-keys.sample.json` on first backend start.

Edit `join-keys.json` to add keys:

```json
{
  "ocj_yourteam_01": { "max_agents": 3, "label": "Team Alpha" },
  "ocj_yourteam_02": { "max_agents": 5, "label": "Team Beta" }
}
```

Share a key with a guest agent. They configure `office-agent-push.py`:

```python
JOIN_KEY   = "ocj_yourteam_01"
AGENT_NAME = "Colleague Bot"
OFFICE_URL = "https://your-office.example.com"
```

Guests appear on the board and walk to the correct zone based on their state.

---

## Public Access via Cloudflare Tunnel

```bash
# One-liner — no account needed for temporary URLs
cloudflared tunnel --url http://127.0.0.1:19000
```

You get `https://random-name.trycloudflare.com` — shareable immediately.

For a permanent URL, set up a named tunnel with your Cloudflare account.

---

## Smoke Test (verify deployment)

```bash
python3 scripts/smoke_test.py --base-url http://127.0.0.1:19000
```

All lines should show `OK`. Useful after config changes or upgrades.

---

## Desktop Pet (optional Electron)

```bash
cd desktop-pet
npm install
npm run dev
```

- Automatically starts the Python backend
- Renders the office as a transparent overlay window
- Default URL: `http://127.0.0.1:19000/?desktop=1`
- Customize paths via env vars — see `desktop-pet/README.md`

> macOS primary; experimental on other platforms.

---

## Memory / Yesterday Memo

Place daily work log files in `memory/` as Markdown:

```
memory/
  2026-03-09.md
  2026-03-10.md   ← most recent is shown as "Yesterday's memo"
```

The backend sanitizes content before serving it to the frontend card.

---

## Project Structure

```
Star-Office-UI/
├── backend/
│   ├── app.py               # Flask app entry point
│   ├── requirements.txt
│   └── run.sh
├── frontend/
│   ├── index.html           # Main office view
│   ├── join.html            # Guest join page
│   ├── invite.html          # Invite page
│   └── layout.js            # Phaser scene + state rendering
├── desktop-pet/             # Electron wrapper (optional)
├── docs/screenshots/
├── memory/                  # Daily memo markdown files
├── office-agent-push.py     # Guest push script
├── set_state.py             # State CLI
├── state.sample.json        # Copy to state.json on first run
├── join-keys.sample.json    # Copy/edit for multi-agent keys
├── SKILL.md                 # OpenClaw skill file
└── .env.example             # Copy to .env for production config
```

---

## Troubleshooting

**`SyntaxError: invalid syntax` on startup**
→ You are using Python 3.9 or older. Upgrade to Python 3.10+.
```bash
python3 --version   # must be 3.10+
```

**Port already in use (19000)**
→ Kill the existing process or change the port:
```bash
# Find and kill
lsof -ti:19000 | xargs kill -9
# Or set a different port in backend/app.py or via env
```

**State not updating in browser**
→ The frontend polls `/status` via SSE/polling. Hard-refresh (`Cmd+Shift+R`) clears stale cache. Check CDN headers if behind a proxy.

**Weak password blocked in production**
→ Set a strong `ASSET_DRAWER_PASS` in `.env`. The server rejects common/short passwords when `FLASK_ENV=production`.

**Guest agent shows as offline immediately**
→ Stale agents auto-revert to `idle` after a timeout. Ensure `office-agent-push.py` is running and pushing at least every 30 seconds.

**`join-keys.json` not found error**
→ The file is auto-generated on first start. If it errors, manually copy:
```bash
cp join-keys.sample.json join-keys.json
```

**AI background generation hangs**
→ Poll the progress endpoint:
```bash
curl http://127.0.0.1:19000/assets/generate-rpg-background/poll
```
Generation is async — the frontend polls this until done. Ensure `GEMINI_API_KEY` is set correctly.

---

## License

- **Code / logic**: MIT
- **Art assets**: Non-commercial only (learning / demo / community use)

For commercial use, replace all art assets with your own original work.
```
