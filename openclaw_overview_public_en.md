# OpenClaw — Platform Overview and Deployment Guide

## Table of Contents

- [What is OpenClaw](#what-is-openclaw)
- [Key Features](#key-features)
  - [Browser Automation](#browser-automation)
  - [Ways to Interact with OpenClaw](#ways-to-interact-with-openclaw)
- [Current Security Status](#current-security-status)
  - [Critical Vulnerabilities](#critical-vulnerabilities)
  - [Security Assessment](#security-assessment)
- [System Requirements](#system-requirements)
- [Choosing a VPS for Deployment](#choosing-a-vps-for-deployment)
  - [Scaling Strategy](#scaling-strategy)
- [Secure VPS Installation](#secure-vps-installation)
  - [Stage 1. VPS Preparation](#stage-1-vps-preparation)
  - [Stage 2. Installing OpenClaw](#stage-2-installing-openclaw)
  - [Stage 3. Secure Configuration](#stage-3-secure-configuration)
  - [Stage 4. Remote Access](#stage-4-remote-access)
  - [Stage 5. Post-Installation](#stage-5-post-installation)
- [Architecture and Access](#architecture-and-access)
  - [Access Diagram](#access-diagram)
  - [Communication Protocol](#communication-protocol)
- [Service Management](#service-management)
  - [Daemon Installation](#daemon-installation)
  - [Management Commands](#management-commands)
- [Pricing Model](#pricing-model)
  - [Component Costs](#component-costs)
  - [API Cost Estimates](#api-cost-estimates)
- [Technical Billing and API Workflow](#technical-billing-and-api-workflow)
  - [Interaction Flow](#interaction-flow)
  - [API Key Storage](#api-key-storage)
  - [How an LLM Request Works](#how-an-llm-request-works)
  - [Provider-Side Billing](#provider-side-billing)
  - [Cost Control](#cost-control)
  - [API Key Leak Protection](#api-key-leak-protection)
  - [Local Models as an Alternative](#local-models-as-an-alternative)
- [Component Distribution](#component-distribution)
- [Monitoring and Maintenance](#monitoring-and-maintenance)
  - [A. Continuous Monitoring](#a-continuous-monitoring)
  - [B. Event-Driven Monitoring](#b-event-driven-monitoring)
  - [C. Trend Monitoring](#c-trend-monitoring)
  - [D. Monitoring Tools](#d-monitoring-tools)
- [Important Operational Considerations](#important-operational-considerations)
  - [Backup and Recovery](#backup-and-recovery)
  - [Updates](#updates)
  - [Dependency on External Services](#dependency-on-external-services)
  - [Project Sustainability](#project-sustainability)
  - [Multiple Instances](#multiple-instances)
  - [Messenger Sessions](#messenger-sessions)
  - [API Key Leak](#api-key-leak)
  - [Legal Considerations](#legal-considerations)
- [What Not To Do](#what-not-to-do)

---

## What is OpenClaw

OpenClaw (formerly ClawdBot, MoltBot) is an open-source AI assistant (MIT license) that runs locally and can:

- Connect to messengers: Telegram, Discord, Slack, WhatsApp, Signal, iMessage, Google Chat, Microsoft Teams, and others
- Execute shell commands on the host system
- Manage files and directories
- **Automate browser interactions** (clicks, forms, screenshots, web scraping)
- Integrate with 100+ services via Model Context Protocol (MCP)
- Work with AI models: Claude (Anthropic), GPT (OpenAI), local models via Ollama

OpenClaw uses a **"Fat Gateway"** architecture — a long-running daemon on your hardware acts as the central hub, while the LLM provides stateless inference.

**Project**: GitHub — 60,000+ stars, actively developed as of February 2026.

---

## Key Features

### Browser Automation

OpenClaw has built-in web page interaction capabilities through a **dedicated isolated browser** — an isolated Chromium instance controlled via the Chrome DevTools Protocol (CDP).

#### What OpenClaw Can Do in the Browser

| Action | Description |
|---|---|
| **Navigation** | Open URLs, follow links, navigate forward/back in history |
| **Clicks** | Click buttons, links, any interactive elements |
| **Text input** | Fill forms, type data into fields |
| **Scrolling** | Scroll the page up/down |
| **Drag & drop** | Drag and drop elements on the page |
| **Screenshots** | Take PNG screenshots of the page or a specific element |
| **PDF export** | Save the page as a PDF |
| **Tab management** | Open, close, switch between tabs |
| **Content reading** | Extract text, element attributes, DOM structure |

#### Snapshot System — OpenClaw's "Vision"

OpenClaw uses a **snapshot system** to understand web pages:

1. **Automatic element numbering** — when a page opens, the system analyzes the DOM and assigns numbers to interactive elements:
   ```
   [1] Button "Log In"
   [2] Field "Email"
   [3] Field "Password"
   [4] Link "Forgot password?"
   ```

2. **Two snapshot modes**:
   - **AI Snapshot** — for complex scenarios where the AI decides what to interact with
   - **Role Snapshot** — for specific tasks (forms, navigation)

3. **Screenshots + Vision** — OpenClaw can take a screenshot and send it to a vision model (Claude Sonnet, GPT-4 Vision) for visual analysis

**Important**: OpenClaw does not "see" pixels like a human — it analyzes the DOM structure and automatically marks up elements. This is faster and more reliable than visual recognition.

#### Browser Module Architecture

```
OpenClaw Gateway
    ↓
Browser Control Service (HTTP API)
    ↓
Chrome DevTools Protocol (CDP)
    ↓
Isolated Chromium (openclaw profile)
```

#### Operating Modes

| Mode | Description | When to use |
|---|---|---|
| `openclaw` | Isolated browser managed by OpenClaw | Default, for automation |
| `chrome` | Extension in the system browser | When you need bookmarks, saved passwords |

#### Configuration

In `~/.openclaw/openclaw.json`:

```json
{
  "browser": {
    "profile": "openclaw",
    "headless": true,
    "port": 18791
  }
}
```

On a VPS, `headless: true` is typically used — the browser runs without a graphical interface.

#### Practical Use Cases

**Web scraping:**
```
Task: "Collect prices for products from site X"
OpenClaw:
1. Opens the catalog page
2. Takes a snapshot (numbers elements containing prices)
3. Extracts price text
4. Navigates to the next page
5. Returns structured data
```

**Form automation:**
```
Task: "Register an account on service Y"
OpenClaw:
1. Opens the registration page
2. Fills in email, password, name
3. Passes validation (unless CAPTCHA)
4. Clicks "Register"
```

**Change monitoring:**
```
Task: "Check every hour whether a news item has appeared on site Z"
OpenClaw:
1. Opens the site
2. Looks for a heading with specific text
3. If found — sends a notification to Telegram
```

#### Playwright Integration

OpenClaw supports **Playwright MCP** (Model Context Protocol) for advanced browser automation:
- Working with authenticated flows (login, OAuth)
- Managing dynamic UI and client-side state
- Complex interaction scenarios

#### Limitations

| What it **cannot** do | Why | Workaround |
|---|---|---|
| Solve CAPTCHA | Anti-bot protection | Services like 2captcha |
| Work with Canvas/WebGL out of the box | Pixel graphics, no DOM | Screenshot + vision model |
| Bypass advanced anti-bot (Cloudflare Turnstile) | Automation detection | Additional tools |

### Ways to Interact with OpenClaw

OpenClaw supports many interaction methods — from the command line to messengers. **Telegram is not required**; you can work without it and without a SIM card.

#### 1. CLI (Command Line Interface)

**Direct work via terminal:**

```bash
openclaw chat "Show the contents of the workspace folder"
openclaw chat
> Hello, what can you do?
> [agent responds]
```

#### 2. Web UI

Access via `http://localhost:18790` or `http://<tailscale-ip>:18790`

#### 3. Native Applications

| Platform | Availability | Connection method |
|---|---|---|
| macOS/iOS | Available | Connect to gateway via Tailscale |
| Android | Via messengers | Discord, Slack, Telegram, etc. |
| Windows | CLI / Web UI | Via browser or terminal |
| Linux | CLI / Web UI | Via terminal |

#### 4. API Calls

```bash
curl -X POST http://<tailscale-ip>:18789/api/chat \
  -H "Authorization: Bearer <your-token>" \
  -d '{"message": "Hello, OpenClaw"}'
```

#### 5. Messengers

| Messenger | Requires SIM? | Works on mobile? | Free? | Setup complexity |
|---|---|---|---|---|
| **Telegram** | No* | Excellent | Free | Very easy |
| **Discord** | No | Excellent | Free | Easy |
| **Slack** | No | Excellent | Free plan** | Easy |
| **WhatsApp** | Yes | Yes | Free | Medium |
| **Signal** | Yes | Yes | Free | Medium |
| **Matrix** | No | Yes | Free | Medium |
| **iMessage** | No*** | iOS only | Free | macOS only |

\* Creating a Telegram bot via @BotFather requires a personal Telegram account (which may require a phone number), but **the bot itself is not tied to a phone number**.

\*\* Slack free plan has message history limits (90 days, 10K messages).

\*\*\* iMessage requires an Apple ID but not a SIM card.

**Can you work without messengers?** Yes, fully. OpenClaw works as a standalone service via CLI and Web UI.

---

## Current Security Status

### Critical Vulnerabilities

Based on 2026 security audits:

- **CVE-2026-25253** (CVSS 8.8) — remote code execution via WebSocket hijacking on exposed instances
- **135,000+ installations** found open to the internet without protection (82 countries)
- **12% of skills on ClawHub** (the extension marketplace) contain malicious code
- **40% of public configurations** on GitHub contain API keys in plaintext
- **Issue #9627** — `openclaw update` / `doctor` / `configure` commands expand `${VAR}` environment variables and write real values into the config file (patched in version ≥ 2026.1.29)

### Security Assessment

- **Score: 2 out of 100** — 84% of data extraction attempts and 91% of prompt injection attacks succeed
- Campaign "ClawHavoc" — reverse shells embedded in skills disguised as legitimate tools
- Root cause: self-hosted architecture — each user is responsible for their own security

---

## System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04+ | Ubuntu 22.04 / 24.04 LTS |
| Node.js | v22.0+ | v22.x LTS |
| RAM | 2 GB | 4 GB+ |
| CPU | 1 core | 2+ cores |
| Disk | 10 GB | 40 GB+ (Docker + images + logs) |
| Docker | Required for sandbox | Latest stable version |

---

## Choosing a VPS for Deployment

Recommended VPS specifications:

| Option | CPU | RAM | Disk | Assessment |
|---|---|---|---|---|
| **Minimum** | 2 cores | **4 GB** | 60 GB NVMe | Minimum viable configuration |
| **Optimal** | 4 cores | **4 GB** | 80 GB NVMe | CPU and disk headroom |
| **With reserve** | 4 cores | 6 GB | 100 GB NVMe | For additional services |

**Why at least 4 GB RAM:**
- 1–2 GB RAM — insufficient for Docker sandbox in "all" mode + Node.js + OpenClaw simultaneously
- 2 GB RAM — works at the limit, Docker will use swap

### Scaling Strategy

Start with the minimum configuration, monitor load, and scale up when needed.

**Signals to scale:**
- RAM > 85% consistently (5+ minutes)
- CPU load average > 80% (1 min avg)
- Disk > 80% full

---

## Secure VPS Installation

### Stage 1. VPS Preparation

**1. Update the system:**

```bash
sudo apt update && sudo apt upgrade -y
```

**2. Create a dedicated non-root user:**

```bash
sudo useradd -r -m -s /bin/bash openclaw-user
sudo passwd openclaw-user
```

**3. Configure firewall (UFW):**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

Do **not** expose OpenClaw ports to the internet.

**4. Install Docker:**

```bash
sudo apt install -y docker.io
sudo usermod -aG docker openclaw-user
```

**5. Install Node.js 22:**

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### Stage 2. Installing OpenClaw

```bash
sudo su - openclaw-user
npm install -g openclaw@latest
```

### Stage 3. Secure Configuration

**1. Bind to localhost:**

Set `host: 127.0.0.1` in the config (not `0.0.0.0`).

**2. Authentication — token:**

```bash
openssl rand -hex 32
```

Set the token in the config and rotate it every 30 days.

**3. File permissions:**

```bash
chmod 700 ~/.openclaw/
chmod 600 ~/.openclaw/config.yaml
```

**4. API keys via environment variables:**

```bash
# ~/.openclaw/.env (chmod 600)
ANTHROPIC_API_KEY=sk-ant-api03-...
OPENAI_API_KEY=sk-...
```

Use references in the config:

```json
{
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "${ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

**Critical**:
- Place API keys in the root `models.providers` section, **not** in `skills.entries`
- Do not hardcode keys directly in the config (risk of Issue #9627)
- Rotate keys every 3–6 months
- After every update, verify: `grep -r "sk-ant\|sk-" ~/.openclaw/openclaw.json`

**5. Docker sandbox — "all" mode:**

Run all operations (including core ones) inside containers.

**6. Access policy:**

```yaml
denyByDefault: true
```

Whitelist specific tools. Deny the `group:runtime` group (exec, bash).

### Stage 4. Remote Access

The OpenClaw gateway listens on `127.0.0.1:18789` (localhost only) by default. Two options for remote access:

#### Option A: SSH Tunnel

```bash
ssh -L 8080:127.0.0.1:18789 openclaw-user@<your-vps>
# Now localhost:8080 on your machine → gateway on VPS
```

**Drawbacks:**
- Must keep the SSH connection open
- Manual reconnection after disconnection
- Inconvenient on mobile

#### Option B: Tailscale (recommended)

Tailscale is a mesh VPN based on WireGuard. Creates a private network between your devices.

**Installation:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

**OpenClaw configuration:**

```json
{
  "gateway": {
    "host": "<TAILSCALE_IP>",
    "port": 18789
  }
}
```

**Advantages:**
- Always available (no need to remember SSH tunnels)
- End-to-end WireGuard encryption
- Works on phone, tablet, any device
- Automatic reconnection when switching networks
- No need to open ports to the internet
- Free for up to 100 devices (Personal use)

### Stage 5. Post-Installation

1. **Run onboarding**:
   ```bash
   openclaw onboard --install-daemon
   ```

2. **Check token monitoring**:
   ```bash
   openclaw status
   openclaw usage
   ```

3. **Do not install** skills from ClawHub without manually reviewing the source code

4. **After every `openclaw update`**, verify keys haven't been exposed:
   ```bash
   grep -r "sk-ant\|sk-" ~/.openclaw/openclaw.json
   ```

5. **Set usage limits** in the provider console — protection against key leaks

6. Enable automatic OS security updates:
   ```bash
   sudo apt install -y unattended-upgrades
   sudo dpkg-reconfigure -plow unattended-upgrades
   ```

---

## Architecture and Access

### Access Diagram

```
You (laptop/phone)
    │
    ├─→ Telegram/WhatsApp/Discord ──→ Messenger servers ──→ VPS (gateway :18789)
    │
    ├─→ CLI / Web UI ──→ SSH tunnel/Tailscale ──→ VPS (gateway :18789)
    │
    └─→ macOS/iOS app ──→ Tailscale ──→ VPS (gateway :18789)
```

**Two connection types:**
- **Messengers** — connect via their own servers; gateway maintains the connection to messenger APIs
- **Control clients** (CLI, web UI, mobile app) — connect to gateway directly over WebSocket

### Communication Protocol

**WebSocket** (port `18789` by default), not REST. WebSocket was chosen because of the event-driven nature of the agent — persistent bidirectional channels are needed for streaming results of long-running tasks.

**Protocol security:**
- Schema typing via TypeBox
- Validation via AJV
- Mandatory handshake on all connections
- Gateway listens on `127.0.0.1:18789` by default

---

## Service Management

### Daemon Installation

OpenClaw is installed as a **systemd user unit** on Linux:

```bash
openclaw onboard --install-daemon
```

The wizard automatically:
- Creates a unit file in `~/.config/systemd/user/`
- Enables `loginctl enable-linger` (runs after logout)

### Management Commands

| Action | Command |
|---|---|
| Start | `clawdbot gateway start` |
| Stop | `clawdbot gateway stop` |
| Restart | `clawdbot gateway restart` |
| Status | `clawdbot gateway status` |
| Logs | `clawdbot gateway logs` |

Systemd commands also work: `systemctl --user start/stop/restart clawdbot-gateway.service`

**Important**: the gateway must run **24/7**. Stopping it means all messengers and tasks stop being processed.

---

## Pricing Model

### Component Costs

OpenClaw is **completely free** (MIT license). No subscriptions, paywall, or limitations.

| Cost item | Who you pay | Amount |
|---|---|---|
| VPS | VPS provider | Per plan |
| AI model API (Claude, GPT) | Anthropic / OpenAI / OpenRouter | $5–100/month * |
| ClawHub skills | Nobody | Free |
| OpenClaw updates | Nobody | Free |

### API Cost Estimates

Using Claude Sonnet:

| Usage level | Messages/day | Monthly cost |
|---|---|---|
| Light usage | 10–50 | ~$5–10 |
| Regular usage | 50–200 | ~$15–30 |
| Active usage | 200–500 | ~$40–100 |
| Heavy usage | 500+ | $100–800+ |

**Ways to reduce costs:**
- Claude Haiku instead of Sonnet (60–70% savings)
- Local models via Ollama (Llama, etc.) — $0, but requires a more powerful VPS or lower quality

---

## Technical Billing and API Workflow

### Interaction Flow

OpenClaw makes **direct API calls** to Anthropic/OpenAI using **your personal API key**. The provider sees every request, counts tokens, and charges **directly to your account**. OpenClaw is not involved in payments — it is only a client.

```
You → register with AI provider → get API key → attach payment method
                                                           ↓
                                              Create billing account with provider
                                                           ↓
You → enter API key into OpenClaw (on VPS) → OpenClaw makes requests to provider API
                                                           ↓
                               Provider sees every request:
                               - counts tokens (input + output + cache)
                               - records in your billing account
                               - charges your card at month-end or threshold
```

**Who pays whom:**

| Who | To whom | For what | How |
|---|---|---|---|
| **You** | **AI provider** | LLM tokens | Card/crypto linked to billing account |
| **You** | **Hosting** | VPS | Payment through provider panel |
| **OpenClaw** | Nobody | — | Open-source, MIT license |

### API Key Storage

**Location**: on your VPS, in one of two places.

#### Option A: Environment variable (recommended)

```bash
# ~/.openclaw/.env (chmod 600)
ANTHROPIC_API_KEY=sk-ant-api03-...
OPENAI_API_KEY=sk-...
```

Reference in `~/.openclaw/openclaw.json`:

```json
{
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "${ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

**Critical**: API keys must be at the root `models.providers` level, **not** inside `skills.entries`.

#### Option B: Directly in config (insecure)

**Problem**: Issue #9627 — `openclaw update` / `doctor` / `configure` commands may expand `${VAR}` and write the real values into the config, exposing the key.

### How an LLM Request Works

1. **You write a message** in Telegram/CLI/Web UI → OpenClaw gateway on VPS
2. **OpenClaw assembles context**: tools, history, workspace files, system prompt
3. **OpenClaw makes an HTTPS request** to the provider API
4. **The provider** verifies the key, processes the request, counts tokens, records in usage account, returns the response
5. **OpenClaw** delivers the response back to you

### Provider-Side Billing

#### Anthropic

- **Daily usage** — how many tokens were consumed
- **Cost breakdown**: Uncached input tokens, Cached input tokens (90% cheaper), Output tokens
- **Rates (Claude Sonnet)**: ~$3 per 1M input tokens, ~$15 per 1M output tokens

#### OpenAI

- **Daily usage** — tokens, requests, cost
- **By model**: GPT-4o, GPT-4, GPT-3.5 — different pricing
- **Rates (GPT-4o)**: ~$2.50 per 1M input tokens, ~$10 per 1M output tokens

### Cost Control

**On the provider side:**
- Set usage limits / hard limits
- Email alerts when threshold is reached

**On the OpenClaw side:**

```bash
openclaw status       # Show tokens for current session
openclaw usage        # Enable per-response cost footer
```

### API Key Leak Protection

**Protection measures:**

1. Store key in `~/.openclaw/.env` with `chmod 600`
2. Never commit to git
3. After `openclaw update`, verify the key hasn't been exposed in config
4. Rotate key every 3–6 months
5. Enable hard limits at the provider
6. Do not use the same key for production and testing

**If the key leaks:**

1. Immediately revoke in the provider console
2. Generate a new key
3. Update in OpenClaw (`~/.openclaw/.env`)
4. Review billing for suspicious requests
5. Change the provider account password

### Local Models as an Alternative

**Ollama** with local models (Llama 3, Mistral, DeepSeek, etc.) is an option:

- **No API costs** ($0)
- Full privacy — data never leaves your infrastructure
- Requires **more powerful VPS**: 8+ GB RAM, preferably GPU
- Quality is lower than Claude Sonnet/GPT-4o

**Recommendation**: hybrid approach — Ollama for simple tasks, cloud models for complex ones.

---

## Component Distribution

| Component | Location | Controlled by |
|---|---|---|
| Gateway (OpenClaw core) | **Your VPS** | You |
| Configuration, API keys | **Your VPS** (`~/.openclaw/`) | You |
| Conversation history, sessions | **Your VPS** (`~/.openclaw/`) | You |
| Workspace (files, memory, prompts) | **Your VPS** (`~/.openclaw/workspace/`) | You |
| Docker sandbox | **Your VPS** | You |
| OpenClaw source code | GitHub (moltbot/clawdbot) | Authors + community |
| openclaw npm package | npm registry | Authors |
| ClawHub (skill registry) | Authors' servers | Authors |
| LLM inference (Claude, GPT) | Anthropic / OpenAI servers | Anthropic / OpenAI |
| Messenger servers | Telegram, WhatsApp, etc. | Messenger platforms |
| Telemetry | **Nowhere** — not collected | — |

**Key points**:

1. **Zero telemetry**: OpenClaw does not collect data about you. However, when using cloud LLMs, your prompts are sent to the model provider's servers.

2. **Direct calls**: OpenClaw makes HTTPS requests directly to the provider API. No intermediate servers from OpenClaw's authors.

3. **Your billing**: The provider sees your API key and charges your account. OpenClaw is not involved in payments.

---

## Monitoring and Maintenance

### A. Continuous Monitoring

#### A1. VPS Resources

| Metric | Tool | Action threshold | What to do |
|---|---|---|---|
| RAM usage | `free -h`, htop | > 85% consistently (5+ min) | Scale up or optimize Docker |
| CPU load average | `uptime`, htop | > 80% (1 min avg) | Scale up |
| Disk — usage | `df -h` | > 80% | Clean Docker logs/images or expand disk |
| Disk — inodes | `df -i` | > 80% | Rotate small files (logs) |
| Swap usage | `free -h` | Active use > 100 MB | RAM insufficient |
| Network traffic | `vnstat` | Anomalous spikes | Possible attack or data leak |

#### A2. Docker — Container Health

| Metric | Tool | What to check |
|---|---|---|
| Container status | `docker ps -a` | Containers in `Restarting`, `Exited` state |
| RAM/CPU consumption | `docker stats` | One container consuming disproportionately |
| Image count | `docker system df` | Accumulation of unused images |
| Restart count | `docker inspect --format='{{.RestartCount}}'` | Growth — memory leak or crash loop |

#### A3. Node.js / OpenClaw

| Metric | What to check |
|---|---|
| Event loop delay | Delays > 100 ms — main thread blocked |
| Heap memory | Growth without returning to baseline — memory leak |
| GC frequency | Increasing GC → application under memory pressure |
| Response time | Latency degradation — early warning sign |
| Log errors | `ERROR`, `FATAL`, stack traces |

#### A4. Security — Continuous Control

| What to monitor | How | Why |
|---|---|---|
| SSH login attempts | `/var/log/auth.log` (fail2ban) | Brute-force attacks |
| Open ports | `ss -tlnp` | Nothing except SSH should listen on `0.0.0.0` |
| WebSocket connections | Gateway logs | CVE-2026-25253 — hijacking |
| Gateway authentication | Auth event logs | Unauthorized access attempts |
| Config integrity | `sha256sum` of configs | Unauthorized changes |
| Keys in config | `grep -r "sk-ant\|sk-\|bot[0-9]" ~/.openclaw/openclaw.json` | Credential leakage after updates |
| `.env` permissions | `ls -l ~/.openclaw/.env` | Must be `600` |
| API spending | Provider console | Unexplained growth → key leak |

### B. Event-Driven Monitoring

| Trigger event | What to check immediately |
|---|---|
| **After `openclaw update`** | 1) Keys not exposed in config. 2) Version ≥ 2026.1.29. 3) Docker sandbox in "all" mode. 4) `600` permissions on config and `.env`. 5) API keys in `models.providers`, not in `skills.entries` |
| **After installing a skill** | 1) Code reviewed manually. 2) No reverse shell, `exec()`, `fetch()` to external URLs. 3) Does not request `group:runtime` |
| **After VPS reboot** | 1) Docker daemon running. 2) Containers in `running`. 3) Firewall active. 4) Ports not listening on `0.0.0.0` |
| **Suspicious activity** | 1) Logs — who sent commands. 2) `allowedUsers` unchanged. 3) No new sessions from unknown sources |
| **Log errors** | 1) Error type (crash, OOM, auth failure, tool error). 2) `docker logs`. 3) `dmesg` — OOM killer |
| **CVE received** | 1) Does it affect your version. 2) Is a patch available. 3) Temporary mitigation measures |
| **API cost spike** | 1) Logs — request volume. 2) Has the key leaked. 3) Check for prompt injection |

### C. Trend Monitoring

| What to track | Period | Warning signal |
|---|---|---|
| RAM usage baseline | Week → month | Gradual growth → memory leak |
| Docker layer size | Weekly | Continuous growth → `docker system prune` |
| Log size | Weekly | Without rotation will fill disk within a month |
| Container restart count | Week | Growth → crash loop |
| Response latency | Week | Degradation → resources at limit |
| API token consumption | Week → month | Unexplained growth → prompt injection or leak |
| Failed SSH attempts | Week | Growth → VPS discovered by scanners |
| Swap usage | Day | Appearance and growth → RAM insufficient |

### D. Monitoring Tools

| Tool | Which checks it covers | Complexity |
|---|---|---|
| **htop** | A1 (RAM, CPU, swap) | Minimal |
| **vnstat** | A1 (network traffic) | Minimal |
| **docker stats / ps -a** | A2 (all Docker metrics) | Already available |
| **docker system df + cron prune** | A2 (images/layers) | Low |
| **fail2ban** | A4 (SSH brute-force) | Low |
| **ufw** | A4 (open ports) | Minimal |
| **logrotate** | C (log growth) | Low |
| **journalctl** | A3 (errors), A4 (auth) | Already available |
| **Cron script with alerts** | A1, A2, A4, C (thresholds) | Medium |
| **Uptime Kuma** | C (response time), A3 (latency) | Medium |

**Cron script** — a custom bash script running every 5 minutes. Checks thresholds and sends notifications to Telegram/email. Replaces Prometheus + Grafana for small VPS instances.

---

## Important Operational Considerations

### Backup and Recovery

**What to back up:**
- `~/.openclaw/openclaw.json` — configuration
- `~/.openclaw/.env` — API keys (store encrypted!)
- `~/.openclaw/workspace/` — agent files, memory, prompts
- OAuth tokens and messenger channel state

**Backup security:**
- Encrypt `.env` before sending to remote storage (GPG)
- Do not keep unencrypted backups with API keys on public servers

**Recommendation**: automated daily cron backup.

### Updates

**Risks:**
- Issue #9627 — an update may overwrite `${VAR}` in the config with the actual value
- May break skill compatibility
- May change sandbox behavior

**Strategy**: update manually, not automatically. Take a VPS snapshot before updating. Verify with the checklist from Section B afterward.

### Dependency on External Services

**If the AI provider goes down?** The gateway keeps running, but the agent cannot generate responses.

**Mitigation:**
- Configure fallback models in the config
- Keep a local model via Ollama as an emergency fallback
- For critical use cases — accounts with two providers

### Project Sustainability

**If OpenClaw's authors abandon the project?**

- Code is open (MIT) — can be forked and maintained
- npm package can be pinned to a specific version
- Installed skills will continue to work
- Critical dependency — npm registry (keep a local copy of the package)

### Multiple Instances

OpenClaw supports Multiple Gateways — each gateway is a separate systemd unit, separate port, separate configuration.

### Messenger Sessions

- **Telegram**: bot session is tied to the token and survives restarts
- **WhatsApp**: session is stored in `~/.openclaw/` and survives restarts, but migration may require re-pairing with a QR code

### API Key Leak

**Signs of a leak:**
- Unexplained increase in API spending
- Requests during hours when you were not working
- Unusual usage patterns

**Actions when detected:**

1. Immediately revoke the key in the provider console
2. Generate a new key
3. Update in OpenClaw
4. Review billing
5. Enable usage limits

### Legal Considerations

If OpenClaw processes data belonging to customers or third parties (emails, orders), this creates obligations under data protection law (Federal Law No. 152-FZ in Russia, GDPR in the EU).

Self-hosting reduces the risk but does not eliminate responsibility — data is sent to the model provider's API servers on every LLM request.

---

## What Not To Do

| Action | Why it's dangerous |
|---|---|
| Run as root | Full system access upon compromise |
| Open port 18789 to the internet | CVE-2026-25253 — remote code execution |
| Install skills from ClawHub without review | 12% contain malicious code |
| Hardcode API keys in config | Issue #9627 — may be exposed after update |
| Commit `.env` or key-containing config to git | API keys become public |
| Use the same API key for production and testing | A leak from the test environment compromises production |
| Use `auth: "none"` | Anyone on the network gets full access |
| Bind to `0.0.0.0` | Exposes gateway to the entire network/internet |
| Place API keys in `skills.entries` | Incorrect config structure |
| Ignore OS security updates | Kernel/SSH-level vulnerabilities |
| Skip backups | Data loss on VPS failure |
| Skip usage limits at the provider | Unlimited charges if the key leaks |
