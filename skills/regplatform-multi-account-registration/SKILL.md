```markdown
---
name: regplatform-multi-account-registration
description: Multi-platform batch account registration system supporting OpenAI, Grok, Kiro, Gemini with task scheduling, HuggingFace worker nodes, Cloudflare routing, credits system, and admin management
triggers:
  - set up regplatform registration system
  - configure batch account registration
  - deploy huggingface worker nodes for registration
  - set up cloudflare worker routing for regplatform
  - configure openai grok kiro account registration
  - manage hf space worker nodes
  - set up credits and user management for regplatform
  - troubleshoot regplatform worker deployment
---

# RegPlatform Multi-Platform Account Registration

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

RegPlatform is a Go-based multi-platform batch account registration system. It orchestrates automated registration across OpenAI, Grok, Kiro, and Gemini via distributed HuggingFace Space worker nodes routed through Cloudflare Workers, with a full admin panel, credits system, JWT auth, and real-time WebSocket log streaming.

---

## Architecture

```
Browser → Nginx (443) → Vue 3 SPA
                          │
                      Go Backend (:8000)
                          │
                    TaskEngine (worker pool)
                          │
                      CF Worker
                     /    |    \    \
               /openai/ /grok/ /kiro/ /ts/
                  │       │      │      │
                HFNP    HFGS   HFKR   HFTS
              (OpenAI) (Grok) (Kiro) (Turnstile)
```

---

## Prerequisites

- Go 1.25+
- Node.js 18+
- PostgreSQL 16+
- Docker & Docker Compose
- HuggingFace account + tokens
- Cloudflare Worker account

---

## Installation & Local Development

### 1. Clone and configure

```bash
git clone https://github.com/your-username/regplatform.git
cd regplatform
cp .env.example .env
```

### 2. Edit `.env`

```env
DATABASE_URL=postgres://user:password@localhost:5432/regplatform
JWT_SECRET=your-32-char-minimum-secret-here
PORT=8000
GIN_MODE=release
DEV_MODE=false
ADMIN_USERNAME=youradminuser
SSO_SECRET=
REDIS_URL=
JWT_EXPIRE_HOURS=72
CORS_ORIGINS=https://yourdomain.com
```

### 3. Start backend

```bash
go mod tidy
go run cmd/server/main.go
```

### 4. Start frontend

```bash
cd web
npm install
npm run dev
```

### 5. Docker (production)

```bash
docker-compose -f docker-compose.prod.yml up -d
```

---

## First-Time Setup

1. Navigate to `http://localhost:8000`
2. Register the first account — it **automatically becomes admin**
3. Log in and go to Admin → System Settings
4. Configure mail services, captcha solvers, and registration URLs

---

## Project Structure

```
regplatform/
├── cmd/
│   ├── server/              # Main server entrypoint
│   ├── openai-worker/       # OpenAI worker binary → HFNP
│   ├── grok-worker/         # Grok worker binary → HFGS
│   └── grpctest/            # gRPC-web debug tool
├── internal/
│   ├── config/              # Viper config loading
│   ├── model/               # GORM data models
│   ├── service/             # TaskEngine, HFSpaceService, etc.
│   ├── worker/              # Platform worker implementations
│   ├── handler/             # HTTP route handlers
│   ├── middleware/          # JWT auth, CORS, rate limiting
│   ├── dto/                 # Request/response structs
│   └── pkg/                 # Internal utilities
├── web/                     # Vue 3 frontend
├── services/
│   ├── aws-builder-id-reg/  # Kiro registration microservice
│   └── turnstile-solver/    # Turnstile solver microservice
├── cloudflare/              # CF Worker source
├── HFNP/                    # OpenAI HF Space template
├── HFGS/                    # Grok HF Space template
├── HFKR/                    # Kiro HF Space template
└── HFTS/                    # Turnstile HF Space template
```

---

## API Reference

### Authentication

```bash
# Register (rate limited: 3/min per IP)
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"myuser","password":"MyPass123"}'

# Login
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"myuser","password":"MyPass123"}'
# Returns: {"token":"eyJ..."}

# Get current user
curl http://localhost:8000/api/me \
  -H "X-Auth-Token: $TOKEN"
```

