```markdown
---
name: regplatformm-multi-platform-registration
description: Multi-platform account batch registration system supporting OpenAI, Grok, Kiro, Gemini with task engine, HuggingFace worker nodes, Cloudflare routing, credits system, and admin management
triggers:
  - set up regplatform registration system
  - configure batch account registration
  - deploy HuggingFace worker spaces
  - set up OpenAI Grok Kiro account registration
  - configure Cloudflare worker routing for registration
  - manage HF space worker nodes
  - set up credits and user management for regplatform
  - integrate regplatform with external systems
---

# RegPlatformm — Multi-Platform Account Batch Registration System

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

RegPlatform is a production-ready multi-platform account batch registration system built in Go. It supports automated account registration for OpenAI, Grok, Kiro, and Gemini via a distributed worker architecture: a Go/Gin backend orchestrates tasks, HuggingFace Space nodes execute registrations, and a Cloudflare Worker handles load balancing and routing.

## Architecture

```
Browser → Nginx → Vue 3 SPA
                → Go Backend (:8000)
                      ├── TaskEngine (worker pool, scheduling)
                      ├── HFSpaceService (elastic node management)
                      ├── WebSocket (real-time logs)
                      └── PostgreSQL
                            ↓
                      CF Worker (path-prefix routing)
                      ├── /openai/* → HFNP pool
                      ├── /grok/*   → HFGS pool
                      ├── /kiro/*   → HFKR pool
                      └── /ts/*     → HFTS pool
```

## Tech Stack

- **Backend**: Go 1.25, Gin, GORM, PostgreSQL 16
- **Frontend**: Vue 3, Vite, Pinia, TailwindCSS
- **CI/CD**: GitHub Actions → GHCR Docker + GitHub Release
- **Deployment**: Docker Compose + HuggingFace Spaces (worker nodes)
- **Routing**: Cloudflare Worker (path-prefix load balancing)

## Installation & Setup

### Prerequisites

- Go 1.25+
- Node.js 18+
- PostgreSQL 16+
- Docker & Docker Compose
- HuggingFace account(s) with API tokens
- Cloudflare Worker account

### 1. Clone and Configure

```bash
git clone https://github.com/xiaolajiaoyyds/regplatformm.git
cd regplatformm
cp .env.example .env
```

### 2. Environment Variables (.env)

```env
# Required
DATABASE_URL=postgres://user:password@localhost:5432/regplatform
JWT_SECRET=your-secret-key-at-least-32-chars

# Optional
PORT=8000
GIN_MODE=release
DEV_MODE=false
ADMIN_USERNAME=your_admin_username
SSO_SECRET=optional_sso_secret
REDIS_URL=redis://localhost:6379
JWT_EXPIRE_HOURS=72
CORS_ORIGINS=https://yourdomain.com,https://app.yourdomain.com
```

### 3. Local Development

```bash
# Backend
go mod tidy
go run cmd/server/main.go

# Frontend (separate terminal)
cd web
npm install
npm run dev
```

### 4. Production Docker Deployment

```bash
docker-compose -f docker-compose.prod.yml up -d
```

### 5. First-Time Setup

1. Visit `http://localhost:8000`
2. Register first account — **automatically becomes admin**
3. Go to admin panel → System Settings to configure services

## Project Structure

```
regplatformm/
├── cmd/
│   ├── server/              # Main server entrypoint
│   ├── openai-worker/       # OpenAI worker binary (→ HFNP)
│   ├── grok-worker/         # Grok worker binary (→ HFGS)
│   └── grpctest/            # gRPC-web debug tool
├── internal/
│   ├── config/              # Viper config loading
│   ├── model/               # GORM data models
│   ├── service/             # Business logic (TaskEngine, HFSpaceService)
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
├── HFTS/                    # Turnstile HF Space template
├── Dockerfile
├── docker-compose.yml
└── docker-compose.prod.yml
```

## API Reference

### Authentication Endpoints (Public)

```go
// Register — rate limited 3/min per IP
POST /api/auth/register
// Body: { "username": "user", "password": "Password1" }
// Password: min 8 chars, uppercase + lowercase + digit required
// First registered user auto-assigned admin (Role=100, IsAdmin=true)

// Login — rate limited 10/min per IP
POST /api/auth/login
// Body: { "username": "user", "password": "Password1" }
// Returns: JWT token

// SSO (optional, requires SSO_SECRET env var)
GET /api/auth/sso

// Dev login (DEV_MODE=true only)
GET /api/auth/dev-login
```

### Auth Token Passing

JWT token can be passed three ways:
```
Cookie: token=<jwt>
Header: X-Auth-Token: <jwt>
Header: Authorization: Bearer <jwt>
```

### User Endpoints (Authenticated)

```go
GET  /api/auth/me                    // Current user info
GET  /api/init                       // Batch initialization load
POST /api/tasks                      // Create registration task
POST /api/tasks/:id/start            // Start task
POST /api/tasks/:id/stop             // Stop task
GET  /api/tasks/current              // Current running task
GET  /api/results                    // Registration results list
GET  /api/results/:taskId/export     // Export results
GET  /api/credits/balance            // Credit balance
POST /api/credits/redeem             // Redeem credit code
GET  /api/proxies                    // List proxies
POST /api/proxies                    // Add proxy
PUT  /api/proxies/:id                // Update proxy
DELETE /api/proxies/:id              // Delete proxy
```

### Admin Endpoints (Admin Role Required)

```go
GET  /api/admin/users                // User list
POST /api/admin/credits/recharge     // Recharge user credits
GET  /api/admin/settings             // Get system settings
POST /api/admin/settings             // Update system settings
GET  /api/admin/hf/overview          // HF Space overview
POST /api/admin/hf/spaces/deploy     // Batch deploy spaces
POST /api/admin/hf/spaces/health     // Health check
POST /api/admin/hf/autoscale         // Elastic scaling
POST /api/admin/hf/sync-cf           // Sync CF Worker env vars
```

### Real-Time Endpoints

```go
GET /ws/logs/:taskId/stream   // SSE real-time logs
GET /ws/logs/:taskId          // WebSocket logs
```

## Configuration: Admin System Settings

Configure in the admin panel (persisted to DB):

| Key | Description |
|-----|-------------|
| `yydsmail_base_url` | YYDS Mail temp email API URL |
| `gptmail_api_key` | GPTMail API Key |
| `gptmail_base_url` | GPTMail service URL |
| `turnstile_solver_url` | Turnstile solver service URL |
| `cf_bypass_solver_url` | CF Bypass solver URL |
| `yescaptcha_key` | YesCaptcha API Key |
| `openai_reg_url` | OpenAI registration service URL |
| `grok_reg_url` | Grok registration service URL |
| `kiro_reg_url` | Kiro registration service URL |
| `default_proxy` | Default proxy address |
| `new_user_bonus` | Credits granted to new users |
| `max_threads_per_user` | Max concurrent threads per user |
| `max_target_per_task` | Max targets per task |

## Supported Platforms

| Platform | Method | HF Space | Release Tag |
|----------|--------|----------|-------------|
| OpenAI | Go HTTP (Sentinel PoW + PKCE OAuth + Auth0) | HFNP | `inference-runtime-latest` |
| Grok | Go HTTP (gRPC-web + Turnstile + Server Actions) | HFGS | `stream-worker-latest` |
| Kiro | Python (AWS Cognito + Camoufox) | HFKR | `browser-agent-latest` |
| Gemini | Python (Camoufox browser automation) | HFGM | `gemini-agent-latest` |
| Turnstile | Python (Camoufox + Patchright) | HFTS | `net-toolkit-latest` |

## HuggingFace Space Worker Setup

### Worker Binaries (via GitHub Releases)

Each platform's worker is compiled and released separately:

```
inference-runtime.zip  → deploy to HFNP spaces (OpenAI)
stream-worker.zip      → deploy to HFGS spaces (Grok)
browser-agent.zip      → deploy to HFKR spaces (Kiro)
gemini-agent.zip       → deploy to HFGM spaces (Gemini)
net-toolkit.zip        → deploy to HFTS spaces (Turnstile)
```

### Space Management via Admin Panel

1. Add HuggingFace tokens in admin → HF Token Management
2. Auto-discover existing spaces or batch deploy new ones
3. Health check and autoscale from admin panel
4. Sync CF Worker env vars after adding/removing spaces

## Cloudflare Worker Configuration

The CF Worker routes by URL path prefix to the appropriate HF Space pool:

```javascript
// cloudflare/ directory — key env vars to set in CF dashboard:
// OPENAI_SPACES = comma-separated HFNP space URLs
// GROK_SPACES   = comma-separated HFGS space URLs
// KIRO_SPACES   = comma-separated HFKR space URLs
// TS_SPACES     = comma-separated HFTS space URLs
```

Use admin panel → **Sync CF Worker** to auto-update these from the DB.

## Code Examples

### Creating a Registration Task (API Client)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type CreateTaskRequest struct {
    Platform string `json:"platform"` // "openai" | "grok" | "kiro" | "gemini"
    Count    int    `json:"count"`
    ProxyID  *int   `json:"proxy_id,omitempty"`
}

func createAndStartTask(baseURL, token string, req CreateTaskRequest) error {
    client := &http.Client{}

    // Create task
    body, _ := json.Marshal(req)
    httpReq, _ := http.NewRequest("POST", baseURL+"/api/tasks", bytes.NewBuffer(body))
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("X-Auth-Token", token)

    resp, err := client.Do(httpReq)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    taskID := result["id"]

    // Start task
    startReq, _ := http.NewRequest(
        "POST",
        fmt.Sprintf("%s/api/tasks/%v/start", baseURL, taskID),
        nil,
    )
    startReq.Header.Set("X-Auth-Token", token)
    _, err = client.Do(startReq)
    return err
}
```

### Consuming Real-Time Logs via SSE

```go
package main

import (
    "bufio"
    "fmt"
    "net/http"
)

func streamLogs(baseURL, token, taskID string) {
    req, _ := http.NewRequest(
        "GET",
        fmt.Sprintf("%s/ws/logs/%s/stream", baseURL, taskID),
        nil,
    )
    req.Header.Set("X-Auth-Token", token)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() {
        line := scanner.Text()
        if line != "" {
            fmt.Println(line) // SSE data lines
        }
    }
}
```

### Exporting Results

```go
func exportResults(baseURL, token, taskID string) ([]byte, error) {
    req, _ := http.NewRequest(
        "GET",
        fmt.Sprintf("%s/api/results/%s/export", baseURL, taskID),
        nil,
    )
    req.Header.Set("X-Auth-Token", token)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var buf bytes.Buffer
    buf.ReadFrom(resp.Body)
    return buf.Bytes(), nil
}
```

### Admin: Recharge User Credits

```go
func rechargeCredits(baseURL, adminToken, username string, amount int) error {
    payload := map[string]interface{}{
        "username": username,
        "amount":   amount,
    }
    body, _ := json.Marshal(payload)

    req, _ := http.NewRequest("POST", baseURL+"/api/admin/credits/recharge", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Auth-Token", adminToken)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}
```

### SSO Integration (External Systems)

```go
// SSO_SECRET must be set in .env
// External system redirects user to:
// GET /api/auth/sso?token=<hmac_signed_token>&username=<user>

// Generate SSO token (external system side):
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func generateSSOToken(username, secret string) string {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write([]byte(username))
    return hex.EncodeToString(mac.Sum(nil))
}
```

## CI/CD: GitHub Actions

Push to `main` triggers automatic builds:

```yaml
# .github/workflows/ builds these artifacts:
# - Docker image → GHCR (main service)
# - inference-runtime.zip → GitHub Release (OpenAI worker)
# - stream-worker.zip     → GitHub Release (Grok worker)
# - browser-agent.zip     → GitHub Release (Kiro worker)
# - gemini-agent.zip      → GitHub Release (Gemini worker)
# - net-toolkit.zip       → GitHub Release (Turnstile solver)
```

Workers are then deployed from Release artifacts to HF Spaces.

## Common Patterns

### Rate Limiting Awareness

```
Register: 3 requests/minute per IP
Login:    10 requests/minute per IP
```

Handle 429 responses with exponential backoff in clients.

### Admin Role Assignment

```
Strategy 1: First registered user → auto admin
Strategy 2: Set ADMIN_USERNAME=yourusername in .env
            → that username gets admin on registration
```

### Proxy Configuration

Proxies are managed per-user. Assign a proxy when creating tasks:
```json
POST /api/tasks
{
  "platform": "openai",
  "count": 10,
  "proxy_id": 3
}
```

Or set `default_proxy` in system settings as a global fallback.

### WebSocket vs SSE for Logs

- Use `GET /ws/logs/:taskId/stream` (SSE) for simple HTTP clients
- Use `GET /ws/logs/:taskId` (WebSocket) for bidirectional or browser use cases

## Troubleshooting

### Database Connection Issues

```bash
# Verify DATABASE_URL format
DATABASE_URL=postgres://username:password@host:5432/dbname?sslmode=disable

# Check PostgreSQL is running
docker-compose ps
```

### Workers Not Receiving Tasks

1. Check CF Worker is deployed and env vars are set
2. Admin panel → HF Overview → verify spaces show as healthy
3. Admin panel → Sync CF Worker to push latest space URLs
4. Check `openai_reg_url` / `grok_reg_url` etc. in system settings point to CF Worker

### JWT Issues

```bash
# Ensure JWT_SECRET is set and ≥32 chars in production
JWT_SECRET=your-very-long-secret-key-at-least-32-characters

# Check JWT_EXPIRE_HOURS (default 72)
JWT_EXPIRE_HOURS=72
```

### CORS Errors

```bash
# Set allowed origins explicitly in production
CORS_ORIGINS=https://yourdomain.com,https://app.yourdomain.com

# DEV_MODE=true allows all origins (development only)
DEV_MODE=true
```

### First User Not Getting Admin

```bash
# Option 1: Ensure you're first to register on fresh DB
# Option 2: Set ADMIN_USERNAME before registering
ADMIN_USERNAME=your_desired_admin_username
```

### HF Space Health Check Failing

- Verify HF tokens are valid and have write access
- Check space build logs in HuggingFace dashboard
- Ensure worker binaries match platform (HFNP=OpenAI, HFGS=Grok, etc.)
- Use admin panel → Health Check to get per-space status

### Dev Mode Login

```bash
# Enable only for local testing
DEV_MODE=true

# Then: GET /api/auth/dev-login
# Returns a JWT without password (development convenience)
```

## Deployment Checklist

- [ ] `DATABASE_URL` set with valid PostgreSQL connection
- [ ] `JWT_SECRET` set to 32+ character random string
- [ ] `CORS_ORIGINS` set to your actual frontend domain(s)
- [ ] HuggingFace tokens added in admin panel
- [ ] HF Spaces deployed for needed platforms
- [ ] Cloudflare Worker deployed from `cloudflare/` directory
- [ ] CF Worker synced from admin panel
- [ ] System settings configured (mail service, captcha keys, reg URLs)
- [ ] `GIN_MODE=release` for production
- [ ] `DEV_MODE=false` for production
```
