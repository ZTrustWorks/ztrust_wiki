# Nginx Reverse Proxy Setup

This guide provides step-by-step instructions for setting up Nginx as a reverse proxy for the **ZTrust Controller (Backend)** and **ZTrust Portal (Frontend)**.

## Prerequisites

Before you begin, ensure you have the following:

-   **Nginx** installed on your server.
-   **SSL Certificates** for your domains.
-   **Network Access**:
    -   Backend domain must be accessible via HTTP (`80`, `443`) and GRPC (`50051`).
    -   Internal API should only be accessible from the frontend server.
-   **Service Ports**:
    -   Backend (ZTrust Controller): `8090`
    -   Frontend (ZTrust Portal): `33000`
    -   Headscale: `8080`

---
## 1. Certificate Setup (Skip if you have a public domain with valid SSL certificate)

### Step 1: Install Certbot

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

### Step 2: Generate Self-Signed Certificate

```bash
# For ZTrust Controller and Portal
sudo certbot --nginx -d ztcontroller.nextzero.vn -d ztrust.nextzero.vn

```

Enter your email address and agree to the terms of service.


## 2. Backend Setup (ZTrust Controller)

### Step 1: Prepare Directories and Certificates

Create the necessary directories for Nginx configuration and SSL certificates.

```bash
sudo mkdir -p /etc/nginx/sites-available /etc/nginx/sites-enabled
sudo mkdir -p /etc/ssl/certs /etc/ssl/private
```

Copy your SSL certificate and private key to the appropriate locations.
*Replace `ztcontroller.nextzero.vn` with your actual domain name.*

```bash
sudo cp ztcontroller.nextzero.vn.crt /etc/ssl/certs/
sudo cp ztcontroller.nextzero.vn.key /etc/ssl/private/
sudo chmod 640 /etc/ssl/private/ztcontroller.nextzero.vn.key
sudo chown root:nginx /etc/ssl/private/ztcontroller.nextzero.vn.key
```

### Step 2: Create Nginx Configuration

Create a new configuration file for the backend.

```bash
sudo nano /etc/nginx/sites-available/ztcontroller.nextzero.vn.conf
```

Paste the following configuration. **Note:** This configuration handles Public API, Internal API, and Headscale traffic.

<details>
<summary><strong>Click to view `ztcontroller.nextzero.vn.conf`</strong></summary>

