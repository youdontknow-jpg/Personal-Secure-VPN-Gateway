Real issues encountered during 3+ months of operation and how I solved them.

Issue 1: DNS Migration from DuckDNS to Namecheap Domain
Timeline
July 22, 2025: Started with DuckDNS (fujilegend.duckdns.org)
July 28, 2025: DNS resolution became unreliable
July 29, 2025 01:42: Migrated to vpn.fujilegend.xyz
Symptoms
DNS response was very slow (500ms+) or timing out completely. Client devices couldn't connect to VPN even though server-side services (Xray, Cloudflare Tunnel) were running normally.

Copydig fujilegend.duckdns.org
# Response time: 536ms (should be <50ms)
# Sometimes: SERVFAIL or timeout
Root Cause
DuckDNS is a free service with no SLA. DNS updates were delayed or failing silently. My home IP had changed but DuckDNS didn't update the record in time.

DuckDNS limitations I discovered:

DNS update delays
Occasional service instability
Long TTL means slow propagation
Not suitable for production use
Solution
Purchased domain from Namecheap (~$10/year) and moved DNS management to Cloudflare.

Copy# Check DNS propagation
dig vpn.fujilegend.xyz

# Expected output:
# ANSWER SECTION:
# vpn.fujilegend.xyz. 300 IN A <Cloudflare-IP>

# Reissue certificate
acme.sh --issue --dns dns_cf -d vpn.fujilegend.xyz
Time spent debugging: about 24 hours (including waiting for DNS propagation and hoping it was temporary)

Evidence
Copyls -la ~/.acme.sh/
# fujilegend.duckdns.org_ecc  - 2025-07-22 00:56:29
# vpn.fujilegend.xyz_ecc      - 2025-07-29 01:42:32
These timestamps show the 7-day period I used DuckDNS before migrating.

Learning
Free services are great for learning but insufficient for production. The $10/year investment eliminated a major reliability concern. Always check DNS first when connectivity fails - it's often the simplest explanation.

Issue 2: Xray Container Failed to Start - JSON Config Error
Timeline
July 28, 2025, 18:21-18:25 JST (4 minutes of continuous failures)

Symptoms
Docker container kept restarting. Every startup attempt failed with EOF error.

Copydocker logs xray
# Output: Failed to start: main: failed to load config files: 
# [/etc/xray/config.json] > infra/conf/serial: failed to decode config > EOF
Container failed 13 consecutive times before I fixed it.

Root Cause
I was editing the config file and probably saved it at a bad moment - incomplete JSON or missing bracket. EOF error typically means the file is truncated or malformed.

Solution
Fixed the JSON syntax error. Container started successfully at 18:25:20.

Copy# Validate JSON syntax
cat /etc/xray/config.json | jq .
# If this returns error, JSON is malformed

# After fixing:
docker restart xray
docker logs xray | tail -20
# Check for successful startup
What I should have done
Copy# Always backup before editing
cp /etc/xray/config.json /etc/xray/config.json.bak

# Validate before restarting
jq . /etc/xray/config.json

# Only restart if validation passes
Evidence
Docker logs showing exact timestamps and 13 consecutive failures:

2025-07-28T18:21:18.095415383Z Failed to start...
2025-07-28T18:21:18.970090956Z Failed to start...
[... 11 more failures ...]
2025-07-28T18:25:05.216456974Z Failed to start...
2025-07-28T18:25:20.820605517Z [Warning] ... (successful start)
Learning
Always validate config files before applying. Use version control for important configs. Keep backups. Don't edit production configs directly - test changes in development first.

Also learned that Docker restart policies are both helpful (auto-retry) and dangerous (can mask problems with infinite restart loops).

Issue 3: Cloudflare Tunnel Token Update Required
Timeline
July 29, 2025, around 01:44 JST (during DNS migration)

Symptoms
After switching from DuckDNS to new domain, Cloudflare Tunnel status showed INACTIVE. The tunnel existed but couldn't establish connection.

Copycloudflared tunnel info fujitsu-vpn
# Output: Status: INACTIVE
Root Cause
DNS change required regenerating the tunnel authentication token. The old token was tied to the previous DNS configuration.

Solution
Backed up old credentials and generated new token:

Copy# Backup old token
cp ~/.cloudflared/cert.pem ~/.cloudflared/cert.pem.bak

# Generate new token
cloudflared tunnel token fujitsu-vpn

# Update config
sudo vim /etc/cloudflared/config.yml
# (update with new token)

# Restart service
sudo systemctl restart cloudflared

# Verify
cloudflared tunnel info fujitsu-vpn
# Output: Status: ACTIVE
Time to resolve: about 30 minutes

