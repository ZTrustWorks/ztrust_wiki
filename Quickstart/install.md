# Installation

## System Requirements

| Component | Minimum (Lab / PoC) ~100 agents | Recommended 1000+ agents (Production) |
|-----------|----------------------|---------------------------|
| CPU       | 4 cores              | 16 cores                 |
| RAM       | 8 GB                 | 24 GB (≥ 32 GB if many service, users, devices) |
| Storage   | 30 GB SSD            | 100 GB SSD / NVMe (expandable) |
| OS        | Linux (Ubuntu/CentOS recommended) | Linux (Ubuntu/CentOS recommended) |
| Docker    | Docker & Docker Compose installed | Docker & Docker Compose installed |
| Domain    | Optional (can use local IP) | Public domain with valid SSL certificate (no self-signed) for Controller and Headscale, internal domain for Frontend |
| Network   | Local network        | Public network, Firewall, Reverse Proxy (Nginx/Caddy) |

## Quick Start
Config `.env` file first. 
- Replace your server IP to `HOST_IP`. 
- Ensure `MONGO_URI` and credentials are correct.
- Ensure `POSTGRES_URL` and credentials are correct.
- Ensure `REDIS_SENTINEL_ADDRS` and credentials are correct.

The `setup.sh` script automates the entire setup for both **Controller** and **Frontend**.

### 1. Run Setup Script

```bash
chmod +x ./setup.sh
sudo ./setup.sh
```

This will:
- Generate necessary keys and configurations.
- Create `.env` from defaults if missing.
- Start all services (Mongo, Redis, Postgres, Headscale, Controller, Frontend).

### 2. Access Services

After the script completes, access your services at:

- **Frontend**: `http://<YOUR_HOST_IP>:33000`
- **Controller**: `http://<YOUR_HOST_IP>:8090`
- **Headscale**: `http://<YOUR_HOST_IP>:8080`

> **Note**: Replace `<YOUR_HOST_IP>` with your server's actual IP address (e.g., `10.110.86.17`).


## Configuration

The system is configured via the `.env` file. If it doesn't exist, `setup.sh` will create one from `env.sample`.

### Minimum Configuration
To ensure the system runs correctly, update these in `.env`:
1. **`HOST_IP`**: Your server's IP address.
2. **Database**: `MONGO_URI`, `POSTGRES_URL`, `REDIS_SENTINEL_ADDRS`.
3. **Passwords**: `SUPER_ADMIN_PASSWORD`, `MONGO_ROOT_PASSWORD`, `POSTGRES_PASSWORD`, `REDIS_PASSWORD`.
4. **`JWT_KEK`**: A random 32-byte hex string.

<details>
<summary><strong>Environment Variables Reference</strong></summary>

