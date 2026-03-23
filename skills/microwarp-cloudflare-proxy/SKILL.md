```markdown
---
name: microwarp-cloudflare-proxy
description: Ultra-lightweight Cloudflare WARP SOCKS5 proxy in Docker using kernel-level WireGuard and microsocks, consuming under 800KB RAM
triggers:
  - set up cloudflare warp proxy docker
  - lightweight warp socks5 container
  - microwarp setup and configuration
  - replace caomingjun warp with microwarp
  - cloudflare warp wireguard docker low memory
  - warp proxy with authentication docker
  - bypass cloudflare warp regional blocks
  - warp socks5 proxy under 1mb ram
---

# MicroWARP Cloudflare Proxy

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

MicroWARP is an ultra-lightweight Cloudflare WARP SOCKS5 proxy that runs inside Docker using Linux kernel-level WireGuard (`wg0`) and a pure-C `microsocks` server. It uses **~800KB RAM** and a 9MB image — compared to 150MB RAM and 201MB image for `caomingjun/warp`.

## Architecture

- **WireGuard kernel module** (`wg0`): handles WARP tunnel at kernel level, near-zero CPU
- **microsocks**: pure C SOCKS5 server, no Go/Rust overhead
- **Auto-registration**: fetches free WARP credentials from Cloudflare API on first run
- **Persistent volume**: saves WireGuard config to avoid re-registration on restart

## Installation

### Docker Compose (recommended)

```yaml
# docker-compose.yml
version: '3.8'

services:
  microwarp:
    image: ghcr.io/ccbkkb/microwarp:latest
    container_name: microwarp
    restart: always
    ports:
      - "1080:1080"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - warp-data:/etc/wireguard

volumes:
  warp-data:
```

```bash
docker compose up -d
docker compose logs -f   # watch first-run registration
```

### Docker Run (one-liner)

```bash
docker run -d \
  --name microwarp \
  --restart always \
  -p 1080:1080 \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  -v warp-data:/etc/wireguard \
  ghcr.io/ccbkkb/microwarp:latest
```

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `BIND_ADDR` | `0.0.0.0` | SOCKS5 listen address |
| `BIND_PORT` | `1080` | SOCKS5 listen port |
| `SOCKS_USER` | *(empty)* | Auth username (empty = no auth) |
| `SOCKS_PASS` | *(empty)* | Auth password |
| `ENDPOINT_IP` | *(auto)* | Custom WARP endpoint IP:port (bypass DPI) |

## Configuration Examples

### With SOCKS5 Authentication

```yaml
version: '3.8'

services:
  microwarp:
    image: ghcr.io/ccbkkb/microwarp:latest
    container_name: microwarp
    restart: always
    ports:
      - "1080:1080"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    environment:
      - SOCKS_USER=${WARP_USER}
      - SOCKS_PASS=${WARP_PASS}
    volumes:
      - warp-data:/etc/wireguard

volumes:
  warp-data:
```

### Custom Port + Auth

```yaml
environment:
  - BIND_ADDR=127.0.0.1   # localhost only
  - BIND_PORT=9050
  - SOCKS_USER=${WARP_USER}
  - SOCKS_PASS=${WARP_PASS}
```

### Bypass Regional Blocks (HK/US DPI)

If Cloudflare's `reserved` bytes verification blocks your region, inject a clean endpoint IP:

```yaml
environment:
  - ENDPOINT_IP=162.159.193.10:2408
```

Scan for clean endpoints using tools like `warp-endpoint-scanner` and inject the best one.

### Localhost-Only Binding (secure setup)

```yaml
ports:
  - "127.0.0.1:1080:1080"   # only accessible from host
environment:
  - BIND_ADDR=0.0.0.0        # container listens on all interfaces
```

## Usage Patterns

### Test the proxy

```bash
# Basic connectivity test
curl --socks5 127.0.0.1:1080 https://cloudflare.com/cdn-cgi/trace

# With authentication
curl --socks5-hostname 127.0.0.1:1080 \
  --proxy-user "${WARP_USER}:${WARP_PASS}" \
  https://cloudflare.com/cdn-cgi/trace

# Check your IP via WARP
curl --socks5 127.0.0.1:1080 https://ipinfo.io/json
```

### Use with Python (requests)

```python
import requests

proxies = {
    "http": "socks5://127.0.0.1:1080",
    "https": "socks5://127.0.0.1:1080",
}

# With auth
proxies_auth = {
    "http": "socks5://user:pass@127.0.0.1:1080",
    "https": "socks5://user:pass@127.0.0.1:1080",
}

resp = requests.get("https://ipinfo.io/json", proxies=proxies)
print(resp.json())
```

### Use with Node.js

```javascript
import { SocksProxyAgent } from 'socks-proxy-agent';
import fetch from 'node-fetch';

const agent = new SocksProxyAgent('socks5://127.0.0.1:1080');
// With auth: 'socks5://user:pass@127.0.0.1:1080'

