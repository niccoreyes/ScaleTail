# OpenClaw with Tailscale Sidecar Configuration

This Docker Compose configuration sets up [OpenClaw](https://openclaw.ai/) with Tailscale as a sidecar container to keep your personal AI assistant reachable over your Tailnet.

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