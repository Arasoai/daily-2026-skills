```markdown
---
name: clear-code-ai-coding-assistants
description: Comprehensive guide and comparison resource for open-source AI coding assistants including Cline, OpenCode, OpenHands, Aider, Continue, Tabby, Void, and Goose
triggers:
  - set up an open source AI coding assistant
  - compare AI coding tools
  - install aider for my project
  - self-host an AI coding assistant
  - open source alternative to GitHub Copilot
  - set up Cline in VS Code
  - configure OpenHands for my team
  - which AI coding assistant should I use
---

# Clear-Code: Open-Source AI Coding Assistants Guide

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Clear-Code is a community-driven resource hub cataloging, comparing, and promoting the best open-source AI coding assistants. This skill gives AI agents the knowledge to help developers install, configure, and effectively use any of the featured tools.

---

## Tool Overview & Quick Selection

| Tool | Interface | Best For | License |
|------|-----------|----------|---------|
| **Cline** | VS Code Extension | Autonomous agent in editor | Apache 2.0 |
| **OpenCode** | Terminal TUI | CLI-first developers | MIT |
| **OpenHands** | Web UI + CLI | Enterprise/team automation | MIT |
| **Aider** | Terminal CLI | Git-integrated pair programming | Apache 2.0 |
| **Continue** | VS Code/JetBrains | Copilot replacement in IDE | Apache 2.0 |
| **Tabby** | Self-hosted server | Private, self-hosted completion | Apache 2.0 |
| **Void** | Standalone editor | Open-source Cursor alternative | — |
| **Goose** | CLI (by Block) | Extensible CLI agent | Apache 2.0 |

---

## 1. Aider — Terminal AI Pair Programmer

### Installation

```bash
# Via pip (recommended)
pip install aider-chat

# Via pipx (isolated environment)
pipx install aider-chat

# Via Homebrew (macOS)
brew install aider
```

### Configuration

```bash
# Set API keys via environment variables
export OPENAI_API_KEY=$OPENAI_API_KEY
export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
export GEMINI_API_KEY=$GEMINI_API_KEY

# Or create a .env file in your project root
echo "ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY" >> .env
```

### Key Commands

```bash
# Start aider with Claude Sonnet (recommended)
aider --model claude-sonnet-4-5

# Start with GPT-4o
aider --model gpt-4o

# Start with a specific file loaded
aider src/main.py src/utils.py

# Use a local model via Ollama
aider --model ollama/codellama

# Architect mode (uses two models: one plans, one edits)
aider --architect --model claude-opus-4-5 --editor-model claude-sonnet-4-5

# Watch mode — aider monitors files for AI comments
aider --watch-files

# Voice input mode
aider --voice

# Auto-commit off (review before committing)
aider --no-auto-commits
```

### In-Session Commands

```
# Inside an aider session:
/add src/newfile.py          # Add a file to context
/drop src/oldfile.py         # Remove a file from context
/ls                          # List files in context
/diff                        # Show last diff
/undo                        # Undo last commit
/run pytest tests/           # Run a shell command
/ask How does this work?     # Ask without making edits
/voice                       # Toggle voice input
/help                        # Show all commands
```

### Configuration File

```yaml
# .aider.conf.yml in project root or ~/.aider.conf.yml
model: claude-sonnet-4-5
auto-commits: true
dirty-commits: true
attribute-author: true
attribute-committer: true
test-cmd: pytest
lint-cmd: ruff check
auto-lint: true
auto-test: true
```

### Real Usage Example

```bash
# Initialize in a project
cd my-project
git init  # Aider requires a git repo

# Start session with relevant files
aider --model claude-sonnet-4-5 src/api.py tests/test_api.py

# Aider prompt examples:
# "Add input validation to the create_user function"
# "Write tests for the payment processing module"
# "Refactor the database connection to use a connection pool"
# "Fix the bug where null values cause a crash on line 47"
```

### Repository Map Usage

```bash
# Aider automatically builds a repo map — you can tune it
aider --map-tokens 2048    # Increase map size for large repos
aider --map-refresh auto   # Auto-refresh map as files change

