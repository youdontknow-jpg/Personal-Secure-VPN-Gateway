# Troubleshooting Guide

Real issues encountered during 3+ months of operation and how I solved them.

---

## Issue 1: Tunnel Active But Domain Returns 404

### Symptoms

- `cloudflared tunnel info` shows tunnel status as "Active"
- Accessing `https://vpn.fujilegend.xyz` returns Cloudflare 404 error page
- Tunnel appears connected but traffic not routing

### Root Cause

Domain nameservers were still pointing to domain registrar (Namecheap), not Cloudflare. The tunnel was active but DNS wasn't resolving through Cloudflare's infrastructure.

### Solution

1. Log into domain registrar (Namecheap)
2. Navigate to domain settings → Nameservers
3. Change from "Namecheap BasicDNS" to "Custom DNS"
4. Set nameservers to Cloudflare's:
   - `ns1.cloudflare.com`
   - `ns2.cloudflare.com`
5. Wait 5-10 minutes for DNS propagation

**Verification:**
```bash
dig vpn.fujilegend.xyz

# Should show:
# ANSWER SECTION:
# vpn.fujilegend.xyz. 300 IN CNAME xxx.cfargotunnel.com.
```

### Learning

Always ensure domain is **fully managed by Cloudflare** before creating tunnel routes. The tunnel can be active but unreachable if DNS doesn't point to Cloudflare.

---

## Issue 2: WebSocket Connection Refused

### Symptoms

- Client initiates connection successfully
- TLS handshake completes
- WebSocket upgrade request fails with "connection refused"
- Docker logs show "no route to host" or "connection refused"

### Root Cause

Container port mapping misconfigured. Xray was listening on port 8443 inside container, but port wasn't properly mapped to host.

### Solution

**Verify container ports:**
```bash
sudo docker port xray

# Should show:
# 8443/tcp -> 127.0.0.1:8443
```

**Check Xray is listening:**
```bash
sudo docker exec xray netstat -tlnp | grep 8443

# Should show:
# tcp 0 0 0.0.0.0:8443 0.0.0.0:* LISTEN 1/xray
```

**Fix: Recreate container with proper port mapping:**
```bash
docker run -d \
  --name xray \
  --restart unless-stopped \
  -p 127.0.0.1:8443:8443 \  # This line is critical
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  -v /usr/local/etc/xray/certs:/usr/local/etc/xray/certs:ro \
  teddysun/xray
```

### Learning

Always verify port mappings after creating containers. Use `docker port` and `netstat` inside container to confirm services are listening.

---

## Issue 3: Cloudflare WAF Blocking Connections

### Symptoms

- Client receives HTTP 403 Forbidden
- Connection blocked before reaching Xray
- Cloudflare dashboard shows "Challenge" or "Block" events in firewall logs
- Normal HTTPS works but WebSocket upgrade fails

### Root Cause

Cloudflare's Web Application Firewall (WAF) security level set too high. The AI-powered threat detection system flagged WebSocket upgrade requests as suspicious activity.

### Solution

1. Log into Cloudflare Dashboard
2. Select domain → Security → Settings
3. Find "Security Level" setting
4. Change from "High" to "Medium" or "Low"
5. Test connection (should work immediately)

**Optional: Create WAF exception rule**
```
If (Hostname equals vpn.fujilegend.xyz)
Then (Security Level: Essentially Off)
```

### Why This Happens

VPN over WebSocket creates unusual traffic patterns:
- Long-lived WebSocket connections
- High data throughput over single connection
- Binary data frames (not typical web app JSON/text)

Cloudflare's AI may flag these as anomalous behavior.

### Learning

WebSocket-based services often need WAF tuning. Not a bug, just overly aggressive security defaults.

---

## Issue 4: Container Not Restarting After Host Reboot

### Symptoms

- Reboot host machine
- Xray container doesn't auto-start
- VPN unavailable until manual `docker start xray`
- `docker ps` shows no container running

### Root Cause

Container created without restart policy. Docker doesn't know it should auto-start on boot.

### Solution

**Option 1: Update existing container**
```bash
sudo docker update --restart unless-stopped xray
```

**Option 2: Recreate with restart policy**
```bash
# Stop and remove old container
docker stop xray
docker rm xray

# Recreate with restart policy
docker run -d \
  --name xray \
  --restart unless-stopped \  # This is the key
  -p 127.0.0.1:8443:8443 \
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  -v /usr/local/etc/xray/certs:/usr/local/etc/xray/certs:ro \
  teddysun/xray
```

**Verify:**
```bash
docker inspect xray | grep -A 3 RestartPolicy

# Should show:
# "RestartPolicy": {
#     "Name": "unless-stopped",
#     "MaximumRetryCount": 0
# }
```

### Learning

Always set restart policy when creating production containers. Options:
- `no` - Never restart (default)
- `unless-stopped` - Restart unless manually stopped (recommended)
- `always` - Always restart, even if manually stopped
- `on-failure` - Only restart on error exit codes

---

## Issue 5: Certificate Expiration

### Symptoms

- Clients report "Your connection is not private" error
- Browser shows certificate warnings
- VPN connection fails with TLS errors
- `openssl` verification shows expired certificate