Password requirements: ≥8 chars, uppercase + lowercase + digit.

Token can be passed as:
- Cookie
- `X-Auth-Token: <token>` header
- `Authorization: Bearer <token>` header

### Task Management

```bash
# Create a registration task
curl -X POST http://localhost:8000/api/tasks \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "platform": "openai",
    "count": 10,
    "proxy_id": 1
  }'

# Start task
curl -X POST http://localhost:8000/api/tasks/123/start \
  -H "X-Auth-Token: $TOKEN"

# Stop task
curl -X POST http://localhost:8000/api/tasks/123/stop \
  -H "X-Auth-Token: $TOKEN"

# Get current running task
curl http://localhost:8000/api/tasks/current \
  -H "X-Auth-Token: $TOKEN"
```

### Results

```bash
# List results
curl http://localhost:8000/api/results \
  -H "X-Auth-Token: $TOKEN"

# Export results for a task
curl http://localhost:8000/api/results/123/export \
  -H "X-Auth-Token: $TOKEN" \
  -o results.csv
```

### Credits

```bash
# Check balance
curl http://localhost:8000/api/credits/balance \
  -H "X-Auth-Token: $TOKEN"

# Redeem a code
curl -X POST http://localhost:8000/api/credits/redeem \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code":"REDEEM-CODE-HERE"}'
```

### Proxy Management

```bash
# Add proxy
curl -X POST http://localhost:8000/api/proxies \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url":"http://user:pass@host:port","name":"proxy1"}'

# List proxies
curl http://localhost:8000/api/proxies \
  -H "X-Auth-Token: $TOKEN"

# Delete proxy
curl -X DELETE http://localhost:8000/api/proxies/1 \
  -H "X-Auth-Token: $TOKEN"
```

### Admin Endpoints

```bash
# List users
curl http://localhost:8000/api/admin/users \
  -H "X-Auth-Token: $ADMIN_TOKEN"

# Recharge credits for user
curl -X POST http://localhost:8000/api/admin/credits/recharge \
  -H "X-Auth-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id":2,"amount":1000}'

# Get/set system settings
curl http://localhost:8000/api/admin/settings \
  -H "X-Auth-Token: $ADMIN_TOKEN"

curl -X POST http://localhost:8000/api/admin/settings \
  -H "X-Auth-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"new_user_bonus","value":"100"}'

# HF Space overview
curl http://localhost:8000/api/admin/hf/overview \
  -H "X-Auth-Token: $ADMIN_TOKEN"

# Deploy HF Spaces
curl -X POST http://localhost:8000/api/admin/hf/spaces/deploy \
  -H "X-Auth-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"openai","count":3}'

# Run health check
curl -X POST http://localhost:8000/api/admin/hf/spaces/health \
  -H "X-Auth-Token: $ADMIN_TOKEN"

# Autoscale spaces
curl -X POST http://localhost:8000/api/admin/hf/autoscale \
  -H "X-Auth-Token: $ADMIN_TOKEN"

# Sync CF Worker env vars
curl -X POST http://localhost:8000/api/admin/hf/sync-cf \
  -H "X-Auth-Token: $ADMIN_TOKEN"
```

### Real-Time Logs

```bash
# SSE stream
curl -N http://localhost:8000/ws/logs/123/stream \
  -H "X-Auth-Token: $TOKEN"

# WebSocket (use wscat or browser)
wscat -c "ws://localhost:8000/ws/logs/123" \
  -H "X-Auth-Token: $TOKEN"
```

---

## System Settings Reference

Configure these in Admin → System Settings (persisted to DB):

| Key | Description |
|-----|-------------|
| `yydsmail_base_url` | YYDS Mail temp email API URL |
| `gptmail_api_key` | GPTMail API key |
| `gptmail_base_url` | GPTMail service URL |
| `turnstile_solver_url` | Turnstile solver service URL |
| `cf_bypass_solver_url` | CF bypass solver URL |
| `yescaptcha_key` | YesCaptcha API key |
| `openai_reg_url` | OpenAI registration service URL |
| `grok_reg_url` | Grok registration service URL |
| `kiro_reg_url` | Kiro registration service URL |
| `default_proxy` | Default proxy address |
| `new_user_bonus` | Credits given to new users |
| `max_threads_per_user` | Max concurrent threads per user |
| `max_target_per_task` | Max targets per task |