```nginx
# Upstream definitions
upstream ztrust_be {
    server 127.0.0.1:8090 max_fails=2 fail_timeout=10s;
}

upstream headscale_ctl {
    server 127.0.0.1:8080 max_fails=2 fail_timeout=10s;
}

# WebSocket upgrade mapping
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name ztcontroller.nextzero.vn internal-ztcontroller.nextzero.vn; # Replace with your domains

    # NOTE:
    # If you DON'T have a wildcard certificate (like *.nextzero.vn),
    # do NOT reuse one certificate for multiple domains.
    #
    # Each domain must have its OWN server block and certificate.
    # Example:
    #   server_name ztcontroller.nextzero.vn
    #   server_name internal-ztcontroller.nextzero.vn
    #
    # Using the wrong certificate will cause TLS / certificate mismatch errors. 
    
    # SSL Configuration
    ssl_certificate     /etc/ssl/certs/ztcontroller.nextzero.vn.crt;
    ssl_certificate_key /etc/ssl/private/ztcontroller.nextzero.vn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # --- WebSocket Support ---
    location ~* ^/internal/api/v1/.*/ws$ {
        # Access Control - Update IPs as needed
        allow 127.0.0.1;
        allow 10.110.73.90/32; # Example IP gateway of frontend
        allow 172.23.0.0/16;   # Gateway testing
        deny all;

        proxy_pass         http://ztrust_be;
        proxy_http_version 1.1;

        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "Upgrade";
        proxy_set_header   Host $host;

        # Forward Sec-WebSocket-* headers
        proxy_set_header   Sec-WebSocket-Key         $http_sec_websocket_key;
        proxy_set_header   Sec-WebSocket-Version     $http_sec_websocket_version;
        proxy_set_header   Sec-WebSocket-Extensions  $http_sec_websocket_extensions;
        proxy_set_header   Sec-WebSocket-Protocol    $http_sec_websocket_protocol;

        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   Authorization $http_authorization;
        
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
        proxy_buffering    off;
    }

    # --- Internal API Routes ---
    # Restricted access for Frontend and ZTrust connection
    location /internal/ {
        allow 127.0.0.1;
        allow 10.110.85.45; # Example IP
        allow 10.110.73.90; # Example IP
        allow 172.23.0.0/16;
        allow 172.21.0.0/16;
        deny all;

        proxy_pass http://ztrust_be;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # --- Public API Routes ---
    location /public/ {
        proxy_pass http://ztrust_be;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # --- Callback Login Agent ---
    location /open/ {
        proxy_pass http://127.0.0.1:33000/open/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # --- Static Files (Next.js) ---
    location /_next/ {
        proxy_pass http://127.0.0.1:33000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # --- Next.js Internals ---
    location /__nextjs_original-stack-frame {
        proxy_pass http://127.0.0.1:33000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
    }

    location /_next/webpack-hmr {
        proxy_pass http://127.0.0.1:33000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
    }

    # --- Headscale Routes ---
    location / {
        proxy_pass http://headscale_ctl;
        proxy_http_version 1.1;

        # Keep long-lived connections (noise handshake + DERP)
        proxy_request_buffering off;
        proxy_buffering off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- GRPC Endpoint (Optional) ---
    location /Headscale.v1.HeadscaleService/ {
        grpc_pass grpc://127.0.0.1:50443;
        grpc_set_header X-Real-IP $remote_addr;
        grpc_read_timeout 3600s;
        grpc_send_timeout 3600s;
    }

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}

# HTTP Redirect to HTTPS
server {
    listen 80;
    server_name ztcontroller.nextzero.vn;
    return 301 https://$host$request_uri;
}
```
</details>

### Step 3: Enable and Verify

Enable the configuration and test it.

```bash
sudo ln -s /etc/nginx/sites-available/ztcontroller.nextzero.vn.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

If the test is successful, restart Nginx:

```bash
sudo systemctl restart nginx
```

### Step 4: Verification

Run the following commands to verify the backend setup:

```bash
# Check if port 443 is listening
sudo ss -tlnp | grep 443

# Check SSL Certificate
openssl s_client -connect ztcontroller.nextzero.vn:443 -servername ztcontroller.nextzero.vn </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Check API Routes
curl -I https://ztcontroller.nextzero.vn/public/healthz
curl -X POST https://internal-ztcontroller.nextzero.vn/internal/api/v1/auth/login -k -H "Content-Type: application/json" -d '{"username":"admin","password":"admin"}'  # Should work from allowed IPs
curl -I https://ztcontroller.nextzero.vn/                          # Headscale check
```

---

## 2. Frontend Setup (ZTrust Portal)

### Step 1: Create Nginx Configuration

Create a configuration file for the frontend (e.g., `/etc/nginx/sites-available/ztrust.nextzero.vn.conf`).

<details>
<summary><strong>Click to view `ztrust.nextzero.vn.conf`</strong></summary>

```nginx
server {
    listen 443 ssl;
    server_name ztrust.nextzero.vn; # Replace with your domain

    ssl_certificate     /etc/ssl/certs/ztrust.nextzero.vn.crt; # Path to your certificate
    ssl_certificate_key /etc/ssl/private/ztrust.nextzero.vn.key; # Path to your key
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:33000; # Forward to frontend container
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name ztrust.nextzero.vn; # Replace with your domain
    return 301 https://$host$request_uri;
}
```
</details>

### Step 2: Enable and Restart

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/ztrust.nextzero.vn.conf /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

---

## Troubleshooting

If you encounter issues, use these commands to debug:

-   **Check Nginx Status**: `sudo systemctl status nginx`
-   **View Nginx Logs**: `sudo tail -f /var/log/nginx/error.log`
-   **Check Access Logs**: `sudo tail -f /var/log/nginx/access.log`
