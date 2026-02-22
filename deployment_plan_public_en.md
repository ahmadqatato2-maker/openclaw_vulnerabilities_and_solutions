# OpenClaw — VPS Deployment Plan

## Table of Contents

- [Initial Requirements](#initial-requirements)
- [API Access Solution](#api-access-solution)
- [Architectural Solution](#architectural-solution)
- [Deployment Stages](#deployment-stages)
- [Important Findings](#important-findings)
- [Deferred Tasks](#deferred-tasks)

---

## Initial Requirements

- **VPS**: 2 CPU 3.0 GHz, 4 GB RAM, 60 GB NVMe
- **OS**: Ubuntu 24.04 LTS
- **AI providers**: OpenRouter (API aggregator — access to Claude, GPT, DeepSeek, Gemini, Llama and 300+ models via a single key)
- **Access**: CLI (SSH), Web UI (browser), Telegram bot (mobile access)
- **Remote access**: Tailscale (mesh VPN)
- **Multi-agent**: 5–10 parallel agents (scenarios: project team / development team)
- **Browser automation**: required from day one
- **Users**: 1 now, 1–2 in the future
- **Monitoring**: basic (security + minimal load tracking)
- **Backup**: GPG encryption, daily
- **Priority**: maximum security

---

## API Access Solution

### Choosing an AI API Provider

**Provider**: OpenRouter (openrouter.ai)

**Reasons for choosing**:
- Direct access to Anthropic/OpenAI may be geo-restricted
- Payment cards from sanctioned regions are not accepted directly
- OpenRouter is an API aggregator with native OpenClaw integration
- Access to 300+ models via a single API key
- Automatic fallback when a provider goes down
- No direct contract with the model provider — lower risk of geo-based ban

**Starting point**: free models with tool calling support (required for OpenClaw)

**Risks**:
- OpenRouter account ban: **low** (no geo-restrictions in ToS)
- Individual model refusals due to geolocation: **medium** (ToS section 5.7 — country is forwarded to providers)
- Future sanction escalation: **unknown**

**Mitigation**:
- If one model refuses — switch to another
- OpenRouter is an aggregator, not dependent on a single provider

**OpenClaw configuration**:

```bash
openclaw onboard --auth-choice apiKey --token-provider openrouter --token "$OPENROUTER_API_KEY" --install-daemon
```

**Recommended models**:
- **For orchestration**: Claude Sonnet 4.5
- **For routine tasks**: DeepSeek R1 (free)
- **For code/analysis**: GPT-4o — best price/quality balance
- **For drafts**: Gemini 2.0 Flash (free)

---

## Architectural Solution

```
┌──────────────────────────────────────────────────────────────┐
│                       Public Internet                        │
│  ┌────────────────┐          ┌──────────────────────────┐    │
│  │ Telegram Bot   │          │ OpenRouter API aggregator │   │
│  │ API            │          │                          │   │
│  └────────────────┘          └──────────────────────────┘    │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
           │                              │ HTTPS
           │                              │
┌──────────┼──────────────────────────────┼───────────────────┐
│          │     Tailscale Mesh VPN       │                   │
│  ┌───────▼──────┐              ┌────────▼─────────┐         │
│  │ Phone         │              │ Laptop / PC      │         │
│  │ (Telegram app)│              │                  │         │
│  └───────────────┘              └──────────────────┘         │
└──────────────────────────────────────────────────────────────┘
           │                              │
           │                              │ Tailscale VPN
           │                              │
┌──────────┼──────────────────────────────┼───────────────────┐
│          │           VPS               │                   │
│          │                              │                   │
│  ┌───────▼────────────────────────┬─────▼──────────┐        │
│  │ UFW Firewall                   │                │        │
│  │ ├─ SSH :22 (key-only)          │ Tailscale      │        │
│  │ └─ Tailscale daemon            │ daemon         │        │
│  └────────────────────────────────┴────────────────┘        │
│                       │                                      │
│  ┌────────────────────▼───────────────────────────┐         │
│  │          OpenClaw Stack                        │         │
│  │  ┌─────────────────────────────────────────┐   │         │
│  │  │ Gateway :18789 (localhost/tailscale)    │   │         │
│  │  └──┬──────────────────────────────────┬───┘   │         │
│  │     │                                  │       │         │
│  │  ┌──▼────────┐  ┌────────────┐  ┌─────▼─────┐ │         │
│  │  │ Web UI    │  │ Telegram   │  │ Headless  │ │         │
│  │  │ :18790    │  │ Bot channel│  │ Chromium  │ │         │
│  │  └───────────┘  └────────────┘  │ :18791    │ │         │
│  │                                  └───────────┘ │         │
│  │  ┌──────────────────────────────────────────┐ │         │
│  │  │ Docker Sandbox (mode=all)                │ │         │
│  │  └──────────────────────────────────────────┘ │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │          Security Layer                        │         │
│  │  - fail2ban (SSH brute-force protection)       │         │
│  │  - UFW deny-all-incoming                       │         │
│  │  - Auth Token (rotation 30d)                   │         │
│  │  - .env chmod 600 (API keys)                   │         │
│  │  - API key spending limit ($15-20/mo)          │         │
│  │  - API key rotation (30d)                      │         │
│  │  - npm version pinning (no @latest)            │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │          Monitoring                            │         │
│  │  - Cron script (5min) → Telegram Bot alerts    │         │
│  │  - logrotate, docker prune                     │         │
│  └────────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

1. **OpenClaw does NOT expose itself to the public internet**: gateway and Web UI listen only on localhost and Tailscale IP. No open ports except SSH.

2. **Telegram** — primary mobile access channel. The bot connects to the Telegram Bot API outbound (OpenClaw initiates via long polling); no inbound connections required.

3. **Tailscale** — primary channel for CLI and Web UI. End-to-end WireGuard encryption, zero trust.

4. **Docker sandbox mode "all"** — all agent operations run inside containers, isolated from the host system.

5. **OpenRouter** — single access point for AI models. Gateway makes HTTPS requests to OpenRouter API; OpenRouter routes to specific providers.

### Resource Assessment

**2 CPU, 4 GB** is sufficient to start:
- 1–3 simultaneous agents
- Basic browser automation
- Cloud LLMs (OpenRouter)

**Signals to scale up** (4 CPU, 4–6 GB):
- RAM > 85% consistently (5+ minutes)
- CPU load average > 80% (1 min avg)
- 5–10 parallel agents with browser automation

---

## Deployment Stages

### Stage 0. Preparation (before VPS)

**What to do:**
- Order VPS with Ubuntu 24.04 LTS
- Create a Tailscale account (free Personal plan)
- Create a Telegram bot via @BotFather, obtain bot token
- Configure DNS: A record for domain → VPS IP
- Register on openrouter.ai, obtain API key

**Result:** VPS accessible via SSH, domain resolves, Tailscale and Telegram bot ready, OpenRouter API key obtained.

### Stage 1. VPS Hardening (OS security)

**What to do:**
1. Update system: `apt update && apt upgrade -y`
2. Create a dedicated non-root user
3. Configure SSH:
   - Key-based authentication only (disable password login)
   - Disable root login
   - Change SSH port (optional, reduces log noise)
4. Install and configure UFW:
   - `default deny incoming`, `default allow outgoing`
   - Allow SSH only
5. Install fail2ban (SSH brute-force protection)
6. Enable automatic security updates: `unattended-upgrades`
7. Configure logrotate for system logs

**Result:** VPS accepts only SSH with key auth. Everything else is closed.

### Stage 2. Docker

**What to do:**
1. Install Docker Engine (from official Docker repository, not `docker.io` from Ubuntu)
2. Add the working user to the `docker` group
3. Configure Docker daemon:
   - Disable userland-proxy
   - Limit container log size (max-size, max-file)
   - Do not bind container ports to `0.0.0.0`

**Result:** Docker ready, securely configured.

### Stage 3. Node.js

**What to do:**
1. Install Node.js 22 LTS via NodeSource
2. Verify version: `node -v`, `npm -v`

**Result:** Runtime ready.

### Stage 4. Tailscale

**What to do:**
1. Install Tailscale on VPS
2. Authorize VPS in the Tailscale network
3. Get the Tailscale IP of the VPS
4. Enable MagicDNS (optional)
5. Configure ACL in Tailscale admin console:
   - Allow access to ports 18789, 18790 from your devices only
6. Install Tailscale on laptop/phone, authorize
7. Test connectivity: ping VPS by Tailscale IP

**Result:** Private network between your devices and VPS. OpenClaw will be accessible only within it.

### Stage 5. Installing OpenClaw

**What to do:**

1. `npm install -g openclaw@<VERSION>` (pinned version — not `@latest`)
2. Run onboarding with OpenRouter:
   ```bash
   openclaw onboard --auth-choice apiKey --token-provider openrouter --token "$OPENROUTER_API_KEY" --install-daemon
   ```
   - Skip messenger setup (Telegram will be connected later)
3. Verify that the systemd user unit is created and gateway starts
4. Test: send a request via CLI to a free model

**Result:** OpenClaw installed, daemon running, connected to OpenRouter.

### Stage 6. Secure OpenClaw Configuration

**What to do:**

1. **Bind to Tailscale IP**: in `~/.openclaw/openclaw.json` set `host: "<TAILSCALE_IP>"` (not `0.0.0.0`, not public IP)
2. **Auth token**: generate with `openssl rand -hex 32`, set in config
3. **OpenRouter API key**:
   - Create `~/.openclaw/.env` with: `OPENROUTER_API_KEY=sk-or-v1-...`
   - `chmod 600 ~/.openclaw/.env`
   - In `openclaw.json` `env` section — use reference `${OPENROUTER_API_KEY}` (supported)
   - In `auth-profiles.json` — **actual key** (variable substitution NOT supported)
   - Key goes in `credentials.openrouter`, NOT in `skills.entries`
   - **Do not store key in `.bak` files** — delete config backups that contain keys
4. **File permissions**:
   - `chmod 700 ~/.openclaw/`
   - `chmod 700 ~/.openclaw/agents/main/agent/`
   - `chmod 600 ~/.openclaw/openclaw.json`
   - `chmod 600 ~/.openclaw/.env`
   - `chmod 600 ~/.openclaw/agents/main/agent/auth-profiles.json`
5. **Docker sandbox mode "all"**: all operations run inside containers
6. **Access policy**: `denyByDefault: true`, whitelist specific tools
7. **Deny `group:runtime`** (exec, bash) by default
8. **Browser config**: `headless: true`, port bound to localhost
9. **Spending limit on OpenRouter API key**:
   - In key settings on openrouter.ai → set Credit limit
   - Caps financial losses in case of key leak
10. **OpenRouter API key rotation** (every 30 days or when leak suspected):
    1. Create new key with the same spending limit
    2. Update `~/.openclaw/.env`
    3. Update `~/.openclaw/agents/main/agent/auth-profiles.json`
    4. `systemctl --user restart openclaw-gateway`
    5. Verify agent works
    6. Delete old key
11. **Gateway auth token rotation** (every 30 days):
    1. `openssl rand -hex 32` — generate new token
    2. Update in gateway config
    3. `systemctl --user restart openclaw-gateway`
12. **Supply chain attack protection**:
    - Pin OpenClaw version at install and update time (no `@latest`)
    - `npm audit --global` before each update
    - VPS snapshot before updates
    - Docker images: pin tags (not `latest`), scan with `docker scout cves`
    - Subscribe to OpenClaw GitHub releases — update only after reviewing changelog

**Result:** OpenClaw isolated, protected with auth token, key stored in .env, sandbox enabled. API key has spending limit, key and token rotation configured, supply chain under control.

### Stage 7. Telegram Bot (chat with agent)

**What to do:**
1. Create a **separate** Telegram bot via @BotFather (NOT the one used for alerts)
2. Configure Telegram channel in OpenClaw: set `botToken` in `channels.telegram`
3. Set `allowFrom` — your Telegram ID only (format: `tg:NUMBER`)
4. Set `dmPolicy: "pairing"` — bot responds only in direct messages
5. Test: send a message to the bot → receive a response from the agent

**Two Telegram bots:**
- **Alert bot** — one-way notifications from cron scripts. Works independently of OpenClaw (curl from bash)
- **Chat bot** — two-way chat with the AI agent via OpenClaw channel. Depends on the running gateway

The separation ensures: if OpenClaw goes down, alerts keep arriving.

**Result:** Mobile access to OpenClaw via Telegram.

### Stage 8. Browser Automation

**What to do:**

1. Install Chromium via Playwright (`npx playwright install chromium`) in user space
2. Install system dependencies (`sudo apt install` — libatk, libcups, libgbm, libcairo, libpango, libasound, etc.)
3. Configure in `openclaw.json`:
   ```json
   {
     "browser": {
       "executablePath": "<PATH_TO_PLAYWRIGHT_CHROMIUM>",
       "headless": true,
       "noSandbox": true,
       "defaultProfile": "openclaw"
     }
   }
   ```
4. Test: `openclaw browser open https://example.com` + `openclaw browser screenshot`

⚠️ **Important**: `noSandbox: true` is required for headless VPS without a graphical environment.

**Result:** Browser automation working.

### Stage 9. Multi-agent

**What to do:**

1. Switch to a paid model with tool calling support
2. Configure subagent model (`agents.defaults.subagents.model`)
3. Configure fallback models
4. Tool profile: `coding`, `tools.deny: ["group:runtime"]`
5. Remove `sessions_send` from deny list (required for multi-agent)
6. Test: spawn subagents in parallel

**Result:** Multi-agent operation configured.

### Stage 10. Monitoring

**What to do:**

1. Create monitoring script (checks: RAM, CPU, swap, disk, gateway, Docker, sandbox, permissions, ports, key leak)
2. Configure logrotate for monitoring logs
3. Schedule `docker system prune` (weekly)
4. Alerts via Telegram Bot API on WARN/CRITICAL
5. Cron `*/5 * * * *` for monitoring
6. Monthly cron reminder for key rotation

**Result:** Monitoring with active alerting.

### Stage 11. Backup

**What to do:**

1. GPG key pair (RSA 4096): private key stored locally, public key on VPS
2. Backup script: `tar` → `gpg --encrypt` → `~/backups/`, 7-day rotation, Telegram notification
3. Backup: `~/.openclaw/` (config, keys, agents, workspace) + `dpkg --get-selections` + `docker images`
4. Daily cron
5. Pull script to download backups to local machine via scp
6. VPS snapshots before each OpenClaw update

**Result:** Daily automated encrypted backup.

### Stage 12. Proxy for Model Unblocking

**Goal:** configure an outbound proxy for the OpenClaw gateway so that requests to OpenRouter originate from a non-blocked IP. Unblocks Anthropic and OpenAI models when geo-restrictions apply.

**What to do:**

1. Cloudflare WARP (SOCKS5 `127.0.0.1:40000`) — free, quick to set up
2. Privoxy (HTTP proxy `127.0.0.1:8118`) — wrapper over WARP SOCKS5, because Node.js undici does not support `socks5h://` in `HTTPS_PROXY`
3. Proxy chain: OpenClaw → Privoxy (HTTP) → WARP (SOCKS5) → internet
4. `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` in systemd unit `openclaw-gateway.service`:
   ```ini
   Environment=HTTPS_PROXY=http://127.0.0.1:8118
   Environment=HTTP_PROXY=http://127.0.0.1:8118
   Environment=NO_PROXY=localhost,127.0.0.1,::1
   ```
5. Privoxy config: `forward-socks5t / 127.0.0.1:40000 .`
6. Switch models, run tests

⚠️ **Important**: OpenClaw (Node.js undici) does not support `socks5h://` in `HTTPS_PROXY` environment variables. Connecting directly via SOCKS5 causes a gateway crash (`Invalid URL protocol`). An intermediate HTTP proxy (Privoxy) is required.

**Fallback strategy**: if WARP becomes unstable — Google and open-source models in fallbacks work without a proxy. Long-term — SSH SOCKS5 tunnel through a VPS in another region.

**Result:** Access to all paid models via OpenRouter.

---

## Stage Order

```
Stage 0   Preparation
Stage 1   VPS Hardening
Stage 2   Docker
Stage 3   Node.js
Stage 4   Tailscale
Stage 5   Install OpenClaw
Stage 6   Secure Configuration
    ↓
Stage 10  Monitoring + Alerts
    ↓
Stage 11  Backup
    ↓
Stage 7   Telegram Bot (chat with agent)
    ↓
Stage 8   Browser Automation
    ↓
Stage 9   Multi-agent
    ↓
Stage 12  Proxy for models (if needed)
```

**Rationale for order**: security and observability first (monitoring, backup), then functional features (Telegram, browser, multi-agent). Without monitoring, attacks go undetected; without backup, recovery is impossible.

---

## Important Findings

### Tool Calling Requirement

**OpenClaw requires models to support tool/function calling.** Models without this capability silently fail — the agent exits without output and without any error in the TUI.

Before switching models, verify tool support via a curl test with the `"tools"` parameter.

### Provider Geo-blocking via OpenRouter

When hosting VPS in regions with sanction restrictions:
- Anthropic and OpenAI may return 403 through OpenRouter
- Google (Gemini) and open-source models work without restrictions
- A proxy with an IP from an unrestricted region fully removes the block
- The block is tied to the IP of the incoming request, not to the OpenRouter account

### Side Effect of `openclaw config set`

On every call to `openclaw config set`:
- The `${OPENROUTER_API_KEY}` variable in the `env` section is replaced with the actual key value
- File permissions on `openclaw.json` are reset from `600` to `664`

Solution: use `jq` to modify the config instead of `openclaw config set`, then restore with `chmod 600`.

---

## Deferred Tasks

1. **MCP integrations research** — exploring available integrations via Model Context Protocol
2. **Ollama on a separate server** — local models for privacy and zero API costs (8+ GB RAM required)
3. **Extended monitoring** (Uptime Kuma, etc.) — dashboards, graphs, history
4. **Multi-user access** — `allowedUsers`, separate auth tokens, roles
5. **SSL certificate and nginx** — if a public Web UI is needed in the future
6. **User separation** — run gateway under a dedicated system user, API key isolated from SSH user
7. **Wrapper around `openclaw config set`** — automatic restoration of key reference and file permissions
