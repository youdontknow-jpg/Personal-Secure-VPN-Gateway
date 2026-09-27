# Setup Guide

Complete step-by-step instructions to deploy this VPN architecture.

---

## Prerequisites

### Hardware Requirements

- Linux server (physical or VM)
- 2GB RAM minimum (1GB for Docker container)
- 10GB storage minimum
- Network connection (WiFi or Ethernet)

### Software Requirements

- Ubuntu 22.04 LTS (or similar Linux distro)
- Docker 20.10+ and Docker Compose
- Root/sudo access

### External Services

- Domain name (e.g., from Namecheap, ~$10/year)
- Cloudflare account (free tier sufficient)

---

## Step 1: System Preparation

### Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker

```bash
# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
```

### Install Cloudflared

```bash
# Download latest release
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Install
sudo dpkg -i cloudflared-linux-amd64.deb

# Verify
cloudflared --version
```

---

## Step 2: Domain Setup

### Register Domain

1. Purchase domain from registrar (Namecheap, GoDaddy, etc.)
2. Example: `yourdomain.com`

### Add Domain to Cloudflare

1. Log into Cloudflare dashboard
2. Click "Add Site"
3. Enter your domain name
4. Choose Free plan
5. Cloudflare will scan DNS records
6. Click "Continue"

### Update Nameservers

1. Cloudflare shows 2 nameservers (e.g., `ns1.cloudflare.com`, `ns2.cloudflare.com`)
2. Go to your domain registrar
3. Find "Nameservers" or "DNS Settings"
4. Change to Custom Nameservers
5. Enter Cloudflare's nameservers
6. Save changes
7. Wait 5-30 minutes for propagation

**Verify:**
```bash
dig yourdomain.com NS

# Should show Cloudflare nameservers
```

---

## Step 3: TLS Certificate

### Install acme.sh

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
```

### Obtain Certificate

```bash
# Set Cloudflare API credentials
export CF_Token="YOUR_CLOUDFLARE_API_TOKEN"
export CF_Account_ID="YOUR_ACCOUNT_ID"

# Issue certificate
acme.sh --issue --dns dns_cf -d vpn.yourdomain.com

# Install certificate
sudo mkdir -p /usr/local/etc/xray/certs
acme.sh --install-cert -d vpn.yourdomain.com \
  --cert-file /usr/local/etc/xray/certs/cert.cer \
  --key-file /usr/local/etc/xray/certs/key.cer \
  --fullchain-file /usr/local/etc/xray/certs/fullchain.cer
```

**Get Cloudflare API Token:**
1. Cloudflare Dashboard → My Profile → API Tokens
2. Create Token → Edit Zone DNS template
3. Zone Resources: Include → Specific Zone → yourdomain.com
4. Copy token

---

## Step 4: Xray Configuration

### Generate UUID

```bash
# Install Xray temporarily to generate UUID
docker run --rm teddysun/xray xray uuid

# Output example: a1b2c3d4-e5f6-7890-abcd-ef1234567890
# Save this for config file
```

### Create Configuration File

```bash
sudo mkdir -p /etc/xray
sudo nano /etc/xray/config.json
```

**Copy from [`configs/xray-config-example.json`](../configs/xray-config-example.json) and modify:**

- Replace `YOUR-UUID-HERE` with generated UUID
- Replace `vpn.example.com` with your actual domain
- Adjust paths if needed

### Verify Configuration

```bash
# Test config syntax
docker run --rm -v /etc/xray/config.json:/etc/xray/config.json teddysun/xray xray test -c /etc/xray/config.json

# Should output: Configuration OK
```

---

## Step 5: Cloudflare Tunnel Setup

### Authenticate with Cloudflare

```bash
cloudflared tunnel login
```

This opens browser for authentication. Grant permissions when prompted.

### Create Tunnel

```bash
# Create tunnel
cloudflared tunnel create my-vpn-tunnel

# Save output! Shows:
# - Tunnel ID (e.g., abc123-def456-ghi789)
# - Credentials file location

# List tunnels to verify
cloudflared tunnel list
```

### Configure Tunnel

```bash
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

**Copy from [`configs/cloudflared-config.yml`](../configs/cloudflared-config.yml) and modify:**

- Replace `YOUR_TUNNEL_ID` with your tunnel ID
- Replace `vpn.example.com` with your domain

### Route DNS

```bash
# Create DNS record
cloudflared tunnel route dns my-vpn-tunnel vpn.yourdomain.com

# Verify in Cloudflare dashboard:
# DNS → Records → Should see CNAME: vpn.yourdomain.com → xxx.cfargotunnel.com
```

### Install as System Service

```bash
# Install service
sudo cloudflared service install

# Start service
sudo systemctl start cloudflared

# Enable auto-start
sudo systemctl enable cloudflared

# Check status
sudo systemctl status cloudflared
```

---

## Step 6: Deploy Xray Container

### Option A: Docker Run Command

```bash
docker run -d \
  --name xray \
  --restart unless-stopped \
  -p 127.0.0.1:8443:8443 \
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  -v /usr/local/etc/xray/certs:/usr/local/etc/xray/certs:ro \
  teddysun/xray
```

### Option B: Docker Compose (Recommended)

```bash
# Copy compose file
sudo cp configs/docker-compose.yml /opt/xray/docker-compose.yml
cd /opt/xray

# Start services
docker compose up -d

# Check status
docker compose ps
```

### Verify Container

```bash
# Check container is running
docker ps | grep xray

# Check logs
docker logs xray

# Should see: "Xray 1.x.x started"
```

