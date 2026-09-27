# Troubleshooting

Real issues encountered while running this gateway and how I solved them.

---

## Issue 1: DNS Migration from DuckDNS to Namecheap Domain

### Timeline
- **July 22, 2025**: Started with DuckDNS (example.duckdns.org)
- **July 28, 2025**: DNS resolution became unreliable
- **July 29, 2025 01:42**: Migrated to vpn.example.com

### Symptoms
DNS response was very slow (500ms+) or timing out completely. Client devices couldn't connect to VPN even though server-side services (Xray, Cloudflare Tunnel) were running normally.

```bash
dig example.duckdns.org
# Response time: 536ms (should be <50ms)
# Sometimes: SERVFAIL or timeout
```

### Root Cause
DuckDNS is a free service with no SLA. DNS updates were delayed or failing silently. My home IP had changed but DuckDNS didn't update the record in time.

DuckDNS limitations I discovered:
- DNS update delays
- Occasional service instability
- Long TTL means slow propagation
- Not suitable for production use

### Solution
Purchased domain from Namecheap (~$10/year) and moved DNS management to Cloudflare.

```bash
# Check DNS propagation
dig vpn.example.com

# Expected output:
# ANSWER SECTION:
# vpn.example.com. 300 IN A <Cloudflare-IP>

# Reissue certificate
acme.sh --issue --dns dns_cf -d vpn.example.com
```

Time spent debugging: about 24 hours (including waiting for DNS propagation and hoping it was temporary)

### Evidence
```bash
ls -la ~/.acme.sh/
# example.duckdns.org_ecc  - 2025-07-22 00:56:29
# vpn.example.com_ecc      - 2025-07-29 01:42:32
```
These timestamps show the 7-day period I used DuckDNS before migrating.

### Learning
Free services are great for learning but insufficient for production. The $10/year investment eliminated a major reliability concern. Always check DNS first when connectivity fails - it's often the simplest explanation.

---

## Issue 2: Xray Container Failed to Start - JSON Config Error

### Timeline
**July 28, 2025, 18:21-18:25 JST** (4 minutes of continuous failures)

### Symptoms
Docker container kept restarting. Every startup attempt failed with EOF error.

```bash
docker logs xray
# Output: Failed to start: main: failed to load config files: 
# [/etc/xray/config.json] > infra/conf/serial: failed to decode config > EOF
```

Container failed 13 consecutive times before I fixed it.

### Root Cause
I was editing the config file and probably saved it at a bad moment - incomplete JSON or missing bracket. EOF error typically means the file is truncated or malformed.

### Solution
Fixed the JSON syntax error. Container started successfully at 18:25:20.

```bash
# Validate JSON syntax
cat /etc/xray/config.json | jq .
# If this returns error, JSON is malformed

# After fixing:
docker restart xray
docker logs xray | tail -20
# Check for successful startup
```

### What I Should Have Done
```bash
# Always backup before editing
cp /etc/xray/config.json /etc/xray/config.json.bak

# Validate before restarting
jq . /etc/xray/config.json

# Only restart if validation passes
```

### Evidence
Docker logs showing exact timestamps and 13 consecutive failures:
```
2025-07-28T18:21:18.095415383Z Failed to start...
2025-07-28T18:21:18.970090956Z Failed to start...
[... 11 more failures ...]
2025-07-28T18:25:05.216456974Z Failed to start...
2025-07-28T18:25:20.820605517Z [Warning] ... (successful start)
```

### Learning
Always validate config files before applying. Use version control for important configs. Keep backups. Don't edit production configs directly - test changes in development first.

Also learned that Docker restart policies are both helpful (auto-retry) and dangerous (can mask problems with infinite restart loops).

---

## Issue 3: Cloudflare Tunnel Token Update Required

### Timeline
**July 29, 2025, around 01:44 JST** (during DNS migration)

### Symptoms
After switching from DuckDNS to new domain, Cloudflare Tunnel status showed INACTIVE. The tunnel existed but couldn't establish connection.

```bash
cloudflared tunnel info my-vpn-tunnel
# Output: Status: INACTIVE
```

### Root Cause
DNS change required regenerating the tunnel authentication token. The old token was tied to the previous DNS configuration.

### Solution
Backed up old credentials and generated new token:

```bash
# Backup old token
cp ~/.cloudflared/cert.pem ~/.cloudflared/cert.pem.bak

# Generate new token
cloudflared tunnel token my-vpn-tunnel

# Update config
sudo vim /etc/cloudflared/config.yml
# (update with new token)

# Restart service
sudo systemctl restart cloudflared

# Verify
cloudflared tunnel info my-vpn-tunnel
# Output: Status: ACTIVE
```

Time to resolve: about 30 minutes

