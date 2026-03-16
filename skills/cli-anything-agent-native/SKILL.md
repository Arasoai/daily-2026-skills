```markdown
---
name: cli-anything-agent-native
description: Make any software agent-controllable by generating a complete CLI harness with JSON output, REPL, tests, and docs in one command
triggers:
  - make this software agent-native
  - generate a CLI harness for this app
  - use cli-anything to wrap this software
  - build an agent-ready CLI for this project
  - cli-anything build harness
  - make this tool work with AI agents
  - wrap this application with a CLI using cli-anything
  - use cli-anything to expose this software to agents
---

# CLI-Anything: Making ALL Software Agent-Native

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

CLI-Anything generates a full agent-ready CLI harness for any software in one command. It analyzes source code or repos, designs a Click-based CLI with structured JSON output, implements REPL interaction, writes comprehensive tests, and publishes to PATH — bridging AI agents and the world's software through the universal CLI interface.

---

## What CLI-Anything Does

Given any software (local path or GitHub repo), CLI-Anything runs a **7-phase pipeline**:

1. 🔍 **Analyze** — Scans source, maps GUI actions to APIs
2. 📐 **Design** — Architects command groups, state model, output formats
3. 🔨 **Implement** — Builds Click CLI with REPL, JSON output, undo/redo
4. 📋 **Plan Tests** — Creates `TEST.md` with unit + E2E test plans
5. 🧪 **Write Tests** — Implements comprehensive test suite
6. 📝 **Document** — Updates `TEST.md` with results
7. 📦 **Publish** — Creates `setup.py`, installs to PATH

The output lives in `<software>/agent-harness/` and is immediately usable by any AI agent.

---

## Installation

### Prerequisites

- Python 3.10+
- Target software installed (GIMP, Blender, LibreOffice, or any app)
- A supported AI coding agent (Claude Code, OpenCode, OpenClaw, Codex, Qodercli)

### Claude Code (Recommended)

```bash
# Add the CLI-Anything marketplace
/plugin marketplace add HKUDS/CLI-Anything

# Install the plugin
/plugin install cli-anything
```

> **Windows users:** Claude Code uses `bash`. Install Git for Windows (includes `bash` + `cygpath`) or use WSL to avoid `cygpath: command not found` errors.

### Manual Installation (Claude Code)

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
cp -r CLI-Anything/cli-anything-plugin ~/.claude/plugins/cli-anything
/reload-plugins
```

### OpenCode

```bash
git clone https://github.com/HKUDS/CLI-Anything.git

# Global install
cp CLI-Anything/opencode-commands/*.md ~/.config/opencode/commands/
cp CLI-Anything/cli-anything-plugin/HARNESS.md ~/.config/opencode/commands/

# Or project-level
cp CLI-Anything/opencode-commands/*.md .opencode/commands/
cp CLI-Anything/cli-anything-plugin/HARNESS.md .opencode/commands/
```

### OpenClaw

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
mkdir -p ~/.openclaw/skills/cli-anything
cp CLI-Anything/openclaw-skill/SKILL.md ~/.openclaw/skills/cli-anything/SKILL.md
```

Then invoke: `@cli-anything build a CLI for ./gimp`

### Codex

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
bash CLI-Anything/codex-skill/scripts/install.sh
# Windows: .\CLI-Anything\codex-skill\scripts\install.ps1
```

### Qodercli

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
bash CLI-Anything/qoder-plugin/setup-qodercli.sh
```

---

## Core Commands

### Building a Harness

```bash
# Claude Code — generate full CLI for a local app
/cli-anything:cli-anything ./gimp

# Claude Code — generate from a GitHub repo
/cli-anything:cli-anything https://github.com/blender/blender

# OpenCode
/cli-anything ./gimp
/cli-anything https://github.com/blender/blender

# Codex (natural language)
# "Use CLI-Anything to build a harness for ./gimp"
```

> **Claude Code < 2.x:** Use `/cli-anything` instead of `/cli-anything:cli-anything`.

### Refining an Existing Harness

```bash
# Broad refinement — agent analyzes all gaps
/cli-anything:refine ./gimp

# Focused refinement — target a specific area
/cli-anything:refine ./gimp "image batch processing and filters"

# OpenCode equivalents
/cli-anything-refine ./gimp
/cli-anything-refine ./gimp "batch processing and filters"