```dotenv
# ==============================================================================
# PROJECT & ENVIRONMENT
# ==============================================================================
COMPOSE_PROJECT_NAME=ztrust_be
# Environment: prod / testing / dev
ENV=prod
# Enable debug mode: true / false
AGENT_DEBUG=false
# Gin framework mode: release / debug
GIN_MODE=release

# ==============================================================================
# NETWORK CONFIGURATION
# ==============================================================================
# Network name of docker-compose
COMMON_NET=zt-stack-net
HOST_IP=10.110.86.17
# Hostname for BE
HOSTNAME=zerotrust-be
# Hostname of BE in Tailnet
TS_HOSTNAME=zerotrust-be

# Control API Address (Bind Address) (Controller running on this address)
ENDPOINT_CONTROLLER_ADDR=0.0.0.0:8090
# Public URL for Controller (e.g., https://ztcontroller.ghtklab.com)
# Ensure your public URL is accessible from the internet (for agent to connect to controller)
ENDPOINT_CONTROLLER_URL=https://ztcontroller.ghtklab.com
# Frontend URL (e.g., https://ztrust.ghtklab.com)
# The portal domain for admin to access the portal
ENDPOINT_CONTROLLER_FE_URL=https://ztrust.ghtklab.com

# gRPC Address (Port for gRPC server)
GRPC_ADDR=:50051
# Metrics Address (127.0.0.1 for production)
GIN_METRICS_ADDR=127.0.0.1:9099

# ==============================================================================
# DATABASE: MONGODB (Required)
# ==============================================================================
# MongoDB connection for controller
MONGO_URI=mongodb://root:changeme!@10.110.86.17:27027,10.110.86.17:27028,10.110.86.17:27029/?authSource=admin&replicaSet=rs0
DB_NAME=ztrust_testing_v1
MONGO_ROOT_USERNAME=root
MONGO_ROOT_PASSWORD=changeme!
MONGO_REPLICA_SET_NAME=rs0

# ==============================================================================
# DATABASE: POSTGRESQL (Required)
# ==============================================================================
# Postgres connection for headscale and controller
POSTGRES_URL=postgres://postgres:changeme!@10.110.86.17:5432/headscale?sslmode=disable
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changeme!
POSTGRES_DB=headscale

# ==============================================================================
# DATABASE: REDIS (Required)
# ==============================================================================
# Redis connection for controller
REDIS_SENTINEL_ADDRS=10.110.86.17:26379,10.110.86.17:26380,10.110.86.17:26381
REDIS_MASTER_NAME=redis-master
REDIS_PASSWORD=changeme!
REDIS_MASTER_HOST=10.110.86.17
REDIS_MASTER_PORT=36379

# ==============================================================================
# HEADSCALE & TAILSCALE
# ==============================================================================
# Headscale Server URL (Controller call headscale via internal network)
HEADSCALE_SERVER_URL=http://headscale_zt:8080
# API Key for Headscale (Generated automatically)
HEADSCALE_APIKEY=xxx
# Auth Key for Tailscale (Generated automatically)
TAILSCALE_AUTHKEY=xxx
# Headscale User for controller
HEADSCALE_USER_BE=ep_controller
# Headscale Database Type: sqlite / postgres
HEADSCALE_DB_TYPE=postgres
# Path to extra DNS records (Keep it default)
EXTRA_DNS_PATH=/var/lib/headscale/extra-records.json
# Tailscale State Directory (Keep it default)
TS_NET_DIR=/app/.tsnet-data

# ==============================================================================
# GOOGLE OAUTH
# ==============================================================================
# Follow this link for get client ID https://developers.google.com/identity/oauth2/web/guides/get-google-api-clientid
# Google Client ID
GOOGLE_CLIENT_ID=xxx
# Google Client Secret
GOOGLE_CLIENT_SECRET=xxx
# Google Redirect URL, e.g. https://ztcontroller.ghtklab.com/public/api/v1/auth/google/callback
# You must add this callback URL in Google API Console
# Format: https://<your-controller-domain>/public/api/v1/auth/google/callback
GOOGLE_REDIRECT_URL=

# ==============================================================================
# SECURITY & TOKENS
# ==============================================================================
# Static Agent Token
# Static token for agent to authenticate with controller
AGENT_STATIC_TOKEN=5^rSXTyBCRQnN^sH86!SXtTc88vuT%dXTS5UoW9BqH%SyTJcQW62EFXTRbp9EUJxXggCEig45qEQxfVKJnB2*tB74MMog2wM64Kaa7zXfx6hC2D4YS6zAd5c4UVr
# JWT Key Encryption Key (32-bytes hex)
JWT_KEK=1ceea52ff71b878c458cc0632f9dbc9506aaca822090922bfe63c2d56277a8e4

# ==============================================================================
# ADMINISTRATION (Required)
# ==============================================================================
# Super Admin Credentials
# Account name of super admin default: superadmin
SUPER_ADMIN_USERNAME=superadmin 
# Email of super admin default: superadmin@ghtk.co
SUPER_ADMIN_EMAIL=superadmin@ghtk.co 
# Password of super admin default: changeme!
SUPER_ADMIN_PASSWORD=changeme! 

# ==============================================================================
# INTEGRATIONS: EMAIL (Optional)
# ==============================================================================
SMTP_HOST=<Your SMTP Host>
SMTP_PORT=<Your SMTP Port>
SMTP_USERNAME=<Your SMTP Username>
SMTP_PASSWORD=<Your SMTP Password>
EMAIL_FROM=<Your Email From>

# ==============================================================================
# INTEGRATIONS: TELEGRAM (Required)
# ==============================================================================
TELEGRAM_BOT_TOKEN=<Your Telegram Bot Token>
TELEGRAM_CHAT_ID=<Your Telegram Chat ID>
TELEGRAM_ENABLE_COMMAND=true

# ==============================================================================
# INTEGRATIONS: KAFKA & SIEM (Optional)
# ==============================================================================
SEND_LOG_TO_SIEM=false
KAFKA_BROKERS=<Your Kafka Brokers eg. broker1:9092,broker2:9092,broker3:9092    >
KAFKA_TOPIC=<Your Kafka Topic>
# Retry limit for send log to SIEM default: 3
KAFKA_SIEM_RETRY_LIMIT=3 

# ==============================================================================
# AGENT CONFIGURATION & TUNING (Default values are recommended)
# ==============================================================================
AGENT_STREAM_SEND_BUF=100
AGENT_CONTROL_HIGH_PRIORITY_CHAN_SIZE=10000
AGENT_CONTROL_LOW_PRIORITY_CHAN_SIZE=20000
AGENT_CONTROL_POOL_SIZE=10000
AGENT_CONTROL_WORKER_COUNT=32
POSTURE_REPORT_CHAN_SIZE=16384
POSTURE_VIO_CHAN_SIZE=8192
POSTURE_EVAL_WORKERS=16

# Custom Protocol Agent Default: ztna-ghtk-agent://auth/callback
CUSTOM_PROTOCOL_AGENT=ztna-ghtk-agent://auth/callback 

# Cert for GRPC (Required for secure gRPC communication)
# Contact admin to get the proper certs.
# Backend TLS certificate (issued by ZTrust Root CA)
CERT_FILE_PATH=/etc/ztrust/certs/ztcontroller.ghtklab.com.crt
# Private key corresponding to the certificate above
KEY_CERT_FILE_PATH=/etc/ztrust/certs/ztcontrollerghtklab.com.key

# ==============================================================================
# MONITORING
# ==============================================================================
GRAFANA_PASSWORD=admin
```

</details>


## Advanced Usage

The `setup.sh` script supports several options for flexible deployment:

```bash
Usage: ./setup.sh [OPTIONS]
Options:
  --skip-mongo        Skip MongoDB setup and container start (use external Mongo)
  --skip-redis        Skip Redis setup and container start (use external Redis)
  --skip-postgres     Skip Postgres setup and container start (use external Postgres)
  --skip-frontend     Skip Frontend setup and container start
  -h, --help          Show this help message
```


## Verification

After running the setup script, check the status of your deployment.

1. **Check Containers**:
   ```bash
   docker compose ps
   ```
   *Ensure all containers are `Up`.*

2. **Check Logs**:
   ```bash
   # Headscale
   docker logs -f headscale_zt

   # Controller
   docker logs -f ztrust_be
   
   # Frontend
   docker logs -f ztrust_fe
   ```

## Uninstall

To uninstall the application, run the following command:

> [!IMPORTANT]
> This will remove all containers, networks, volumes, and images created by the setup script.
>


```bash
sudo ./cleanup.sh
```