# Installation Guide

## 1. System Requirements

| Component | Minimum (Lab / PoC) | Recommended ~500 agents (Production) |
|-----------|----------------------|---------------------------|
| CPU       | 4 cores              | 16 cores                 |
| RAM       | 8 GB                 | 24 GB (≥ 32 GB if many service, users, devices) |
| Storage   | 30 GB SSD            | 100 GB SSD / NVMe (expandable) |
| OS        | Linux (Ubuntu/CentOS recommended) | Linux (Ubuntu/CentOS recommended) |
| Docker    | Docker & Docker Compose installed | Docker & Docker Compose installed |
| Domain    | Optional (can use local IP) | Public domain with valid SSL certificate (no self-signed) for Backend and Headscale, internal domain for Frontend |
| Network   | Local network        | Public network, Firewall, Reverse Proxy (Nginx/Caddy) - Port 80/443 HTTP, 50051 TCP/GRPC|

## 2. Download

```bash

git clone https://github.com/ZTrustWorks/ztrust_deploy.git
```

## 3. Configuration

```bash

cd ztrust_deploy
```

## 3.1. Config Headscale

You can get the Headscale config file from the 

```bash

```
##