# Codex
# "Use CLI-Anything to refine ./shotcut for picture-in-picture workflows"
```

Refine is **incremental and non-destructive** — run it multiple times to expand coverage.

### Validation & Testing

```bash
# OpenCode: validate the generated harness
/cli-anything-validate ./gimp

# OpenCode: run tests
/cli-anything-test ./gimp

# Codex
# "Use CLI-Anything to validate ./libreoffice"
```

### Listing Available Harnesses

```bash
# OpenCode
/cli-anything-list
```

---

## Using a Generated CLI

After the pipeline completes, the harness is in `<software>/agent-harness/`.

### Install to PATH

```bash
cd gimp/agent-harness
pip install -e .
```

### Basic Usage

```bash
# Get help (self-documenting for agents)
cli-anything-gimp --help
cli-anything-gimp layer --help

# Create a new project
cli-anything-gimp project new --width 1920 --height 1080 -o poster.json

# Add a layer (JSON output mode)
cli-anything-gimp --json layer add -n "Background" --type solid --color "#1a1a2e"

# Open a file
cli-anything-gimp file open ./image.png

# Export
cli-anything-gimp file export --format png --output ./out.png
```

### JSON Output Mode

Every command supports `--json` for structured agent consumption:

```bash
cli-anything-gimp --json project new --width 800 --height 600
```

```json
{
  "status": "ok",
  "command": "project new",
  "result": {
    "width": 800,
    "height": 600,
    "layers": [],
    "id": "proj_abc123"
  }
}
```

On error:

```json
{
  "status": "error",
  "command": "layer add",
  "error": "No active project. Run `project new` or `file open` first.",
  "code": "NO_ACTIVE_PROJECT"
}
```

### Interactive REPL

```bash
# Enter REPL (stateful session)
cli-anything-gimp

# Inside REPL:
# > project new --width 1920 --height 1080
# > layer add -n "Sky" --type gradient
# > layer add -n "Text" --type text --content "Hello"
# > undo
# > redo
# > file export --format jpeg --quality 90
# > exit
```

### Undo / Redo

```bash
cli-anything-gimp undo
cli-anything-gimp redo
cli-anything-gimp history --last 10
```

---

## Generated Harness Structure

```
gimp/
└── agent-harness/
    ├── setup.py                  # pip-installable package
    ├── TEST.md                   # Test plan + results
    ├── cli_anything_gimp/
    │   ├── __init__.py
    │   ├── main.py               # Click CLI entrypoint
    │   ├── commands/
    │   │   ├── project.py        # project new/open/save/close
    │   │   ├── layer.py          # layer add/remove/reorder/merge
    │   │   ├── filter.py         # filter apply/list/preview
    │   │   ├── file.py           # file open/export/import
    │   │   └── ...
    │   ├── state.py              # Application state model
    │   ├── output.py             # JSON + human output formatting
    │   └── repl.py               # Interactive REPL
    └── tests/
        ├── unit/
        │   ├── test_project.py
        │   ├── test_layer.py
        │   └── ...
        └── e2e/
            ├── test_workflow_poster.py
            └── ...
```

---

## Writing a Custom Harness (Python Pattern)

If you need to manually create or extend a harness, here's the standard pattern CLI-Anything follows:

### Entry Point (`main.py`)

```python
import click
import json
import sys
from .state import AppState
from .output import output_result, output_error
from .commands import project, layer, file_ops, filters

@click.group()
@click.option("--json", "json_mode", is_flag=True, default=False,
              help="Output results as JSON for agent consumption")
@click.pass_context
def cli(ctx, json_mode):
    """CLI-Anything harness for GIMP — agent-ready interface."""
    ctx.ensure_object(dict)
    ctx.obj["json_mode"] = json_mode
    ctx.obj["state"] = AppState()

cli.add_command(project.group)
cli.add_command(layer.group)
cli.add_command(file_ops.group)
cli.add_command(filters.group)

if __name__ == "__main__":
    cli()
```

### Command Group (`commands/layer.py`)

```python
import click
from ..output import output_result, output_error

@click.group(name="layer")
def group():
    """Manage image layers."""
    pass

@group.command("add")
@click.option("-n", "--name", required=True, help="Layer name")
@click.option("--type", "layer_type",
              type=click.Choice(["solid", "gradient", "text", "image"]),
              default="solid")
