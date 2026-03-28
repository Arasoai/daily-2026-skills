```markdown
---
name: openilink-hub-wechat-bot
description: Self-hosted WeChat Bot management platform using iLink protocol with WebSocket/Webhook/AI relay, App Marketplace, and 7-language SDKs
triggers:
  - set up openilink hub for wechat bot management
  - integrate wechat messages with webhook or websocket
  - build an openilink app for wechat automation
  - configure ai auto-reply for wechat bot
  - deploy openilink hub with docker
  - use openilink sdk to send wechat messages
  - create custom app for openilink marketplace
  - forward wechat messages to slack or feishu
---

# OpeniLink Hub — WeChat Bot Management Platform

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

OpeniLink Hub is an open-source self-hosted platform for managing WeChat bots via the official **iLink (智联) protocol** introduced with WeChat ClawBot in March 2026. It handles context_token lifecycle, CDN encryption, 24-hour session renewal, multi-bot management, and message dispatch — so you focus on your business logic.

## Installation

### One-line install (Linux/macOS)
```bash
curl -fsSL https://raw.githubusercontent.com/openilink/openilink-hub/main/install.sh | sh
oih
```

### Docker (quickstart)
```bash
docker run -d -p 9800:9800 ghcr.io/openilink/openilink-hub:latest
```

### Docker Compose (production with PostgreSQL)
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: openilink
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: openilink
    volumes:
      - pgdata:/var/lib/postgresql/data

  hub:
    image: ghcr.io/openilink/openilink-hub:latest
    ports:
      - "9800:9800"
    environment:
      DATABASE_URL: postgres://openilink:${POSTGRES_PASSWORD}@postgres:5432/openilink?sslmode=disable
      RP_ORIGIN: https://hub.example.com
      RP_ID: hub.example.com
      SECRET: ${HUB_SECRET}
    depends_on:
      - postgres

volumes:
  pgdata:
```

```bash
docker compose up -d
```

### Build from source
```bash
git clone https://github.com/openilink/openilink-hub.git
cd openilink-hub
cd web && pnpm install && pnpm run build && cd ..
go build -o oih .
./oih
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `oih` | Run in foreground |
| `oih install` | Register as systemd (Linux) or launchd (macOS) service |
| `oih uninstall` | Remove system service |
| `oih version` | Show version |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | SQLite (auto) | `postgres://user:pass@host/db?sslmode=disable` |
| `SECRET` | — | Random string for signing tokens |
| `RP_ORIGIN` | `http://localhost:9800` | WebAuthn relying party origin (full URL) |
| `RP_ID` | `localhost` | WebAuthn relying party ID (domain only) |
| `PORT` | `9800` | HTTP server port |

**SQLite data locations:**
- Linux: `~/.local/share/openilink-hub/openilink.db`
- macOS: `~/Library/Application Support/openilink-hub/openilink.db`
- root/service: `/var/lib/openilink-hub/openilink.db`

## SDK Usage

### Go SDK
```bash
go get github.com/openilink/openilink-sdk-go
```

```go
package main

import (
    "context"
    "fmt"
    "log"

    openilink "github.com/openilink/openilink-sdk-go"
)

func main() {
    client := openilink.NewClient(openilink.Config{
        HubURL: "http://localhost:9800",
        Token:  os.Getenv("OPENILINK_TOKEN"),
    })

    // Listen for incoming messages via WebSocket
    err := client.OnMessage(context.Background(), func(msg *openilink.Message) error {
        fmt.Printf("[%s] %s: %s\n", msg.BotID, msg.Sender, msg.Content)

        // Reply to the message
        return client.SendText(context.Background(), &openilink.SendTextRequest{
            BotID:    msg.BotID,
            ToUserID: msg.Sender,
            Content:  "收到你的消息: " + msg.Content,
        })
    })
    if err != nil {
        log.Fatal(err)
    }
}
```

### Node.js SDK
```bash
npm install @openilink/openilink-sdk-node
```

```javascript
import { OpeniLinkClient } from '@openilink/openilink-sdk-node';

const client = new OpeniLinkClient({
  hubUrl: 'http://localhost:9800',
  token: process.env.OPENILINK_TOKEN,
});

// Subscribe to messages
client.onMessage(async (msg) => {
  console.log(`[${msg.botId}] ${msg.sender}: ${msg.content}`);

  // Send a reply
  await client.sendText({
    botId: msg.botId,
    toUserId: msg.sender,
    content: `Echo: ${msg.content}`,
  });
});

await client.connect();
```

### Python SDK
```bash
pip install openilink-sdk-python
```

```python
import os
import asyncio
from openilink import OpeniLinkClient

async def main():
    client = OpeniLinkClient(
        hub_url="http://localhost:9800",
        token=os.environ["OPENILINK_TOKEN"],
    )

    @client.on_message
    async def handle(msg):
        print(f"[{msg.bot_id}] {msg.sender}: {msg.content}")
        await client.send_text(
            bot_id=msg.bot_id,
            to_user_id=msg.sender,
            content=f"你好，我收到了: {msg.content}",
        )

    await client.connect()

asyncio.run(main())
```