---

## HuggingFace Space Workers

Each platform uses a dedicated HF Space type:

| Platform | Space Type | Release Tag | Notes |
|----------|-----------|-------------|-------|
| OpenAI | HFNP | `inference-runtime-latest` | Go HTTP, Sentinel PoW + PKCE OAuth |
| Grok | HFGS | `stream-worker-latest` | Go HTTP, gRPC-web + Turnstile |
| Kiro | HFKR | `browser-agent-latest` | Python, AWS Cognito + Camoufox |
| Gemini | HFGM | `gemini-agent-latest` | Python, Camoufox browser automation |
| Turnstile | HFTS | `net-toolkit-latest` | Python, Camoufox + Patchright |

### Deploy Worker Binaries to HF Space

Worker binaries are released via GitHub Actions to GitHub Releases. The HF Space templates in `HFNP/`, `HFGS/`, `HFKR/`, `HFTS/` pull from the latest release tags.

```bash
# Build OpenAI worker locally
go build -o inference-runtime ./cmd/openai-worker/

# Build Grok worker locally
go build -o stream-worker ./cmd/grok-worker/
```

CI/CD auto-builds on push to `main`:
- Docker image → GHCR
- Worker zips → GitHub Release (tagged `inference-runtime-latest`, `stream-worker-latest`, etc.)

---

## Cloudflare Worker Setup

The CF Worker routes requests by path prefix to the correct HF Space pool.

```javascript
// cloudflare/ — key routing logic (simplified)
// Environment variables set via CF dashboard or sync-cf API:
// OPENAI_SPACES = comma-separated HF Space URLs
// GROK_SPACES   = comma-separated HF Space URLs
// KIRO_SPACES   = comma-separated HF Space URLs
// TS_SPACES     = comma-separated HF Space URLs

// Path prefix routing:
// /openai/* → round-robin across OPENAI_SPACES
// /grok/*   → round-robin across GROK_SPACES
// /kiro/*   → round-robin across KIRO_SPACES
// /ts/*     → round-robin across TS_SPACES
```

After deploying HF Spaces, use `POST /api/admin/hf/sync-cf` to automatically update CF Worker environment variables with active Space URLs.

---

## Go Code Patterns

### Adding a new platform worker

```go
// internal/worker/myplatform.go
package worker

import (
    "context"
    "github.com/your-username/regplatform/internal/dto"
)

type MyPlatformWorker struct {
    cfg *Config
}

func NewMyPlatformWorker(cfg *Config) *MyPlatformWorker {
    return &MyPlatformWorker{cfg: cfg}
}

func (w *MyPlatformWorker) Register(ctx context.Context, req *dto.RegisterRequest) (*dto.RegisterResult, error) {
    // 1. Get temp email from mail service
    // 2. Submit registration form / API
    // 3. Solve captcha if needed (via turnstile_solver_url)
    // 4. Verify email
    // 5. Return credentials
    return &dto.RegisterResult{
        Email:    req.Email,
        Password: req.Password,
        Token:    "oauth-token-here",
    }, nil
}
```

### Calling the task engine from a handler

```go
// internal/handler/task.go
func (h *TaskHandler) CreateTask(c *gin.Context) {
    var req dto.CreateTaskRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    userID := c.GetUint("user_id") // set by JWT middleware

    task, err := h.taskService.Create(c.Request.Context(), userID, &req)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    c.JSON(201, task)
}
```

### JWT middleware pattern

```go
// internal/middleware/auth.go
func JWTAuth(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := extractToken(c) // checks Cookie, X-Auth-Token, Authorization
        claims, err := validateJWT(token, secret)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }
        c.Set("user_id", claims.UserID)
        c.Set("is_admin", claims.IsAdmin)
        c.Next()
    }
}

func AdminRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        if !c.GetBool("is_admin") {
            c.AbortWithStatusJSON(403, gin.H{"error": "admin required"})
            return
        }
        c.Next()
    }
}
```