---

## Step 7: Firewall Configuration

### Install UFW

```bash
sudo apt install -y ufw
```

### Configure Rules

```bash
# Allow SSH (important! Don't lock yourself out)
sudo ufw allow 22/tcp

# Allow HTTP (only needed for HTTP-01 validation -
# not required when using DNS validation with dns_cf as in Step 3)
sudo ufw allow 80/tcp

# Allow HTTPS (for general web access)
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

**Note:** No need to open port 8443. All VPN traffic goes through Cloudflare Tunnel.

### Install Fail2Ban (Optional but Recommended)

```bash
sudo apt install -y fail2ban

# Start and enable
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status
```

---

## Step 8: Testing

### Test from Host

```bash
# Test Cloudflare Tunnel
curl -I https://vpn.yourdomain.com

# Should return HTTP 200 or 400 (not 404 or 503)
```

### Test VPN Connection

**Using V2Ray/Xray client (e.g., v2rayN, Qv2ray):**

1. **Install client** on device (Windows/Mac/Linux/iOS/Android)

2. **Add server:**
   - Protocol: VLESS
   - Address: vpn.yourdomain.com
   - Port: 443
   - UUID: (your generated UUID)
   - Encryption: none
   - Network: ws
   - Path: /chatws
   - TLS: enabled
   - SNI: vpn.yourdomain.com

3. **Connect and test:**
   ```bash
   # Check your IP (should show home network IP)
   curl ifconfig.me
   
   # Test accessing home network resource
   ping 192.168.1.1  # Your router
   ```

---

## Step 9: Certificate Auto-Renewal

### Setup Renewal Hook

```bash
# Edit acme.sh renewal hook
acme.sh --install-cert -d vpn.yourdomain.com \
  --cert-file /usr/local/etc/xray/certs/cert.cer \
  --key-file /usr/local/etc/xray/certs/key.cer \
  --fullchain-file /usr/local/etc/xray/certs/fullchain.cer \
  --reloadcmd "docker restart xray"
```

### Verify Cron Job

```bash
# Check cron
crontab -l | grep acme

# Should show exactly ONE daily renewal check
```

**Don't discard the output.** acme.sh installs its cron job with `> /dev/null`, so a failed renewal leaves no trace - that is how my certificate expired twice ([Issue 4](troubleshooting.md#issue-4-full-vpn-outage---expired-certificate--broken-client), [Issue 5](troubleshooting.md#issue-5-server-frozen-by-a-restart-loop-of-leftover-services)). Send it to a log file instead:

```bash
crontab -l | grep -v 'acme.sh --cron' > /tmp/cron.new
echo '7 17 * * * "$HOME/.acme.sh"/acme.sh --cron --home "$HOME/.acme.sh" >> "$HOME/.acme.sh/cron.log" 2>&1' >> /tmp/cron.new
crontab /tmp/cron.new
```

### Test Renewal

```bash
# Force renewal (for testing)
acme.sh --renew -d vpn.yourdomain.com --force

# Check if Xray restarted
docker logs xray --tail 20
```

---

## Step 10: Monitoring Setup (Optional)

### Basic Health Check Script

```bash
sudo nano /usr/local/bin/vpn-health-check.sh
```

**Content:**
```bash
#!/bin/bash

# Check Xray container
if ! docker ps | grep -q xray; then
    echo "Xray container not running! Starting..."
    docker start xray
fi

# Check Cloudflared service
if ! systemctl is-active --quiet cloudflared; then
    echo "Cloudflared not running! Starting..."
    systemctl start cloudflared
fi

# Check connectivity
if ! curl -s -o /dev/null -w "%{http_code}" https://vpn.yourdomain.com | grep -q "200\|400"; then
    echo "VPN endpoint not responding!"
fi
```

**Make executable and schedule:**
```bash
sudo chmod +x /usr/local/bin/vpn-health-check.sh

# Add to crontab (runs every 5 minutes)
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/vpn-health-check.sh") | crontab -
```

---

## Troubleshooting

If something doesn't work, check:

1. **DNS propagation:** `dig vpn.yourdomain.com` shows CNAME to cfargotunnel.com
2. **Cloudflared status:** `systemctl status cloudflared`
3. **Xray logs:** `docker logs xray`
4. **Cloudflare dashboard:** Check firewall events
5. **Port mapping:** `docker port xray` shows 8443

**See [Troubleshooting Guide](troubleshooting.md) for specific issues.**

---

## Maintenance

### Regular Updates

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Xray image
docker pull teddysun/xray:latest
docker restart xray

# Update Cloudflared
sudo apt install --only-upgrade cloudflared
```

### Backup Configuration

```bash
# Create backup directory
sudo mkdir -p /backup/vpn-config

# Backup configs
sudo cp /etc/xray/config.json /backup/vpn-config/
sudo cp /etc/cloudflared/config.yml /backup/vpn-config/
sudo cp -r /usr/local/etc/xray/certs /backup/vpn-config/

# Backup tunnel credentials
sudo cp /root/.cloudflared/*.json /backup/vpn-config/
```

---

## Next Steps

- Read [Architecture Documentation](architecture.md) to understand how it works
- Review [Security Considerations](security.md)
- Check [Alternatives](alternatives.md) to understand design choices
- Set up monitoring and alerting

---

**Related Documentation:**
- [Architecture](architecture.md)
- [Troubleshooting](troubleshooting.md)
- [Security](security.md)
- [Alternatives](alternatives.md)