## Webhook Configuration

Configure a Webhook endpoint in the Hub dashboard under **Settings → Webhooks**. Hub will POST events to your URL:

```json
{
  "event": "message.received",
  "bot_id": "bot_abc123",
  "message": {
    "id": "msg_xyz",
    "sender": "user_openid",
    "content": "Hello",
    "type": "text",
    "timestamp": 1711900800
  }
}
```

### Express.js webhook handler
```javascript
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

app.post('/webhook', (req, res) => {
  // Verify Hub signature
  const sig = req.headers['x-openilink-signature'];
  const expected = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(JSON.stringify(req.body))
    .digest('hex');

  if (sig !== `sha256=${expected}`) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const { event, bot_id, message } = req.body;

  if (event === 'message.received') {
    console.log(`Bot ${bot_id} received: ${message.content}`);
    // process message...
  }

  res.json({ ok: true });
});

app.listen(8080);
```

### Go webhook handler
```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
)

type WebhookEvent struct {
    Event   string  `json:"event"`
    BotID   string  `json:"bot_id"`
    Message Message `json:"message"`
}

type Message struct {
    ID        string `json:"id"`
    Sender    string `json:"sender"`
    Content   string `json:"content"`
    Type      string `json:"type"`
    Timestamp int64  `json:"timestamp"`
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    var body []byte
    body, _ = io.ReadAll(r.Body)

    // Verify signature
    sig := r.Header.Get("X-OpeniLink-Signature")
    mac := hmac.New(sha256.New, []byte(os.Getenv("WEBHOOK_SECRET")))
    mac.Write(body)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))
    if sig != expected {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    var event WebhookEvent
    if err := json.Unmarshal(body, &event); err != nil {
        http.Error(w, "Bad Request", http.StatusBadRequest)
        return
    }

    if event.Event == "message.received" {
        fmt.Printf("Bot %s received from %s: %s\n",
            event.BotID, event.Message.Sender, event.Message.Content)
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte(`{"ok":true}`))
}

func main() {
    http.HandleFunc("/webhook", webhookHandler)
    http.ListenAndServe(":8080", nil)
}
```

## Building a Custom App

Apps integrate with Hub via WebSocket or Webhook and are authorized via PKCE OAuth.

### Minimal Go App (based on openilink-app-echo)
```bash
git clone https://github.com/openilink/openilink-app-echo.git my-app
cd my-app
```

```go
// main.go
package main

import (
    "context"
    "log"
    "os"

    app "github.com/openilink/openilink-app-sdk-go"
)

func main() {
    a := app.New(app.Config{
        AppID:     os.Getenv("APP_ID"),
        AppSecret: os.Getenv("APP_SECRET"),
        HubURL:    os.Getenv("HUB_URL"),
    })

    // Declare a command: users type /ping in WeChat
    a.RegisterCommand("ping", "Reply with pong", func(ctx context.Context, req *app.CommandRequest) error {
        return req.Reply("pong 🏓")
    })

    // Handle all messages
    a.OnMessage(func(ctx context.Context, msg *app.Message) error {
        log.Printf("Received: %s", msg.Content)
        return nil
    })

    if err := a.Start(context.Background()); err != nil {
        log.Fatal(err)
    }
}
```

**App manifest** (`app.json`) required for marketplace listing:
```json
{
  "id": "my-awesome-app",
  "name": "My Awesome App",
  "version": "1.0.0",
  "description": "Does something great with WeChat messages",
  "webhook_url": "https://my-app.example.com/webhook",
  "commands": [
    {
      "name": "ping",
      "description": "Replies with pong"
    }
  ],
  "scopes": ["messages:read", "messages:write"]
}
```

### Local Development with Mock Server
```bash
# Terminal 1: Start Mock Server (no real WeChat needed)
go run ./cmd/appmock --webhook-url http://localhost:8080/webhook

# Terminal 2: Run your App
APP_ID=test APP_SECRET=test HUB_URL=http://localhost:9801 go run .

# Terminal 3: Inject a test message
curl -X POST http://localhost:9801/mock/event \
  -H "Content-Type: application/json" \
  -d '{"sender":"alice","content":"/ping"}'

# Check what your App sent back
curl http://localhost:9801/mock/messages
```

## AI Auto-Reply Configuration

In the Hub dashboard → **Settings → AI**:

```env
# Set via env or dashboard
OPENAI_API_KEY=...        # Set in environment
OPENAI_BASE_URL=...       # For compatible APIs (Ollama, DeepSeek, etc.)
OPENAI_MODEL=gpt-4o-mini
AI_SYSTEM_PROMPT="You are a helpful WeChat assistant."
```