# For monorepos, run from subdirectory
cd packages/backend
aider --model claude-sonnet-4-5
```

---

## 2. Cline — VS Code Autonomous Agent

### Installation

```
1. Open VS Code
2. Go to Extensions (Ctrl+Shift+X)
3. Search "Cline"
4. Install "Cline" by Saoud Rizwan
```

Or install via CLI:
```bash
code --install-extension saoudrizwan.claude-dev
```

### Configuration

After installing, open Cline settings:
1. Click the Cline icon in the sidebar
2. Click the settings gear icon
3. Select your API provider and enter credentials

Supported providers:
- Anthropic (Claude models)
- OpenAI (GPT models)
- Google Gemini
- OpenRouter (access many models)
- Ollama (local models)
- LM Studio (local models)
- AWS Bedrock
- Azure OpenAI

```json
// VS Code settings.json — Cline config
{
  "cline.apiProvider": "anthropic",
  "cline.apiModelId": "claude-opus-4-5",
  // API key set in the Cline UI, not here
}
```

### MCP (Model Context Protocol) Integration

```json
// Add to Cline MCP settings (accessible in Cline UI)
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "$GITHUB_TOKEN"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "$DATABASE_URL"
      }
    }
  }
}
```

### Effective Cline Prompts

```
# Good Cline task descriptions:
"Create a REST API endpoint for user authentication using JWT tokens.
 Use Express.js, add input validation with Zod, and write Jest tests."

"Refactor the payment processing module to handle webhooks from Stripe.
 Look at the existing stripe.js file and the webhook handler."

"Debug why the React component in src/components/DataTable.tsx causes
 a memory leak. Check the useEffect hooks."

"Set up a complete CI/CD pipeline with GitHub Actions for this Node.js
 project. Include lint, test, and deploy stages."
```

### Custom Instructions

```markdown
<!-- Cline custom instructions (set in Cline UI) -->
You are working on a TypeScript Node.js project.
- Always use TypeScript strict mode
- Follow the existing code patterns in src/
- Write tests for all new functions
- Use the project's existing logger (import from src/utils/logger)
- Never modify package.json without asking first
- Prefer async/await over callbacks
```

---

## 3. OpenHands — Self-Hosted AI Software Engineer

### Installation via Docker (Recommended)

```bash
# Pull and run OpenHands
docker pull docker.all-hands.dev/all-hands-ai/openhands:0.40

docker run -it --rm \
  --pull=always \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.40-nikolaik \
  -e LOG_ALL_EVENTS=true \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.openhands-state:/.openhands-state \
  -p 3000:3000 \
  --add-host host.docker.internal:host-gateway \
  --name openhands-app \
  docker.all-hands.dev/all-hands-ai/openhands:0.40
```

Access at: `http://localhost:3000`

### CLI Usage

```bash
# Install CLI
pip install openhands-ai

# Run a task non-interactively
openhands run \
  --task "Fix the failing tests in tests/test_api.py" \
  --model claude-opus-4-5 \
  --api-key $ANTHROPIC_API_KEY \
  --workspace /path/to/project

# Run with GitHub issue
openhands run \
  --github-token $GITHUB_TOKEN \
  --task "Fix issue #123 in this repository" \
  --repo owner/repo-name
```

### Python SDK Usage

```python
from openhands import OpenHandsClient

client = OpenHandsClient(
    model="claude-opus-4-5",
    api_key=os.environ["ANTHROPIC_API_KEY"]
)

# Run an autonomous task
result = client.run(
    task="Add pagination to the /api/users endpoint in src/routes/users.py",
    workspace="/path/to/project",
    max_iterations=30
)

print(result.summary)
print(result.files_changed)
```

### Docker Compose for Teams

