# ZTrust — All-in-One Installation Guide

End-to-end, foolproof deployment of ZTrust Controller + Frontend + Nginx + SSL on a single Linux server. Follow the steps **in order**. Each step has a verification — **do not proceed if verification fails**.

> **Target audience**: fresh Ubuntu/CentOS server, one-shot production install.
> **Time**: ~20–30 minutes (plus DNS propagation if using Let's Encrypt).
> **Result**: ZTrust accessible at 3 domains over HTTPS, agents connecting via gRPC.

---

## Step 0 — Prerequisites (5 min)

### 0.1 Server spec

| | Minimum | Recommended (~500 agents) |
|---|---|---|
| CPU | 4 cores | 16 cores |
| RAM | 8 GB | 24+ GB |
| Disk | 30 GB SSD | 100 GB SSD |
| OS | Ubuntu 22.04+ / CentOS 9+ | Same |

### 0.2 Install Docker + Docker Compose

```bash
# Ubuntu / Debian
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker
```

**Verify:**
```bash
docker --version && docker compose version
```
Both must print a version. If not, reinstall Docker before continuing.

### 0.3 Open firewall ports

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 50051/tcp   # gRPC — agents connect directly
sudo ufw --force enable
```

**Verify:**
```bash
sudo ufw status | grep -E '80|443|50051'
```
Must show 3 `ALLOW` lines.

### 0.4 Plan your 3 domains

ZTrust uses 3 domains behind nginx:

| Purpose | Example | Visibility |
|---|---|---|
| **Public Controller** (agents connect) | `ztcontroller.example.com` | **Public DNS** |
| **Internal Controller** (FE → BE) | `inztcontroller.example.com` | Private (after cert issued) |
| **Frontend Portal** | `ztrust.example.com` | Private (after cert issued) |

**Decision — pick ONE cert strategy:**

- **A. Let's Encrypt (recommended for prod)**: all 3 domains must have **valid public DNS → this server's IP** at install time. After certs are issued, you can move the 2 private domains to internal-only DNS.
- **B. Self-signed (lab / internal)**: no DNS required. Browsers will warn — acceptable for testing.

**If choosing A, set DNS records NOW and wait for propagation (`dig <domain> +short` must return your server IP).**

---

## Step 1 — Download (1 min)

Get the ZTrust deployment bundle from [nextzero.vn/download](https://nextzero.vn/download) → **Server - Community Edition**.

```bash
# Extract wherever you want — e.g. /opt
cd /opt
# ... extract ztrust_deploy here ...
cd ztrust_deploy
```

**Verify:**
```bash
ls setup.sh backend/scripts/setup-nginx.sh env.sample
```
All 3 files must exist.

---

## Step 2 — Create `.env` and set your 3 domains (2 min)

```bash
cp env.sample .env
```

Edit `.env` and set **only these 3 variables** (everything else is auto-configured):

```bash
ENDPOINT_CONTROLLER_URL=https://ztcontroller.example.com
ENDPOINT_CONTROLLER_INTERNAL_URL=https://inztcontroller.example.com
ENDPOINT_CONTROLLER_FE_URL=https://ztrust.example.com
```

> `setup.sh` will auto-generate: `HOST_IP`, all database passwords, `JWT_KEK`, `SUPER_ADMIN_PASSWORD`, and all database URIs. **Do not edit them manually.**

**Optional (skip for minimum install):**
- `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` — for Google SSO
- `SMTP_*` — for email alerts
- `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` — for Telegram alerts

**Verify:**
```bash
grep -E '^ENDPOINT_CONTROLLER_(URL|INTERNAL_URL|FE_URL)' .env
```
Must show 3 lines with your domains.

---

## Step 3 — Provide gRPC TLS certificate (skip for lab)

Agents connect to the Controller over **TLS-encrypted gRPC** on port `50051`. Any valid TLS cert for your public controller domain works.

**Pick one:**

- **A. Reuse Let's Encrypt cert (recommended)** — the same cert nginx uses in Step 5. Do this AFTER running `setup-nginx.sh` with LE:
  ```bash
  mkdir -p backend/certs
  sudo cp /etc/letsencrypt/live/<public-domain>/fullchain.pem \
          backend/certs/<public-domain>.crt
  sudo cp /etc/letsencrypt/live/<public-domain>/privkey.pem \
          backend/certs/<public-domain>.key
  sudo chmod 600 backend/certs/<public-domain>.key
  ```
- **B. Self-signed (lab only)** — agents must add it to their trust store manually.

Update `.env`:

```bash
CERT_FILE_PATH=/etc/ztrust/certs/<public-domain>.crt
KEY_CERT_FILE_PATH=/etc/ztrust/certs/<public-domain>.key
```

> **Skip entirely**: leave both paths empty to disable gRPC TLS (agents cannot connect in prod — lab only).

**Verify (if using Option A/C):**
```bash
ls -l backend/certs/
```
Must show `.crt` and `.key` files.

> **Note — Option A ordering**: Let's Encrypt certs don't exist until Step 5 runs. If you're using Option A, skip this step now, finish Step 5 first, then come back and copy the certs + restart backend: `sudo docker compose restart ztrust_be`.

---

## Step 4 — Run `setup.sh` (10 min)

```bash
chmod +x setup.sh
sudo ./setup.sh
```

The script will:
1. Auto-detect `HOST_IP`
2. Generate all missing passwords / secrets
3. Create headscale config with your public domain
4. Start all containers: `mongo_rs{1,2,3}`, `postgres`, `haproxy`, `redis_*`, `headscale_zt`, `ztrust_be`, `ztrust_fe`

Press `Enter` at prompts to accept defaults (anything you've already set in `.env` is preserved).

**Verify (all containers must be `Up`):**
```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```
Expected containers: `ztrust_be`, `ztrust_fe`, `headscale_zt`, `mongo_rs1`, `postgres`, `redis_zt`, `haproxy`.

**If any container is restarting or exited:**
```bash
sudo docker logs <container_name> --tail 50
```
Most common cause: missing `.env` variable → re-check Step 2.

---

## Step 5 — Run `setup-nginx.sh` (5 min)

```bash
sudo ./backend/scripts/setup-nginx.sh
```

The script will:
1. Read your 3 domains from `.env`
2. Prompt: `Let's Encrypt (A)` or `Self-signed (B)` — pick per Step 0.4
3. If A: run certbot for all 3 domains (all must resolve publicly)
4. Generate 3 nginx configs (public BE + Headscale, internal BE, frontend)
5. Enable sites, test config, reload nginx

**Preview without installing:**
```bash
sudo ./backend/scripts/setup-nginx.sh --dry-run
```

**Force self-signed (skip prompt):**
```bash
sudo ./backend/scripts/setup-nginx.sh --self-signed
```

**Verify:**
```bash
sudo nginx -t
sudo systemctl status nginx | grep Active
```
Must show `syntax is ok`, `test is successful`, and `Active: active (running)`.

---

## Step 6 — Configure frontend `env-config.json` (2 min)

```bash
sudo cp frontend/config/env-config.example.json frontend/config/env-config.json
sudo nano frontend/config/env-config.json
```

Set:

```json
{
  "NEXT_PUBLIC_API_BASE_URL": "https://inztcontroller.example.com",
  "NEXT_PUBLIC_API_PUBLIC_URL": "https://ztcontroller.example.com",
  "NEXT_PUBLIC_WS_BASE_URL": "wss://inztcontroller.example.com",
  "NEXT_PUBLIC_DOMAIN_APP": "https://ztrust.example.com",
  "NEXT_PUBLIC_EMAIL_ORG_DOMAIN": "@example.com",
  "NEXT_PUBLIC_LOCATION": "",
  "NEXT_PUBLIC_VERSION_APP": "1.0.0"
}
```

Rules:
- `NEXT_PUBLIC_API_BASE_URL` = **internal** domain (FE → BE private calls)
- `NEXT_PUBLIC_API_PUBLIC_URL` = **public** domain (OAuth callbacks, shown to users)
- `NEXT_PUBLIC_WS_BASE_URL` = `wss://` + internal domain

Restart frontend:
```bash
sudo docker compose restart ztrust_fe
```

---

## Step 7 — End-to-end verification (3 min)

Run each check. **All 4 must pass.**

### 7.1 Public Controller (agents endpoint)
```bash
curl -sI https://ztcontroller.example.com/public/healthz
```
Expect: `HTTP/2 200` or `HTTP/2 204`.

### 7.2 Internal Controller (blocked from public = good)
```bash
curl -sI https://inztcontroller.example.com/internal/api/v1/health
```
Expect: `HTTP/2 403` (private IP ACL — confirms restriction works).
From the server itself: `curl -sI http://127.0.0.1:8090/internal/api/v1/health` → `200`.

### 7.3 Frontend portal
```bash
curl -sI https://ztrust.example.com
```
Expect: `HTTP/2 200`.

### 7.4 gRPC port (agent connection)
```bash
nc -vz <server-ip> 50051
```
Expect: `succeeded` / `open`.

### 7.5 Login to portal

Open `https://ztrust.example.com` in your browser:
- Username: `superadmin`
- Password: check `.env` — `SUPER_ADMIN_PASSWORD` (auto-generated)

```bash
grep SUPER_ADMIN_PASSWORD .env
```

**If login works → installation complete.** ✅

---

## Troubleshooting — Fix Table

| Symptom | Cause | Fix |
|---|---|---|
| Container keeps restarting | Bad `.env` value | `docker logs <name>` → fix var → `docker compose up -d` |
| `502 Bad Gateway` on any domain | Backend/FE not running | `docker ps` — confirm all `Up`; restart if needed |
| `certbot failed` during Step 5 | DNS not propagated | `dig <domain> +short` → wait → rerun `sudo ./backend/scripts/setup-nginx.sh` |
| Browser `ERR_CERT_AUTHORITY_INVALID` | Self-signed cert | Use Let's Encrypt OR install cert in OS trust store |
| `403 Forbidden` on `/internal/` from FE server | Server not in private IP range | Check `allow` blocks in `/etc/nginx/sites-available/<internal-domain>.conf` |
| Frontend blank / API errors | `env-config.json` wrong | Re-verify Step 6 URLs |
| Agent can't connect on :50051 | Firewall blocks, wrong cert | `ufw status` shows 50051 allowed; verify `CERT_FILE_PATH` points to existing file |
| `nginx -t` fails | Cert files missing | Check paths in `/etc/nginx/sites-available/*.conf` exist |
| Forgot superadmin password | Lost `.env` password | `docker exec -it ztrust_be ./reset-admin-password.sh` (if available) or redeploy |

### Debug commands

```bash
# All container status
sudo docker ps

# Live logs for backend / frontend
sudo docker logs -f ztrust_be
sudo docker logs -f ztrust_fe

# Nginx logs
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Test nginx config
sudo nginx -t

# Reload nginx without downtime
sudo systemctl reload nginx

# Listening ports
sudo ss -tlnp | grep -E '80|443|8090|8080|33000|50051'
```

---

## Upgrade / Restart / Uninstall

```bash
# Restart all
sudo docker compose restart

# Stop all
sudo docker compose down

# Update images (after pulling new deploy bundle)
sudo docker compose pull && sudo docker compose up -d

# Full uninstall (DESTRUCTIVE — deletes all data)
sudo docker compose down -v
sudo rm -rf /etc/nginx/sites-enabled/{<your-3-domains>}.conf
sudo systemctl reload nginx
```

---

## Reference — What gets auto-configured

`setup.sh` writes sane defaults for everything below — you should **never** edit them manually:

- `HOST_IP` (auto-detected)
- `MONGO_URI`, `POSTGRES_URL`, `REDIS_*` (internal container names)
- `MONGO_ROOT_PASSWORD`, `POSTGRES_PASSWORD`, `REDIS_PASSWORD` (random)
- `JWT_KEK` (random 32-byte hex)
- `SUPER_ADMIN_PASSWORD` (random)
- `HEADSCALE_APIKEY`, `TAILSCALE_AUTHKEY` (generated against Headscale)
- Headscale `server_url`, DB connection, DNS records

For deep-dive manual nginx reference, see [Nginx Reverse Proxy Setup](setup-nginx.md).
For the step-by-step with screenshots, see [Installation System](install.md).
