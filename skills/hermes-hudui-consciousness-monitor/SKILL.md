```markdown
---
name: hermes-hudui-consciousness-monitor
description: Web UI consciousness monitor for Hermes AI agent with persistent memory — FastAPI backend + React frontend dashboard
triggers:
  - set up hermes hud web ui
  - monitor hermes agent in browser
  - hermes hudui dashboard
  - show hermes agent consciousness monitor
  - hermes web ui not working
  - add token cost tracking hermes
  - hermes hud websocket updates
  - configure hermes hudui themes
---

# ☤ Hermes HUD Web UI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A browser-based consciousness monitor for the Hermes AI agent. Reads from `~/.hermes/` data files and serves a real-time React dashboard via a FastAPI backend with WebSocket support.

## Architecture Overview

```
React Frontend (Vite + SWR)
    ↓ /api/* + WebSocket /ws
FastAPI Backend (Python 3.11+)
    ↓ collectors/*.py + cache + file watcher
~/.hermes/ (agent data files)
```

- **Backend**: FastAPI with collectors, mtime-based cache, watchfiles watcher
- **Frontend**: React + Vite + SWR with silent background updates and auto-reconnect WebSocket
- **Data source**: `~/.hermes/` directory — no database, no external APIs

## Installation

### Quick Install

```bash
git clone https://github.com/joeynyc/hermes-hudui.git
cd hermes-hudui
python3.11 -m venv venv
source venv/bin/activate
./install.sh
hermes-hudui
```

Open http://localhost:3001

### Manual Install

```bash
python3.11 -m venv venv
source venv/bin/activate

# Install Python package
pip install -e .

# Build frontend and copy to backend static dir
cd frontend
npm install
npm run build
cp -r dist/* ../backend/static/

# Start server
hermes-hudui
```

### Subsequent Runs

```bash
source venv/bin/activate
hermes-hudui
```

### With TUI Support (Optional)

```bash
pip install hermes-hudui[tui]
```

## Requirements

- Python 3.11+
- Node.js 18+
- A running Hermes agent with data written to `~/.hermes/`

## Key CLI Commands

| Command | Description |
|---------|-------------|
| `hermes-hudui` | Start the web server on port 3001 |
| `./install.sh` | Full install: venv setup + pip install + frontend build |

## Backend: Collectors

Collectors are Python modules in `backend/collectors/` that read `~/.hermes/` and return dataclasses. Each collector corresponds to a dashboard panel.

### Collector Pattern

```python
# backend/collectors/identity.py
from dataclasses import dataclass
from pathlib import Path
import json

HERMES_DIR = Path.home() / ".hermes"

@dataclass
class IdentityData:
    designation: str
    substrate: str
    runtime: str
    days_conscious: int
    brain_size_mb: float

def collect_identity() -> IdentityData:
    config_path = HERMES_DIR / "config.json"
    if not config_path.exists():
        return IdentityData(
            designation="Unknown",
            substrate="unknown",
            runtime="unknown",
            days_conscious=0,
            brain_size_mb=0.0,
        )
    data = json.loads(config_path.read_text())
    return IdentityData(
        designation=data.get("designation", "Hermes"),
        substrate=data.get("substrate", "unknown"),
        runtime=data.get("runtime", "unknown"),
        days_conscious=data.get("days_conscious", 0),
        brain_size_mb=data.get("brain_size_mb", 0.0),
    )
```

### Registering a New Collector in FastAPI

```python
# backend/main.py (simplified pattern)
from fastapi import FastAPI
from backend.collectors.identity import collect_identity
from backend.cache import cached

app = FastAPI()

@app.get("/api/identity")
async def identity():
    return cached("identity", collect_identity)
```

## Backend: Caching

The cache uses mtime-based invalidation. TTLs by data type:

| Data Type | TTL |
|-----------|-----|
| Sessions | 30s |
| Skills | 60s |
| Patterns | 60s |
| Profiles | 45s |

### Using the Cache

```python
from backend.cache import cached

# Simple usage — key + callable
result = cached("sessions", collect_sessions)

# With TTL override
result = cached("identity", collect_identity, ttl=10)
```

### Cache Invalidation

The file watcher (`watchfiles`) monitors `~/.hermes/` and invalidates relevant cache keys when files change, triggering WebSocket broadcasts to all clients.

## Backend: WebSocket

The WebSocket endpoint at `/ws` broadcasts `data_changed` events when `~/.hermes/` files change.

### Server-Side Broadcast Pattern

```python
# backend/ws.py (simplified)
from fastapi import WebSocket
import asyncio
import json

connected_clients: list[WebSocket] = []

async def broadcast_change(event: str = "data_changed"):
    message = json.dumps({"type": event})
    dead = []
    for ws in connected_clients:
        try:
            await ws.send_text(message)
        except Exception:
            dead.append(ws)
    for ws in dead:
        connected_clients.remove(ws)

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    connected_clients.append(websocket)
    try:
        while True:
            await websocket.receive_text()  # keep alive
    except Exception:
        connected_clients.remove(websocket)
```

## Frontend: SWR Data Fetching

Each panel fetches its own API endpoint with `keepPreviousData` to avoid loading flashes.

```typescript
// frontend/src/hooks/useIdentity.ts
import useSWR from "swr";

const fetcher = (url: string) => fetch(url).then((r) => r.json());

export function useIdentity() {
  return useSWR("/api/identity", fetcher, {
    keepPreviousData: true,
    revalidateOnFocus: false,
  });
}
```

## Frontend: WebSocket Hook

Auto-reconnects with exponential backoff and triggers SWR revalidation on `data_changed` events.

```typescript
// frontend/src/hooks/useHermesSocket.ts
import { useEffect, useRef } from "react";
import { mutate } from "swr";

export function useHermesSocket() {
  const wsRef = useRef<WebSocket | null>(null);
  const retryDelay = useRef(1000);

  function connect() {
    const ws = new WebSocket(`ws://${location.host}/ws`);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      if (msg.type === "data_changed") {
        // Revalidate all SWR keys
        mutate(() => true, undefined, { revalidate: true });
      }
    };

    ws.onclose = () => {
      setTimeout(() => {
        retryDelay.current = Math.min(retryDelay.current * 2, 30000);
        connect();
      }, retryDelay.current);
    };

    ws.onopen = () => {
      retryDelay.current = 1000; // reset on success
    };
  }

  useEffect(() => {
    connect();
    return () => wsRef.current?.close();
  }, []);
}
```

## Frontend: Panel Component Pattern

```typescript
// frontend/src/panels/IdentityPanel.tsx
import { useIdentity } from "../hooks/useIdentity";

export function IdentityPanel() {
  const { data, isLoading } = useIdentity();

  // Show stale data while refreshing — no loading flash
  if (!data) return <div className="panel-loading">Loading…</div>;

  return (
    <div className="panel">
      <h2>Identity</h2>
      <dl>
        <dt>Designation</dt>
        <dd>{data.designation}</dd>
        <dt>Days Conscious</dt>
        <dd>{data.days_conscious}</dd>
        <dt>Brain Size</dt>
        <dd>{data.brain_size_mb.toFixed(1)} MB</dd>
      </dl>
    </div>
  );
}
```

## Token Cost Pricing

Costs are calculated from token counts using hardcoded per-model pricing in the backend.

### Supported Models and Pricing

| Provider | Model | Input | Output | Cache Read |
|----------|-------|------:|-------:|-----------:|
| Anthropic | Claude Opus 4 | $15/M | $75/M | $1.50/M |
| Anthropic | Claude Sonnet 4 | $3/M | $15/M | $0.30/M |
| Anthropic | Claude Haiku 3.5 | $0.80/M | $4/M | $0.08/M |
| OpenAI | GPT-4o | $2.50/M | $10/M | $1.25/M |
| OpenAI | o1 | $15/M | $60/M | $7.50/M |
| DeepSeek | V3 | $0.27/M | $1.10/M | $0.07/M |
| xAI | Grok 3 | $3/M | $15/M | $0.75/M |
| Google | Gemini 2.5 Pro | $1.25/M | $10/M | $0.31/M |

Unknown models fall back to Claude Opus pricing. Local/free models are detected and priced at $0.

### Cost Calculation Example

```python
# backend/collectors/costs.py (pattern)
PRICING = {
    "claude-opus-4": {"input": 15.0, "output": 75.0, "cache_read": 1.50},
    "claude-sonnet-4": {"input": 3.0, "output": 15.0, "cache_read": 0.30},
    "gpt-4o": {"input": 2.50, "output": 10.0, "cache_read": 1.25},
    # ... etc
}

def calculate_cost(model: str, input_tokens: int, output_tokens: int, cache_tokens: int = 0) -> float:
    # Detect local/free models
    if any(tag in model.lower() for tag in ["local", "ollama", "lmstudio"]):
        return 0.0

    pricing = PRICING.get(model, PRICING["claude-opus-4"])  # fallback
    cost = (
        (input_tokens / 1_000_000) * pricing["input"]
        + (output_tokens / 1_000_000) * pricing["output"]
        + (cache_tokens / 1_000_000) * pricing["cache_read"]
    )
    return round(cost, 6)
```

## Themes

Switch themes with the `t` key or the theme picker UI.

| Theme | Key | Description |
|-------|-----|-------------|
| Neural Awakening | `ai` | Cyan/blue on deep navy |
| Blade Runner | `blade-runner` | Amber/orange on warm black |
| fsociety | `fsociety` | Green on pure black |
| Anime | `anime` | Purple/violet on indigo |

CRT scanline overlay is togglable via the theme picker.

### Applying a Theme Programmatically

```typescript
// frontend/src/theme.ts
type Theme = "ai" | "blade-runner" | "fsociety" | "anime";

export function setTheme(theme: Theme) {
  document.documentElement.setAttribute("data-theme", theme);
  localStorage.setItem("hermes-theme", theme);
}

export function getTheme(): Theme {
  return (localStorage.getItem("hermes-theme") as Theme) ?? "ai";
}
```

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `1`–`9`, `0` | Switch dashboard tabs |
| `t` | Toggle theme picker |
| `Ctrl+K` | Open command palette |

## Dashboard Panels / Tabs

| Tab | Data Source | Description |
|-----|-------------|-------------|
| Identity | `~/.hermes/config.json` | Designation, substrate, uptime |
| What I Know | Session logs | Conversations, messages, actions, skills |
| What I Remember | Memory files | Capacity bars, user profile, corrections |
| What I See | API key files | Key presence, service health |
| What I'm Learning | Skills directory | Recently modified skills with categories |
| What I'm Working On | Project files | Active projects, dirty file status |
| While You Sleep | Cron config | Scheduled jobs |
| How I Think | Tool logs | Tool usage patterns with gradient bars |
| My Rhythm | Activity logs | Daily activity sparkline |
| Growth Delta | Snapshots | Diffs showing what changed |
| Token Costs | Token logs | Per-model USD cost with daily trend |

## Adding a New API Endpoint

```python
# 1. Create collector
# backend/collectors/my_feature.py
from dataclasses import dataclass
from pathlib import Path

HERMES_DIR = Path.home() / ".hermes"

@dataclass
class MyFeatureData:
    value: str

def collect_my_feature() -> MyFeatureData:
    path = HERMES_DIR / "my_feature.json"
    if not path.exists():
        return MyFeatureData(value="none")
    import json
    return MyFeatureData(**json.loads(path.read_text()))

# 2. Register route in backend/main.py
from backend.collectors.my_feature import collect_my_feature
from backend.cache import cached

@app.get("/api/my-feature")
async def my_feature():
    return cached("my_feature", collect_my_feature, ttl=30)

# 3. Create frontend hook
# frontend/src/hooks/useMyFeature.ts
import useSWR from "swr";
const fetcher = (url: string) => fetch(url).then(r => r.json());
export const useMyFeature = () =>
  useSWR("/api/my-feature", fetcher, { keepPreviousData: true });
```

## Common Patterns

### Check if `~/.hermes/` Exists

```python
from pathlib import Path

HERMES_DIR = Path.home() / ".hermes"

if not HERMES_DIR.exists():
    raise RuntimeError(
        "No ~/.hermes/ directory found. "
        "Make sure the Hermes agent has run at least once."
    )
```

### Graceful Missing File Handling

```python
def safe_read_json(path: Path, default=None):
    try:
        return json.loads(path.read_text()) if path.exists() else default
    except (json.JSONDecodeError, OSError):
        return default
```

### Frontend: Rebuild After Backend Changes

```bash
cd frontend
npm run build
cp -r dist/* ../backend/static/
```

During development, run the Vite dev server separately (proxy configured in `vite.config.ts` to forward `/api` and `/ws` to the FastAPI backend on port 3001).

```bash
# Terminal 1 — backend
source venv/bin/activate
hermes-hudui

# Terminal 2 — frontend dev server with HMR
cd frontend
npm run dev
# Opens http://localhost:5173 with proxy to :3001
```

## Troubleshooting

### Port Already in Use

```bash
# Find and kill the process using port 3001
lsof -ti:3001 | xargs kill -9
hermes-hudui
```

### Frontend Not Loading (blank page)

The frontend static files may not be built or copied:

```bash
cd frontend
npm install
npm run build
cp -r dist/* ../backend/static/
hermes-hudui
```

### WebSocket Not Connecting ("● live" badge missing)

1. Check the browser console for WebSocket errors
2. Ensure the backend is running (`hermes-hudui`)
3. Confirm no firewall is blocking ws://localhost:3001/ws
4. The hook auto-reconnects with exponential backoff — wait a moment

### No Data Showing / Empty Dashboard

```bash
# Check that ~/.hermes/ exists and has files
ls ~/.hermes/

# Check backend logs for collector errors
hermes-hudui  # errors printed to stdout
```

### `watchfiles` Not Detecting Changes

```bash
pip install --upgrade watchfiles
```

On Linux, you may need to increase inotify limits:

```bash
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### WSL: Browser Can't Reach Server

The install script detects WSL automatically. If the browser can't connect:

```bash
# Get WSL IP
hostname -I | awk '{print $1}'
# Open http://<wsl-ip>:3001 instead of localhost
```

### Python Version Mismatch

```bash
python3.11 --version  # must be 3.11+
python3.11 -m venv venv  # explicitly use 3.11
```

## Platform Support

| Platform | Status |
|----------|--------|
| macOS | Native via `./install.sh` |
| Linux | Native via `./install.sh` |
| Windows | Via WSL only |
| WSL | Auto-detected by install script |

## Relationship to hermes-hud (TUI)

Both read from the same `~/.hermes/` directory independently. The Web UI (`hermes-hudui`) is fully standalone — it ships its own collectors. It adds features the TUI lacks:

- Dedicated Memory, Skills, and Sessions tabs
- Per-model token cost tracking with daily trend
- Command palette (`Ctrl+K`)
- Live theme switcher with preview
- WebSocket real-time updates

Install both side by side:

```bash
pip install hermes-hud       # TUI version
pip install hermes-hudui[tui] # Web UI + TUI bridge
```
```
