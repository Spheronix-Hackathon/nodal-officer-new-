# Spheronix Hackathon - AWS Production Deployment Guide

This guide details the step-by-step process for deploying this FastAPI + MongoDB Hackathon application to **Amazon Web Services (AWS)** using **EC2 + Nginx + Systemd + Certbot (SSL)**.

---

## Architecture Overview

* **Cloud Server:** AWS EC2 (Ubuntu 22.04 LTS or 24.04 LTS)
* **Web Server / Reverse Proxy:** Nginx (Handles SSL termination, port 80 -> 443 redirect, compression)
* **Application Server:** Gunicorn with Uvicorn async workers (`127.0.0.1:8000`)
* **Process Manager:** Linux `systemd` (`hackathon.service`)
* **Database:** MongoDB Atlas (Cloud) or Self-Hosted MongoDB
* **SSL / TLS:** Let's Encrypt (Certbot) with auto-renewal
* **App Directory:** `/opt/hackathon/app`

---

## Step 1: AWS EC2 Instance Setup

1. **Log in to AWS Console** and navigate to **EC2** > **Launch Instance**.
2. **Configuration**:
   * **Name**: `spheronix-hackathon-prod`
   * **OS / AMI**: `Ubuntu Server 22.04 LTS (HVM)` or `24.04 LTS` (64-bit x86)
   * **Instance Type**: `t3.small` (2 vCPU, 2 GB RAM) or `t3.medium` (2 vCPU, 4 GB RAM recommended for traffic surges).
   * **Key Pair**: Create a new key pair or choose an existing `.pem` key (e.g., `hackathon-key.pem`). Download and store it securely.
3. **Network & Security Group**:
   * Create a security group with the following inbound rules:
     * **SSH (Port 22)**: Source = `My IP` (recommended for security) or `0.0.0.0/0`
     * **HTTP (Port 80)**: Source = `Anywhere (0.0.0.0/0)`
     * **HTTPS (Port 443)**: Source = `Anywhere (0.0.0.0/0)`
4. **Storage**: 20 GB to 30 GB gp3 SSD.
5. Click **Launch Instance**.

### Allocate an Elastic IP (Static Public IP)
1. Go to **EC2** > **Network & Security** > **Elastic IPs**.
2. Click **Allocate Elastic IP address**.
3. Select the IP > **Actions** > **Associate Elastic IP address**.
4. Choose your running EC2 instance and click **Associate**.
*(This ensures your server IP never changes when rebooted).*

---

## Step 2: Configure DNS (Domain Name)

1. Open your domain DNS provider (Route 53, GoDaddy, Cloudflare, Namecheap, etc.).
2. Add an **A Record**:
   * **Host / Name**: `@` (or your subdomain, e.g., `hackathon`)
   * **Value / Points to**: Your EC2 Elastic IP
3. Add a **CNAME** or **A Record** for `www`:
   * **Host**: `www`
   * **Value**: Your Elastic IP or root domain.

---

## Step 3: Connect to EC2 & Install System Dependencies

Open terminal (or PowerShell/PuTTY) on your local machine:

```bash
chmod 400 hackathon-key.pem
ssh -i hackathon-key.pem ubuntu@<YOUR_ELASTIC_IP>
```

Once connected, run system updates and install required packages:

```bash
sudo apt update && sudo apt upgrade -y

# Install Python 3, venv, pip, Git, Nginx, and Certbot
sudo apt install -y python3-pip python3-venv git nginx certbot python3-certbot-nginx
```

---

## Step 4: Clone Application & Setup Virtual Environment

1. **Create application directory and dedicated user**:

```bash
# Create hackathon system user with www-data group
sudo useradd -rs /bin/bash -g www-data hackathon || true

# Prepare application directory
sudo mkdir -p /opt/hackathon/app
sudo chown -R ubuntu:www-data /opt/hackathon
```

2. **Clone the repository**:

```bash
git clone <YOUR_GIT_REPOSITORY_URL> /opt/hackathon/app
cd /opt/hackathon/app
```

3. **Create Python virtual environment & install dependencies**:

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

---

## Step 5: Configure Production Environment (`.env`)