```yaml
# docker-compose.yml
version: '3.8'
services:
  openhands:
    image: docker.all-hands.dev/all-hands-ai/openhands:0.40
    ports:
      - "3000:3000"
    environment:
      - SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.40-nikolaik
      - LLM_MODEL=claude-opus-4-5
      - LLM_API_KEY=$ANTHROPIC_API_KEY
      - SANDBOX_USER_ID=1000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - openhands-data:/.openhands-state
    restart: unless-stopped

volumes:
  openhands-data:
```

---

## 4. OpenCode — Terminal TUI Assistant

### Installation

```bash
# Via npm
npm install -g opencode-ai

# Via Homebrew
brew install opencode-ai/tap/opencode

# From source
git clone https://github.com/opencode-ai/opencode
cd opencode
go build -o opencode .
```

### Configuration

```json
// ~/.config/opencode/config.json
{
  "provider": "anthropic",
  "model": "claude-sonnet-4-5",
  "providers": {
    "anthropic": {
      "apiKey": "$ANTHROPIC_API_KEY"
    },
    "openai": {
      "apiKey": "$OPENAI_API_KEY"
    },
    "ollama": {
      "baseUrl": "http://localhost:11434"
    }
  },
  "theme": "dark",
  "keybindings": "vim"
}
```

### Key Commands

```bash
# Start in current directory
opencode

# Start with specific model
opencode --model claude-opus-4-5

# Start with a file in context
opencode --file src/main.go

# Continue a previous session
opencode --session my-feature-work

# List sessions
opencode sessions list

# Run in non-interactive mode
opencode run "Add error handling to the database connection"
```

### TUI Keyboard Shortcuts

```
Ctrl+N        New conversation
Ctrl+O        Open file picker
Ctrl+S        Save session
Ctrl+L        Clear screen
Tab           Cycle through panels
Enter         Send message
Ctrl+C        Cancel/Exit
/help         Show help
/model        Switch model
/context      Show current context
```

---

## 5. Continue — IDE Copilot Replacement

### Installation

**VS Code:**
```bash
code --install-extension Continue.continue
```

**JetBrains:** Install "Continue" from the JetBrains Marketplace.

### Configuration

```json
// ~/.continue/config.json
{
  "models": [
    {
      "title": "Claude Sonnet",
      "provider": "anthropic",
      "model": "claude-sonnet-4-5",
      "apiKey": "$ANTHROPIC_API_KEY"
    },
    {
      "title": "Local Llama",
      "provider": "ollama",
      "model": "codellama:13b"
    },
    {
      "title": "GPT-4o",
      "provider": "openai",
      "model": "gpt-4o",
      "apiKey": "$OPENAI_API_KEY"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Starcoder2",
    "provider": "ollama",
    "model": "starcoder2:3b"
  },
  "embeddingsProvider": {
    "provider": "transformers.js"
  },
  "contextProviders": [
    { "name": "code" },
    { "name": "docs" },
    { "name": "diff" },
    { "name": "terminal" },
    { "name": "problems" },
    { "name": "folder" },
    { "name": "codebase" }
  ],
  "slashCommands": [
    { "name": "edit", "description": "Edit selected code" },
    { "name": "comment", "description": "Write comments for code" },
    { "name": "share", "description": "Export conversation" },
    { "name": "cmd", "description": "Generate a shell command" }
  ]
}
```

### Usage Patterns

```
# In VS Code with Continue:

# Inline edit (select code, then):
Ctrl+I (or Cmd+I on Mac) → type instruction → Enter

# Chat panel:
Ctrl+Shift+L → opens chat

# Context shortcuts in chat:
@file src/api.py          # Reference a file
@code myFunction          # Reference a function
@docs                     # Reference documentation
@terminal                 # Include terminal output
@diff                     # Include current git diff
@codebase                 # Search entire codebase

# Example prompts:
"@file src/auth.py Explain how the JWT validation works"
"@diff Write a commit message for these changes"
"@terminal Fix the error shown above"
```