### Root Cause

TLS certificate expired. Let's Encrypt certificates valid for 90 days, need renewal.

### Solution

**Check certificate expiry:**
```bash
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates

# Shows:
# notBefore=Aug 15 00:00:00 2025 GMT
# notAfter=Nov 13 23:59:59 2025 GMT  # EXPIRED
```

**Renew with acme.sh:**
```bash
# Force renewal
acme.sh --renew -d vpn.fujilegend.xyz --force

# Verify new certificate
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates

# Restart Xray to load new cert
sudo docker restart xray
```

### Prevention

**Set up auto-renewal cron job:**
```bash
# Add to root crontab
sudo crontab -e

# Add line (runs daily at 3 AM):
0 3 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh > /dev/null
```

**Add renewal hook to restart Xray:**
```bash
acme.sh --install-cert -d vpn.fujilegend.xyz \
  --cert-file /usr/local/etc/xray/certs/cert.cer \
  --key-file /usr/local/etc/xray/certs/key.cer \
  --fullchain-file /usr/local/etc/xray/certs/fullchain.cer \
  --reloadcmd "docker restart xray"
```

### Learning

Certificate management is critical for production services. Always set up auto-renewal, don't rely on manual intervention.

---

## Issue 6: Slow Performance from Specific Regions

### Symptoms

- VPN works but very slow from certain geographic locations
- High latency (>200ms) despite good internet connection
- Frequent disconnections from specific cities/countries
- Other locations work fine

### Root Cause

Cloudflare routing client to suboptimal Point of Presence (PoP). Client might be routed to distant datacenter instead of nearest one.

### Why This Happens

Cloudflare uses Anycast routing, which usually picks nearest PoP but can make suboptimal choices based on:
- BGP routing tables
- PoP load balancing
- Internet backbone congestion
- ISP peering relationships

### Solution (Limited)

This is mostly outside your control, but can try:

**1. Change DNS resolver**
```bash
# Try different Cloudflare resolvers
# 1.1.1.1 (standard)
# 1.0.0.1 (alternative)
```

**2. Try different times of day**
- CDN routing changes based on load
- Peak hours may route differently

**3. Consider Cloudflare Argo** (paid service)
- Optimizes routing through Cloudflare network
- Costs extra but improves latency

**4. Contact Cloudflare support**
- Report routing issues with traceroute data
- They can investigate BGP routing

### Workaround

For critical connections from problematic regions, consider:
- Alternative tunnel endpoint (different domain/tunnel)
- Second VPN instance in different region
- Hybrid approach (multiple VPN options)

### Learning

CDN routing is complex. Can't control everything. Focus on monitoring and having backup plans.

**Note:** This hasn't been a major issue for Asia-Pacific clients connecting to Japan-based server. Most Cloudflare PoPs in region provide good routing.

---

## Diagnostic Commands

### Quick Health Check

```bash
# Check cloudflared status
sudo systemctl status cloudflared

# Check Xray container status
docker ps | grep xray
docker logs xray --tail 50

# Check connectivity
curl -I https://vpn.fujilegend.xyz

# Check tunnel info
cloudflared tunnel info <TUNNEL_ID>
```

### Performance Testing

```bash
# Test latency
ping vpn.fujilegend.xyz

# Trace route
traceroute vpn.fujilegend.xyz

# Test throughput (from client)
# Download test
curl -o /dev/null https://vpn.fujilegend.xyz/largefile.bin

# Upload test  
curl -F "file=@largefile.bin" https://vpn.fujilegend.xyz/upload
```

### Log Analysis

```bash
# Xray logs
docker logs xray -f

# Cloudflared logs
sudo journalctl -u cloudflared -f

# System logs
sudo journalctl -xe

# Filter for errors
docker logs xray 2>&1 | grep -i error
```

---

## Common Error Messages

| Error Message | Likely Cause | Solution |
|---------------|--------------|----------|
| `connection refused` | Port mapping issue | Check `docker port xray` |
| `certificate verify failed` | Cert expired/invalid | Renew certificate with acme.sh |
| `403 Forbidden` | Cloudflare WAF blocking | Lower security level |
| `404 Not Found` | DNS not pointing to Cloudflare | Fix nameservers |
| `timeout` | Firewall blocking | Check UFW rules, cloudflared status |
| `bad gateway` | Xray not running | Start container: `docker start xray` |

---

## Prevention Strategies

### Monitoring

- Set up uptime monitoring (e.g., UptimeRobot)
- Alert on certificate expiration (30 days before)
- Monitor container resource usage
- Track Cloudflare WAF events

### Maintenance

- Regular system updates (`apt update && upgrade`)
- Keep Docker images updated
- Review logs weekly for anomalies
- Test backup/restore procedures

### Documentation

- Document all configuration changes
- Keep notes on what works/doesn't work
- Track troubleshooting steps that worked
- Version control for config files (Git)

---

**Related Documentation:**
- [Architecture](architecture.md) - How the system works
- [Setup Guide](setup-guide.md) - Deployment instructions
- [Security](security.md) - Security considerations