Compatible with any OpenAI-spec API (Ollama, DeepSeek, Qwen, etc.).

## REST API Patterns

Hub exposes a REST API at `http://localhost:9800/api/v1`.

```bash
# List connected bots
curl -H "Authorization: Bearer $OPENILINK_TOKEN" \
  http://localhost:9800/api/v1/bots

# Send a text message
curl -X POST \
  -H "Authorization: Bearer $OPENILINK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"bot_id":"bot_abc","to_user_id":"user_openid","content":"Hello!"}' \
  http://localhost:9800/api/v1/messages/send

# Send an image
curl -X POST \
  -H "Authorization: Bearer $OPENILINK_TOKEN" \
  -F "bot_id=bot_abc" \
  -F "to_user_id=user_openid" \
  -F "image=@/path/to/image.png" \
  http://localhost:9800/api/v1/messages/send-image

# Get message trace (debugging)
curl -H "Authorization: Bearer $OPENILINK_TOKEN" \
  http://localhost:9800/api/v1/messages/trace/msg_xyz
```

## Common Patterns

### Pattern: Forward WeChat messages to Slack
Install the **Bridge** app from the marketplace, configure:
```env
BRIDGE_TARGET=slack
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
BRIDGE_BOT_ID=bot_abc123   # which bot to bridge
```

### Pattern: Multi-bot message routing
```go
client.OnMessage(context.Background(), func(msg *openilink.Message) error {
    switch msg.BotID {
    case "bot_customer_service":
        return handleCustomerService(msg)
    case "bot_sales":
        return handleSales(msg)
    default:
        return handleDefault(msg)
    }
})
```

### Pattern: Command parsing in webhook
```python
@client.on_message
async def handle(msg):
    if msg.content.startswith("/"):
        parts = msg.content.split(maxsplit=1)
        cmd = parts[0][1:]   # strip leading /
        args = parts[1] if len(parts) > 1 else ""

        if cmd == "help":
            await client.send_text(
                bot_id=msg.bot_id,
                to_user_id=msg.sender,
                content="Commands: /help /status /ping"
            )
        elif cmd == "ping":
            await client.send_text(
                bot_id=msg.bot_id,
                to_user_id=msg.sender,
                content="pong!"
            )
```

## Troubleshooting

### Bot shows offline / session expired
- Hub auto-renews 24-hour iLink sessions. If a bot goes offline, check **Dashboard → Bots → Status**.
- Re-scan QR code from the bot detail page.
- Check logs: `journalctl -u openilink-hub -f` (if installed as service).

### Messages not delivered to Webhook
- Check **Dashboard → Messages → Trace** for the message — shows each delivery step with latency.
- Verify webhook URL is reachable from Hub (especially in Docker: use `host.docker.internal` not `localhost`).
- Confirm `WEBHOOK_SECRET` matches on both sides.

### Passkey login not working
- `RP_ORIGIN` must exactly match the browser URL (including port), e.g. `https://hub.example.com`.
- `RP_ID` must be the domain only, e.g. `hub.example.com`.
- Passkey requires HTTPS in production (localhost is exempt).

### App OAuth flow fails
- Ensure App's redirect URI is registered in Hub dashboard under **Apps → OAuth Settings**.
- PKCE is required — `code_challenge_method` must be `S256`.

### Docker networking: App can't reach Hub
```yaml
# Use service name, not localhost
environment:
  HUB_URL: http://hub:9800
```

### Database migration issues (PostgreSQL)
```bash
# Hub runs migrations automatically on start.
# To reset (destructive!):
docker compose down -v
docker compose up -d
```

## Project Structure (for contributors)

```
openilink-hub/
├── cmd/
│   ├── main.go          # CLI entrypoint (oih)
│   └── appmock/         # Mock server for App development
├── internal/
│   ├── api/             # REST API handlers
│   ├── broker/          # Message dispatch (App/WS/Webhook/AI)
│   ├── provider/        # iLink protocol adapter
│   └── store/           # SQLite/PostgreSQL data layer
├── web/                 # React 19 frontend
├── docs/
│   └── mock-server.md   # Mock server documentation
└── docker-compose.yml
```

## Related Repositories

| Repo | Purpose |
|------|---------|
| [openilink-sdk-go](https://github.com/openilink/openilink-sdk-go) | Go SDK |
| [openilink-sdk-node](https://github.com/openilink/openilink-sdk-node) | Node.js SDK |
| [openilink-sdk-python](https://github.com/openilink/openilink-sdk-python) | Python SDK |
| [openilink-app-echo](https://github.com/openilink/openilink-app-echo) | App development template |
| [openilink-app-command-service](https://github.com/openilink/openilink-app-command-service) | 20+ built-in commands |
| [openclaw-channels](https://github.com/openilink/openclaw-channels) | OpenClaw AI Agent integration |
```