### Evidence
Token comparison shows different values:
```bash
# Old token (backed up):
cat ~/.cloudflared/old_cred.json.bak
# Token: <redacted>

# Current token:
cat ~/.cloudflared/cert.pem
# Token: <redacted>
```
Backup files created at 2025-07-29 01:44:51.

### Learning
DNS changes can have cascading effects - not just certificates but also authentication tokens. One big change (DNS migration) triggered multiple smaller issues:
- Certificate reissue
- Token regeneration
- Config file updates

Always think about component dependencies when making system changes. Backup everything before major changes. Validate each step before moving to next.

---

## Issue 4: Full VPN Outage - Expired Certificate + Broken Client

### Timeline
**June 2026** - VPN became completely unusable. Two independent failures happened at the same time, which made the diagnosis much harder.

### Symptoms
The macOS client (Hiddify) failed to start:
```
failed to start core / no such file Working/configs/*.json
127.0.0.1:59202 refused
```
On top of that, connections from other paths worked intermittently at best.

### Root Cause
**Failure 1 - Client: broken Hiddify build**
The installed app was the Mac App Store Catalyst port (`apple.hiddify.com~iosmac`, labelled "Not verified for macOS"). macOS had cleaned up its compiled config cache, so the core could not start.

**Failure 2 - Server: expired TLS certificate**
Probing the service locally, bypassing the client completely:
```bash
curl -vk https://127.0.0.1:8443
# expire date: <in the past>
# HTTP/1.1 400 Bad Request
# sec-websocket-version: 13
```
The certificate had expired, but the `sec-websocket-version: 13` header proved Xray itself was alive. That ruled out "the service is down" and pointed at the certificate layer.

The real cause: the Cloudflare API token (`CF_Token`) that acme.sh uses for DNS validation had become invalid. **Auto-renewal had been failing silently** until the certificate finally expired.

**Red herrings**
- A leftover systemd-managed Xray service (disabled)
- An orphaned `/usr/bin/xray` process

For a while it was unclear which Xray was actually serving traffic. It turned out to be the Docker container; the redundant ones were removed.

### Solution
**Client:**
```bash
# Remove the App Store Catalyst version, install the official GitHub release (.dmg, v4.1.1)
xattr -dr com.apple.quarantine /Applications/Hiddify.app
```

**Server:**
```bash
# 1. Create a new Cloudflare API token (Zone:DNS:Edit + Zone:Zone:Read)
#    and update CF_Token for acme.sh
# 2. Force renewal
acme.sh --renew -d vpn.example.com --force
# 3. Le_ReloadCmd restarts the container automatically
#    (docker restart xray)
```
Certificate renewed successfully and Xray picked it up. Note: Let's Encrypt certificates are valid for 90 days, so this one ran until September 21, 2026 - which matters in [Issue 5](#issue-5-server-frozen-by-a-restart-loop-of-leftover-services).

### Security Hardening
During the fix I realized the VPN UUID had once been accidentally posted in a chat app (the message was later deleted). **Deleting a message does not un-leak a secret**, so I rotated the UUID and invalidated the old one.

### Result

| Item | Before | After |
|---|---|---|
| Client | Catalyst port, core failed to start | Official GitHub release, working |
| Server certificate | Expired | Renewed, auto-renewal restored |
| Xray processes | Redundant / orphaned instances | Single instance (Docker container) |
| VPN credential (UUID) | Previously exposed | Rotated |

### Learning
1. **Separate variables when two failures overlap.** Test client and server independently instead of assuming a single cause. `curl` against the local port cuts the client out of the picture.
2. **Silent automation failures are the most dangerous.** acme.sh renewal failed with no alert at all. Automation needs alerting on failure, not just on success.
3. **Leaked credentials must be rotated, not deleted.** Once exposure is suspected, rotation is the only reliable fix.
4. **Distribution channel defines the trust boundary.** The same open-source client can be reliable from the official GitHub release and broken as a third-party App Store port - especially important for security tools.

### Follow-up
- [ ] Rotate the Cloudflare Tunnel token (credential hygiene)
- [ ] Make sure Xray / cloudflared start on boot (`restart: unless-stopped` / `systemctl enable`)
- [ ] Verify the acme.sh renewal cron runs cleanly with the new token
- [ ] Certificate expiry check N days before expiration, with notification
- [ ] Backup route that does not depend on Cloudflare (Xray Reality)
- [ ] Move server credentials (tokens, UUID, SSH) into a password manager

---

## Issue 5: Server Frozen by a Restart Loop of Leftover Services

### Timeline
- **August 16, 2026**: last journal entry the server managed to write
- **August 21, 2026**: scheduled certificate renewal failed - no trace, because cron output went to `/dev/null`
- **September 21, 2026**: certificate expired
- **September 27, 2026**: noticed while trying to SSH in; fixed the same day

