```markdown
---
name: nanobot-ai-assistant
description: Ultra-lightweight personal AI assistant framework with multi-channel support, MCP integration, and agent capabilities
triggers:
  - set up nanobot AI assistant
  - configure nanobot with telegram discord
  - install nanobot agent framework
  - connect nanobot to chat platforms
  - nanobot MCP tool integration
  - deploy lightweight AI agent
  - nanobot provider configuration
  - build AI assistant with nanobot
---

# nanobot AI Assistant

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

nanobot is an ultra-lightweight personal AI assistant framework (~99% fewer lines of code than OpenClaw) that delivers full agent functionality including multi-channel chat integrations, MCP (Model Context Protocol) support, memory, scheduled tasks, web search, and extensible skills.

## Installation

### From Source (recommended for development)
```bash
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
pip install -e .
```

### From PyPI (stable)
```bash
pip install nanobot-ai
```

### With uv (fast, isolated)
```bash
uv tool install nanobot-ai
```

### Update
```bash
pip install -U nanobot-ai
# or
uv tool upgrade nanobot-ai
```

**Requirements:** Python ≥ 3.11

## Quick Start

```bash
# 1. Initialize workspace
nanobot onboard

# 2. Edit config at ~/.nanobot/config.json
# 3. Start chatting
nanobot agent
```

## Configuration

All configuration lives in `~/.nanobot/config.json`.

### Minimal Configuration

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "$OPENROUTER_API_KEY"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5",
      "provider": "openrouter"
    }
  }
}
```

### Full Configuration Example

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "$OPENROUTER_API_KEY"
    },
    "openai": {
      "apiKey": "$OPENAI_API_KEY"
    },
    "anthropic": {
      "apiKey": "$ANTHROPIC_API_KEY"
    },
    "deepseek": {
      "apiKey": "$DEEPSEEK_API_KEY"
    },
    "ollama": {
      "baseUrl": "http://localhost:11434"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5",
      "provider": "openrouter"
    }
  },
  "memory": {
    "enabled": true
  },
  "search": {
    "provider": "tavily",
    "apiKey": "$TAVILY_API_KEY"
  }
}
```

### Supported Providers

| Provider | Config Key | Notes |
|----------|-----------|-------|
| OpenRouter | `openrouter` | Recommended for global access |
| OpenAI | `openai` | Direct OpenAI API |
| Anthropic | `anthropic` | Direct Anthropic API |
| DeepSeek | `deepseek` | |
| Qwen | `qwen` | |
| Moonshot/Kimi | `moonshot` | |
| MiniMax | `minimax` | |
| VolcEngine | `volcengine` | |
| Azure OpenAI | `azure` | |
| Ollama | `ollama` | Local models via `baseUrl` |
| vLLM | `vllm` | Local LLMs |
| OpenAI Codex | `codex` | OAuth login |

## CLI Reference

```bash
# Initialize workspace
nanobot onboard

# Start interactive agent chat
nanobot agent

# Start all configured channels (Telegram, Discord, etc.)
nanobot channels start

# Login / setup a specific channel
nanobot channels login

# List available skills
nanobot skills list

# Install a skill from ClawHub
nanobot skills install <skill-name>

# Show version
nanobot --version

# Restart bot (useful in channel sessions)
/restart
```

## Channel Setup

### Telegram (Recommended)

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "$TELEGRAM_BOT_TOKEN",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    }
  }
}
```

Setup steps:
1. Message `@BotFather` on Telegram → `/newbot`
2. Copy the token into config
3. Find your User ID in Telegram settings
4. Run `nanobot channels start`

### Discord

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "$DISCORD_BOT_TOKEN",
      "allowFrom": ["YOUR_DISCORD_USER_ID"]
    }
  }
}
```

Requires **Message Content Intent** enabled in Discord Developer Portal.

### Slack

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "$SLACK_BOT_TOKEN",
      "appToken": "$SLACK_APP_TOKEN"
    }
  }
}
```

### WhatsApp

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true
    }
  }
}
```

```bash
nanobot channels login   # Scan QR code to authenticate
```

After upgrading nanobot with WhatsApp:
```bash
rm -rf ~/.nanobot/bridge
nanobot channels login
```

### Feishu (Lark)

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "$FEISHU_APP_ID",
      "appSecret": "$FEISHU_APP_SECRET"
    }
  }
}
```

### DingTalk

```json
{
  "channels": {
    "dingtalk": {
      "enabled": true,
      "appKey": "$DINGTALK_APP_KEY",
      "appSecret": "$DINGTALK_APP_SECRET"
    }
  }
}
```

### Email

```json
{
  "channels": {
    "email": {
      "enabled": true,
      "imap": {
        "host": "imap.gmail.com",
        "port": 993,
        "user": "$EMAIL_USER",
        "password": "$EMAIL_PASSWORD"
      },
      "smtp": {
        "host": "smtp.gmail.com",
        "port": 587,
        "user": "$EMAIL_USER",
        "password": "$EMAIL_PASSWORD"
      }
    }
  }
}
```

## MCP (Model Context Protocol)

nanobot supports MCP for connecting external tools and data sources.

### MCP Configuration

```json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
      },
      "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {
          "GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_TOKEN"
        }
      },
      "remote-sse": {
        "url": "https://your-mcp-server.com/sse",
        "headers": {
          "Authorization": "Bearer $MCP_TOKEN"
        }
      }
    }
  }
}
```

MCP servers can expose tools, prompts, and resources that nanobot's agent can use automatically during conversations.

## Web Search

```json
{
  "search": {
    "provider": "tavily",
    "apiKey": "$TAVILY_API_KEY"
  }
}
```

