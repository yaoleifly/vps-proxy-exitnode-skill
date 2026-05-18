---
name: vps-proxy-exitnode
description: Deploy and maintain a Linux VPS as a Tailscale exit node plus a 3X-UI/Xray VLESS Reality proxy node, including SSH bootstrap from IP/username/password, Tailscale installation/login/exit-node advertising, proxy-fleet or 3X-UI setup, Nginx HTTPS panel/subscription hosting, Cloudflare DNS considerations, DERP coexistence, and health checks. Use when the user provides VPS login details and asks to configure Tailscale exit node, x-ui/3x-ui, VLESS Reality, Clash/Mihomo subscription, HTTPS panel access, or to inspect/fix this server stack.
---

# VPS Proxy Exit Node

## Operating Model

Use this skill when the user gives a VPS IP, SSH username, and password and expects you to configure the server end to end. Treat credentials as sensitive: use them only for login/bootstrap, do not repeat them in user-facing replies, and prefer installing a local SSH key immediately so subsequent commands do not expose the password.

Default target stack:

- Debian/Ubuntu VPS
- Tailscale logged in and advertising `0.0.0.0/0` and `::/0` as an exit node
- 3X-UI/Xray with VLESS Reality inbound
- Nginx on public `80/443`
- HTTPS panel at a hidden path such as `https://domain/xui-<token>/`
- Mihomo/Clash subscription YAML at `https://domain/config.yaml` and a longer random path

Read [runbook.md](references/runbook.md) before performing a live deployment or repair.

## Workflow

1. Bootstrap SSH access.
   - Add a dedicated local SSH key to the remote `authorized_keys`.
   - Add a `~/.ssh/config` alias for the host.
   - Verify non-interactive SSH works before changing the server.

2. Inspect before changing.
   - Record OS, architecture, listening ports, running services, DNS status, and whether `tailscaled`, `derper`, `nginx`, or `x-ui` already exist.
   - If `80/443` are occupied, identify the owner before installing Nginx.

3. Configure Tailscale exit node.
   - If Tailscale is installed and logged in, preserve the identity and confirm `AdvertiseRoutes` includes `0.0.0.0/0` and `::/0`.
   - If not logged in, install Tailscale and run login. Ask the user for an auth key or provide the login URL when interactive approval is required.
   - Ensure IP forwarding is enabled and persistent.
   - Tell the user when route approval may still be required in the Tailscale admin console.

4. Configure 3X-UI and VLESS Reality.
   - Prefer the existing `/Users/go/proxy-fleet` workflow when present; otherwise clone `https://github.com/oaker-io/proxy-fleet`.
   - Patch local tooling as needed for current Xray output fields such as `Password (PublicKey)`.
   - Generate strong panel credentials and a hidden panel path.
   - Bind the panel to `127.0.0.1` after HTTPS reverse proxy is working.

5. Configure HTTPS and subscription hosting.
   - If a domain is provided and DNS resolves to the VPS, use Let’s Encrypt with `certbot`.
   - Put Nginx in front of the panel and subscription.
   - If DERP already uses `80/443`, move DERP to backend ports such as `8080/8443` and proxy default traffic back to DERP.
   - Keep the subscription URL stable and update `proxy-fleet/config.json`.

6. Verify and report.
   - Verify `systemctl is-active tailscaled nginx x-ui` and `derper` if present.
   - Verify `tailscale status` shows `offers exit node`.
   - Verify the panel returns `200` over HTTPS and login succeeds.
   - Verify the subscription returns `200` and `proxy-fleet status` shows the node OK.
   - Provide only final URLs, health status, and any required user action.

## Safety Rules

- Do not disable an existing service without first identifying how traffic will be preserved.
- Do not leave the 3X-UI panel publicly exposed over plain HTTP.
- Do not overwrite unrelated user config; back up service files before edits.
- Do not run destructive cleanup commands.
- If Tailscale auth or admin route approval is required, stop and ask the user for that action.

## Common Final Outputs

Report these items when complete:

- HTTPS panel URL
- Subscription URL
- Tailscale IP and exit-node status
- Proxy node address/port
- Services checked
- Any remaining manual step, especially Cloudflare proxy mode or Tailscale route approval
