# **How to Set Up Your Own Custom DERP Server**

Welcome! This guide will walk you through building and running your very own **DERP (Detoured Encrypted Routing Protocol) server** using Tailscale's source code. This is perfect for enterprise environments where you need a private relay infrastructure to boost performance and keep latency low.

------

## **1. System Requirements**

- **For ~1000 users**, we recommend:
  - **CPU:** 4 vCPU
  - **RAM:** 4 GB
  - **Disk:** 30 GB (SSD)
  - **Bandwidth:** 500 Mbps – 1 Gbps
  - **NIC:** 1 Gbps+
  - **High Availability (HA):** A cluster of 3 DERP servers running in a mesh.
- **OS:** Ubuntu 20.04/22.04 (or similar).
- **Go:** Version 1.22 or newer.
- **Firewall Rules** (if applicable):
  - **443/TCP** – WebSocket & HTTPS
  - **UDP/3478** – STUN
- **DNS:** Point your hostname to the server's public IP.

Example:

```
ztrust-relay-02.ghtklab.com 157.66.96.198
```

------

## **2. Setting Up Go**

First, let's get Go installed:

```
sudo apt update
sudo apt install -y golang git
```

Let's verify the installation:

```
go version
```

------

## **3. Build `derper` from Source**

Now, we'll clone the Tailscale repository and build the `derper` binary:

```
git clone https://github.com/tailscale/tailscale.git
cd tailscale/cmd/derper
go build
```

Once built, you'll find the **derper** binary here:

```
./derper
```

We recommend moving it to `$HOME/go/bin` or `/usr/local/bin` for easy access:

```
mkdir -p $HOME/go/bin
cp derper $HOME/go/bin/
```

------

## **4. Prepare Data & Mesh Keys**

Create directories for certificates and the mesh key:

```
sudo mkdir -p /var/lib/derper
sudo mkdir -p /home/dongtv28/.cache/tailscale/derper-certs
sudo chown -R root:root /var/lib/derper
```

Generate a shared mesh key for your DERP servers:

```
openssl rand -base64 32 | sudo tee /var/lib/derper/mesh.key
sudo chmod 600 /var/lib/derper/mesh.key
```

**Note:** This key must be identical across all nodes in your DERP mesh.

------

## **5. Configure Firewall**

If you have a firewall enabled, allow the necessary ports:

```
sudo ufw allow 443/tcp
sudo ufw allow 3478/udp
```

------

## **6. Run DERP as a Service**

Create the systemd service file:

```
sudo nano /etc/systemd/system/derper.service
```

Paste in the following configuration (adjust paths as needed):

```bash
[Unit]
Description=Custom DERP Server
After=network.target

[Service]
ExecStart=/home/dongtv28/go/bin/derper \
  --hostname ztrust-relay-02.ghtklab.com \
  --certmode manual \
  --certdir /home/dongtv28/.cache/tailscale/derper-certs \ # Directory containing certs
  -mesh-with ztrust-relay-01.ghtklab.com,ztrust-relay-03.ghtklab.com \ # Other custom DERPs in the org to mesh with
  -mesh-psk-file /var/lib/derper/mesh.key # Private key for encryption between org DERPs
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

------

## **7. Start the Service**

Enable and start your new service:

```
sudo systemctl daemon-reload
sudo systemctl enable derper
sudo systemctl start derper
sudo systemctl status derper
```

------

## **8. Configure SSL Certificates**

Since we're using `--certmode manual`, you'll need to provide your own TLS certificates.

### Option 1: Let's Encrypt + Certbot

```
sudo apt install certbot
sudo certbot certonly --standalone -d ztrust-relay-02.ghtklab.com
```

Copy the certificates to your `certdir`:

```
/home/dongtv28/.cache/tailscale/derper-certs/
```

Ensure they are named as follows:

- `ztrust-relay-02.ghtklab.com.crt`
- `ztrust-relay-02.ghtklab.com.key`

------

### Option 2: Enterprise Internal Certificates

Simply place your certificate files into the `certdir` folder.

<div>
 <img src="https://cache.giaohangtietkiem.vn/d/b929a6bc3460f8cb108facddf00126cb.png">
</div>

------

## **9. Verify Installation**

### Check STUN Port:

```
sudo ss -ulpn | grep 3478
```

### Check HTTPS/WebSocket:

Open your browser and visit:

```
https://ztrust-relay-02.ghtklab.com
```

If you see "DERP server OK", you're good to go!

------

## **10. Update Headscale Configuration**

Create a `DERPMAP` file with the following content:

```json
{       
       "999": {
            "RegionID": 999,
            "RegionCode": "zlab",
            "RegionName": "GHTK ZLab",
            "Latitude": 21.1393,
            "Longitude": 105.8666,
            "Nodes": [               
                {
                    "Name": "z",
                    "RegionID": 999,
                    "HostName": "ztrust-relay-02.ghtklab.com",
                    "IPv4": "157.66.96.198", <!-- Public IP of derper server -->
                    "IPv6": "",
                    "CanPort80": true
                }
            ]
        }
    }
}
```

Host this `DERPMAP` file on your organization's CDN, then update your Headscale `config.yaml` to point to it. (Make sure your Headscale controller can reach this URL):

![image-20251126170922098](https://cache.giaohangtietkiem.vn/d/9a7c5161dc52c7b3804ff3bb42295ecb.png)

------

## **11. Operations & Monitoring**

- **View Logs:**

```
sudo journalctl -u derper -f
```

- **Restart Service:**

```
sudo systemctl restart derper
```

- **Monitor Metrics:** Keep an eye on CPU, RAM, Disk, and Network traffic. Since DERP handles a lot of WebRTC/DERP/Noise traffic, monitoring bandwidth and network errors is crucial.




------

