

# Setting Up Your AI Stack on Oracle Cloud Infrastructure (OCI) with Docker Compose

This guide provides step-by-step instructions for creating an OCI instance and deploying your multi-service Docker Compose setup on Ubuntu 22.04. It builds on the foundation from [pantaleone-ai/n8n-on-oracle-vm](https://github.com/pantaleone-ai/n8n-on-oracle-vm) but is tailored to your specific configuration, using Traefik for reverse proxy and SSL/TLS. Last updated: **10:49 AM MDT, May 24, 2025**.

## Prerequisites

- An Oracle Cloud account with Free Tier access. Sign up at [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/).
- A domain name (e.g., `yourdomain.com`) with access to its DNS settings (e.g., via Namecheap, GoDaddy, or Google Domains).
- Basic familiarity with SSH and terminal commands.
- Your live `docker-compose.yml` file, which includes services such as `n8n`, `postgres`, `ollama`, `qdrant`, `llamacpp`, `openwebui`, `traefik`, `prometheus`, `grafana`, `redis`, and `crawl4ai`.

## Step 1: Sign Up for Oracle Cloud and Create a Compute Instance

1. **Sign Up for Oracle Cloud**:
   - Visit [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) and create an account.
   - Provide a credit card for identity verification (no charges apply if you stay within Free Tier limits).

2. **Create a Compute Instance**:
   - Log in to the [Oracle Cloud Console](https://cloud.oracle.com/).
   - Navigate to **Compute** > **Instances**.
   - Click **Create Instance** and configure as follows:
     - **Name**: Enter a name (e.g., `ai-stack`).
     - **Compartment**: Use the default compartment.
     - **Placement**: Select any Availability Domain.
     - **Image and Shape**:
       - **Image**: Choose **Canonical Ubuntu 22.04** (or the latest LTS version available).
       - **Shape**: Select **VM.Standard.A1.Flex** (an ARM-based Free Tier option) or another Free Tier-eligible shape.
     - **Networking**: Accept the default settings.
     - **Add SSH Keys**: Upload your public SSH key. If you don’t have one, generate it using [GitHub’s SSH key guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).
     - **Boot Volume**: Leave as default.
   - Click **Create** and wait for the instance to provision.

3. **Record the Public IP**:
   - Once the instance is running (status: green), go to its details page and copy the **Public IP Address**.

---

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
     sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
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

---

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

---

## Step 4: Set Up a Domain Name and DNS

1. **Configure DNS Records**:
   - Log in to your domain provider’s DNS management console.
   - Create A records for each service subdomain pointing to your instance’s public IP. For example:
     - `n8n.yourdomain.com` → `<public_ip>`
     - `openwebui.yourdomain.com` → `<public_ip>`
     - `grafana.yourdomain.com` → `<public_ip>`
     - Repeat for all services (e.g., `prometheus`, `qdrant`, etc.).
   - DNS propagation may take a few hours.

---

## Step 5: Prepare the Docker Compose File

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

3. **Add Your `docker-compose.yml`**:
   - Create the `docker-compose.yml` file:
     ```bash
     nano docker-compose.yml
     ```
   - Paste your live Docker Compose configuration, ensuring it:
     - References variables from the `.env` file (e.g., `${POSTGRES_USER}`).
     - Includes Traefik labels for routing (e.g., `traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)`).
     - Defines volumes and networks as needed.
   - Save and exit.

---

## Step 6: Configure Traefik for Reverse Proxy and SSL/TLS

1. **Create `traefik_dynamic.yml`**:
   - Since your setup uses Traefik, create a dynamic configuration file:
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
     ```
   - Save and exit.

2. **Verify Traefik Configuration in `docker-compose.yml`**:
   - Ensure your Traefik service includes:
     ```yaml
     traefik:
       image: traefik:v2.10
       command:
         - "--api.insecure=true"
         - "--providers.docker=true"
         - "--entrypoints.web.address=:80"
         - "--entrypoints.websecure.address=:443"
         - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
         - "--certificatesresolvers.myresolver.acme.email=${TRAEFIK_ACME_EMAIL}"
         - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock
         - letsencrypt:/letsencrypt
         - ./traefik_dynamic.yml:/etc/traefik/traefik_dynamic.yml
     ```
   - Adjust subdomains in service labels (e.g., `traefik.http.routers.n8n.rule=Host(`n8n.yourdomain.com`)`).

---

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

---

## Step 8: Verify the Setup

1. **Access Services**:
   - Wait a few minutes for services to initialize and Traefik to obtain SSL certificates.
   - Test access via your browser:
     - n8n: `https://n8n.yourdomain.com`
     - OpenWebUI: `https://openwebui.yourdomain.com`
     - Grafana: `https://grafana.yourdomain.com`
     - Repeat for other services.

2. **Troubleshooting**:
   - If a service isn’t accessible:
     - Check its logs: `docker-compose logs -f <service_name>`.
     - Verify DNS propagation: `ping n8n.yourdomain.com`.
     - Ensure Traefik has SSL certificates:
       ```bash
       docker volume inspect ai_letsencrypt
       ```
       Look for `acme.json` in the volume’s mount point.

---

## Additional Notes

- **Resource Limits**: Adjust CPU and memory limits in `docker-compose.yml` if your instance struggles (e.g., `deploy: resources: limits: cpus: '1.0' memory: 1g`).
- **ARM Compatibility**: If using VM.Standard.A1.Flex (ARM), ensure all Docker images support ARM architecture.
- **Security**: Keep your `.env` file private and use strong passwords.
- **Maintenance**: Regularly update your setup:
  ```bash
  sudo apt update && sudo apt upgrade -y
  docker-compose pull
  docker-compose up -d
  ```

---
