# Architecture & Design

Detailed explanation of why this VPN architecture was chosen and how it works.

---

## System Architecture

```
Client Device
   ↓
Cloudflare Domain (vpn.fujilegend.xyz)
   ↓
Cloudflare Tunnel (TLS 1.3 encryption)
   ↓
cloudflared daemon (local)
   ↓
Docker Container: Xray (port 8443)
   ↓
Home Network (file server, media, services)
```

### Traffic Flow

1. **Client connects** to Cloudflare domain via HTTPS
2. **Cloudflare CDN** receives connection, routes to tunnel
3. **Cloudflare Tunnel** establishes encrypted connection to home network
4. **cloudflared daemon** receives traffic, forwards to Docker container
5. **Xray container** decrypts and routes to home network resources

All traffic encrypted end-to-end with TLS 1.3, appears as normal HTTPS/WebSocket to external observers.

---

## Why This Architecture?

### Problem 1: Traditional VPNs Are Easy to Detect

Traditional VPN protocols (WireGuard, OpenVPN, IPSec) have distinctive signatures:

- **WireGuard**: Uses UDP, has recognizable handshake pattern
- **OpenVPN**: Specific packet structure, easily fingerprinted
- **IPSec**: Well-known protocol identifiers

**Result:** Easily blocked by deep packet inspection (DPI) in restrictive networks.

### Solution: WebSocket Over TLS Obfuscation

**Xray with WebSocket transport:**

1. **Looks like normal web traffic**
   - Uses standard HTTPS port (443)
   - WebSocket upgrade request identical to web applications (chat apps, real-time dashboards)
   - No VPN protocol signatures visible

2. **Bypasses DPI systems**
   - To network inspectors, appears as regular website visit
   - WebSocket is legitimate protocol used by millions of websites
   - Blocking would disrupt legitimate services

3. **CDN camouflage**
   - Traffic routes through Cloudflare's massive CDN
   - Benefits from Cloudflare's global IP pool
   - Extremely difficult to block without affecting other Cloudflare services

---

### Problem 2: Exposed Home IP and Open Ports

Traditional VPN setup requires:
- Opening ports on home router (security risk)
- Exposing real home IP address
- Vulnerable to DDoS attacks and port scanning
- Requires static IP or complex DDNS configuration

### Solution: Cloudflare Tunnel (Zero Port Forwarding)

**How Cloudflare Tunnel works:**

```
Home Network → cloudflared initiates outbound connection → Cloudflare edge
Client → Connects to Cloudflare edge → Routed through existing tunnel → Home
```

**Benefits:**

- **No inbound ports opened** - All connections initiated from inside home network
- **Real IP hidden** - Only Cloudflare IP visible to internet
- **DDoS protection** - Cloudflare's 100+ Tbps capacity absorbs attacks
- **Free TLS certificates** - Automatic certificate management
- **Global CDN** - Low latency from anywhere in the world

---

### Problem 3: Service Management and Updates

Traditional VPN installations:
- Software installed directly on host OS
- Dependencies can conflict
- Difficult to rollback after updates
- Messy to uninstall completely

### Solution: Docker Containerization

**Container benefits:**

- **Isolated environment** - No host dependencies
- **Easy updates** - `docker pull` new image, restart container
- **Reproducible** - Same configuration works on any Docker host
- **Resource limits** - Control CPU/memory usage
- **Easy cleanup** - `docker rm` removes completely

---

## Security Layers

### Layer 1: Network Perimeter

- **UFW firewall** - Only essential ports (22, 80, 443) exposed to localhost
- **Fail2Ban** - Automatic IP blocking after failed SSH attempts
- **No internet-facing ports** - All VPN traffic via Cloudflare Tunnel

### Layer 2: Authentication

- **UUID-based client auth** - Similar to API keys
- **TLS certificate verification** - Prevents man-in-the-middle attacks
- **Cloudflare credentials** - Stored securely, not in code

### Layer 3: Encryption

- **TLS 1.3** - Latest encryption standard
- **Perfect Forward Secrecy** - Past communications can't be decrypted even if key compromised
- **WebSocket over TLS** - No plaintext transmission

### Layer 4: Container Isolation

- **Limited privileges** - Container runs without root capabilities
- **Read-only configs** - Configuration files mounted read-only
- **Network namespace** - Separate from host network stack

---

## Performance Characteristics

### Latency Breakdown