### Symptoms
From the Mac:
```
$ ssh fujitsu
websocket: bad handshake
Connection closed by UNKNOWN port 65535
```
```bash
curl -sI https://ssh.example.com | head -1   # HTTP/2 302
curl -sI https://vpn.example.com | head -1   # HTTP/2 530
```
On the server's own console: an endless flood of `systemd-journald: Failed to write entry`. Typing was impossible - switching TTYs and Magic SysRq didn't help either.

**Diagnostic trap:** the 302 on the SSH hostname looks healthy, but it means nothing - Cloudflare Access redirects to its login page *before* the request ever reaches the origin. The 530 on the VPN hostname (same tunnel) was the real signal: Cloudflare could not reach the server at all.

### Recovery
1. Hard power-off (long press), then GRUB → Advanced options → recovery mode
2. Resumed normal boot - the server came back and SSH worked again

### Root Cause
The disk was fine (22% used, mounted read-write, no EXT4 or I/O errors). The previous boot's journal told the real story:
```
hiddify-haproxy.service: Scheduled restart job, restart counter is at 2060030.
hiddify-nginx.service: Failed to locate executable /opt/hiddify-manager/nginx/pre-start.sh: No such file or directory
hiddify-singbox.service: Failed to locate executable /opt/hiddify-manager/singbox/sing-box: No such file or directory
```
When I simplified the stack to Xray + Cloudflare Tunnel (see [Multi-System Integration Challenges](#multi-system-integration-challenges-october-november-2025)), I deleted the Hiddify Manager files but **not its systemd units**. Eight `hiddify-*` units stayed enabled, each failing and restarting about once per second - over 2 million restarts each. The journal grew to 3.9 GB, journald could no longer write, and the machine became unusable.

The expired certificate was a second, related problem: the August 21 renewal failed while the server was in this state, and since the acme.sh cron job discarded all output, there was no record of why.

### Solution
```bash
# 1. Disable leftover units and move them to a backup directory (not deleted)
sudo systemctl disable --now $(systemctl list-unit-files 'hiddify*' --no-legend | awk '{print $1}')
sudo mkdir -p /root/hiddify-units-backup
sudo mv /etc/systemd/system/hiddify-* /root/hiddify-units-backup/
sudo systemctl daemon-reload && sudo systemctl reset-failed

# 2. Shrink the journal and cap its size
sudo journalctl --vacuum-size=500M      # 3.9G -> 416M
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=500M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald

# 3. Renew the certificate (acme.sh runs as the normal user, not root)
~/.acme.sh/acme.sh --renew -d vpn.example.com --ecc --force
sudo openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -enddate
# notAfter=Dec 26 2026
```
Verified afterwards: cloudflared `active` and `enabled`, Xray restart policy `unless-stopped`, VPN hostname no longer returns 530.

### Learning
1. **Removing software means removing its services.** Deleting the files left enabled systemd units behind, and they failed quietly for weeks. After uninstalling anything, check `systemctl list-unit-files`.
2. **Unbounded logs can take down a whole machine.** One restart loop turned into gigabytes of journal. A log size limit is a safety feature, not housekeeping.
3. **`> /dev/null` in a cron job hides failures.** The same "silent automation" lesson as Issue 4, found in a different place.
4. **Know what each response actually proves.** A 302 from an Access-protected hostname says nothing about the origin; a 530 does.
5. **Keep a way in when remote access is gone.** The server's LAN IP and username should be written down somewhere other than the server - I had to dig them out of `~/.ssh/config` and shell history.

### Follow-up
- [ ] Send acme.sh cron output to a log file and remove the duplicate cron entry
- [ ] Remove the stale DuckDNS entry from acme.sh
- [ ] Certificate expiry alert - still the most important missing piece
- [ ] Look for other leftovers from the old multi-service setup (e.g. `/usr/bin/xray`)

---

## Additional Challenges (Summary)

### Automation Script Development (September 2025)
Attempted to create one-click VPN control scripts. Hit multiple issues:

Common mistakes I made:
- Using `/bin/zsh` on Ubuntu (doesn't exist by default)
- Copying macOS commands (`brew`) to Linux
- EOF syntax errors in heredocs
- sudoers file permissions (needs 0440)
- systemd service name confusion (cloudflared vs cloudflared.service)

Bash history limitations (2000 lines) means I lost detailed records of the debugging process, but the backup files prove I worked through these issues.

**Learning**: Test scripts incrementally. Validate syntax before deployment. Understand the differences between development (macOS) and production (Linux) environments.

### Security Evolution (August 2025)

**Initial Setup**  
Started with services exposed on `0.0.0.0` (all interfaces). Didn't understand the implications.

**Realization**  
Discovered that if Cloudflare Tunnel goes down, VPS public ports are directly accessible. This was a turning point in understanding production vs. development security.

**Actions Taken**
- Changed Docker port binding to `127.0.0.1:8443` (localhost only)
- Configured systemd restart policies
- Set up basic UFW firewall rules
- Understood that "defense in depth" means multiple security layers

**Learning**: Even if one layer fails (e.g., tunnel drops), other layers should prevent exposure. Security isn't just about encryption - it's about proper network configuration.

### Multi-System Integration Challenges (October-November 2025)

**The Problem**  
At peak complexity, I was running:
- Xray (VLESS)
- Hiddify-Xray
- WireGuard
- Cloudflare Tunnel

Issues encountered:
- Port conflicts
- Service startup order dependencies
- Configuration file locations scattered
- Unclear which service was actually being used

**Resolution**  
Simplified to just Xray + Cloudflare Tunnel. Removed redundant services - or so I thought. The Hiddify systemd units were left behind and eventually froze the server ([Issue 5](#issue-5-server-frozen-by-a-restart-loop-of-leftover-services)). Learned that more components = more complexity = more failure points.

**Learning**: Keep it simple. Only add complexity when actually needed. Every additional component is a potential maintenance burden.

---

## Prevention Strategies Going Forward

### What I'm Doing Differently Now

1. **Increasing bash history size**:
```bash
export HISTSIZE=10000
export HISTFILESIZE=20000
```

2. **Maintaining a troubleshooting log**:
```bash
echo "$(date): [problem description]" >> ~/troubleshooting.log
```

3. **Other improvements**:
- Taking screenshots of critical steps
- Version control for config files
- Testing changes in staging before production

### Monitoring Setup
- Journal size capped at 500M (after Issue 5)
- Certificate expiration alerts (30 days before) - **planned, not yet in place.** Issue 4 happened exactly because renewal failed silently.
- Cloudflare Tunnel health checks
- Container restart count monitoring
- Weekly log review

---

## Meta-Learning: The Value of Failure

The biggest lesson isn't any specific technical solution. It's that losing detailed logs taught me the importance of documentation.

I encountered 30+ issues in the first 6 months, but only have complete evidence for 3. The others are lost to bash history limits and log rotation. This itself is a valuable lesson about production operations.

In future projects, I'll prioritize documentation from day one. Not just for others, but for "future me" who won't remember why things were done certain ways.

---

## Common Diagnostic Commands

Quick health check:
```bash
# Tunnel status
systemctl status cloudflared

# Container status
docker ps | grep xray
docker logs xray --tail 50

# Connectivity test
curl -I https://vpn.example.com

# Certificate check
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates
```

When something breaks, start with these commands. They catch 90% of issues.

---

**Related Documentation:**
- [Architecture](architecture.md) - How the system works
- [Setup Guide](setup-guide.md) - Deployment instructions
- [What I Learned](what-i-learned.md) - Personal reflections
- [Security](security.md) - Security considerations

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# トラブルシューティング

このゲートウェイの運用中に実際に遭遇した問題とその解決方法の記録。

---

## 問題1：DuckDNSからNamecheapドメインへのDNS移行

### 経緯
- **2025年7月22日**：DuckDNS（example.duckdns.org）で開始
- **2025年7月28日**：DNS解決が不安定になる
- **2025年7月29日 01:42**：vpn.example.comへ移行

### 症状
DNSレスポンスが非常に遅い（500ms以上）か、完全にタイムアウト。サーバー側のサービス（Xray、Cloudflare Tunnel）は正常に動作しているのに、クライアント端末からVPNに接続できない状態。

```bash
dig example.duckdns.org
# レスポンス時間: 536ms（50ms以下であるべき）
# 時々: SERVFAILまたはタイムアウト
```

### 根本原因
DuckDNSは無料サービスでSLA（サービスレベル契約）がない。DNS更新が遅延したり、エラーが出ても通知されない。自宅のIPアドレスが変更されたが、DuckDNSが記録を適時に更新できなかった。

発見したDuckDNSの限界：
- DNS更新の遅延
- 時折のサービス不安定
- 長いTTLによる伝播の遅さ
- 本番環境には不向き

### 解決方法
Namecheapでドメインを購入（年間約10ドル）し、DNS管理をCloudflareに移行。

```bash
# DNS伝播を確認
dig vpn.example.com

# 期待される出力:
# ANSWER SECTION:
# vpn.example.com. 300 IN A <Cloudflare-IP>

# 証明書を再発行
acme.sh --issue --dns dns_cf -d vpn.example.com
```

デバッグにかかった時間：約24時間（DNS伝播待ちと、一時的な問題かもしれないという期待を含む）

### 証拠
```bash
ls -la ~/.acme.sh/
# example.duckdns.org_ecc  - 2025-07-22 00:56:29
# vpn.example.com_ecc      - 2025-07-29 01:42:32
```
これらのタイムスタンプは、移行前にDuckDNSを使用していた7日間を示している。

### 学んだこと
無料サービスは学習には最適だが、本番環境には信頼性が不足。年間10ドルの投資で大きな信頼性の懸念を解消できた。接続障害が発生したら、まずDNSを確認 - 最もシンプルな説明が正しいことが多い。

---

## 問題2：Xrayコンテナの起動失敗 - JSON設定エラー

### 経緯
**2025年7月28日 18:21-18:25（日本時間）**（4分間の連続失敗）

### 症状
Dockerコンテナが再起動を繰り返す。すべての起動試行がEOFエラーで失敗。

```bash
docker logs xray
# 出力: Failed to start: main: failed to load config files: 
# [/etc/xray/config.json] > infra/conf/serial: failed to decode config > EOF
```

修正するまでにコンテナは13回連続で失敗。

### 根本原因
設定ファイルを編集中に、おそらく不適切なタイミングで保存 - JSONが不完全だったか括弧が欠けていた。EOFエラーは通常、ファイルが途中で切れているか形式が不正であることを示す。

### 解決方法
JSON構文エラーを修正。18:25:20にコンテナが正常起動。

```bash
# JSON構文を検証
cat /etc/xray/config.json | jq .
# エラーが返された場合、JSONが不正

# 修正後:
docker restart xray
docker logs xray | tail -20
# 正常起動を確認
```

### やるべきだったこと
```bash
# 編集前に必ずバックアップ
cp /etc/xray/config.json /etc/xray/config.json.bak

# 再起動前に検証
jq . /etc/xray/config.json

# 検証が通った場合のみ再起動
```

### 証拠
正確なタイムスタンプと13回連続失敗を示すDockerログ：
```
2025-07-28T18:21:18.095415383Z Failed to start...
2025-07-28T18:21:18.970090956Z Failed to start...
[... さらに11回の失敗 ...]
2025-07-28T18:25:05.216456974Z Failed to start...
2025-07-28T18:25:20.820605517Z [Warning] ... (起動成功)
```

### 学んだこと
設定ファイルは適用前に必ず検証。重要な設定はバージョン管理を使用。バックアップを保持。本番環境の設定を直接編集しない - まず開発環境で変更をテスト。

また、Dockerの再起動ポリシーは有用（自動リトライ）でもあり危険（無限再起動ループで問題を隠蔽する可能性）でもあると学んだ。

---

## 問題3：Cloudflare Tunnelトークン更新が必要

### 経緯
**2025年7月29日 01:44頃（日本時間）**（DNS移行中）

### 症状
DuckDNSから新しいドメインに切り替えた後、Cloudflare TunnelのステータスがINACTIVEと表示。トンネルは存在するが接続を確立できない。

```bash
cloudflared tunnel info my-vpn-tunnel
# 出力: Status: INACTIVE
```

### 根本原因
DNS変更により、トンネル認証トークンの再生成が必要になった。古いトークンは前のDNS設定に紐付けられていた。

### 解決方法
古い認証情報をバックアップし、新しいトークンを生成：

```bash
# 古いトークンをバックアップ
cp ~/.cloudflared/cert.pem ~/.cloudflared/cert.pem.bak

# 新しいトークンを生成
cloudflared tunnel token my-vpn-tunnel

# 設定を更新
sudo vim /etc/cloudflared/config.yml
# (新しいトークンで更新)

# サービスを再起動
sudo systemctl restart cloudflared

# 確認
cloudflared tunnel info my-vpn-tunnel
# 出力: Status: ACTIVE
```

解決にかかった時間：約30分

### 証拠
トークン比較で異なる値が確認できる：
```bash
# 古いトークン（バックアップ済み）:
cat ~/.cloudflared/old_cred.json.bak
# Token: <redacted>

# 現在のトークン:
cat ~/.cloudflared/cert.pem
# Token: <redacted>
```
バックアップファイルは2025-07-29 01:44:51に作成。

### 学んだこと
DNS変更には連鎖的な影響がある - 証明書だけでなく認証トークンにも影響。一つの大きな変更（DNS移行）が複数の小さな問題を引き起こした：
- 証明書の再発行
- トークンの再生成
- 設定ファイルの更新

システム変更を行う際は、コンポーネント間の依存関係を常に考慮。大きな変更の前にすべてをバックアップ。次のステップに進む前に各ステップを検証。

---

## 問題4：VPN全面停止 - 証明書期限切れとクライアント破損

### タイムライン
**2026年6月** - VPNが完全に使用不能に。2つの独立した障害が同時に発生し、原因の切り分けが非常に難しくなった。

### 症状
macOSクライアント（Hiddify）が起動しない：
```
failed to start core / no such file Working/configs/*.json
127.0.0.1:59202 refused
```
さらに、他の経路からの接続もつながったりつながらなかったりする状態。

### 根本原因
**障害1 - クライアント：Hiddifyのビルド破損**
インストールされていたのはMac App StoreのCatalyst移植版（`apple.hiddify.com~iosmac`、「Not verified for macOS」表記）。macOSがコンパイル済み設定キャッシュを削除したため、コアが起動できなくなっていた。

**障害2 - サーバー：TLS証明書の期限切れ**
クライアントを介さず、ローカルでサービスを直接確認：
```bash
curl -vk https://127.0.0.1:8443
# expire date: <過去の日付>
# HTTP/1.1 400 Bad Request
# sec-websocket-version: 13
```
証明書は期限切れだったが、`sec-websocket-version: 13` ヘッダーによりXray自体は生きていることが確認できた。「サービスが落ちている」という仮説を排除し、証明書レイヤーに絞り込めた。

真の原因：acme.shがDNS検証に使うCloudflare APIトークン（`CF_Token`）が無効になっていた。**自動更新は静かに失敗し続け**、証明書が期限切れになるまで誰も気づかなかった。

**紛らわしかった要素**
- 残っていたsystemd管理のXrayサービス（無効化済み）
- 孤立した `/usr/bin/xray` プロセス

どのXrayが実際に通信を処理しているのか、しばらく判断がつかなかった。最終的にDockerコンテナ内のXrayが本物と確認し、冗長なものは削除した。

### 解決策
**クライアント：**
```bash
# App StoreのCatalyst版を削除し、GitHub公式リリース（.dmg、v4.1.1）をインストール
xattr -dr com.apple.quarantine /Applications/Hiddify.app
```

**サーバー：**
```bash
# 1. Cloudflareで新しいAPIトークンを作成（Zone:DNS:Edit + Zone:Zone:Read）
#    acme.shのCF_Tokenを更新
# 2. 強制更新
acme.sh --renew -d vpn.example.com --force
# 3. Le_ReloadCmdによりコンテナが自動再起動
#    (docker restart xray)
```
証明書の更新に成功し、Xrayに反映された。注：Let's Encryptの証明書は90日間有効のため、この証明書は2026年9月21日まで - これが[問題5](#問題5残存サービスの再起動ループでサーバーが停止)に関わってくる。

### セキュリティ強化
修正中に、VPNのUUIDを過去にチャットアプリへ誤って投稿していたことに気づいた（メッセージは後で削除済み）。**メッセージを削除しても漏洩はなかったことにならない**ため、UUIDをローテーションし、古いものを無効化した。

### 結果

| 項目 | 修正前 | 修正後 |
|---|---|---|
| クライアント | Catalyst移植版、コア起動不可 | GitHub公式リリース、正常動作 |
| サーバー証明書 | 期限切れ | 更新済み、自動更新復旧 |
| Xrayプロセス | 冗長・孤立インスタンス | 単一インスタンス（Dockerコンテナ） |
| VPN認証情報（UUID） | 過去に露出 | ローテーション済み |

### 学んだこと
1. **2つの障害が重なったら、まず変数を分離する。** 単一原因と決めつけず、クライアントとサーバーを別々に検証する。ローカルポートへの `curl` でクライアントを切り離せる。
2. **静かに失敗する自動化が最も危険。** acme.shの更新失敗はアラートを一切出さなかった。自動化には成功時だけでなく失敗時の通知が必要。
3. **漏洩した認証情報は削除ではなくローテーションする。** 露出が疑われた時点で、ローテーションが唯一確実な対処。
4. **配布チャネルが信頼境界を決める。** 同じオープンソースクライアントでも、GitHub公式リリースとサードパーティのApp Store移植版では信頼性が根本的に異なりうる。セキュリティツールでは特に重要。

### 今後の対応
- [ ] Cloudflare Tunnelトークンのローテーション（認証情報の衛生管理）
- [ ] Xray / cloudflared の自動起動設定（`restart: unless-stopped` / `systemctl enable`）
- [ ] 新しいトークンでacme.shの更新cronが正常に動くか検証
- [ ] 証明書期限のN日前チェックと通知
- [ ] Cloudflareに依存しないバックアップ経路（Xray Reality）
- [ ] サーバー認証情報（トークン、UUID、SSH）をパスワードマネージャーに移行

---

## 問題5：残存サービスの再起動ループでサーバーが停止

### タイムライン
- **2026年8月16日**：サーバーが書き込めた最後のジャーナル
- **2026年8月21日**：証明書の定期更新が失敗 - cronの出力が `/dev/null` に捨てられていたため記録なし
- **2026年9月21日**：証明書の期限切れ
- **2026年9月27日**：SSHしようとして発覚、同日中に復旧

### 症状
Macから：
```
$ ssh fujitsu
websocket: bad handshake
Connection closed by UNKNOWN port 65535
```
```bash
curl -sI https://ssh.example.com | head -1   # HTTP/2 302
curl -sI https://vpn.example.com | head -1   # HTTP/2 530
```
サーバー本体のコンソールには `systemd-journald: Failed to write entry` が延々と流れ続け、入力不能。TTY切り替えもMagic SysRqも効かなかった。

**診断の落とし穴：** SSHホスト名の302は正常に見えるが、何の証明にもならない。Cloudflare Accessはリクエストがオリジンに届く*前に*ログインページへリダイレクトするため。同じトンネルを使うVPNホスト名の530こそが本当のシグナルだった：Cloudflareがサーバーにまったく到達できていない。

### 復旧
1. 電源長押しで強制終了し、GRUB → Advanced options → recovery mode
2. 通常起動を再開 - サーバーが復帰し、SSHも使えるようになった

### 根本原因
ディスクは正常（使用率22%、読み書き可能でマウント、EXT4やI/Oエラーなし）。前回起動時のジャーナルが本当の原因を示していた：
```
hiddify-haproxy.service: Scheduled restart job, restart counter is at 2060030.
hiddify-nginx.service: Failed to locate executable /opt/hiddify-manager/nginx/pre-start.sh: No such file or directory
hiddify-singbox.service: Failed to locate executable /opt/hiddify-manager/singbox/sing-box: No such file or directory
```
構成をXray + Cloudflare Tunnelにシンプル化した際（[複数システム統合の課題](#複数システム統合の課題2025年10-11月)を参照）、Hiddify Managerのファイルは削除したが**systemdユニットは残っていた**。8つの `hiddify-*` ユニットが有効なまま、約1秒ごとに失敗と再起動を繰り返し、それぞれ200万回以上再起動。ジャーナルは3.9GBまで膨れ上がり、journaldが書き込めなくなり、マシンが使用不能になった。

期限切れの証明書は、関連するもう一つの問題：8月21日の更新はサーバーがこの状態の間に失敗しており、acme.shのcronジョブが出力をすべて捨てていたため、原因の記録が残っていなかった。

### 解決策
```bash
# 1. 残存ユニットを無効化し、バックアップディレクトリへ移動（削除はしない）
sudo systemctl disable --now $(systemctl list-unit-files 'hiddify*' --no-legend | awk '{print $1}')
sudo mkdir -p /root/hiddify-units-backup
sudo mv /etc/systemd/system/hiddify-* /root/hiddify-units-backup/
sudo systemctl daemon-reload && sudo systemctl reset-failed

# 2. ジャーナルを縮小し、サイズ上限を設定
sudo journalctl --vacuum-size=500M      # 3.9G -> 416M
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=500M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald

# 3. 証明書を更新（acme.shはrootではなく通常ユーザーで動作）
~/.acme.sh/acme.sh --renew -d vpn.example.com --ecc --force
sudo openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -enddate
# notAfter=Dec 26 2026
```
復旧後に確認：cloudflaredは `active` かつ `enabled`、Xrayの再起動ポリシーは `unless-stopped`、VPNホスト名は530を返さなくなった。

### 学んだこと
1. **ソフトウェアの削除にはサービスの削除も含まれる。** ファイルだけ消して有効なsystemdユニットが残り、何週間も静かに失敗し続けた。何かをアンインストールしたら `systemctl list-unit-files` を確認する。
2. **上限のないログはマシン全体を止めうる。** 一つの再起動ループが数GBのジャーナルになった。ログのサイズ上限は片付けではなく安全機能。
3. **cronの `> /dev/null` は失敗を隠す。** 問題4と同じ「静かに失敗する自動化」の教訓を、別の場所で再発見。
4. **各レスポンスが実際に何を証明するかを知る。** Access保護下のホスト名の302はオリジンについて何も語らない。530は語る。
5. **リモートアクセスを失った時の入口を確保する。** サーバーのLAN IPとユーザー名はサーバー以外の場所に記録しておくべき - 今回は `~/.ssh/config` とシェル履歴から掘り出した。

### 今後の対応
- [ ] acme.shのcron出力をログファイルに残し、重複したcronエントリを削除
- [ ] acme.shから古いDuckDNSのエントリを削除
- [ ] 証明書期限アラート - 依然として最も重要な欠落
- [ ] 旧マルチサービス構成の他の残骸を探す（例：`/usr/bin/xray`）

---

## その他の課題（概要）

### 自動化スクリプト開発（2025年9月）
ワンクリックでVPNを制御するスクリプトを作成しようと試みたが、複数の問題に遭遇：

よくやった間違い：
- Ubuntuで`/bin/zsh`を使用（デフォルトでは存在しない）
- macOSのコマンド（`brew`）をLinuxにコピー
- heredocでのEOF構文エラー
- sudoersファイルのパーミッション（0440が必要）
- systemdサービス名の混乱（cloudflared vs cloudflared.service）

Bash履歴の制限（2000行）により、デバッグプロセスの詳細な記録は失われたが、バックアップファイルがこれらの問題を解決した証拠として残っている。

**学んだこと**：スクリプトは段階的にテスト。デプロイ前に構文を検証。開発環境（macOS）と本番環境（Linux）の違いを理解する。

### セキュリティの進化（2025年8月）

**初期設定**  
サービスを`0.0.0.0`（すべてのインターフェース）で公開していた。その影響を理解していなかった。

**気づき**  
Cloudflare Tunnelがダウンすると、VPSの公開ポートが直接アクセス可能になることを発見。これが本番環境と開発環境のセキュリティを理解する転機となった。

**取った対策**
- Dockerポートバインディングを`127.0.0.1:8443`（ローカルホストのみ）に変更
- systemd再起動ポリシーの設定
- 基本的なUFWファイアウォールルールの設定
- 「多層防御」が複数のセキュリティレイヤーを意味することを理解

**学んだこと**：一つのレイヤーが失敗しても（例：トンネルがダウン）、他のレイヤーが露出を防ぐべき。セキュリティは暗号化だけではない - 適切なネットワーク設定も含む。

### 複数システム統合の課題（2025年10-11月）

**問題**  
最も複雑な時期には、以下を同時に実行していた：
- Xray（VLESS）
- Hiddify-Xray
- WireGuard
- Cloudflare Tunnel

遭遇した問題：
- ポート競合
- サービス起動順序の依存関係
- 設定ファイルの場所が分散
- どのサービスが実際に使用されているか不明

**解決**  
XrayとCloudflare Tunnelのみにシンプル化。冗長なサービスを削除 - したつもりだった。Hiddifyのsystemdユニットが残っており、最終的にサーバーを停止させた（[問題5](#問題5残存サービスの再起動ループでサーバーが停止)）。コンポーネントが多い = 複雑さが増す = 障害ポイントが増える、と学んだ。

**学んだこと**：シンプルに保つ。実際に必要な場合のみ複雑さを追加。追加コンポーネントはすべて潜在的なメンテナンス負担。

---

## 今後の予防策

### 現在異なるやり方でやっていること

1. **bash履歴サイズの増加**:
```bash
export HISTSIZE=10000
export HISTFILESIZE=20000
```

2. **トラブルシューティングログの保持**:
```bash
echo "$(date): [問題の説明]" >> ~/troubleshooting.log
```

3. **その他の改善**:
- 重要なステップのスクリーンショット撮影
- 設定ファイルのバージョン管理
- 本番環境への適用前にステージングでテスト

### 監視の設定
- ジャーナルサイズを500Mに制限（問題5の後）
- 証明書有効期限アラート（30日前） - **計画中、未導入。** 問題4はまさに更新が静かに失敗したことが原因。
- Cloudflare Tunnelヘルスチェック
- コンテナ再起動回数の監視
- 週次ログレビュー

---

## メタ学習：失敗の価値

最大の教訓は、特定の技術的解決策ではない。詳細なログを失ったことで、ドキュメンテーションの重要性を学んだ。

最初の6ヶ月で30以上の問題に遭遇したが、完全な証拠があるのは3つのみ。残りはbash履歴の制限とログローテーションで失われた。これ自体が本番運用における貴重な教訓。

今後のプロジェクトでは、初日からドキュメント作成を優先する。他の人のためだけでなく、なぜそのような方法を取ったか覚えていない「未来の自分」のためにも。

---

## よく使う診断コマンド

クイックヘルスチェック：
```bash
# トンネルのステータス
systemctl status cloudflared

# コンテナのステータス
docker ps | grep xray
docker logs xray --tail 50

# 接続テスト
curl -I https://vpn.example.com

# 証明書チェック
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates
```

何かが壊れた場合、これらのコマンドから始める。90%の問題はこれで捕捉できる。

---

**関連ドキュメント：**
- [Architecture](architecture.md) - システムの仕組み
- [Setup Guide](setup-guide.md) - デプロイ手順
- [What I Learned](what-i-learned.md) - 個人的な振り返り
- [Security](security.md) - セキュリティの考慮事項
