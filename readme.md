***

# n8n on Google Cloud Free Tier (Podman + Traefik)

This guide details how to host [n8n](https://n8n.io/) on a **Google Cloud Platform (GCP) Always Free** instance (`e2-micro`) using **Podman** (Rootless) and **Traefik** for automatic SSL.

## Prerequisites
1.  A Google Cloud Platform account.
2.  A domain name (e.g., `example.com`) with DNS records managed (e.g., Cloudflare, Namecheap).

---

## Step 1: Create the VM Instance
1.  Go to **Compute Engine** > **VM Instances** > **Create Instance**.
2.  **Region:** `us-west1`, `us-central1`, or `us-east1` (Required for Free Tier).
3.  **Machine Type:** `e2-micro` (2 vCPU, 1 GB memory).
4.  **Boot Disk:**
    *   OS: **Debian GNU/Linux 12 (bookworm)**
    *   Disk Type: **Standard persistent disk**
    *   Size: **30 GB**
5.  **Firewall:** Check "Allow HTTP traffic" and "Allow HTTPS traffic".
6.  Click **Create**.

## Step 2: Configure Firewall
1.  Go to **VPC Network** > **Firewall**.
2.  Create a rule named `allow-http-https`.
3.  **Targets:** All instances in the network.
4.  **Source IPv4 ranges:** `0.0.0.0/0`.
5.  **Protocols and ports:** TCP `80`, `443`.

## Step 3: DNS Configuration
Point your domain's **A Record** to the VM's **External IP**.
*   **Type:** A
*   **Name:** `n8n` (or `@` for root)
*   **Value:** `YOUR_GCP_EXTERNAL_IP`
*   *Note: If using Cloudflare, set Proxy status to "DNS Only" (Grey Cloud) initially to ensure certificate generation works.*

## Step 4: Server Setup (SSH)
Connect to your VM via SSH and run the following commands to prepare the environment.

### 1. Create Swap File (Crucial for 1GB RAM)
Prevent crashes by adding 2GB of swap memory.
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
# Adjust swappiness
sudo sysctl vm.swappiness=20
echo 'vm.swappiness=20' | sudo tee -a /etc/sysctl.conf
```

### 2. Install Podman & Dependencies
```bash
sudo apt-get update
sudo apt-get install -y podman podman-compose dbus-user-session uidmap
```

### 3. Configure Rootless Ports & Persistence
Allow the user to bind ports 80/443 and keep processes running after logout.
```bash
# Allow binding port 80/443
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Enable Podman socket
systemctl --user enable --now podman.socket

# Enable Linger (Keep server running when SSH closes)
sudo loginctl enable-linger $USER
```

**Recommended:** Reboot the VM now (`sudo reboot`) to ensure all user permissions and groups are applied correctly. Reconnect after 1 minute.

## Step 5: Installation

### 1. Create Directory Structure
```bash
mkdir -p n8n-docker-caddy/local-files
cd n8n-docker-caddy
```

### 2. Create `.env` File
Create a file named `.env` and add your details:
```bash
nano .env
```
**Content:**
```ini
DOMAIN_NAME=yourdomain.com
SUBDOMAIN=n8n
SSL_EMAIL=youremail@gmail.com
GENERIC_TIMEZONE=UTC
UID=1000
```

### 3. Create `docker-compose.yml`
```bash
nano docker-compose.yml
```
**Content:**
```yaml
services:
  traefik:
    image: "docker.io/traefik"
    restart: always
    security_opt:
      - label=disable
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      # HTTP Challenge (More reliable for this setup)
      - "--certificatesresolvers.mytlschallenge.acme.httpchallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt:z
      # Map the rootless podman socket to the internal docker socket
      - /run/user/${UID}/podman/podman.sock:/var/run/docker.sock:z

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      # Fix for Mixed Content / UI loading issues
      - N8N_EDITOR_BASE_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
      # Optimization for 1GB RAM Server
      - EXECUTIONS_PROCESS=main
    volumes:
      - n8n_data:/home/node/.n8n:z
      - ./local-files:/files:z

volumes:
  n8n_data:
  traefik_data:
```

### 4. Start the Services
```bash
podman-compose up -d
```

## Verification
1.  Wait 30-60 seconds for the SSL certificate to generate.
2.  Navigate to `https://n8n.yourdomain.com`.
3.  Create your owner account.

## Troubleshooting
*   **Check Logs:** `podman logs traefik` or `podman logs n8n`.
*   **Restart:** `podman-compose down` followed by `podman-compose up -d`.
*   **Reset SSL:** If certificates fail, remove the volume: `podman volume rm n8n-docker-caddy_traefik_data`.
