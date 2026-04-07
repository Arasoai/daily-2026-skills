```markdown
---
name: port-whisperer-cli
description: CLI tool for visualizing and managing processes running on network ports with framework detection and Docker support
triggers:
  - "what's running on my ports"
  - "install port-whisperer"
  - "check which process is using port"
  - "kill process on port"
  - "show dev server ports"
  - "port whisperer commands"
  - "find what's listening on a port"
  - "manage ports from terminal"
---

# Port Whisperer CLI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`port-whisperer` is a beautiful CLI tool that shows every process listening on your network ports — with color-coded status, framework detection (Next.js, Vite, Django, etc.), Docker container identification, and interactive process management. Runs in ~0.2s via three shell calls.

## Install

```bash
# Global install (recommended — gives you `ports` and `whoisonport` commands)
npm install -g port-whisperer

# Run without installing
npx port-whisperer
```

Requires macOS. Linux support is planned; Windows is not supported.

## Core Commands

### List active dev ports

```bash
ports
```

Shows dev servers, Docker containers, and databases. Filters out system apps (Spotify, Raycast, etc.) by default.

### Show all listening ports (including system)

```bash
ports --all
```

### Inspect a specific port

```bash
ports 3000
# or
whoisonport 3000
```

Outputs full process tree, repository path, git branch, memory usage, and an interactive kill prompt.

### Kill processes

```bash
ports kill 3000               # kill by port number
ports kill 3000 5173 8080     # kill multiple ports at once
ports kill 42872              # kill by PID
ports kill -f 3000            # force kill (SIGKILL) for stubborn processes
```

### Show all dev processes (not just port-bound)

```bash
ports ps           # dev processes only
ports ps --all     # every running process
```

Displays PID, CPU%, memory, project name, detected framework, uptime, and a description column. Docker processes are collapsed into a single summary row.

### Clean up orphaned/zombie processes

```bash
ports clean
```

Finds and kills orphaned or zombie dev server processes. Only targets dev runtimes (node, python, ruby, etc.) — safe for desktop apps.

### Watch for real-time port changes

```bash
ports watch
```

Notifies whenever a port starts or stops listening.

## What the Output Looks Like

```
┌───────┬─────────┬───────┬──────────────────────┬────────────┬────────┬───────────┐
│ PORT  │ PROCESS │ PID   │ PROJECT              │ FRAMEWORK  │ UPTIME │ STATUS    │
├───────┼─────────┼───────┼──────────────────────┼────────────┼────────┼───────────┤
│ :3000 │ node    │ 42872 │ frontend             │ Next.js    │ 1d 9h  │ ● healthy │
│ :5432 │ docker  │ 58351 │ backend-postgres-1   │ PostgreSQL │ 10d 3h │ ● healthy │
│ :6379 │ docker  │ 58351 │ backend-redis-1      │ Redis      │ 10d 3h │ ● healthy │
└───────┴─────────┴───────┴──────────────────────┴────────────┴────────┴───────────┘
```

**Status colors:**
- 🟢 Green = healthy
- 🟡 Yellow = orphaned (parent process gone)
- 🔴 Red = zombie

## Framework Detection

Port Whisperer auto-detects frameworks by reading `package.json` dependencies and inspecting process command lines:

| Category | Detected Frameworks |
|----------|-------------------|
| JavaScript | Next.js, Vite, Express, Angular, Remix, Astro |
| Python | Django, FastAPI |
| Ruby | Rails |
| Docker images | PostgreSQL, Redis, MongoDB, LocalStack, nginx |

## How It Works Internally

Three shell calls, batched for performance:

```
1. lsof -iTCP -sTCP:LISTEN       → all TCP listeners
2. ps (batched)                   → command, uptime, memory, PPID, status per PID
3. lsof -d cwd (batched)         → working directory → project/framework detection
4. docker ps                      → maps host ports to container names/images
```

## Common Workflows

### Free up port 3000 before starting a dev server

```bash
# Check what's there
ports 3000

# Kill it
ports kill 3000

# Or force kill if it won't die
ports kill -f 3000
```

### Kill everything before a clean restart

```bash
ports kill 3000 5173 8080 4000
```

### Check Docker service ports

```bash
ports --all
# or just:
ports
# Docker containers appear automatically with container name and image detected
```

### Find and clean up leftover processes after a crash

```bash
ports clean
```

### Monitor ports during development

```bash
ports watch
# Streams notifications as ports open/close
```

### Scripting: check if a port is in use

```bash
# Exit code 0 if port is active, non-zero if free
ports 3000 && echo "port in use" || echo "port free"
```

## Troubleshooting

**`ports` command not found after install**
```bash
# Check npm global bin is in PATH
npm bin -g
# Add to ~/.zshrc or ~/.bashrc:
export PATH="$(npm bin -g):$PATH"
```

**Permission denied when killing a process**
```bash
# Use force kill flag
ports kill -f 3000
# Or use sudo if process is owned by another user
sudo ports kill 3000
```

**Docker containers not showing up**
- Ensure Docker Desktop is running
- Verify `docker ps` works in your terminal (Docker CLI must be accessible)

**Port shows as orphaned (yellow)**
- The parent process has exited but the child is still holding the port
- Run `ports clean` to sweep these up, or `ports kill <port>` to target directly

**Process won't die with graceful kill**
```bash
ports kill -f 3000   # sends SIGKILL instead of SIGTERM
```

**Only macOS is supported**
- Linux support is planned but not yet available
- Windows is not planned — use WSL as a workaround
```