---

## 6. Tabby — Self-Hosted Code Completion Server

### Installation via Docker

```bash
# CPU only
docker run -it \
  --env TABBY_DOWNLOAD_HOST=huggingface.co \
  -p 8080:8080 \
  -v $HOME/.tabby:/data \
  tabbyml/tabby \
  serve --model TabbyML/StarCoder-1B

# With NVIDIA GPU
docker run -it \
  --gpus all \
  -p 8080:8080 \
  -v $HOME/.tabby:/data \
  tabbyml/tabby \
  serve --model TabbyML/DeepseekCoder-6.7B --device cuda
```

### Configuration

```toml
# ~/.tabby/config.toml
[server]
completion_timeout = 5000  # ms

[[repositories]]
name = "my-project"
git_url = "file:///path/to/project"

[[repositories]]
name = "shared-lib"
git_url = "https://github.com/org/shared-lib"
```

### VS Code Integration

```json
// VS Code settings.json
{
  "tabby.api.endpoint": "http://localhost:8080",
  "tabby.inlineCompletion.triggerMode": "automatic"
}
```

---

## 7. Goose — Extensible CLI Agent (by Block)

### Installation

```bash
# Via installer script
curl -fsSL https://github.com/block/goose/releases/latest/download/install.sh | sh

# Via Homebrew
brew install block/tap/goose

# Via pip
pip install goose-ai
```

### Configuration

```yaml
# ~/.config/goose/profiles.yaml
default:
  provider: anthropic
  processor: claude-opus-4-5
  accelerator: claude-haiku-4-5
  moderator: passive
  toolkits:
    - name: developer
    - name: github
      requires:
        GITHUB_TOKEN: $GITHUB_TOKEN
    - name: jira
      requires:
        JIRA_URL: $JIRA_URL
        JIRA_TOKEN: $JIRA_TOKEN
```

### Key Commands

```bash
# Interactive session
goose session start

# One-shot task
goose run "Add a Dockerfile to this Node.js project"

# With a specific profile
goose session start --profile my-team-config

# Resume session
goose session resume

# List available toolkits
goose toolkit list

# Check version
goose version
```

---

## Choosing the Right Tool

### Decision Tree

```
Q: Do you prefer working in an IDE or terminal?
├── IDE → Cline (VS Code) or Continue (VS Code/JetBrains)
└── Terminal → Aider, OpenCode, or Goose

Q: Do you need autonomous multi-step task execution?
├── Yes, enterprise-scale → OpenHands
├── Yes, in VS Code → Cline
├── Yes, in terminal → Goose or Aider
└── No, just assistance → Continue or Tabby

Q: Is data privacy critical (no code leaves your network)?
├── Yes → Tabby (self-hosted) + local model via Ollama
├── Somewhat → OpenHands (self-hosted) or Aider with Ollama
└── No → Any tool with your preferred cloud provider

Q: What's your primary use case?
├── Code completion/autocomplete → Tabby, Continue
├── Chat + editing → Aider, Cline, Continue
├── Autonomous task completion → OpenHands, Cline
├── Git-integrated refactoring → Aider
└── Full software engineering tasks → OpenHands
```

### Local Model Setup (Ollama)

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull a code model
ollama pull codellama:13b
ollama pull deepseek-coder:6.7b
ollama pull starcoder2:15b

# Use with Aider
aider --model ollama/codellama:13b

# Use with Continue (in config.json)
{
  "provider": "ollama",
  "model": "deepseek-coder:6.7b"
}

# Ollama OpenAI-compatible endpoint
# Base URL: http://localhost:11434/v1
# API Key: ollama (any string)
```

---

## Common Patterns & Workflows

### Pattern 1: Feature Development with Aider

```bash
# 1. Create a feature branch
git checkout -b feature/user-notifications

# 2. Start aider with relevant files
aider src/notifications.py src/models/user.py tests/test_notifications.py