@click.option("--color", default="#ffffff", help="Fill color (hex)")
@click.pass_context
def add(ctx, name, layer_type, color):
    """Add a new layer to the active project."""
    state = ctx.obj["state"]
    json_mode = ctx.obj["json_mode"]

    if not state.active_project:
        output_error(ctx, "NO_ACTIVE_PROJECT",
                     "No active project. Run `project new` or `file open` first.")
        return

    try:
        layer = state.active_project.add_layer(name, layer_type, color)
        output_result(ctx, {
            "layer_id": layer.id,
            "name": layer.name,
            "type": layer.type,
            "index": layer.index,
        }, message=f"Added layer '{name}'")
    except Exception as e:
        output_error(ctx, "LAYER_ADD_FAILED", str(e))
```

### Output Formatter (`output.py`)

```python
import click
import json
import sys

def output_result(ctx, data: dict, message: str = ""):
    json_mode = ctx.obj.get("json_mode", False)
    if json_mode:
        click.echo(json.dumps({"status": "ok", "result": data}, indent=2))
    else:
        if message:
            click.echo(f"✓ {message}")
        for key, val in data.items():
            click.echo(f"  {key}: {val}")

def output_error(ctx, code: str, message: str):
    json_mode = ctx.obj.get("json_mode", False)
    if json_mode:
        click.echo(json.dumps({
            "status": "error",
            "code": code,
            "error": message
        }, indent=2))
    else:
        click.echo(f"✗ Error [{code}]: {message}", err=True)
    sys.exit(1)
```

### State Model (`state.py`)

```python
from dataclasses import dataclass, field
from typing import Optional, List
import uuid

@dataclass
class Layer:
    id: str = field(default_factory=lambda: f"layer_{uuid.uuid4().hex[:8]}")
    name: str = ""
    type: str = "solid"
    color: str = "#ffffff"
    index: int = 0
    visible: bool = True

@dataclass
class Project:
    id: str = field(default_factory=lambda: f"proj_{uuid.uuid4().hex[:8]}")
    width: int = 1920
    height: int = 1080
    layers: List[Layer] = field(default_factory=list)
    _history: List[dict] = field(default_factory=list)

    def add_layer(self, name: str, layer_type: str, color: str) -> Layer:
        layer = Layer(name=name, type=layer_type, color=color,
                      index=len(self.layers))
        self._history.append({"action": "add_layer", "layer": layer})
        self.layers.append(layer)
        return layer

    def undo(self):
        if not self._history:
            raise ValueError("Nothing to undo")
        last = self._history.pop()
        if last["action"] == "add_layer":
            self.layers.remove(last["layer"])

@dataclass
class AppState:
    active_project: Optional[Project] = None

    def new_project(self, width: int, height: int) -> Project:
        self.active_project = Project(width=width, height=height)
        return self.active_project
```

### `setup.py`

```python
from setuptools import setup, find_packages

setup(
    name="cli-anything-gimp",
    version="0.1.0",
    packages=find_packages(),
    install_requires=["click>=8.0"],
    entry_points={
        "console_scripts": [
            "cli-anything-gimp=cli_anything_gimp.main:cli",
        ],
    },
)
```

---

## Testing the Generated Harness

```bash
# Run all tests
cd gimp/agent-harness
pytest tests/ -v

# Run only unit tests
pytest tests/unit/ -v

# Run E2E workflow tests
pytest tests/e2e/ -v

# With coverage
pytest tests/ --cov=cli_anything_gimp --cov-report=term-missing
```

### Example Unit Test

```python
# tests/unit/test_layer.py
from click.testing import CliRunner
from cli_anything_gimp.main import cli
import json

def test_add_layer_json_output():
    runner = CliRunner()
    # First create a project
    result = runner.invoke(cli, ["--json", "project", "new",
                                 "--width", "800", "--height", "600"])
    assert result.exit_code == 0

    result = runner.invoke(cli, ["--json", "layer", "add",
                                 "-n", "Background", "--type", "solid",
                                 "--color", "#ff0000"])
    assert result.exit_code == 0
    data = json.loads(result.output)
    assert data["status"] == "ok"
    assert data["result"]["name"] == "Background"

def test_add_layer_no_project_returns_error():
    runner = CliRunner()
    result = runner.invoke(cli, ["--json", "layer", "add", "-n", "Test"])
    data = json.loads(result.output)
    assert data["status"] == "error"
    assert data["code"] == "NO_ACTIVE_PROJECT"
```

### Example E2E Test

```python
# tests/e2e/test_workflow_poster.py
from click.testing import CliRunner
from cli_anything_gimp.main import cli
import json

