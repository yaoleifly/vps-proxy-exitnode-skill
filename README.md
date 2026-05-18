# VPS Proxy Exit Node Skill

[中文说明](README.zh-CN.md)

`vps-proxy-exitnode` is a Codex skill for turning a fresh Linux VPS into a combined Tailscale exit node and 3X-UI/Xray VLESS Reality proxy node.

It captures the deployment workflow we want Codex to repeat when given a server IP address, SSH username, and password: bootstrap SSH access, configure Tailscale exit-node routing, deploy 3X-UI and Xray, publish a Clash/Mihomo subscription, and expose the management panel safely through HTTPS.

## What It Does

- Bootstraps password-based SSH access into key-based access.
- Installs or verifies Tailscale and advertises the VPS as an exit node.
- Enables persistent IPv4 and IPv6 forwarding.
- Deploys 3X-UI/Xray with a VLESS Reality inbound.
- Generates and syncs a Clash/Mihomo subscription through `proxy-fleet`.
- Configures Nginx and Let’s Encrypt HTTPS for the panel and subscription.
- Supports hidden 3X-UI panel paths and local-only panel binding.
- Handles coexistence with an existing DERP server on the same VPS.
- Provides health checks for Tailscale, Nginx, 3X-UI, Xray, and subscription access.

## When To Use It

Use this skill when you want Codex to configure or inspect a VPS for:

- Tailscale exit-node service
- 3X-UI / Xray proxy service
- VLESS Reality nodes
- Clash or Mihomo subscription hosting
- HTTPS panel access through a domain
- DERP and Nginx coexistence on ports `80` and `443`

## Repository Layout

```text
.
├── SKILL.md                 # Skill trigger metadata and high-level workflow
├── references/
│   └── runbook.md           # Detailed deployment and troubleshooting runbook
├── agents/
│   └── openai.yaml          # Codex UI metadata
├── README.md                # English introduction
├── README.zh-CN.md          # Chinese introduction
└── LICENSE
```

## Install

Copy the skill into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/yaoleifly/vps-proxy-exitnode-skill.git ~/.codex/skills/vps-proxy-exitnode
```

Then ask Codex to configure a VPS as a Tailscale exit node and 3X-UI proxy node.

Example request:

```text
My server IP is <ip>, username is <user>, password is <password>.
Configure it as a Tailscale exit node and 3X-UI proxy node using the VPS Proxy Exit Node skill.
```

## Required Inputs

At minimum, Codex needs:

- VPS IP address
- SSH username
- SSH password

Optional but recommended:

- A domain or subdomain for HTTPS
- A Tailscale auth key, or permission for you to complete the Tailscale login link
- A preferred node name and region emoji

## Security Notes

This repository intentionally contains no real server passwords, private keys, panel credentials, GitHub tokens, Tailscale keys, or machine-specific secrets.

During actual use:

- Treat server credentials as temporary and sensitive.
- Rotate or revoke tokens after use.
- Keep the 3X-UI panel behind HTTPS.
- Prefer binding the panel to `127.0.0.1` and exposing it only through Nginx.
- Keep Cloudflare proxy mode off for non-HTTP proxy traffic unless you know the exact implications.

## License

MIT
