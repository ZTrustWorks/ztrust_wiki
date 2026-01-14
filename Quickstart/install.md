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
4. **`ZTRUST_LICENSE_ACTIVATION_CODE`**: A code for active license.
   
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