const res = await fetch('https://ipinfo.io/json', { agent });
console.log(await res.json());
```

### Convert to HTTP Proxy with gost

MicroWARP provides SOCKS5 only. Chain with `gost` for HTTP:

```bash
# Install gost
curl -fsSL https://github.com/go-gost/gost/releases/latest/download/gost_linux_amd64.tar.gz | tar xz

# Chain: HTTP :8081 → SOCKS5 microwarp :1080
# IMPORTANT: use socks5:// NOT socks5h:// to resolve DNS locally
nohup ./gost \
  -F "socks5://${WARP_USER}:${WARP_PASS}@127.0.0.1:1080" \
  -L "http://:8081" \
  > /dev/null 2>&1 &
```

> ⚠️ Always use `socks5://` not `socks5h://` — `socks5h` delegates DNS to the proxy, which can deadlock during WireGuard UDP handshake cold start causing `503` errors.

### Multi-instance setup (different ports)

```yaml
version: '3.8'

services:
  warp-1:
    image: ghcr.io/ccbkkb/microwarp:latest
    ports: ["1080:1080"]
    cap_add: [NET_ADMIN, SYS_MODULE]
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - warp-data-1:/etc/wireguard

  warp-2:
    image: ghcr.io/ccbkkb/microwarp:latest
    ports: ["1081:1080"]
    cap_add: [NET_ADMIN, SYS_MODULE]
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - warp-data-2:/etc/wireguard

volumes:
  warp-data-1:
  warp-data-2:
```

## Common Integration Patterns

### With Xray/V2Ray outbound

```json
{
  "outbounds": [
    {
      "tag": "warp",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "127.0.0.1",
            "port": 1080,
            "users": [
              { "user": "admin", "pass": "yourpass" }
            ]
          }
        ]
      }
    }
  ]
}
```

### With Telegram (via proxychains)

```bash
# /etc/proxychains4.conf
[ProxyList]
socks5 127.0.0.1 1080
```

```bash
proxychains4 telegram-desktop
```

## Monitoring & Debugging

```bash
# Check container resource usage
docker stats microwarp

# Expected output:
# CONTAINER ID   NAME       CPU %   MEM USAGE / LIMIT     MEM %
# 2fa58f84c517   microwarp  0.25%   800KiB / 967.4MiB     0.08%

# View logs (first-run registration)
docker logs microwarp

# Inspect WireGuard interface inside container
docker exec microwarp wg show

# Check if SOCKS5 port is listening
docker exec microwarp ss -tlnp | grep 1080

# Test from inside container
docker exec microwarp wget -qO- --timeout=10 \
  -e "use_proxy=yes" \
  -e "https_proxy=socks5://127.0.0.1:1080" \
  https://cloudflare.com/cdn-cgi/trace
```

## Troubleshooting

### Container fails to start — missing kernel module

```
Error: WireGuard kernel module not found
```

**Fix**: Ensure host kernel has WireGuard support (Linux 5.6+ has it built-in):

```bash
# Check host kernel
uname -r
# Should be 5.6+

# Verify wireguard module
modprobe wireguard && echo "OK"
```

Add `SYS_MODULE` capability and the sysctl:

```yaml
cap_add:
  - NET_ADMIN
  - SYS_MODULE
sysctls:
  - net.ipv4.conf.all.src_valid_mark=1
```

### Registration fails / cannot connect to Cloudflare

**Fix**: Inject a known-good WARP endpoint:

```yaml
environment:
  - ENDPOINT_IP=162.159.193.10:2408
```

Popular clean endpoints:
- `162.159.193.10:2408`
- `162.159.195.1:2408`
- `188.114.96.1:2408`

### SOCKS5 returns 503 intermittently

This is the WireGuard UDP handshake cold-start issue. **Fix**: Use `socks5://` not `socks5h://` in your client. The difference:

- `socks5://` — client resolves DNS, sends IP to proxy ✅
- `socks5h://` — proxy resolves DNS, hits WireGuard during cold start ❌

### Volume permission issues on restart

```bash
# Check volume contents
docker run --rm -v warp-data:/data alpine ls -la /data

# If empty after accidental volume deletion, container will re-register automatically
# Just restart and wait ~10 seconds
docker compose restart microwarp
```

### High CPU on arm64 (Oracle Cloud)

Ensure you're pulling the multi-arch image correctly:

```bash
# Force arm64 pull
docker pull --platform linux/arm64 ghcr.io/ccbkkb/microwarp:latest
```

## Required Docker Capabilities

| Capability | Reason |
|---|---|
| `NET_ADMIN` | Create/configure `wg0` WireGuard interface |
| `SYS_MODULE` | Load WireGuard kernel module if not already loaded |
| `net.ipv4.conf.all.src_valid_mark=1` | Required for WireGuard routing marks |

## Architecture Notes for AI Agents

- The image is ~9MB (Alpine-based)
- No `warp-cli`, no Rust daemon — pure kernel WireGuard + C microsocks
- Config persisted at `/etc/wireguard/wg0.conf` inside the volume
- First-run script auto-calls Cloudflare WARP API to register and get keypair
- SOCKS5 server binds inside container; map host port via Docker `-p`
- Multi-arch: `linux/amd64` and `linux/arm64` supported natively
```