Evidence
Token comparison shows different values:

Copy# Old token (backed up):
cat ~/.cloudflared/old_cred.json.bak
# Token: x2H4mWhQ5DT4milyylOyuQ90...

# Current token:
cat ~/.cloudflared/cert.pem
# Token: P1I42eS8tYSSL3aoLAcPZJ_TB...
Backup files created at 2025-07-29 01:44:51.

Learning
DNS changes can have cascading effects - not just certificates but also authentication tokens. One big change (DNS migration) triggered multiple smaller issues:

Certificate reissue
Token regeneration
Config file updates
Always think about component dependencies when making system changes. Backup everything before major changes. Validate each step before moving to next.

Additional Challenges (Summary)
Automation Script Development (September 2025)
Attempted to create one-click VPN control scripts. Hit multiple issues:

Common mistakes I made:

Using /bin/zsh on Ubuntu (doesn't exist by default)
Copying macOS commands (brew) to Linux
EOF syntax errors in heredocs
sudoers file permissions (needs 0440)
systemd service name confusion (cloudflared vs cloudflared.service)
Bash history limitations (2000 lines) means I lost detailed records of the debugging process, but the backup files prove I worked through these issues.

Learning: Test scripts incrementally. Validate syntax before deployment. Understand the differences between development (macOS) and production (Linux) environments.

Security Evolution (August 2025)
Initial Setup
Started with services exposed on 0.0.0.0 (all interfaces). Didn't understand the implications.

Realization
Discovered that if Cloudflare Tunnel goes down, VPS public ports are directly accessible. This was a turning point in understanding production vs. development security.

Actions Taken
Changed Docker port binding to 127.0.0.1:8443 (localhost only)
Configured systemd restart policies
Set up basic UFW firewall rules
Understood that "defense in depth" means multiple security layers
Learning: Even if one layer fails (e.g., tunnel drops), other layers should prevent exposure. Security isn't just about encryption - it's about proper network configuration.

Multi-System Integration Challenges (October-November 2025)
The Problem
At peak complexity, I was running:

Xray (VLESS)
Hiddify-Xray
WireGuard
Cloudflare Tunnel
Issues encountered:

Port conflicts
Service startup order dependencies
Configuration file locations scattered
Unclear which service was actually being used
Resolution
Simplified to just Xray + Cloudflare Tunnel. Removed redundant services. Learned that more components = more complexity = more failure points.

Learning: Keep it simple. Only add complexity when actually needed. Every additional component is a potential maintenance burden.

Prevention Strategies Going Forward
What I'm Doing Differently Now
Increasing bash history size:

Copyexport HISTSIZE=10000
export HISTFILESIZE=20000
Maintaining a troubleshooting log:

Copyecho "$(date): [problem description]" >> ~/troubleshooting.log
Taking screenshots of critical steps

Version control for config files

Testing changes in staging before production

Monitoring Setup
Certificate expiration alerts (30 days before)
Cloudflare Tunnel health checks
Container restart count monitoring
Weekly log review
Meta-Learning: The Value of Failure
The biggest lesson isn't any specific technical solution. It's that losing detailed logs taught me the importance of documentation.

I encountered 30+ issues over 6 months, but only have complete evidence for 3. The others are lost to bash history limits and log rotation. This itself is a valuable lesson about production operations.

In future projects, I'll prioritize documentation from day one. Not just for others, but for "future me" who won't remember why things were done certain ways.

Common Diagnostic Commands
Quick health check:

Copy# Tunnel status
systemctl status cloudflared

# Container status
docker ps | grep xray
docker logs xray --tail 50

# Connectivity test
curl -I https://vpn.fujilegend.xyz

# Certificate check
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates
When something breaks, start with these commands. They catch 90% of issues.

Related Documentation:

Architecture - How the system works
Setup Guide - Deployment instructions
What I Learned - Personal reflections
Security - Security considerations
トラブルシューティングガイド
3ヶ月以上の運用で遭遇した実際の問題とその解決方法。

問題1：DuckDNSからNamecheapドメインへのDNS移行
時系列
2025年7月22日：DuckDNS（fujilegend.duckdns.org）で開始
2025年7月28日：DNS解決が不安定になる
2025年7月29日 01:42：vpn.fujilegend.xyzへ移行
症状
DNS応答が非常に遅い（500ms以上）か完全にタイムアウト。サーバー側のサービス（Xray、Cloudflare Tunnel）は正常に動作しているのに、クライアントデバイスがVPNに接続できない。

Copydig fujilegend.duckdns.org
# 応答時間: 536ms（50ms未満であるべき）
# 時々: SERVFAILまたはタイムアウト
根本原因
DuckDNSはSLAのない無料サービス。DNS更新が遅延するか静かに失敗していた。自宅IPが変わったが、DuckDNSが間に合わずレコードを更新しなかった。

私が発見したDuckDNSの限界：

DNS更新の遅延
時々のサービス不安定性
長いTTLは伝播が遅い
本番環境には不向き
解決方法
Namecheapからドメインを購入（年間約$10）し、DNS管理をCloudflareに移行。

Copy# DNS伝播を確認
dig vpn.fujilegend.xyz

# 期待される出力:
# ANSWER SECTION:
# vpn.fujilegend.xyz. 300 IN A <Cloudflare-IP>

# 証明書を再発行
acme.sh --issue --dns dns_cf -d vpn.fujilegend.xyz
デバッグに費やした時間：約24時間（DNS伝播を待ち、一時的な問題であることを期待していた時間を含む）

証拠
Copyls -la ~/.acme.sh/
# fujilegend.duckdns.org_ecc  - 2025-07-22 00:56:29
# vpn.fujilegend.xyz_ecc      - 2025-07-29 01:42:32
これらのタイムスタンプは、移行前にDuckDNSを使用した7日間を示している。

学び
無料サービスは学習には最適だが、本番環境には不十分。年間$10の投資で大きな信頼性の懸念を解消した。接続が失敗したらまずDNSを確認 - それが最もシンプルな説明であることが多い。

問題2：Xrayコンテナの起動失敗 - JSON設定エラー
時系列
2025年7月28日 18:21-18:25 JST（4分間の連続失敗）

症状
Dockerコンテナが再起動を繰り返す。すべての起動試行がEOFエラーで失敗。

Copydocker logs xray
# 出力: Failed to start: main: failed to load config files: 
# [/etc/xray/config.json] > infra/conf/serial: failed to decode config > EOF
修正するまでコンテナが13回連続で失敗した。

根本原因
設定ファイルを編集していて、おそらく悪いタイミングで保存した - JSONが不完全だったか括弧が不足していた。EOFエラーは通常、ファイルが切り詰められたか不正な形式であることを意味する。

解決方法
JSON構文エラーを修正。コンテナは18:25:20に正常起動。

Copy# JSON構文を検証
cat /etc/xray/config.json | jq .
# エラーが返される場合、JSONが不正

# 修正後:
docker restart xray
docker logs xray | tail -20
# 正常起動を確認
するべきだったこと
Copy# 編集前に必ずバックアップ
cp /etc/xray/config.json /etc/xray/config.json.bak

# 再起動前に検証
jq . /etc/xray/config.json

# 検証が通った場合のみ再起動
証拠
正確なタイムスタンプと13回の連続失敗を示すDockerログ：

2025-07-28T18:21:18.095415383Z Failed to start...
2025-07-28T18:21:18.970090956Z Failed to start...
[... 11回の失敗 ...]
2025-07-28T18:25:05.216456974Z Failed to start...
2025-07-28T18:25:20.820605517Z [Warning] ... (正常起動)
学び
適用前に常に設定ファイルを検証。重要な設定にはバージョン管理を使用。バックアップを保持。本番設定を直接編集しない - まず開発環境で変更をテスト。

Dockerの再起動ポリシーは便利（自動リトライ）でもあり危険（無限再起動ループで問題を隠す可能性）でもあることも学んだ。

問題3：Cloudflare Tunnelトークン更新が必要
時系列
2025年7月29日 01:44頃 JST（DNS移行中）

症状
DuckDNSから新しいドメインに切り替えた後、Cloudflare TunnelのステータスがINACTIVEに。トンネルは存在するが接続を確立できない。

Copycloudflared tunnel info fujitsu-vpn
# 出力: Status: INACTIVE
根本原因
DNS変更により、トンネル認証トークンの再生成が必要だった。古いトークンは以前のDNS設定に紐付けられていた。

解決方法
古い認証情報をバックアップし、新しいトークンを生成：

Copy# 古いトークンをバックアップ
cp ~/.cloudflared/cert.pem ~/.cloudflared/cert.pem.bak

# 新しいトークンを生成
cloudflared tunnel token fujitsu-vpn

# 設定を更新
sudo vim /etc/cloudflared/config.yml
# (新しいトークンで更新)

# サービスを再起動
sudo systemctl restart cloudflared

# 確認
cloudflared tunnel info fujitsu-vpn
# 出力: Status: ACTIVE
解決までの時間：約30分

証拠
トークンの比較で異なる値が表示される：

Copy# 古いトークン（バックアップ済み）:
cat ~/.cloudflared/old_cred.json.bak
# Token: x2H4mWhQ5DT4milyylOyuQ90...

# 現在のトークン:
cat ~/.cloudflared/cert.pem
# Token: P1I42eS8tYSSL3aoLAcPZJ_TB...
バックアップファイルは2025-07-29 01:44:51に作成された。

学び
DNS変更は連鎖的な影響を持つ - 証明書だけでなく認証トークンにも。一つの大きな変更（DNS移行）が複数の小さな問題を引き起こした：

証明書の再発行
トークンの再生成
設定ファイルの更新
システム変更を行う際は常にコンポーネントの依存関係を考慮。重要な変更前にすべてをバックアップ。次に進む前に各ステップを検証。

その他の課題（概要）
自動化スクリプト開発（2025年9月）
ワンクリックVPN制御スクリプトの作成を試みた。複数の問題に遭遇：

私が犯した一般的なミス：

Ubuntuで/bin/zshを使用（デフォルトでは存在しない）
macOSコマンド（brew）をLinuxにコピー
heredocのEOF構文エラー
sudoersファイルのパーミッション（0440が必要）
systemdサービス名の混同（cloudflared vs cloudflared.service）
Bash履歴の制限（2000行）により、デバッグプロセスの詳細な記録を失ったが、バックアップファイルがこれらの問題を解決したことを証明している。

学び：スクリプトを段階的にテスト。デプロイ前に構文を検証。開発環境（macOS）と本番環境（Linux）の違いを理解。

セキュリティの進化（2025年8月）
初期設定
0.0.0.0（すべてのインターフェース）でサービスを公開して開始。その影響を理解していなかった。

気づき
Cloudflare Tunnelがダウンすると、VPSのパブリックポートに直接アクセスできることを発見。これは本番環境と開発環境のセキュリティを理解する転機だった。

取った行動
Dockerポートバインディングを127.0.0.1:8443（ローカルホストのみ）に変更
systemd再起動ポリシーを設定
基本的なUFWファイアウォールルールを設定
「多層防御」が複数のセキュリティレイヤーを意味することを理解
学び：一つの層が失敗しても（例：トンネルがダウン）、他の層が露出を防ぐべき。セキュリティは暗号化だけではなく - 適切なネットワーク設定が重要。

マルチシステム統合の課題（2025年10月-11月）
問題
最も複雑な時期には、以下を実行していた：

Xray（VLESS）
Hiddify-Xray
WireGuard
Cloudflare Tunnel
遭遇した問題：

ポート競合
サービス起動順序の依存関係
設定ファイルの場所が散在
実際にどのサービスが使用されているか不明確
解決策
Xray + Cloudflare Tunnelのみに簡素化。冗長なサービスを削除。より多くのコンポーネント = より多くの複雑性 = より多くの障害点であることを学んだ。

学び：シンプルに保つ。実際に必要な時だけ複雑性を追加。追加コンポーネントはすべて潜在的なメンテナンス負担。

今後の予防戦略
現在、私が異なる方法で行っていること
bash履歴サイズの増加：

Copyexport HISTSIZE=10000
export HISTFILESIZE=20000
トラブルシューティングログの維持：

Copyecho "$(date): [問題の説明]" >> ~/troubleshooting.log
重要なステップのスクリーンショット撮影

設定ファイルのバージョン管理

本番前にステージング環境で変更をテスト

モニタリング設定
証明書有効期限アラート（30日前）
Cloudflare Tunnelヘルスチェック
コンテナ再起動回数の監視
週次ログレビュー
メタ学習：失敗の価値
最大の教訓は特定の技術的解決策ではない。詳細なログを失ったことが、ドキュメンテーションの重要性を教えてくれた。

6ヶ月間で30以上の問題に遭遇したが、完全な証拠があるのは3つだけ。他はbash履歴制限とログローテーションで失われた。これ自体が本番運用についての貴重な教訓。

今後のプロジェクトでは、初日からドキュメンテーションを優先する。他人のためだけでなく、なぜそのような方法で行ったかを覚えていない「未来の自分」のためにも。

一般的な診断コマンド
クイックヘルスチェック：

Copy# Tunnelステータス
systemctl status cloudflared

# コンテナステータス
docker ps | grep xray
docker logs xray --tail 50

# 接続テスト
curl -I https://vpn.fujilegend.xyz

# 証明書確認
openssl x509 -in /usr/local/etc/xray/certs/fullchain.cer -noout -dates
何かが壊れたら、これらのコマンドから始める。90%の問題を捕捉できる。

関連ドキュメント：

Architecture - システムの仕組み
Setup Guide - デプロイ手順
What I Learned - 個人的な振り返り
Security - セキュリティに関する考慮事項
