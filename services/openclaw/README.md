# OpenClaw with Tailscale Sidecar Configuration

This Docker Compose configuration sets up [OpenClaw](https://openclaw.ai/) with Tailscale as a sidecar container to keep your personal AI assistant reachable over your Tailnet.

## Quick Start

**Prerequisites:**
- Docker installed with user in `docker` group
- Tailscale account with MagicDNS enabled
- API keys for preferred AI provider (Claude, OpenAI, etc.)

**Steps:**
1. Set up `.env` with Tailscale auth key and preferences
2. Run `docker compose up -d`
3. Complete OpenClaw onboarding: `docker compose exec gateway node dist/index.js onboard`
4. Access via: `https://openclaw.tailnet-name.ts.net`

**For detailed setup, see the [OpenClaw Initial Configuration](#openclaw-initial-configuration) section below.**

## OpenClaw

[OpenClaw](https://openclaw.ai/) is an open-source personal AI assistant that can actually automate tasks for you—clearing your inbox, sending emails, managing your calendar, checking you in for flights, and much more—all from WhatsApp, Telegram, or any chat app you already use. Unlike hosted AI services, OpenClaw's context, skills, and memory live on YOUR computer, giving you full data ownership and the ability to extend it with custom skills. Pairing OpenClaw with Tailscale gives you private, secure access to your AI assistant from anywhere on your Tailnet without exposing it to the public internet.

## Key Features

- Personal AI assistant with persistent memory and context
- Automates tasks like email management, calendar scheduling, travel bookings
- Works with WhatsApp, Telegram, and other chat interfaces
- Uses multiple AI models (Claude, both premium and self-hosted)
- Create custom skills and extensions
- Full data ownership and self-hosted
- Multi-agent capabilities for complex workflows
- Tailscale integration for private, secure remote access

## Configuration Overview

In this setup, the `tailscale-openclaw` service runs Tailscale and manages secure networking for OpenClaw. The `gateway` service shares that network stack via Docker's `network_mode: service:` configuration, which keeps your AI assistant accessible only over your Tailnet unless you intentionally expose it via Funnel or host port mappings.

## Prerequisites

- User should be in the `docker` group for proper file permissions
- Pre-create volume directories to avoid root-owned folders:
  ```bash
  mkdir -p openclaw-config openclawworkspace
  chown $USER:$USER openclaw-config openclawworkspace
  ```
- Enable MagicDNS in your Tailscale admin console for proper HTTPS access
- Get a Tailscale auth key from https://tailscale.com/admin/settings/keys

## OpenClaw-specific Gotchas

- **Initial Setup**: First launch requires configuring your Claude API keys or local model endpoints through OpenClaw's CLI interface
- **Authentication**: 
  - `OPENCLAW_GATEWAY_TOKEN` is OpenClaw's internal authentication (NOT the same as Tailscale's `TS_AUTHKEY`)
  - Generate a random token with `openssl rand -base64 32` for additional security
  - Can be left empty for private Tailnet usage since Tailscale provides network isolation
  - `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` should generally be left empty for security; set to "true" only for specific debugging needs
- **Workspace Permissions**: Ensure the `openclaw-workspace` directory has proper permissions for the user running the container
- **Claude Configuration**: `CLAUDE_AI_SESSION_KEY`, `CLAUDE_WEB_SESSION_KEY`, and `CLAUDE_WEB_COOKIE` are optional—your assistant can help you obtain them interactively
- **MagicDNS**: Set `TS_ACCEPT_DNS=true` in the compose.yaml environment variables if you want HTTPS access via MagicDNS
- **Time Zone**: Set `OPENCLAW_TZ` to match your local timezone for accurate scheduling and time-based automations

## MagicDNS and Serve Configuration

When using MagicDNS with Tailscale Serve:

1. Enable MagicDNS in your Tailscale admin console (your tailnet's DNS settings)
2. Set `TS_CERT_DOMAIN=openclaw` in `.env` (or your preferred hostname)
3. Set `TS_ACCEPT_DNS=true` in `.env` to enable MagicDNS
4. The `ts-serve` config automatically routes HTTPS port 443 to the OpenClaw gateway on port 18789
5. Access your OpenClaw assistant at `https://openclaw.tailnet-name.ts.net`

**Note:** If you encounter connection refused issues, ensure:
- MagicDNS is enabled in your Tailscale admin console
- HTTPS is enabled for your tailnet (required for Tailscale Serve)
- The Tailscale container has a valid Tailnet IP and is healthy (check with `docker compose ps`)
- The OpenClaw gateway container is healthy and running on port 18789

## OpenClaw Initial Configuration

After `docker compose up`, you need to complete OpenClaw's initial configuration:

### Step 1: Verify Services Are Running
```bash
docker compose ps
# Both containers should show "Up (healthy)"
```

### Step 2: Run OpenClaw Onboarding Wizard (Recommended)

OpenClaw provides an interactive setup wizard:

```bash
# Interactive onboarding
docker compose exec gateway node dist/index.js onboard --mode local --no-install-daemon

# Or use the simplified wizard
docker compose exec gateway node dist/index.js onboard
```

The wizard will configure:
- API provider and model selection (Claude, OpenAI, etc.)
- Gateway token generation
- Basic gateway settings

### Step 3: Manual Configuration (Alternative)

If you prefer manual setup or the wizard fails:

**Configure Tailscale mode:**
```bash
docker compose exec gateway node dist/index.js config set --batch-json '[
  {"path":"gateway.mode","value":"local"},
  {"path":"gateway.bind","value":"tailnet"},
  {"path":"gateway.tailscale","value":{"mode":"serve"}}
]'
```

**Set up allowed origins:**
```bash
docker compose exec gateway node dist/index.js config set --batch-json '[
  {"path":"gateway.controlUi.allowedOrigins","value":["https://*.ts.net","http://*.ts.net"]}
]'
```

**Generate gateway token:**
```bash
# Generate random token
TOKEN=$(openssl rand -base64 32)
echo "Your Gateway Token: $TOKEN"

# Set via config
docker compose exec gateway node dist/index.js config set gateway.token "$TOKEN"

# Or update .env file
echo "OPENCLAW_GATEWAY_TOKEN=$TOKEN" >> .env
docker compose up -d
```

**Configure model provider:**
```bash
# For Anthropic/Claude
docker compose exec gateway node dist/index.js config set --batch-json '[
  {"path":"agent.model","value":"anthropic/claude-opus-4-6"},
  {"path":"agents.defaults.model","value":"anthropic/claude-opus-4-6"}
]'

# For OpenAI
docker compose exec gateway node dist/index.js config set agent.model "openai/gpt-4-turbo"
```

### Step 4: Access the Web UI

**Via MagicDNS (recommended):**
```
https://openclaw.tailnet-name.ts.net
```

**Via dashboard command:**
```bash
docker compose exec gateway node dist/index.js dashboard --no-open
```

**Via Tailscale IP (fallback):**
```bash
# Get Tailscale IP
docker compose exec tailscale tailscale ip -4
# Access via: http://<IP>:18789
```

### Step 5: Configure Messaging Channels (Optional)

**WhatsApp (shows QR code to scan):**
```bash
docker compose exec gateway node dist/index.js channels login whatsapp
```

**Telegram (requires bot token):**
```bash
docker compose exec gateway node dist/index.js channels add telegram --token "YOUR_BOT_TOKEN"
```

**Discord (requires bot token):**
```bash
docker compose exec gateway node dist/index.js channels add discord --token "YOUR_BOT_TOKEN"
```

### Step 6: Connect Remote Devices (Optional)

From another device (phone, laptop, etc.) running OpenClaw:
```bash
# On the Pi, get connection code
docker compose exec gateway node dist/index.js devices list
```

From your other device:
```bash
docker compose exec gateway node dist/index.js devices approve <connection-id>
```

### Step 7: Test Configuration

**Check gateway health:**
```bash
docker compose exec gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

**Test simple message:**
```bash
docker compose exec gateway node dist/index.js agent --message "Hello from OpenClaw!"
```

**View current config:**
```bash
docker compose exec gateway node dist/index.js config get
```

### Common Post-Setup Tasks

**Enable persistent memory:**
```bash
docker compose exec gateway node dist/index.js config set agents.defaults.memory.enabled true
```

**Enable file access:**
```bash
docker compose exec gateway node dist/index.js config set agents.defaults.tools.read.enabled true
docker compose exec gateway node dist/index.js config set agents.defaults.tools.write.enabled true
```

**Check OpenClaw doctor for issues:**
```bash
docker compose exec gateway node dist/index.js doctor
```

## Configuration Persistence

OpenClaw stores all configuration, state, and workspace data in mounted volumes:

- **`openclaw-config/`** - Main configuration directory
  - `openclaw.json` - Gateway and agent settings
  - `agents/<agentId>/agent/auth-profiles.json` - API keys and authentication profiles
  - `.env` - Environment variables and secrets

- **`openclaw-workspace/`** - Workspace and skills directory
  - `skills/<skill>/SKILL.md` - Custom skill definitions
  - Agent sessions and memory data
  - Persistent workspaces files

- **Tailscale state** - Stored in `ts/state/`
  - Tailscale node keys and connection state
  - Persistent across container restarts

**Manual configuration editing:**
```bash
# View current configuration
cat openclaw-config/openclaw.json

# Edit configuration (requires container restart)
nano openclaw-config/openclaw.json
docker compose restart gateway
```

## Advanced Configuration

### Enable Agent Sandboxing
To run agent tool execution in isolated Docker containers:
```bash
# Enable sandboxing
docker compose exec gateway node dist/index.js config set --batch-json '[
  {"path":"agents.defaults.sandbox.mode","value":"non-main"},
  {"path":"agents.defaults.sandbox.scope","value":"agent"}
]'

# Build sandbox image (optional, for custom sandbox)
# From OpenClaw repo: ./scripts/sandbox-setup.sh
```

### Tailscale Funnel (Public Access)
To expose OpenClaw publicly with password protection:
```bash
# Enable Funnel mode
docker compose exec gateway node dist/index.js config set gateway.tailscale.mode funnel

# Set password auth (required for Funnel)
docker compose exec gateway node dist/index.js config set gateway.auth.mode password

# Set password
PASS=$(openssl rand -base64 16)
docker compose exec gateway node dist/index.js config set gateway.password "$PASS"

# Note: Edit ts-serve config to set AllowFunnel to true
```

### Multiple Agent Workspaces
Create separate agent instances with different settings:
```bash
# Create new agent
docker compose exec gateway node dist/index.js agents new --id "personal"
docker compose exec gateway node dist/index.js agents new --id "work"

# Configure different models
docker compose exec gateway node dist/index.js config set agents.personal.model "anthropic/claude-opus-4-6"
docker compose exec gateway node dist/index.js config set agents.work.model "openai/gpt-4-turbo"
```

## Port Exposure

The `ports` section is commented out by default for Tailnet-only access. To expose OpenClaw locally:

1. Uncomment the `ports` section in the compose.yaml
2. Set `SERVICEPORT=18789` in `.env`
3. Access locally at `http://localhost:18789`

## Optional LAN Exposure

If you need LAN access while keeping Tailnet connectivity, uncomment the ports mapping in compose.yaml:

```yaml
ports:
  - 0.0.0.0:18789:18789
```

This allows local network access while maintaining Tailscale connectivity.

## Troubleshooting

**Connection refused on MagicDNS URL:**
```bash
# Check container status
docker compose ps

# Check Tailscale logs
docker compose logs tailscale

# Check OpenClaw logs  
docker compose logs gateway

# Verify Tailscale connectivity
docker compose exec tailscale tailscale status

# Test local access
docker compose exec gateway curl -I http://127.0.0.1:18789/healthz
```

**Common issues:**
- MagicDNS not enabled in Tailscale admin console
- HTTPS not enabled for your tailnet
- Tailscale Serve configuration incomplete
- Missing `TS_AUTHKEY` or invalid auth key
- OpenClaw gateway failing to start (check logs)

**"Unauthorized" when accessing web UI:**
```bash
# Verify gateway token is set
docker compose exec gateway node dist/index.js config get gateway.token

# Check .env matches configured token
grep OPENCLAW_GATEWAY_TOKEN .env

# Try accessing via Tailscale IP directly
docker compose exec tailscale tailscale ip -4
# Access: http://<IP>:18789
```

**Model not responding:**
```bash
# Check API key configuration
docker compose exec gateway node dist/index.js config get agent.model

# Verify network connectivity
docker compose exec gateway ping api.anthropic.com

# Check gateway logs
docker compose logs gateway --tail=50
```

**Permission errors in config directories:**
```bash
# Fix permissions for OpenClaw user (uid 1000)
sudo chown -R 1000:1000 openclaw-config openclaw-workspace

# Restart containers
docker compose down && docker compose up -d
```

**Quick diagnostic command:**
```bash
# Comprehensive health check
docker compose exec gateway node dist/index.js doctor
```

## Upstream Documentation

- [OpenClaw Website](https://openclaw.ai/)
- [OpenClaw GitHub Repository](https://github.com/openclaw/openclaw)
- [OpenClaw Tailscale Documentation](https://docs.openclaw.ai/gateway/tailscale)
- [OpenClaw Docker Installation](https://docs.openclaw.ai/install/docker)
- [OpenClaw Configuration Guide](https://docs.openclaw.ai/)
- [Tailscale Serve Overview](https://tailscale.com/kb/1312/serve)

## Docker Image Note

The default image `ghcr.io/openclaw/openclaw:latest` is provided from the GitHub Container Registry. Alternative tags include: `main`, `latest`, or version tags (e.g., `2026.2.26`). For custom builds, follow the [OpenClaw Docker guide](https://docs.openclaw.ai/install/docker) to build locally as `openclaw:local`.

## Files to Check

Please check the following contents for validity as some variables need to be defined upfront:

- `.env` // Main variables: `TS_AUTHKEY`, `OPENCLAW_GATEWAY_TOKEN`