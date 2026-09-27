# Security Considerations

Current security implementation, known limitations, and planned improvements.

---

## Current Security Measures

### Network Layer Protection

**Firewall (UFW):**
- Host firewall allows only essential ports (22, 80, 443) - none of them are forwarded on the home router, so they are reachable from the LAN only
- Default deny policy for inbound traffic
- VPN port (8443) bound to 127.0.0.1, reachable only through cloudflared

**Intrusion Prevention (Fail2Ban):**
- Monitors SSH login attempts
- Automatically blocks IPs after repeated failures
- Running since 2025, blocked 200+ malicious IPs

**Zero Port Forwarding:**
- No ports opened on home router
- All VPN traffic via Cloudflare Tunnel (outbound connections only)
- Eliminates common attack vector (exposed ports)

### Authentication Layer

**Cloudflare Access for SSH:**
- Remote SSH goes through Cloudflare Access (Google login + 2FA) before it reaches the tunnel
- No SSH port exposed to the internet

**UUID-Based Authentication:**
- Clients authenticate with a UUID (similar to an API key) - currently one UUID shared by all devices (see Known Limitations)
- 128-bit random identifier
- No username/password system to brute-force

**TLS Certificate Verification:**
- Clients verify server certificate
- Prevents man-in-the-middle attacks
- Certificates from trusted CA (Let's Encrypt)

**Cloudflare Tunnel Credentials:**
- Stored securely in `/root/.cloudflared/`
- File permissions: 600 (owner read/write only)
- Never committed to version control

### Encryption Layer

**TLS 1.3:**
- Latest TLS version
- Modern cipher suites (ChaCha20-Poly1305, AES-256-GCM)
- Minimum TLS version 1.2 (for compatibility)

**Perfect Forward Secrecy:**
- Session keys not derived from single secret
- Past communications remain secure even if key compromised

**End-to-End Encryption:**
- Traffic encrypted from client to Xray server
- Cloudflare can see encrypted tunnel, not plaintext

### Container Isolation

**Docker Security:**
- Default Docker isolation (namespaces, default seccomp profile)
- Separate network namespace from host
- Not configured yet: non-root user / dropped capabilities (planned)

**Read-Only Mounts:**
- Configuration files mounted read-only
- Container can't modify its own config
- Reduces risk of persistence if compromised

**Resource Limits:**
- CPU and memory limits set
- Prevents resource exhaustion attacks
- Container can't consume all host resources

---

## What This Protects Against

### Network Attacks

**DDoS (Distributed Denial of Service):**
- Cloudflare absorbs attack traffic (100+ Tbps capacity)
- Real IP hidden, can't be directly targeted
- Traffic filtering at CDN level

**Port Scanning:**
- No ports exposed on home router
- Scanners see Cloudflare IP, not home IP
- VPN port not directly accessible

**Man-in-the-Middle:**
- TLS certificate verification prevents impersonation
- Perfect Forward Secrecy protects past sessions
- Client validates server identity

### Protocol Detection

**Deep Packet Inspection (DPI):**
- WebSocket obfuscation masks VPN signatures
- Traffic appears as normal HTTPS/WebSocket
- No distinctive protocol patterns

**Traffic Analysis:**
- CDN integration provides natural traffic camouflage
- Difficult to distinguish from legitimate Cloudflare traffic
- Benefits from Cloudflare's massive IP pool

### Access Control

**Brute Force Attacks:**
- No password-based authentication to brute-force
- UUID too long to guess (2^128 possibilities)
- Fail2Ban blocks repeated SSH attempts

**Unauthorized Access:**
- UUID required for connection
- TLS certificate verification
- Firewall blocks unexpected traffic

---

## Known Limitations

### Current Limitations

This is a learning project. I'm aware of these security limitations:

#### 1. Single Shared Credential

**Issue:**
- All client devices use same UUID
- One compromised device requires UUID regeneration for all devices
- No way to revoke access for individual device

**Impact:**
- Lost/stolen device requires updating all other devices
- Can't track which device made which connection
- All-or-nothing access control

**Mitigation (current):**
- Limit UUID sharing to trusted devices only
- UUID rotated after suspected exposure (2026); no fixed rotation schedule yet
- Monitor logs for suspicious activity

#### 2. Limited Traffic Analysis Protection

**Issue:**
- While traffic obfuscated, persistent monitoring could correlate patterns
- Single tunnel endpoint makes traffic analysis easier
- No built-in traffic randomization or dummy traffic

**Impact:**
- Sophisticated adversary might identify VPN usage
- Connection patterns (timing, volume) still visible
- No protection against long-term statistical analysis

**Mitigation (current):**
- Cloudflare CDN provides some pattern noise
- Mix VPN traffic with other HTTPS traffic when possible
- Not a concern for my threat model (personal use)

#### 3. Basic Access Control

**Issue:**
- Once connected, client has access to entire home network
- No service-level restrictions (can't limit to only NAS access)
- No time-based access controls

**Impact:**
- Compromised client device has full network access
- Can't enforce principle of least privilege
- No segmentation between resources

**Mitigation (current):**
- Only connect from trusted devices
- Keep client devices secure (updated, antivirus)
- Monitor for unusual network activity

#### 4. Single Point of Failure

**Issue:**
- All traffic depends on one Docker container
- If container crashes, VPN unavailable until restart
- No redundancy or failover mechanism

**Impact:**
- Service downtime if Xray container fails
- Manual intervention required for recovery
- No high availability

**Mitigation (current):**
- Docker restart policy (unless-stopped) and cloudflared enabled in systemd - auto-start verified after [Issue 5](troubleshooting.md#issue-5-server-frozen-by-a-restart-loop-of-leftover-services)
- No external health check yet - both 2026 outages were noticed by me, not by an alert

#### 5. Cloudflare Dependency

**Issue:**
- Entire setup depends on Cloudflare Tunnel service
- Cloudflare can technically see connection metadata
- Must trust Cloudflare with tunnel infrastructure

**Impact:**
- If Cloudflare Tunnel discontinued, setup breaks
- Cloudflare knows when connections made (not content)
- Lock-in to Cloudflare ecosystem

**Mitigation (current):**
- End-to-end encryption (Cloudflare can't see traffic content)
- Cloudflare Tunnel free tier stable for years
- Can migrate to alternative tunnel if needed

---

## Threat Model

### What I'm Protecting Against

**In-scope threats:**
1. **Network eavesdropping** - ISP, public WiFi operators
2. **Censorship** - Network-level blocking of VPN protocols
3. **Port scanning** - Attackers scanning for vulnerable services
4. **DDoS attacks** - Attempts to overwhelm home connection

**Successfully mitigated by current architecture.**

### What I'm NOT Protecting Against

**Out-of-scope threats:**
1. **Nation-state adversaries** - Advanced persistent threats (APT)
2. **Sophisticated traffic analysis** - Long-term pattern correlation
3. **Cloudflare compromise** - Cloudflare itself being malicious
4. **Physical access** - Attacker with physical access to server

**Would require different architecture/approach.**

---

## Planned Improvements

### Phase 1: Enhanced Monitoring

**Goals:**
- Better visibility into system behavior
- Proactive problem detection
- Performance metrics tracking

**Implementation:**
- Certificate expiry alerts (highest priority - see Issues 4 and 5)
- Prometheus metrics collection from Xray and Docker
- Grafana dashboards for visualization
- Automated alerting (email/Telegram) for anomalies
- Traffic flow analysis and anomaly detection

**Security benefit:** Detect attacks and issues faster

### Phase 2: Advanced Access Control

**Goals:**
- Per-device authentication
- Service-level access restrictions
- Identity-based authorization

**Implementation:**
- Extend Cloudflare Access (already used for SSH) to other services
- Multiple UUIDs for different devices
- iptables rules for service-level restrictions
- Separate VLANs for different access levels

**Security benefit:** Principle of least privilege, device revocation

### Phase 3: Redundancy & Failover

**Goals:**
- Eliminate single point of failure
- Automatic recovery from failures
- Load distribution

**Implementation:**
- Second Xray instance on different host
- DNS-based load balancing (or HAProxy)
- Health checking and automatic failover
- Synchronized configuration

**Security benefit:** High availability, resilience

### Phase 4: Advanced Anti-Censorship

**Goals:**
- Even stronger traffic obfuscation
- Multiple fallback mechanisms
- Harder to detect and block

**Implementation:**
- Backup route independent of Cloudflare (Xray Reality)
- Domain fronting techniques
- Multiple CDN providers (not just Cloudflare)
- Traffic randomization and dummy traffic
- Pluggable transports (obfs4, meek)

**Security benefit:** Resilient against advanced censorship

---

## Why Not Implement Now?

**Honest assessment of priorities:**

1. **Learning approach:**
   - Want to understand Zero-Trust principles properly before building more of it
   - Building knowledge foundation before complex implementations

2. **Fix the basics first:**
   - Two full outages in 2026 ([Issue 4](troubleshooting.md#issue-4-full-vpn-outage---expired-certificate--broken-client), [Issue 5](troubleshooting.md#issue-5-server-frozen-by-a-restart-loop-of-leftover-services)) came from basics, not missing features
   - One credential exposure (UUID posted in a chat app) - handled by rotation
   - Alerts and cleanup matter more right now than new components

3. **Time management:**
   - Complex implementations need focused time

4. **Risk management:**
   - "If it ain't broke, don't fix it" principle
   - Changes introduce risk of breaking working setup
   - Stability more important than features right now

**This is a learning journey.** Current architecture taught me VPN fundamentals, Docker, and networking basics. Future phases will teach enterprise security, high availability, and advanced threat mitigation.

---

## Security Best Practices I Follow

### Account Security

- **Strong unique passwords** for all accounts (Cloudflare, registrar)
- **2FA enabled** using authenticator app (not SMS)
- **Regular access log review** in Cloudflare dashboard
- **API token least privilege** (DNS edit only, not full account)

### System Hardening

- **Minimal software** installed on host (only essentials)
- **Regular updates** via unattended-upgrades
- **SSH key-only** authentication (password auth disabled)
- **Firewall default-deny** policy

### Operational Security

- **UUID rotation** after suspected exposure (no fixed schedule yet)
- **Certificate monitoring** - planned, not in place yet (see Issues 4 and 5)
- **Log review** weekly for anomalies
- **Configuration backup** before changes

### Development Security

- **Never commit secrets** to version control
- **Configuration templates** use placeholders
- **Sensitive values** stored securely (not in code)
- **Documentation** doesn't include real credentials

---

## Security Resources

### Learning Materials

- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Mozilla TLS Configuration](https://ssl-config.mozilla.org/)
- [Cloudflare Security Documentation](https://developers.cloudflare.com/fundamentals/basic-tasks/protect-your-origin-server/)

### Tools Used

- [acme.sh](https://github.com/acmesh-official/acme.sh) - Certificate management
- [fail2ban](https://www.fail2ban.org/) - Intrusion prevention
- [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall) - Firewall management

### Monitoring

- [Docker logs](https://docs.docker.com/engine/reference/commandline/logs/) - Container logging
- Cloudflare Analytics - Traffic analysis
- systemd journal - System logs

---

## Incident Response Plan

### If UUID Compromised

1. Generate new UUID: `xray uuid`
2. Update `/etc/xray/config.json`
3. Restart Xray: `docker restart xray`
4. Update all client devices with new UUID
5. Monitor logs for old UUID usage

### If Certificate Compromised

1. Revoke certificate: `acme.sh --revoke -d vpn.yourdomain.com`
2. Issue new certificate: `acme.sh --issue --dns dns_cf -d vpn.yourdomain.com`
3. Install new certificate
4. Restart Xray: `docker restart xray`

### If Server Compromised

1. Disconnect from network immediately
2. Take snapshots/backups if possible
3. Rebuild server from scratch
4. Rotate all credentials (UUID, certificates, API tokens)
5. Review logs to understand attack vector
6. Implement additional hardening

---

**Related Documentation:**
- [Architecture](architecture.md) - How security is implemented
- [Setup Guide](setup-guide.md) - Secure deployment steps
- [Troubleshooting](troubleshooting.md) - Security-related issues
