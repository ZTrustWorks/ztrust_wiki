# Nginx Reverse Proxy Setup

ZTrust uses **3 domains** behind nginx:

| Domain | Purpose | Visibility |
|--------|---------|-----------|
| `ENDPOINT_CONTROLLER_URL` | Public controller — agents connect + public API + Headscale | Public internet |
| `ENDPOINT_CONTROLLER_INTERNAL_URL` | Internal controller — frontend → backend internal API + WebSocket | Private / internal network |
| `ENDPOINT_CONTROLLER_FE_URL` | Frontend portal — Next.js admin portal | Private / internal network |

gRPC (port `50051`) is **not** proxied through nginx — agents connect directly.

---

## 0. Automated Setup (Recommended)

Run the setup script. It reads all domain values from `.env` and generates correct configs automatically:

```bash
sudo ./backend/scripts/setup-nginx.sh
```

**What it does:**
- Reads `ENDPOINT_CONTROLLER_URL`, `ENDPOINT_CONTROLLER_INTERNAL_URL`, `ENDPOINT_CONTROLLER_FE_URL` from `.env`
- Prompts for SSL certificate strategy (Let's Encrypt or self-signed)
- Generates 3 nginx configs (public BE, internal BE, frontend)
- Enables sites, tests config, reloads nginx
- Optionally configures ufw firewall (ports 80, 443, 50051)

**Preview without installing:**
```bash
sudo ./backend/scripts/setup-nginx.sh --dry-run
```

**Force self-signed certs (skip Let's Encrypt prompt):**
```bash
sudo ./backend/scripts/setup-nginx.sh --self-signed
```

---

## 1. SSL Certificate Strategy

### Option A — Let's Encrypt (production)

All 3 domains must have **valid public DNS** pointing to the server at time of issuance.
After certs are issued, the 2 private domains (`internal BE`, `frontend`) can be moved to internal-only DNS.

```bash
# Install certbot
sudo apt install -y certbot python3-certbot-nginx

# Issue certs (all 3 domains must resolve publicly at this moment)
sudo certbot certonly --nginx -d nextzeroctl.nextzero.vn
sudo certbot certonly --nginx -d innextzeroctl.nextzero.vn
sudo certbot certonly --nginx -d nextzero.nextzero.vn
```

Certbot stores certs at `/etc/letsencrypt/live/<domain>/`. Auto-renewal runs via systemd timer.

### Option B — Self-signed (lab / internal)

No DNS requirement. Browsers will show a security warning; configure client trust manually if needed.

```bash
# The setup script handles this automatically with --self-signed
sudo ./backend/scripts/setup-nginx.sh --self-signed

# Or generate manually:
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nextzeroctl.nextzero.vn.key \
  -out /etc/ssl/certs/nextzeroctl.nextzero.vn.crt \
  -subj "/CN=nextzeroctl.nextzero.vn/O=ZTrust/C=VN" \
  -addext "subjectAltName=DNS:nextzeroctl.nextzero.vn"
```

---

## 2. Nginx Configs (Manual Reference)

### 2.1 Public Controller (`nextzeroctl.nextzero.vn`)

Handles: agents calling public API, Headscale control plane, DERP.

<details>
<summary><strong>View config</strong></summary>

```nginx
upstream ztrust_be_pub  { server 127.0.0.1:8090 max_fails=2 fail_timeout=10s; }
upstream headscale_ctl  { server 127.0.0.1:8080 max_fails=2 fail_timeout=10s; }
map $http_upgrade $connection_upgrade_pub { default upgrade; '' close; }

server {
    listen 443 ssl http2;
    server_name nextzeroctl.nextzero.vn;

    ssl_certificate     /etc/letsencrypt/live/nextzeroctl.nextzero.vn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nextzeroctl.nextzero.vn/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Public API
    location /public/ {
        proxy_pass       http://ztrust_be_pub;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Agent OAuth callback (served by Frontend)
    location /open/ {
        proxy_pass       http://127.0.0.1:33000/open/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Headscale — must be last (catches all remaining paths)
    location / {
        proxy_pass              http://headscale_ctl;
        proxy_http_version      1.1;
        proxy_request_buffering off;
        proxy_buffering         off;
        proxy_read_timeout      3600s;
        proxy_send_timeout      3600s;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade_pub;
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server { listen 80; server_name nextzeroctl.nextzero.vn; return 301 https://$host$request_uri; }
```
</details>

### 2.2 Internal Controller (`innextzeroctl.nextzero.vn`)

Handles: frontend → backend internal API calls and WebSocket. **Restricted to private network IPs.**

<details>
<summary><strong>View config</strong></summary>

```nginx
upstream ztrust_be_int { server 127.0.0.1:8090 max_fails=2 fail_timeout=10s; }
map $http_upgrade $connection_upgrade_int { default upgrade; '' close; }

server {
    listen 443 ssl http2;
    server_name innextzeroctl.nextzero.vn;

    ssl_certificate     /etc/letsencrypt/live/innextzeroctl.nextzero.vn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/innextzeroctl.nextzero.vn/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # WebSocket (real-time events) — private IPs only
    location ~* ^/internal/api/v1/.*/ws$ {
        allow 127.0.0.1;
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;
        deny all;

        proxy_pass         http://ztrust_be_int;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "Upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   Authorization $http_authorization;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering    off;
    }

    # Internal API — private IPs only
    location /internal/ {
        allow 127.0.0.1;
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;
        deny all;

        proxy_pass       http://ztrust_be_int;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Public API also accessible via internal domain
    location /public/ {
        proxy_pass       http://ztrust_be_int;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / { return 404; }
}

server { listen 80; server_name innextzeroctl.nextzero.vn; return 301 https://$host$request_uri; }
```
</details>

### 2.3 Frontend Portal (`nextzero.nextzero.vn`)

<details>
<summary><strong>View config</strong></summary>

```nginx
server {
    listen 443 ssl http2;
    server_name nextzero.nextzero.vn;

    ssl_certificate     /etc/letsencrypt/live/nextzero.nextzero.vn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nextzero.nextzero.vn/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass         http://127.0.0.1:33000;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
    }
}

server { listen 80; server_name nextzero.nextzero.vn; return 301 https://$host$request_uri; }
```
</details>

---

## 3. Frontend Config

After setting up nginx, update `frontend/config/env-config.json` to match the 3-domain setup:

```json
{
  "NEXT_PUBLIC_API_BASE_URL": "https://innextzeroctl.nextzero.vn",
  "NEXT_PUBLIC_API_PUBLIC_URL": "https://nextzeroctl.nextzero.vn",
  "NEXT_PUBLIC_WS_BASE_URL": "wss://innextzeroctl.nextzero.vn",
  "NEXT_PUBLIC_LOCATION": "",
  "NEXT_PUBLIC_VERSION_APP": "1.0.0",
  "NEXT_PUBLIC_DOMAIN_APP": "https://nextzero.nextzero.vn",
  "NEXT_PUBLIC_EMAIL_ORG_DOMAIN": "@nextzero.vn"
}
```

> `NEXT_PUBLIC_API_BASE_URL` = internal domain (FE → BE internal calls)
> `NEXT_PUBLIC_API_PUBLIC_URL` = public domain (shown to users, OAuth callbacks)

---

## 4. gRPC Port (Agent Connections)

Agents connect **directly** to the controller on port `50051` — not through nginx. This port must be open in the firewall:

```bash
sudo ufw allow 50051/tcp comment "ZTrust gRPC agent connections"
```

---

## 5. Firewall Summary

| Port | Protocol | Purpose | Exposure |
|------|----------|---------|---------|
| `80` | TCP | HTTP → HTTPS redirect | Public |
| `443` | TCP | HTTPS (all 3 domains via SNI) | Public |
| `50051` | TCP | gRPC (agent → controller) | Agent machines |
| `8090` | TCP | Backend API (direct) | **Internal only — block externally** |
| `33000` | TCP | Frontend (direct) | **Internal only — block externally** |
| `8080` | TCP | Headscale (direct) | **Internal only — block externally** |

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `502 Bad Gateway` | Container not running | `docker compose ps` — check all Up |
| `SSL: no certificate` | Wrong cert path | Verify `ssl_certificate` path exists |
| `Connection refused :50051` | Firewall blocks gRPC | `sudo ufw allow 50051/tcp` |
| `403 Forbidden` on `/internal/` | Client IP not in `allow` list | Check Docker bridge subnet in nginx `allow` blocks |
| Frontend blank page | `env-config.json` URLs wrong | Check `NEXT_PUBLIC_API_BASE_URL` points to internal domain |
| `ERR_CERT_AUTHORITY_INVALID` | Self-signed cert not trusted | Use Let's Encrypt or add cert to browser/OS trust store |

**Debug commands:**
```bash
sudo nginx -t                          # Test config syntax
sudo systemctl reload nginx            # Reload without downtime
sudo tail -f /var/log/nginx/error.log  # Error log
sudo tail -f /var/log/nginx/access.log # Access log
sudo ss -tlnp | grep nginx             # Check listening ports
```