| Stage | Latency | Notes |
|-------|---------|-------|
| Client to Cloudflare | 5-15ms | Depends on client location, Cloudflare PoP |
| Cloudflare Tunnel | 10-20ms | Encryption/decryption overhead |
| Xray processing | 1-5ms | Container overhead |
| **Total** | **30-50ms** | Comparable to commercial VPNs |

### Throughput

**Tested on 1Gbps fiber connection:**

- **Download:** 150-250 Mbps (bottleneck: client connection)
- **Upload:** 80-120 Mbps (bottleneck: home upload bandwidth)
- **Container overhead:** <5% CPU usage

**Scalability:** Current setup handles 5-10 concurrent connections without performance degradation.

---

## Anti-Censorship Effectiveness

### Traffic Analysis Resistance

**What network inspectors see:**

1. **DNS query** - `vpn.fujilegend.xyz` (looks like normal domain)
2. **TLS handshake** - Standard HTTPS connection to Cloudflare
3. **HTTP upgrade** - WebSocket upgrade request (used by many web apps)
4. **Encrypted data** - TLS-encrypted WebSocket frames

**What they DON'T see:**

- No VPN protocol signatures
- No unusual packet patterns
- No distinctive handshakes
- Just normal HTTPS/WebSocket traffic

### Real-World Testing

Tested in:
- Corporate networks with strict firewall rules
- Public WiFi with content filtering
- Networks known to block VPN protocols

**Result:** Successfully maintained connectivity in all tested environments over 3+ months.

---

## Deployment Environment

### Hardware Specifications

- **Host:** Repurposed Fujitsu PC (2014 model)
- **CPU:** Intel Core i5 (sufficient for VPN workload)
- **RAM:** 8GB total, 1GB allocated to Docker container
- **Storage:** 120GB SSD
- **Network:** WiFi connection to home router (1Gbps fiber)

### Software Stack

- **OS:** Ubuntu 22.04 LTS
- **Container Runtime:** Docker 24.x
- **Xray Image:** teddysun/xray:latest (actively maintained)
- **cloudflared:** Latest stable release

### Resource Usage (Typical)

- **CPU:** 1-2% average, 5-10% peak
- **RAM:** 200-300MB (Docker container)
- **Disk:** Logs rotate automatically, <1GB total
- **Network:** Depends on client usage

---

## Design Trade-offs

### What This Architecture Sacrifices

**Performance:**
- Slightly higher latency than WireGuard (kernel-level vs userspace)
- Lower throughput than bare-metal VPN (Docker/tunnel overhead)

**Simplicity:**
- More complex than simple WireGuard setup
- Requires understanding of Docker, Cloudflare, Xray

**Cost:**
- Domain name required (~$10/year)
- Cloudflare Tunnel free tier (currently used)

### What This Architecture Gains

**Anti-censorship:**
- Works in restrictive network environments where WireGuard fails
- Traffic obfuscation prevents protocol detection
- CDN infrastructure provides natural camouflage

**Security:**
- No exposed ports on home network
- Cloudflare DDoS protection
- Real IP address hidden

**Maintainability:**
- Easy updates via Docker
- Reproducible configuration
- Isolated environment

---

## Future Architecture Considerations

### Planned Improvements

**Phase 1: Enhanced Monitoring**
- Prometheus metrics collection
- Grafana dashboards for traffic analysis
- Automated alerting for anomalies

**Phase 2: Access Control**
- Cloudflare Access for identity-based auth
- Per-service access policies with iptables
- Separate VLANs for different access levels

**Phase 3: Redundancy**
- Second Xray instance on different host
- Load balancing between instances
- Automatic failover

**Phase 4: Advanced Anti-Censorship**
- Domain fronting techniques
- Multiple CDN providers (not just Cloudflare)
- Traffic randomization patterns

### Why Not Now?

**Honest assessment:**

1. **Learning priority** - Currently focused on Network+ certification (Dec 28, 2025)
2. **Stability first** - Current setup proven stable over 3+ months
3. **Understanding before complexity** - Want to master Zero-Trust principles before implementing
4. **Risk management** - Avoiding changes during exam prep period

This is a learning journey. Current architecture taught VPN fundamentals and anti-censorship techniques. Next phases will teach enterprise-grade security and high-availability systems.

---

**Related Documentation:**
- [Setup Guide](setup-guide.md) - How to deploy this architecture
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
- [Alternatives](alternatives.md) - Why not other VPN solutions