Supported search providers: `tavily`, and other multi-provider options. Web search enables the agent to retrieve real-time information.

## Memory System

Token-based memory is enabled by default. The agent automatically stores and retrieves relevant context across conversations.

```json
{
  "memory": {
    "enabled": true,
    "maxTokens": 4000
  }
}
```

## Scheduled Tasks (Cron)

nanobot supports natural language cron scheduling. Tell the agent:

```
"Remind me every day at 9am to check emails"
"Send me a market summary every weekday at 8am"
```

The agent will set up recurring tasks stored in your workspace.

## Skills / Plugins

```bash
# Browse available skills on ClawHub
nanobot skills list

# Install a skill
nanobot skills install web-scraper

# Skills live in ~/.nanobot/skills/
```

Custom skills are Python files placed in `~/.nanobot/skills/`. Each skill exposes tools the agent can call.

### Writing a Custom Skill

```python
# ~/.nanobot/skills/my_skill.py

from nanobot.skill import skill, tool

@skill(name="my-skill", description="My custom skill")
class MySkill:

    @tool(description="Fetch data from an API")
    async def fetch_data(self, url: str) -> str:
        """Fetch data from the given URL."""
        import httpx
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            return response.text

    @tool(description="Process text input")
    async def process_text(self, text: str, uppercase: bool = False) -> str:
        """Process the input text."""
        if uppercase:
            return text.upper()
        return text.strip()
```

## Docker Deployment

```bash
# Build image
docker build -t nanobot .

# Run with config volume
docker run -d \
  -v ~/.nanobot:/root/.nanobot \
  -e OPENROUTER_API_KEY=$OPENROUTER_API_KEY \
  --name nanobot \
  nanobot
```

Docker Compose example:

```yaml
version: "3.8"
services:
  nanobot:
    image: hkuds/nanobot:latest
    volumes:
      - ~/.nanobot:/root/.nanobot
    environment:
      - OPENROUTER_API_KEY=${OPENROUTER_API_KEY}
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
    restart: unless-stopped
```

## Linux Service (systemd)

```ini
# /etc/systemd/system/nanobot.service
[Unit]
Description=nanobot AI Assistant
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser
ExecStart=/usr/local/bin/nanobot channels start
Restart=on-failure
RestartSec=5
Environment=OPENROUTER_API_KEY=your_key_here

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable nanobot
sudo systemctl start nanobot
sudo systemctl status nanobot
```

## Multiple Instances

Run multiple nanobot instances with different configs:

```bash
# Specify a custom config directory
NANOBOT_HOME=~/.nanobot-work nanobot agent
NANOBOT_HOME=~/.nanobot-personal nanobot channels start
```

Each instance maintains its own memory, skills, and channel configurations.

## Web Proxy Support

```json
{
  "proxy": {
    "http": "http://proxy.example.com:8080",
    "https": "http://proxy.example.com:8080"
  }
}
```

## LangSmith Integration

```json
{
  "langsmith": {
    "apiKey": "$LANGSMITH_API_KEY",
    "project": "nanobot-traces"
  }
}
```

## Project Structure

```
~/.nanobot/
├── config.json          # Main configuration
├── skills/              # Custom skill plugins
├── memory/              # Persistent agent memory
├── sessions/            # Chat session history
├── bridge/              # WhatsApp bridge data
└── workspace/           # Agent working directory
```

Repository structure:
```
nanobot/
├── nanobot/
│   ├── agent/           # Core agent logic
│   ├── channels/        # Chat platform integrations
│   ├── providers/       # LLM provider adapters
│   ├── skills/          # Built-in skills
│   ├── memory/          # Memory system
│   └── mcp/             # MCP client
├── core_agent_lines.sh  # Line count verification
└── pyproject.toml
```

## Troubleshooting

### Agent not responding
```bash
# Check version
nanobot --version

# Verify config is valid JSON
python -m json.tool ~/.nanobot/config.json

# Test provider connectivity
nanobot agent  # try a simple message
```

### Channel not connecting
```bash
# Re-run channel login
nanobot channels login

# Check channel is enabled in config
cat ~/.nanobot/config.json | python -m json.tool
```

### WhatsApp session lost after upgrade
```bash
rm -rf ~/.nanobot/bridge
nanobot channels login   # scan QR code again
```

### Memory issues
```bash
# Memory files are in ~/.nanobot/memory/
# Clear to reset (loses all stored context)
rm -rf ~/.nanobot/memory/
```

### MCP server not found
Ensure `npx` / `node` is installed for Node.js-based MCP servers:
```bash
node --version
npx --version
```

### Model not available
Check that the model name matches the provider's format:
- OpenRouter: `anthropic/claude-opus-4-5`
- OpenAI direct: `gpt-4o`
- Anthropic direct: `claude-opus-4-5`
- Ollama: `llama3.2`

### Verify line count claim
```bash
cd /path/to/nanobot
bash core_agent_lines.sh
```

## Common Patterns

### Environment variable substitution in config
nanobot supports `$ENV_VAR` syntax in config values:
```json
{
  "providers": {
    "openai": {
      "apiKey": "$OPENAI_API_KEY"
    }
  }
}
```

### Thinking mode (experimental)
```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5",
      "thinking": true
    }
  }
}
```

### Multi-agent / subagents
Subagents are supported in both CLI and channel modes. The main agent can spawn subagents for parallel tasks automatically based on the conversation context.

### Access control for channels
```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "$TELEGRAM_BOT_TOKEN",
      "allowFrom": ["user_id_1", "user_id_2"]
    }
  }
}
```

Leave `allowFrom` empty array `[]` for open access (not recommended for public bots).
```
