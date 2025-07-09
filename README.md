# Homelab Setup Documentation

This documentation serves as a comprehensive guide for setting up and managing a homelab environment using Debian as the base operating system.

## Table of Contents

1. [Initial Debian Setup](#initial-debian-setup)
2. [Network Configuration](#network-configuration)
3. [SSH Access Setup](#ssh-access-setup)
4. [Advanced SSH Setup: Secure Remote Access via DuckDNS (LAN + WAN)](#️-advanced-ssh-setup-secure-remote-access-via-duckdns-lan--wan)
5. [Docker and Harbor Registry](#docker-and-harbor-registry)
6. [Nginx with HTTPS](#nginx-with-https)
7. [System Maintenance](#system-maintenance)

## Initial Debian Setup

### WiFi Connection Setup

In case of network disconnection, use `iw` command to reconnect to WiFi:

```bash
# List available wireless interfaces
iw dev

# Scan for available networks
sudo iw dev wlan0 scan | grep SSID

# Connect to WiFi network
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf

# If configuration file doesn't exist, create it:
sudo wpa_passphrase "YOUR_SSID" "YOUR_PASSWORD" | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf

# Get IP address via DHCP
sudo dhclient wlan0
```

### Static IP Address Configuration

Configure static IP for easier SSH access:

using `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces

# Add configuration:
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4

# Restart networking
sudo systemctl restart networking
```

## SSH Access Setup

### Server-side SSH Configuration

```bash
# Install OpenSSH server
sudo apt update
sudo apt install openssh-server

# Enable and start SSH service
sudo systemctl enable ssh
sudo systemctl start ssh

# Configure SSH (optional security hardening)
sudo nano /etc/ssh/sshd_config

# Recommended changes:
Port 22
PermitRootLogin no
PasswordAuthentication yes  # Change to 'no' after setting up key-based auth
PubkeyAuthentication yes

# Restart SSH service
sudo systemctl restart ssh
```

### Client-side SSH Configuration (macOS)

Create SSH config file on your Mac for easier access:

```bash
# Create SSH config directory if it doesn't exist
mkdir -p ~/.ssh

# Create or edit SSH config file
nano ~/.ssh/config

# Add homelab server configuration:
Host homelab
    HostName 192.168.1.100
    User your-username
    Port 22
    IdentityFile ~/.ssh/id_rsa

# Optional: Create SSH key pair for passwordless access
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# Copy public key to homelab server
ssh-copy-id homelab
```

Now you can SSH into your homelab with simply:
```bash
ssh homelab
```

## 🛠️ Advanced SSH Setup: Secure Remote Access via DuckDNS (LAN + WAN)

This section covers setting up secure SSH access both locally and remotely using DuckDNS for dynamic DNS.

### 🧱 Step 1: Set a Static IP for Debian

using `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces

# Add configuration (example for WiFi):
allow-hotplug wlp2s0
iface wlp2s0 inet static
    address 192.168.1.XXX
    netmask 255.255.255.0
    gateway 192.168.1.1
    wpa-ssid YOUR_WIFI_SSID
    wpa-psk YOUR_WIFI_PASSWORD
    dns-nameservers 1.1.1.1 8.8.8.8

# Restart networking
sudo systemctl restart networking
```

**Purpose:** Ensures the Debian machine always has the same IP address on your local network, required for reliable port forwarding.

### 🔐 Step 2: Install and Start the SSH Server

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Purpose:** Enables remote access to the Debian machine via SSH.

### 🌍 Step 3: Register a DuckDNS Subdomain

1. Visit: https://www.duckdns.org
2. Sign in with GitHub/Google
3. Create a domain: `your-domain.duckdns.org`
4. Copy your personal token

### 📝 Step 4: Set Up DuckDNS Update Script

```bash
# Create DuckDNS directory
mkdir ~/duckdns
nano ~/duckdns/duck.sh
```

Script content:
```bash
echo url="https://www.duckdns.org/update?domains=your-domain&token=YOUR_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Make it executable and install dependencies:
```bash
chmod 700 ~/duckdns/duck.sh
sudo apt install curl
```

Test the script:
```bash
bash ~/duckdns/duck.sh
cat ~/duckdns/duck.log  # should return "OK"
```

### ⏰ Step 5: Schedule Auto-IP Updates with Cron

```bash
crontab -e

# Add this line to update every 5 minutes:
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

**Purpose:** Keeps your DuckDNS domain updated with your current public IP every 5 minutes.

### 🔀 Step 6: Set Up Router Port Forwarding

Configure your router with these settings:

| Setting | Value |
|---------|-------|
| External Port | 2222 |
| Internal IP | 192.168.1.XXX |
| Internal Port | 22 |
| Protocol | TCP |

**Purpose:** Allows access to your Debian machine's SSH server from outside your home network via port 2222.

### 🔑 Step 7: Generate and Install SSH Key

On your Mac:
```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

Install the key on Debian via DuckDNS:
```bash
ssh-copy-id -p 2222 your-username@your-domain.duckdns.org
```

### 🔧 Step 8: Disable Password Authentication in SSH

```bash
sudo nano /etc/ssh/sshd_config

# Add or modify this line:
PasswordAuthentication no

# Restart SSH
sudo systemctl restart ssh
```

**Purpose:** Prevents all password-based SSH logins (only SSH key holders can connect).

### 🚫 Step 9: Disable Root Login

In the same SSH config file:
```bash
# Add or modify this line:
PermitRootLogin no

# Restart SSH again
sudo systemctl restart ssh
```

**Purpose:** Prevents direct SSH login as root for security reasons. Use sudo from a regular user instead.

### 🧠 Step 10: Configure SSH Shortcuts on Mac

```bash
nano ~/.ssh/config

# Add this configuration block:
Host homelab-local
    HostName 192.168.1.XXX
    User your-username
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host homelab-remote
    HostName your-domain.duckdns.org
    User your-username
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

Now you can connect:
- **On LAN:** `ssh homelab-local`
- **On WAN (from anywhere):** `ssh homelab-remote`

### Security Best Practices

1. **Change Default SSH Port** (optional):
   ```bash
   sudo nano /etc/ssh/sshd_config
   # Change: Port 22 to Port XXXX
   sudo systemctl restart ssh
   ```

2. **Enable UFW Firewall**:
   ```bash
   sudo ufw enable
   sudo ufw allow ssh
   sudo ufw allow 2222/tcp  # for remote access
   ```

3. **Monitor SSH Access**:
   ```bash
   # Check recent SSH logins
   sudo journalctl -u ssh -f
   
   # Check failed login attempts
   sudo grep "Failed password" /var/log/auth.log
   ```

4. **Backup SSH Keys**:
   ```bash
   # Backup your SSH keys securely
   cp ~/.ssh/id_ed25519* ~/secure-backup-location/
   ```

## Docker and Harbor Registry

### Docker Installation

```bash
# Update package index
sudo apt update

# Install required packages
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Harbor Registry Setup

```bash
# Create harbor directory
mkdir -p ~/harbor
cd ~/harbor

# Download Harbor installer
wget https://github.com/goharbor/harbor/releases/download/v2.8.0/harbor-online-installer-v2.8.0.tgz
tar xvf harbor-online-installer-v2.8.0.tgz
cd harbor

# Configure Harbor
cp harbor.yml.tmpl harbor.yml
nano harbor.yml

# Key configurations in harbor.yml:
hostname: harbor.homelab.local  # or your domain
http:
  port: 80
https:
  port: 443
  certificate: /path/to/your/certificate.crt
  private_key: /path/to/your/private.key
harbor_admin_password: your-secure-password
database:
  password: your-db-password
data_volume: /data

# Install and start Harbor
sudo ./install.sh
```

### Securing Harbor Behind Host-Level Nginx

This section covers configuring Harbor to run securely behind your host's Nginx proxy instead of exposing Harbor's internal proxy directly.

#### Step 1: Rebind Harbor's Internal Proxy to Localhost

Edit Harbor's docker-compose.yml to limit proxy access to localhost only:

```bash
cd ~/harbor
nano docker-compose.yml

# Find the proxy service and change ports from:
# ports:
#   - "80:8080"
#   - "443:8443"
# To:
ports:
  - "127.0.0.1:8080:8080"
  - "127.0.0.1:8443:8443"
```

**Purpose:** Limits Harbor's built-in Nginx to the loopback interface so it can't be reached directly from the network, only via your host Nginx.

#### Step 2: Restart Harbor Cleanly

```bash
# Stop Harbor and remove orphan containers
sudo docker-compose down --remove-orphans

# Start Harbor with new configuration
sudo docker-compose up -d

# Verify no containers are binding to 0.0.0.0
docker ps | grep proxy
```

#### Step 3: Install and Configure Host Nginx

```bash
# Install Nginx on the host system
sudo apt install nginx

# Test Nginx configuration
sudo nginx -t
```

#### Step 4: Create Harbor Nginx Configuration

```bash
# Create Harbor site configuration
sudo nano /etc/nginx/sites-available/harbor

# Add initial HTTP-only configuration:
server {
    listen 80;
    server_name your-domain.duckdns.org;
    
    # Disable any limits for Harbor
    client_max_body_size 0;
    chunked_transfer_encoding on;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}

# Enable the site
sudo ln -s /etc/nginx/sites-available/harbor /etc/nginx/sites-enabled/
sudo nginx -t
```

#### Step 5: Obtain Let's Encrypt Certificate

```bash
# Stop Nginx temporarily for standalone certificate generation
sudo systemctl stop nginx

# Generate certificate using standalone mode
sudo certbot certonly --standalone -d your-domain.duckdns.org

# Certificate files will be stored in:
# /etc/letsencrypt/live/your-domain.duckdns.org/fullchain.pem
# /etc/letsencrypt/live/your-domain.duckdns.org/privkey.pem
```

#### Step 6: Enable HTTPS in Nginx

```bash
# Update Harbor Nginx configuration with HTTPS
sudo nano /etc/nginx/sites-available/harbor

# Replace with full HTTPS configuration:
server {
    listen 80;
    server_name your-domain.duckdns.org;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.duckdns.org;

    # SSL Configuration
    ssl_certificate     /etc/letsencrypt/live/your-domain.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.duckdns.org/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Harbor-specific settings
    client_max_body_size 0;
    chunked_transfer_encoding on;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}

# Test and start Nginx
sudo nginx -t
sudo systemctl start nginx
```

#### Step 7: Clean Up Any Conflicting Nginx Processes

```bash
# Stop any manually launched Nginx workers
sudo nginx -s quit || true
sudo pkill nginx

# Start Nginx properly via systemctl
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### Step 8: Verify Local Proxy Functionality

```bash
# Test HTTP proxy
curl -I http://127.0.0.1/

# Test HTTPS proxy
curl -Ik https://127.0.0.1/

# Both should return HTTP 200, confirming Nginx → Harbor is working
```

#### Step 9: Configure Router Port Forwarding

Set up port forwarding in your router:

| Service | External Port | Internal IP | Internal Port | Protocol |
|---------|---------------|-------------|---------------|----------|
| HTTP | 80 | 192.168.1.XXX | 80 | TCP |
| HTTPS | 443 | 192.168.1.XXX | 443 | TCP |

#### Step 10: Verify External Access

```bash
# Test external connectivity
nc -vz your-domain.duckdns.org 80
nc -vz your-domain.duckdns.org 443

# Both commands should succeed
```

Now Harbor should be accessible securely via:
- **Local:** `https://your-domain.duckdns.org`
- **External:** `https://your-domain.duckdns.org`

#### Certificate Renewal

Set up automatic certificate renewal:

```bash
# Test renewal
sudo certbot renew --dry-run

# Add to crontab for automatic renewal
sudo crontab -e

# Add this line to renew certificates twice daily:
0 12 * * * /usr/bin/certbot renew --quiet --reload-nginx
```

## Nginx with HTTPS

### Nginx Installation and Configuration

```bash
# Install Nginx
sudo apt update
sudo apt install nginx

# Install Certbot for Let's Encrypt certificates
sudo apt install certbot python3-certbot-nginx

# Create main nginx configuration
sudo nano /etc/nginx/sites-available/homelab

# Basic configuration example:
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    # Root location
    location / {
        root /var/www/html;
        index index.html index.htm;
    }
    
    # Harbor registry proxy
    location /harbor/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Additional service proxies
    location /service1/ {
        proxy_pass http://localhost:3001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /service2/ {
        proxy_pass http://localhost:3002/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Enable the site
sudo ln -s /etc/nginx/sites-available/homelab /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default

# Test nginx configuration
sudo nginx -t

# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com

# Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Multiple HTTP Services Configuration

Create separate configuration files for different services:

```bash
# Service 1 configuration
sudo nano /etc/nginx/sites-available/service1

server {
    listen 80;
    server_name service1.homelab.local;
    
    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Enable service configurations
sudo ln -s /etc/nginx/sites-available/service1 /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

## System Maintenance

### Regular Updates

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Clean up
sudo apt autoremove -y
sudo apt autoclean

# Update Docker images
docker system prune -a
```

### Backup Scripts

```bash
# Create backup script
nano ~/backup.sh

#!/bin/bash
# Backup important configurations
BACKUP_DIR="/home/$(whoami)/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup configurations
cp -r /etc/nginx $BACKUP_DIR/
cp -r ~/.ssh $BACKUP_DIR/
cp /etc/dhcpcd.conf $BACKUP_DIR/
cp /etc/wpa_supplicant/wpa_supplicant.conf $BACKUP_DIR/

# Backup Docker volumes
docker run --rm -v harbor_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/harbor_data.tar.gz -C /data .

echo "Backup completed: $BACKUP_DIR"

# Make executable
chmod +x ~/backup.sh
```

### Monitoring

```bash
# Install basic monitoring tools
sudo apt install htop iotop ncdu

# Check system status
systemctl status nginx
systemctl status ssh
systemctl status docker
docker ps
```

## Troubleshooting

### Common Issues

1. **WiFi Connection Issues**
   ```bash
   # Check wireless interface status
   ip link show
   
   # Restart network manager
   sudo systemctl restart NetworkManager
   ```

2. **SSH Connection Refused**
   ```bash
   # Check SSH service status
   sudo systemctl status ssh
   
   # Check firewall
   sudo ufw status
   ```

3. **Docker/Harbor Issues**
   ```bash
   # Check Docker status
   sudo systemctl status docker
   
   # Check Harbor logs
   docker-compose logs -f
   ```

4. **Nginx Issues**
   ```bash
   # Check nginx configuration
   sudo nginx -t
   
   # Check nginx logs
   sudo tail -f /var/log/nginx/error.log
   ```

---

## Notes for Future Reference

- Always backup configurations before making changes
- Keep track of IP addresses and port mappings
- Document any custom configurations or scripts
- Regular security updates are essential
- Monitor system resources and logs regularly
- Consider implementing automated backup solutions
- Use version control for configuration files

This documentation should be updated as the homelab setup evolves and new services are added. 
