# VPS Proxy Exit Node Skill

A Codex skill for deploying and maintaining a Linux VPS as both:

- a Tailscale exit node
- a 3X-UI/Xray VLESS Reality proxy node

It captures a repeatable workflow for SSH bootstrap from IP/username/password, Tailscale exit-node advertising, 3X-UI setup, Nginx/Let’s Encrypt HTTPS, hidden panel paths, Clash/Mihomo subscription publishing, and DERP coexistence.

## Install

Copy this folder into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R vps-proxy-exitnode-skill ~/.codex/skills/vps-proxy-exitnode
```

Then ask Codex to configure a VPS as a Tailscale exit node and 3X-UI proxy node.

## Contents

- `SKILL.md`: trigger metadata and high-level workflow
- `references/runbook.md`: detailed deployment and verification runbook
- `agents/openai.yaml`: UI metadata for Codex

## Security

The skill intentionally contains no server passwords, private keys, panel credentials, tokens, or machine-specific secrets. Treat any credentials supplied during use as ephemeral and do not commit them.

## License

MIT
