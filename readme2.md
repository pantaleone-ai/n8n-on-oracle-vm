# Setting Up Your AI Stack on Oracle Cloud Infrastructure (OCI) Always Free Tier with Docker Compose

This guide provides step-by-step instructions for creating an OCI instance using the Always Free Tier, deploying a multi-service Docker Compose setup on Ubuntu 22.04, and optimizing for AI workloads with a LLaMA.cpp-optimized boot image. It builds on [pantaleone-ai/n8n-on-oracle-vm](https://github.com/pantaleone-ai/n8n-on-oracle-vm) but is tailored to your configuration, using Traefik for reverse proxy and SSL/TLS. It includes enabling OCI server agents, automatic backups, and example configuration files for `docker-compose.yml` and `traefik_dynamic.yml`. Last updated: **12:01 PM MDT, Saturday, May 24, 2025**.

## Overview of Modules in `docker-compose.yml`

Your Docker Compose setup includes the following services, each serving a specific purpose in your AI and automation stack:

- **postgres**: A PostgreSQL database (version 13) used as the backend for `n8n` to store workflows and data.
- **n8n**: A workflow automation tool (`n8nio/n8n:latest`) for creating and managing automated workflows, accessible at `n8n.yourdomain.com`.
- **ollama**: An AI model server (`ollama/ollama:latest`) for running large language models locally, used by `openwebui`.
- **qdrant**: A vector search engine (`qdrant/qdrant:latest`) for similarity search and embeddings storage, useful for AI applications.
- **llamacpp**: An Ampere-optimized server (`amperecomputingai/llama.cpp:latest`) for running LLaMA models efficiently, also used by `openwebui`.
- **openwebui**: A web interface (`ghcr.io/open-webui/open-webui:main`) for interacting with AI models from `ollama` and `llamacpp`, accessible at `openwebui.yourdomain.com`.
- **traefik**: A reverse proxy (`traefik:v2.9`) that handles routing and SSL/TLS certificates via Let’s Encrypt, directing traffic to services like `n8n`, `openwebui`, and `grafana`.
- **prometheus**: A monitoring system (`prom/prometheus:v2.37.8`) for collecting and storing metrics from your services.
- **grafana**: A visualization platform (`grafana/grafana:9.1.0`) for creating dashboards to monitor metrics from `prometheus`, accessible at `grafana.yourdomain.com`.
- **redis**: An in-memory data store (`redis:7.0.5`) used for caching and fast data access.
- **crawl4ai**: A web scraping tool (`unclecode/crawl4ai:latest`) for extracting data from websites, useful for AI training or data collection.

## Prerequisites

- An Oracle Cloud account with Always Free Tier access. Sign up at [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/).
- A domain name (e.g., `yourdomain.com`) with access to its DNS settings (e.g., via Namecheap, GoDaddy, or Google Domains).
- Basic familiarity with SSH and terminal commands.

## Step 1: Sign Up for Oracle Cloud and Create a Compute Instance with Ampere A1 Cores

The OCI Always Free Tier provides 4 Ampere A1 OCPUs and 24GB of RAM, which can be used to create up to 4 instances. We’ll configure one instance with the full 4 OCPUs and 24GB RAM, using a LLaMA.cpp-optimized boot image.

1. **Sign Up for Oracle Cloud**:
   - Visit [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) and create an account.
   - Provide a credit card for identity verification (no charges apply if you stay within Free Tier limits). A temporary $100 authorization may be placed but will not be charged.

2. **Create a Compute Instance**:
   - Log in to the [Oracle Cloud Console](https://cloud.oracle.com/).
   - Navigate to **Compute** > **Instances**.
   - Click **Create Instance** and configure as follows:
     - **Name**: Enter a name (e.g., `ai-stack`).
     - **Compartment**: Use the default compartment.
     - **Placement**: Select any Availability Domain. If you encounter an "Out of Capacity" error, try a different region or wait a few hours, as capacity fluctuates.
     - **Image and Shape**:
       - **Shape**: Select **VM.Standard.A1.Flex** (an ARM-based Free Tier option). Ensure it displays the "Always Free-eligible" label.
       - **OCPUs and Memory**: Set to 4 OCPUs and 24GB RAM to utilize the full Free Tier allowance for Ampere A1.
       - **Image**: Click **Change Image**. Select **Canonical Ubuntu 22.04** as the base image, which is compatible with LLaMA.cpp and marked as "Always Free Eligible". This image supports ARM architecture and is suitable for AI workloads.
     - **Networking**: Accept the default settings, ensuring the instance is in a public subnet with a public IP assigned.
     - **Add SSH Keys**: Upload your public SSH key. If you don’t have one, generate it using [GitHub’s SSH key guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).
     - **Boot Volume**:
       - Use the default size (47GB minimum). The Free Tier provides 200GB total block volume storage, including boot volumes. This instance will use up to 47GB, leaving room for additional volumes if needed.
       - Enable **Automatic Backups**: Check the box for **Enable automatic backups** to create daily backups of your boot volume. The Free Tier includes 5 volume backups, which you can manage manually if needed.
   - Click **Create** and wait for the instance to provision (status: green).

3. **Enable OCI Server Agents**:
   - Navigate to **Compute** > **Instances** and select your instance (`ai-stack`).
   - Go to **Resources** > **Oracle Cloud Agent**.
   - Enable the following plugins to allow monitoring and management:
     - **Compute Instance Monitoring**: For metrics collection.
     - **Compute Instance Run Command**: For executing commands remotely.
     - **Management Agent**: For deploying management agents to monitor your instance.
   - These agents are part of the Free Tier and help with instance monitoring and management.

4. **Record the Public IP**:
   - From the instance details page, copy the **Public IP Address**.

## Step 2: Connect to Your Instance and Perform Initial Setup

1. **SSH into Your Instance**:
   - Open a terminal on your local machine and connect using:
     ```bash
     ssh ubuntu@<public_ip>
     ```
     Replace `<public_ip>` with the IP address from Step 1.

2. **Update the Operating System**:
   - Ensure your system is up-to-date:
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```

3. **Install Docker**:
   - Install Docker with the following commands:
     ```bash
     sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
     curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
     sudo add-apt-repository "deb [arch=arm64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
     sudo apt update
     sudo apt install -y docker-ce
     ```

4. **Install Docker Compose**:
   - Install Docker Compose:
     ```bash
     sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
     sudo chmod +x /usr/local/bin/docker-compose
     ```

5. **Add User to Docker Group**:
   - Allow your user to run Docker commands without `sudo`:
     ```bash
     sudo usermod -aG docker ubuntu
     ```
   - Log out and back in to apply the group change:
     ```bash
     exit
     ssh ubuntu@<public_ip>
     ```

## Step 3: Configure Networking and Security

1. **Set Up Ingress Rules in OCI**:
   - In the Oracle Cloud Console, go to your instance’s details page.
   - Click the **Virtual Cloud Network (VCN)** link (e.g., `vcn-YYYYMMDD-HHMM`).
   - Navigate to **Security Lists** and select the default security list.
   - Click **Add Ingress Rules** and add:
     - **Rule 1**:
       - **Source CIDR**: `0.0.0.0/0`
       - **IP Protocol**: `TCP`
       - **Destination Port Range**: `80`
     - **Rule 2**:
       - **Source CIDR**: `0.0.0.0/0`
       - **IP Protocol**: `TCP`
       - **Destination Port Range**: `443`
   - Click **Add Ingress Rules** to save.

2. **Optional: Configure UFW (Uncomplicated Firewall)**:
   - Install and configure UFW for additional security:
     ```bash
     sudo apt install -y ufw
     sudo ufw allow OpenSSH
     sudo ufw allow 80/tcp
     sudo ufw allow 443/tcp
     sudo ufw enable
     ```
   - Confirm with `y` when prompted.

## Step 4: Set Up a Domain Name and DNS

1. **Configure DNS Records**:
   - Log in to your domain provider’s DNS management console.
   - Create A records for each service subdomain pointing to your instance’s public IP. For example:
     - `n8n.yourdomain.com` → `<public_ip>`
     - `openwebui.yourdomain.com` → `<public_ip>`
     - `grafana.yourdomain.com` → `<public_ip>`
     - Repeat for all services (e.g., `prometheus`, `qdrant`, etc.).
   - DNS propagation may take a few hours.

## Step 5: Prepare the Configuration Files

1. **Create a Project Directory**:
   - On your instance, create a directory for your project:
     ```bash
     mkdir ~/ai && cd ~/ai
     ```

2. **Create a `.env` File**:
   - Create a `.env` file to store environment variables:
     ```bash
     nano .env
     ```
   - Add the following, replacing placeholders with your values:
     ```env
     POSTGRES_USER=postgres
     POSTGRES_PASSWORD=your_secure_password
     POSTGRES_DB=db
     N8N_POSTGRES_DB=n8n-postgres
     TRAEFIK_ACME_EMAIL=your_email@example.com
     TZ=America/New_York  # Adjust to your timezone
     OPENWEBUI_ADMIN_USER=admin
     OPENWEBUI_ADMIN_PASSWORD=your_openwebui_password
     GRAFANA_ADMIN_USER=admin
     GRAFANA_ADMIN_PASSWORD=your_grafana_password
     ```
   - Save and exit (`Ctrl+X`, `Y`, `Enter`).

3. **Create the `docker-compose.yml` File**:
   - Create the `docker-compose.yml` file:
     ```bash
     nano docker-compose.yml
     ```
   - Add the following example configuration, which mirrors your live setup with recommended default values:
     ```yaml
     version: '3.8'

     services:
       # PostgreSQL Database
       postgres:
         image: postgres:13
         container_name: postgres
         environment:
           POSTGRES_USER: ${POSTGRES_USER}
           POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
           POSTGRES_DB: ${POSTGRES_DB}
         volumes:
           - postgres_data:/var/lib/postgresql/data
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '0.5'
               memory: 2g
         healthcheck:
           test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
           interval: 10s
           timeout: 5s
           retries: 5
         restart: unless-stopped

       # n8n Workflow Automation
       n8n:
         image: n8nio/n8n:latest
         container_name: n8n
         environment:
           N8N_COMMUNITY_PACKAGES_ENABLED: 'true'
           NODE_FUNCTION_ALLOW_EXTERNAL: 'true'
           N8N_PUSH_BACKEND: websocket
           N8N_DIAGNOSTICS_ENABLED: 'false'
           N8N_PROTOCOL: 'https'
           N8N_TEMPLATES_ENABLED: 'true'
           N8N_EDITOR_BASE_URL: https://n8n.yourdomain.com
           DB_TYPE: postgres
           DB_POSTGRES_DB: ${N8N_POSTGRES_DB}
           DB_POSTGRES_USER: ${POSTGRES_USER}
           DB_POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
           DB_POSTGRES_HOST: postgres
           DB_POSTGRES_PORT: '5432'
           TZ: ${TZ}
         volumes:
           - n8n_data:/home/node/.n8n
         labels:
           - "traefik.enable=true"
           - "traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)"
           - "traefik.http.routers.n8n.entrypoints=websecure"
           - "traefik.http.routers.n8n.tls=true"
           - "traefik.http.routers.n8n.tls.certresolver=myresolver"
           - "traefik.http.routers.n8n.tls.options=default"
         networks:
           - internal
         deploy:
           resources:
             limits:
               cpus: '0.5'
               memory: 1g
         restart: unless-stopped

       # Ollama AI Model Server
       ollama:
         image: ollama/ollama:latest
         container_name: ollama
         environment:
           TZ: ${TZ}
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '1'
               memory: 4g
         restart: unless-stopped

       # Qdrant Vector Store
       qdrant:
         image: qdrant/qdrant:latest
         container_name: qdrant
         environment:
           TZ: ${TZ}
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '0.5'
               memory: 1g
         restart: unless-stopped

       # Ampere-Optimized llama.cpp Server
       llamacpp:
         image: amperecomputingai/llama.cpp:latest
         container_name: llamacpp
         environment:
           TZ: ${TZ}
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '1'
               memory: 4g
         restart: unless-stopped

       # OpenWebUI AI Interface
       openwebui:
         image: ghcr.io/open-webui/open-webui:main
         container_name: openwebui
         environment:
           OPENAI_API_BASE_URL: http://ollama:11434,http://llamacpp:8080/v1
           OPENAI_API_KEY: sk-fakekey
           ADMIN_USER: ${OPENWEBUI_ADMIN_USER}
           ADMIN_PASSWORD: ${OPENWEBUI_ADMIN_PASSWORD}
           TZ: ${TZ}
         volumes:
           - openwebui_data:/app/backend/data
         labels:
           - "traefik.enable=true"
           - "traefik.http.routers.openwebui.rule=Host(`openwebui.yourdomain.com`)"
           - "traefik.http.routers.openwebui.entrypoints=websecure"
           - "traefik.http.routers.openwebui.tls=true"
           - "traefik.http.routers.openwebui.tls.certresolver=myresolver"
           - "traefik.http.routers.openwebui.tls.options=default"
         networks:
           - internal
         deploy:
           resources:
             limits:
               cpus: '0.5'
               memory: 1g
         restart: unless-stopped

       # Traefik Reverse Proxy
       traefik:
         image: traefik:v2.9  # Matches your live setup; consider upgrading to v2.10
         container_name: traefik
         command:
           - "--providers.docker=true"
           - "--entrypoints.web.address=:80"
           - "--entrypoints.websecure.address=:443"
           - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
           - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
           - "--certificatesresolvers.myresolver.acme.caServer=https://acme-v02.api.letsencrypt.org/directory"
           - "--certificatesresolvers.myresolver.acme.email=${TRAEFIK_ACME_EMAIL}"
           - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
           - "--log.level=DEBUG"
           - "--providers.file.filename=/etc/traefik/traefik_dynamic.yml"
         ports:
           - "80:80"
           - "443:443"
         volumes:
           - /var/run/docker.sock:/var/run/docker.sock
           - letsencrypt:/letsencrypt
           - ./traefik_dynamic.yml:/etc/traefik/traefik_dynamic.yml
         networks:
           - internal
         labels:
           - "traefik.http.routers.http-catchall.rule=HostRegexp(`{host:.+}`)"
           - "traefik.http.routers.http-catchall.entrypoints=web"
           - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
           - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
         deploy:
           resources:
             limits:
               cpus: '0.25'
               memory: 512m
         restart: unless-stopped

       # Prometheus Monitoring
       prometheus:
         image: prom/prometheus:v2.37.8
         container_name: prometheus
         volumes:
           - prometheus_data:/prometheus
           - ./prometheus.yml:/etc/prometheus/prometheus.yml
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '0.25'
               memory: 256m
         restart: unless-stopped

       # Grafana Visualization
       grafana:
         image: grafana/grafana:9.1.0
         container_name: grafana
         environment:
           GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
           GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
           TZ: ${TZ}
         volumes:
           - grafana_data:/var/lib/grafana
         labels:
           - "traefik.enable=true"
           - "traefik.http.routers.grafana.rule=Host(`grafana.yourdomain.com`)"
           - "traefik.http.routers.grafana.entrypoints=websecure"
           - "traefik.http.routers.grafana.tls=true"
           - "traefik.http.routers.grafana.tls.certresolver=myresolver"
           - "traefik.http.routers.grafana.tls.options=default"
         networks:
           - internal
         deploy:
           resources:
             limits:
               cpus: '0.25'
               memory: 256m
         restart: unless-stopped

       # Redis Caching
       redis:
         image: redis:7.0.5
         container_name: redis
         environment:
           TZ: ${TZ}
         networks:
           - internal
         labels:
           - "traefik.enable=false"
         deploy:
           resources:
             limits:
               cpus: '0.25'
               memory: 512m
         healthcheck:
           test: ["CMD", "redis-cli", "ping"]
           interval: 10s
           timeout: 5s
           retries: 5
         restart: unless-stopped

       # Crawl4AI Web Scraping
       crawl4ai:
         image: unclecode/crawl4ai:latest
         container_name: crawl4ai
         environment:
           PLAYWRIGHT_BROWSERS_PATH: /ms-playwright
           TZ: ${TZ}
         volumes:
           - crawl4ai_data:/app/data
         networks:
           - internal
         deploy:
           resources:
             limits:
               cpus: '0.5'
               memory: 1g
         restart: unless-stopped

     networks:
       internal:
         driver: bridge

     volumes:
       postgres_data:
       n8n_data:
       openwebui_data:
       prometheus_data:
       grafana_data:
       letsencrypt:
       crawl4ai_data:
     ```
   - Save and exit.

## Step 6: Configure Traefik for Reverse Proxy and SSL/TLS

1. **Create `traefik_dynamic.yml`**:
   - Create the `traefik_dynamic.yml` file with recommended default values for TLS and middleware settings:
     ```bash
     nano traefik_dynamic.yml
     ```
   - Add:
     ```yaml
     tls:
       options:
         default:
           minVersion: VersionTLS12
           cipherSuites:
             - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
             - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
           curvePreferences:
             - CurveP256
             - CurveP384

     http:
       middlewares:
         rate-limit:
           rateLimit:
             average: 100
             burst: 50
         security-headers:
           headers:
             sslRedirect: true
             stsSeconds: 31536000
             stsIncludeSubdomains: true
             stsPreload: true
             forceSTSHeader: true
             frameDeny: true
             contentTypeNosniff: true
             browserXssFilter: true
     ```
   - Save and exit.
   - This configuration enforces TLS 1.2+, uses secure ciphers, adds rate limiting, and includes security headers for better protection.

2. **Verify Traefik Configuration in `docker-compose.yml`**:
   - The Traefik service in the example `docker-compose.yml` above matches your live setup:
     - Uses `traefik:v2.9`.
     - Configures HTTP to HTTPS redirection.
     - Uses the `httpchallenge` for Let’s Encrypt SSL certificates.
   - Adjust subdomains in service labels (e.g., `traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)`).
   - **Note**: Optionally upgrade to `traefik:v2.10` for improved stability by changing the `image` tag and testing thoroughly.

## Step 7: Deploy the Docker Compose Stack

1. **Start the Stack**:
   - From the `~/ai` directory, run:
     ```bash
     docker-compose up -d
     ```

2. **Check Container Status**:
   - Verify all services are running:
     ```bash
     docker-compose ps
     ```
   - Look for `State: Up` for each service.

3. **View Logs (if Needed)**:
   - Check logs for a specific service:
     ```bash
     docker-compose logs -f <service_name>
     ```
     Example: `docker-compose logs -f n8n` or `docker-compose logs -f traefik`.

## Step 8: Verify the Setup

1. **Access Services**:
   - Wait a few minutes for services to initialize and Traefik to obtain SSL certificates.
   - Test access via your browser:
     - n8n: `https://n8n.yourdomain.com`
     - OpenWebUI: `https://openwebui.yourdomain.com`
     - Grafana: `https://grafana.yourdomain.com`
     - Repeat for other services.

2. **Verify Automatic Backups**:
   - In the OCI Console, go to **Block Storage** > **Boot Volume Backups**.
   - Confirm that automatic backups are being created daily for your instance’s boot volume. Manage backups if you reach the Free Tier limit of 5.

3. **Monitor Instance with OCI Agents**:
   - Navigate to **Observability & Management** > **Monitoring** in the OCI Console.
   - Check metrics for your instance (e.g., CPU, memory usage) collected by the enabled OCI server agents.

4. **Troubleshooting**:
   - If a service isn’t accessible:
     - Check its logs: `docker-compose logs -f <service_name>`.
     - Verify DNS propagation: `ping n8n.yourdomain.com`.
     - Ensure Traefik has SSL certificates:
       ```bash
       docker volume inspect ai_letsencrypt
       ```
       Look for `acme.json` in the volume’s mount point.

## Additional Notes

- **Free Tier Limits**: The Always Free Tier allows 4 Ampere A1 OCPUs, 24GB RAM, and 200GB block volume storage. This setup uses one instance with the full 4 OCPUs and 24GB RAM, leaving 153GB of block storage for additional volumes or instances.
- **ARM Compatibility**: The `VM.Standard.A1.Flex` shape uses Ampere A1 cores, optimized for LLaMA.cpp. Ensure all Docker images (e.g., `amperecomputingai/llama.cpp:latest`) support ARM architecture.
- **Security**: Keep your `.env` file private and use strong passwords.
- **Maintenance**: Regularly update your setup:
  ```bash
  sudo apt update && sudo apt upgrade -y
  docker-compose pull
  docker-compose up -d
  ```
- **Idle Instances**: OCI may reclaim idle Free Tier instances after 7 days of low usage (CPU <20%). Keep your instance active or use cron jobs to simulate activity.

---