def test_full_poster_workflow():
    runner = CliRunner()

    # 1. New project
    r = runner.invoke(cli, ["--json", "project", "new",
                             "--width", "1920", "--height", "1080"])
    assert json.loads(r.output)["status"] == "ok"

    # 2. Add background
    r = runner.invoke(cli, ["--json", "layer", "add",
                             "-n", "Background", "--color", "#1a1a2e"])
    assert json.loads(r.output)["status"] == "ok"

    # 3. Add text layer
    r = runner.invoke(cli, ["--json", "layer", "add",
                             "-n", "Title", "--type", "text"])
    assert json.loads(r.output)["status"] == "ok"

    # 4. Export
    r = runner.invoke(cli, ["--json", "file", "export",
                             "--format", "png", "--output", "/tmp/poster.png"])
    assert json.loads(r.output)["status"] == "ok"
```

---

## Supported Applications (Built-in Demos)

CLI-Anything has been demonstrated and tested against 11 apps:

| App | Harness Name | Key Command Groups |
|-----|-------------|-------------------|
| GIMP | `cli-anything-gimp` | project, layer, filter, file |
| Blender | `cli-anything-blender` | scene, object, render, animation |
| LibreOffice | `cli-anything-libreoffice` | document, spreadsheet, presentation |
| Shotcut | `cli-anything-shotcut` | project, timeline, clip, export |
| Zoom | `cli-anything-zoom` | meeting, audio, video, participants |
| + 6 more | — | — |

---

## Common Patterns for AI Agents

### Chain Commands via JSON

```bash
# Agent workflow: create → edit → export
cli-anything-gimp --json project new --width 1200 --height 800 | \
  jq -r '.result.id' | \
  xargs -I{} cli-anything-gimp --json layer add -n "BG" --color "#000"

cli-anything-gimp --json file export --format png --output result.png
```

### Check Status Before Acting

```python
import subprocess, json

def run_cli(cmd: list[str]) -> dict:
    result = subprocess.run(
        ["cli-anything-gimp", "--json"] + cmd,
        capture_output=True, text=True
    )
    return json.loads(result.stdout)

state = run_cli(["project", "status"])
if state["status"] == "ok" and state["result"]["has_active_project"]:
    run_cli(["layer", "add", "-n", "Overlay", "--type", "solid"])
else:
    run_cli(["project", "new", "--width", "1920", "--height", "1080"])
```

### Discover Available Commands

```bash
# Agents can always self-discover via --help
cli-anything-gimp --help
cli-anything-gimp layer --help
cli-anything-gimp filter --help
```

---

## Troubleshooting

### `cygpath: command not found` (Windows)

Claude Code runs via `bash`. Install **Git for Windows** (includes bash + cygpath) or use **WSL**.

### `command not found: cli-anything-gimp`

```bash
# Ensure the harness is installed
cd gimp/agent-harness && pip install -e .

# Verify entry point
pip show cli-anything-gimp
which cli-anything-gimp
```

### `No active project` errors

The generated CLI is **stateful within a session**. In scripts, always run `project new` or `file open` before other commands:

```bash
cli-anything-gimp project new --width 1920 --height 1080
cli-anything-gimp layer add -n "Layer 1"
```

### Plugin not found after install (Claude Code)

```bash
/reload-plugins
# Verify with:
/plugin list
```

### OpenCode: HARNESS.md not found

`HARNESS.md` must be in the **same directory** as the command `.md` files:

```bash
cp CLI-Anything/cli-anything-plugin/HARNESS.md ~/.config/opencode/commands/
```

### Tests failing after refine

The refine command adds new commands but existing tests are non-destructive. If tests break, run:

```bash
cd gimp/agent-harness
/cli-anything:refine ./gimp  # re-run to fix gaps
pytest tests/ -v
```

### LibreOffice / GIMP path issues on Windows/macOS

Check that the application binary is on your `$PATH` or use absolute paths when invoking the harness's underlying API. The generated `state.py` typically includes a `BINARY_PATH` config you can override:

```python
# In cli_anything_libreoffice/state.py
BINARY_PATH = "/Applications/LibreOffice.app/Contents/MacOS/soffice"
```

---

## Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Structured output** | Every command supports `--json` flag |
| **Self-describing** | Full `--help` on all commands and groups |
| **Stateful REPL** | `AppState` persists across REPL commands |
| **Undo/redo** | History stack in state model |
| **Agent-first errors** | Errors include `code` + `message` in JSON |
| **Composable** | Commands chain via pipes and scripting |
| **Deterministic** | Same inputs → same outputs, no side effects |
```