# 3. Describe the feature
# "Add an email notification system that sends users alerts
#  when their subscription is about to expire. Use SendGrid.
#  Add configuration via environment variables."

# 4. Review auto-committed changes
git log --oneline -5
git diff HEAD~1
```

### Pattern 2: Bug Fix Workflow with OpenHands

```bash
# Run OpenHands on a specific bug
docker run --rm \
  -e LLM_MODEL=claude-opus-4-5 \
  -e LLM_API_KEY=$ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  docker.all-hands.dev/all-hands-ai/openhands:0.40 \
  run --task "The login endpoint returns 500 when email contains
             a + character. Reproduce, fix, and add a test."
```

### Pattern 3: Code Review with Continue

```
# In VS Code Continue chat:
@diff Review these changes for:
1. Security vulnerabilities
2. Performance issues
3. Missing error handling
4. Test coverage gaps

Provide specific line-by-line feedback.
```

### Pattern 4: Multi-file Refactoring with Aider

```bash
aider --model claude-opus-4-5 \
  src/database.py \
  src/models/*.py \
  src/repositories/*.py

# Prompt:
# "Refactor the database layer to use the Repository pattern.
#  Create a base Repository class, implement UserRepository and
#  ProductRepository. Update all existing code to use the new pattern."
```

---

## Troubleshooting

### Aider Issues

```bash
# Rate limit errors
aider --model claude-sonnet-4-5  # Use a faster/cheaper model

# Context too large
aider --map-tokens 1024  # Reduce repo map size
# Or manually manage files: /drop files you don't need

# Git conflicts
git status  # Check for uncommitted changes
aider --no-auto-commits  # Review before committing

# Model not found
aider --list-models  # Show available models
aider --model openrouter/anthropic/claude-opus-4-5  # Use OpenRouter
```

### Cline Issues

```
# Cline not responding
- Check API key in Cline settings
- Verify model name is correct
- Check rate limits on your provider dashboard

# MCP tools not working  
- Restart VS Code after adding MCP servers
- Check MCP server logs in Cline output panel
- Verify environment variables are set
```

### OpenHands Docker Issues

```bash
# Permission errors
docker run ... --user $(id -u):$(id -g) ...

# Can't access files
# Mount your workspace correctly:
-v /absolute/path/to/project:/workspace

# Port already in use
docker run ... -p 3001:3000 ...  # Use different host port

# Out of memory
docker run ... --memory=8g ...   # Increase memory limit
```

### Continue Not Autocompleting

```json
// Ensure tabAutocompleteModel is set in config.json
{
  "tabAutocompleteModel": {
    "title": "Codestral",
    "provider": "mistral",
    "model": "codestral-latest",
    "apiKey": "$MISTRAL_API_KEY"
  }
}
```

---

## Environment Variables Reference

```bash
# Common API keys needed
export ANTHROPIC_API_KEY="..."      # Claude models
export OPENAI_API_KEY="..."         # GPT models
export GEMINI_API_KEY="..."         # Google Gemini
export OPENROUTER_API_KEY="..."     # OpenRouter (access many models)
export MISTRAL_API_KEY="..."        # Mistral/Codestral
export GROQ_API_KEY="..."           # Groq (fast inference)

# Service integrations
export GITHUB_TOKEN="..."           # GitHub access
export GITLAB_TOKEN="..."           # GitLab access

# Self-hosted endpoints
export OLLAMA_BASE_URL="http://localhost:11434"
export TABBY_ENDPOINT="http://localhost:8080"
```

---

## Community & Resources

- **Discord:** https://discord.com/invite/DsRcA3GwPy
- **Updates:** https://x.com/paidev
- **Aider leaderboard:** https://aider.chat/docs/leaderboards/
- **SWE-Bench results:** https://www.swebench.com
- **MCP tools registry:** https://github.com/modelcontextprotocol/servers
- **Ollama model library:** https://ollama.ai/library
```
