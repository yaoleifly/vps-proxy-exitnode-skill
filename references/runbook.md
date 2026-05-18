# VPS Proxy Exit Node Runbook

## Inputs

Expected user inputs:

- VPS IP
- SSH username
- SSH password
- Optional domain/subdomain for HTTPS
- Optional Tailscale auth key, or permission to have the user complete browser login

Do not echo passwords back to the user.

## SSH Bootstrap

Use a dedicated key per server:

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/<alias> -C '<alias>'
```

If password SSH is needed and `sshpass` is absent, use `SSH_ASKPASS` with a temporary script or `expect`. Append only the public key:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
grep -qxF '<pubkey>' ~/.ssh/authorized_keys || echo '<pubkey>' >> ~/.ssh/authorized_keys
```

Create `~/.ssh/config`:

```sshconfig
Host <alias>
  HostName <ip>
  User <username>
  IdentityFile ~/.ssh/<alias>
  IdentitiesOnly yes
  StrictHostKeyChecking accept-new
```

Verify:

```bash
ssh -o BatchMode=yes <alias> 'uname -a; cat /etc/os-release | sed -n "1,5p"'
```

## First Inspection

Run before making changes:

```bash
ssh <alias> 'hostname; ip -br addr; ss -tulnp; systemctl list-units --type=service --state=running --no-pager'
ssh <alias> 'cat /etc/resolv.conf; getent hosts raw.githubusercontent.com github.com || true'
```

If GitHub DNS fails, temporarily set usable resolvers or fix system DNS before installing software:

```bash
printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" > /etc/resolv.conf
```

## Tailscale Exit Node

Install if missing:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

If already logged in, preserve the node. Otherwise run one of:

```bash
tailscale up --advertise-exit-node --accept-routes=false
tailscale up --auth-key '<auth-key>' --advertise-exit-node --accept-routes=false
```

Enable forwarding persistently:

```bash
cat >/etc/sysctl.d/99-tailscale-exit-node.conf <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sysctl --system
```

Verify:

```bash
tailscale status
tailscale debug prefs
tailscale netcheck
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
```

Expected signals:

- `tailscale status` shows `offers exit node`
- `AdvertiseRoutes` includes `0.0.0.0/0` and `::/0`
- IP forwarding values are `1`

The user may still need to approve the exit-node routes in the Tailscale admin console.

## 3X-UI / proxy-fleet

Use the local repo path `/Users/go/proxy-fleet` if available. Otherwise clone:

```bash
git clone https://github.com/oaker-io/proxy-fleet.git /Users/go/proxy-fleet
```

Create or update `config.json` with:

- panel username/password
- panel port, typically `9453`
- `listen_ip: "127.0.0.1"` once reverse proxy is ready
- hidden `web_base_path`, for example `/xui-<token>/`
- subscription domain/path
- VLESS Reality defaults

Patch compatibility if Xray prints `Password (PublicKey)`:

```python
pub = kv.get("Password") or kv.get("Password (PublicKey)") or kv.get("Public key", "")
```

When the panel uses a non-root base path or HTTPS internally, make sure proxy-fleet API calls use:

- normalized `web_base_path`
- `panel_scheme` of `http` or `https`
- unverified local HTTPS context if the certificate is not valid for `localhost`

Deploy:

```bash
python3 scripts/fleet.py deploy <alias> --name '<Name>' --emoji '🇯🇵'
python3 scripts/fleet.py sync
python3 scripts/fleet.py status
```

Reality checks:

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://<ip>:<vless-port>
```

`400` is normal for Reality when probed by a non-VLESS client.

## Nginx, HTTPS, and DERP Coexistence

If `derper` owns `80/443`, move it behind Nginx:

```systemd
ExecStart=/root/go/bin/derper \
  --hostname=<existing-derp-hostname> \
  --certmode=manual \
  --certdir=/etc/derper \
  --a=:8443 \
  --http-port=8080 \
  --stun=true \
  --stun-port=3478 \
  --verify-clients=true
```

Reload:

```bash
systemctl daemon-reload
systemctl restart derper
```

Install Nginx and certbot:

```bash
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx certbot
```

Issue cert with webroot after DNS resolves to the VPS:

```bash
mkdir -p /var/www/letsencrypt
certbot certonly --webroot -w /var/www/letsencrypt -d <domain> --agree-tos --register-unsafely-without-email --non-interactive
```

Use Nginx vhost shape:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name <domain>;

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name <domain>;

    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;

    root /var/www/sub;
    default_type text/plain;

    location = /<panel-path-no-trailing> {
        return 301 /<panel-path-no-trailing>/;
    }

    location /<panel-path>/ {
        proxy_pass https://127.0.0.1:9453;
        proxy_ssl_verify off;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
    }

    location = /config.yaml {
        alias /var/www/sub/<random-path>/config.yaml;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

If DERP must remain reachable on the old host/IP, add default `80/443` servers that proxy to `127.0.0.1:8080` and `https://127.0.0.1:8443` with `proxy_ssl_verify off`.

## Enabling TLS Inside 3X-UI

Some 3X-UI versions do not set cert paths via CLI reliably. Check:

```bash
/usr/local/x-ui/x-ui setting -getCert
sqlite3 /etc/x-ui/x-ui.db 'select id,key,value from settings order by id;'
```

If needed, set cert records directly:

```bash
sqlite3 /etc/x-ui/x-ui.db "delete from settings where key in ('webCertFile','webKeyFile'); insert into settings(key,value) values ('webCertFile','/etc/letsencrypt/live/<domain>/fullchain.pem'), ('webKeyFile','/etc/letsencrypt/live/<domain>/privkey.pem');"
systemctl restart x-ui
```

Verify logs show:

```text
Web server running HTTPS on 127.0.0.1:9453
```

## Final Verification

Run:

```bash
systemctl is-active tailscaled nginx x-ui derper 2>/dev/null
tailscale status
tailscale debug prefs
curl -I https://<domain>/<panel-path>/
curl -X POST -d 'username=<user>&password=<password>' https://<domain>/<panel-path>/login
curl -I https://<domain>/config.yaml
python3 scripts/fleet.py status
```

Expected:

- panel returns `200`
- login JSON has `"success":true`
- subscription returns `200`
- `proxy-fleet status` shows `✅ OK`
- `tailscale status` shows `offers exit node`

## User-Facing Summary

Keep final responses concise:

- Give panel URL and subscription URL.
- Confirm exit-node status.
- Mention if Tailscale admin route approval is still needed.
- Mention if Cloudflare should remain DNS-only. For proxy protocols, keep Cloudflare orange-cloud off unless deliberately configured.