Create your production environment file at `/opt/hackathon/app/.env`:

```bash
nano /opt/hackathon/app/.env
```

Paste your configuration:

```env
APP_ENV=production
APP_NAME="Spheronix Hackathon Registration API"
APP_BASE_URL=https://yourdomain.com
SERVER_HOST=127.0.0.1
SERVER_PORT=8000
CORS_ORIGINS=https://yourdomain.com,https://www.yourdomain.com

# MongoDB Atlas Connection
MONGO_URI=mongodb+srv://<db_user>:<db_password>@cluster0.xxxx.mongodb.net/?retryWrites=true&w=majority
DB_NAME=hackathon_db
GLOBAL_DB_NAME=global_db

# Security Keys
# Generate with: python3 -c "import secrets; print(secrets.token_urlsafe(48))"
SECRET_KEY=generate_a_random_48_char_secret_key_here
ADMIN_API_KEY=generate_a_random_secure_admin_api_key_here

# SMTP Configuration (for registration confirmation emails)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@spheronix.in
SMTP_PASSWORD=your-app-password
MAIL_FROM=noreply@spheronix.in
```

> **Note on MongoDB**: In MongoDB Atlas, ensure your EC2 Elastic IP is added under **Network Access** > **IP Access List**.

Secure file permissions:
```bash
chmod 600 /opt/hackathon/app/.env
sudo chown -R hackathon:www-data /opt/hackathon/app
```

---

## Step 6: Configure Systemd Service

1. Copy the systemd service template included in the repository:

```bash
sudo cp /opt/hackathon/app/scripts/hackathon.service /etc/systemd/system/hackathon.service
```

2. Verify or edit the file if needed (`sudo nano /etc/systemd/system/hackathon.service`):

```ini
[Unit]
Description=Spheronix Hackathon FastAPI Service
After=network.target

[Service]
Type=simple
User=hackathon
Group=www-data
WorkingDirectory=/opt/hackathon/app
EnvironmentFile=/opt/hackathon/app/.env
ExecStart=/opt/hackathon/app/venv/bin/gunicorn -c gunicorn_conf.py app.main:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

3. Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable hackathon.service
sudo systemctl start hackathon.service

# Check service status
sudo systemctl status hackathon.service
```

Test that the local API responds:
```bash
curl http://127.0.0.1:8000/api/health
```
*(Should return `{"status":"running","db":"connected",...}`)*

---

## Step 7: Configure Nginx Reverse Proxy

1. Create Nginx site configuration:

```bash
sudo nano /etc/nginx/sites-available/hackathon
```

2. Paste the following configuration (replace `yourdomain.com` with your actual domain):

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    client_max_body_size 25M;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (if needed)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
}
```

3. Enable site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/hackathon /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## Step 8: Install Free SSL Certificate (HTTPS)

Run Certbot to automatically configure SSL:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will:
1. Verify domain ownership.
2. Generate free Let's Encrypt certificates.
3. Automatically configure SSL redirects in `/etc/nginx/sites-available/hackathon`.
4. Configure an auto-renewal system timer (`systemctl list-timers | grep certbot`).

Test SSL auto-renewal:
```bash
sudo certbot renew --dry-run
```

---

## Step 9: Post-Deployment Verification Checklist

1. [ ] **Homepage loads over HTTPS**: Open `https://yourdomain.com` in your browser.
2. [ ] **Health endpoint**: Visit `https://yourdomain.com/api/health` -> verify `"status": "running"` and `"db": "connected"`.
3. [ ] **Static Assets**: Verify styles, images, and countdown timer render without console errors.
4. [ ] **Registration Form**: Test submitting a test registration.
5. [ ] **Payment & Email**: Verify payments and receipt emails trigger as expected.

---

## Step 10: How to Deploy Future Updates

Whenever you make code changes and push to GitHub/GitLab, deploy updates to the server with one command:

```bash
cd /opt/hackathon/app
./scripts/deploy.sh
```

Or run manually:
```bash
cd /opt/hackathon/app
git pull origin main
source venv/bin/activate
pip install -r requirements.txt
sudo systemctl restart hackathon.service
sudo systemctl status hackathon.service --no-pager
```