### SSE log streaming

```go
// internal/handler/logs.go
func (h *LogHandler) StreamLogs(c *gin.Context) {
    taskID := c.Param("taskId")

    c.Header("Content-Type", "text/event-stream")
    c.Header("Cache-Control", "no-cache")
    c.Header("Connection", "keep-alive")

    ch := h.logService.Subscribe(taskID)
    defer h.logService.Unsubscribe(taskID, ch)

    c.Stream(func(w io.Writer) bool {
        select {
        case msg, ok := <-ch:
            if !ok {
                return false
            }
            c.SSEvent("log", msg)
            return true
        case <-c.Request.Context().Done():
            return false
        }
    })
}
```

---

## Vue 3 Frontend Patterns

### Calling the API with auth token

```javascript
// web/src/api/client.js
import axios from 'axios'
import { useAuthStore } from '@/stores/auth'

const client = axios.create({ baseURL: '/api' })

client.interceptors.request.use(config => {
  const auth = useAuthStore()
  if (auth.token) {
    config.headers['X-Auth-Token'] = auth.token
  }
  return config
})

export default client
```

### SSE log streaming in Vue

```javascript
// web/src/composables/useLogs.js
import { ref, onUnmounted } from 'vue'

export function useLogs(taskId) {
  const logs = ref([])
  const es = new EventSource(`/ws/logs/${taskId}/stream`, {
    headers: { 'X-Auth-Token': localStorage.getItem('token') }
  })

  es.addEventListener('log', e => {
    logs.value.push(JSON.parse(e.data))
  })

  onUnmounted(() => es.close())

  return { logs }
}
```

---

## Docker Compose (Production)

```yaml
# docker-compose.prod.yml key services
services:
  app:
    image: ghcr.io/your-username/regplatform:latest
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: regplatform
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

---

## Common Troubleshooting

### Database connection fails
```bash
# Verify PostgreSQL is running and DATABASE_URL is correct
psql "$DATABASE_URL" -c "SELECT 1"
# Format: postgres://user:pass@host:5432/dbname?sslmode=disable
```

### JWT errors in production
- Ensure `JWT_SECRET` is ≥32 characters
- Check token expiry: default is 72 hours (`JWT_EXPIRE_HOURS`)
- Verify CORS: set `CORS_ORIGINS` to your frontend domain

### HF Space workers not responding
1. Check space status: `POST /api/admin/hf/spaces/health`
2. Verify HF tokens are valid in admin panel
3. Check CF Worker env vars are synced: `POST /api/admin/hf/sync-cf`
4. Spaces may be sleeping — autoscale wakes them: `POST /api/admin/hf/autoscale`

### Registration tasks stuck
- Check `turnstile_solver_url` is set and reachable
- Verify `default_proxy` or task-specific proxy is working
- Check mail service settings (`yydsmail_base_url` or `gptmail_*`)
- View real-time logs: `GET /ws/logs/:taskId/stream`

### Rate limiting
- Register endpoint: 3 requests/min per IP
- Login endpoint: 10 requests/min per IP
- Task concurrency: controlled by `max_threads_per_user` setting

### First admin not created
- Either be the first registered user, OR
- Set `ADMIN_USERNAME=yourusername` in `.env` before registering

### Dev mode login (for testing)
```bash
# Only available when DEV_MODE=true
curl http://localhost:8000/api/auth/dev-login?username=testuser
```

---

## CI/CD — GitHub Actions

On push to `main`, the workflow builds:

| Job | Output | Tag |
|-----|--------|-----|
| `build-app` | Docker image → GHCR | `latest` |
| `build-openai-worker` | `inference-runtime.zip` | `inference-runtime-latest` |
| `build-grok-worker` | `stream-worker.zip` | `stream-worker-latest` |
| `build-kiro-reg` | `browser-agent.zip` | `browser-agent-latest` |
| `build-gemini-reg` | `gemini-agent.zip` | `gemini-agent-latest` |
| `build-ts-solver` | `net-toolkit.zip` | `net-toolkit-latest` |

HF Space templates reference these release tags to pull the latest worker binaries on startup.
```
