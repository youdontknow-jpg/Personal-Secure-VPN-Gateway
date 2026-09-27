# Personal Secure VPN Gateway

Production VPN gateway with anti-censorship capabilities using Xray and Cloudflare Tunnel.

**Status:** In daily use since August 2025. Not always up - real outages are documented in [Troubleshooting](docs/troubleshooting.md).  
**Purpose:** Secure remote access to home network with traffic obfuscation to bypass restrictive firewalls

📖 **[日本語版はこちら](#日本語版)**

---

## Features

- **Zero port forwarding** - All traffic through Cloudflare Tunnel, no exposed ports on home router
- **Traffic obfuscation** - WebSocket over TLS disguises VPN traffic as regular HTTPS browsing
- **Anti-censorship** - Bypasses deep packet inspection (DPI) and restrictive network filters
- **Containerized** - Docker deployment for easy management and reproducibility
- **Zero Trust SSH** - Admin SSH goes through Cloudflare Access (Google login + 2FA) over the same tunnel, no SSH port exposed to the internet
- **Battle-tested** - Running since August 2025, including recovery from two full outages

## Architecture

```
VPN:   Client → Cloudflare CDN → Cloudflare Tunnel → cloudflared → Docker (Xray) → Home Network
Admin: ssh → Cloudflare Access (Google + 2FA) → same tunnel → SSH on the server
```

All traffic encrypted with TLS 1.3, WebSocket protocol masks VPN signatures from network inspection.

**[📘 Detailed Architecture Documentation](docs/architecture.md)**

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| CDN & Tunnel | Cloudflare Tunnel | Zero port forwarding, DDoS protection, IP hiding |
| VPN Protocol | Xray (VLESS) | WebSocket-based tunnel with traffic obfuscation |
| Container | Docker | Service isolation and management |
| Firewall | UFW + Fail2Ban | Host-level security |
| Admin access | Cloudflare Access | Google login + 2FA in front of SSH |

## Quick Start

```bash
# 1. Setup Cloudflare Tunnel
cloudflared tunnel create my-vpn
cloudflared tunnel route dns my-vpn vpn.example.com

# 2. Deploy Xray container
docker run -d \
  --name xray \
  --restart unless-stopped \
  -p 127.0.0.1:8443:8443 \
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  teddysun/xray

# 3. Start tunnel
cloudflared tunnel run my-vpn
```

**[📗 Complete Setup Guide](docs/setup-guide.md)**

## Configuration Examples

Configuration templates are available in the [`configs/`](configs/) directory:

- [`xray-config-example.json`](configs/xray-config-example.json) - Xray VLESS configuration
- [`cloudflared-config.yml`](configs/cloudflared-config.yml) - Cloudflare Tunnel settings
- [`docker-compose.yml`](configs/docker-compose.yml) - Docker Compose deployment

All sensitive values (UUIDs, domain names, tunnel IDs) are replaced with placeholders.

## Performance

**Measured in daily use:**

- **Latency:** 30-50ms total (10-20ms Cloudflare overhead)
- **Throughput:** 150-250 Mbps download, 80-120 Mbps upload
- **Reliability:** Stable for the first ~3 months. Two full unplanned outages in 2026: an expired certificate plus a broken client ([Issue 4](docs/troubleshooting.md#issue-4-full-vpn-outage---expired-certificate--broken-client)), and a server frozen by leftover services in a restart loop ([Issue 5](docs/troubleshooting.md#issue-5-server-frozen-by-a-restart-loop-of-leftover-services))
- **Anti-censorship:** Successfully maintained connectivity in restrictive network environments

## Documentation

- **[Architecture & Design](docs/architecture.md)** - Why this architecture, how traffic obfuscation works
- **[Setup Guide](docs/setup-guide.md)** - Complete deployment instructions
- **[Troubleshooting](docs/troubleshooting.md)** - Real issues encountered and solutions
- **[Alternative Approaches](docs/alternatives.md)** - Why not WireGuard, Shadowsocks, Tor, or commercial VPNs
- **[Security Considerations](docs/security.md)** - Current limitations and planned improvements

## What I Learned

**Technical skills:**
- Anti-censorship techniques (WebSocket obfuscation, CDN-based masking)
- Cloudflare infrastructure (Tunnel, WAF, global CDN)
- Docker containerization and systemd integration
- Network security fundamentals (defense in depth, firewall design)

**Operational experience:**
- Maintained a self-hosted service since August 2025
- Troubleshot and resolved 8+ major issues, including two full outages
- Recovered a frozen server via GRUB recovery mode and traced it to a systemd restart loop
- Rotated credentials after suspected exposure
- Managed TLS certificates and service updates
- Optimized for various network conditions

**[📙 Detailed Learning Reflections](docs/what-i-learned.md)**

## Current Status (September 2026)

- ✅ Xray (Docker, `restart: unless-stopped`) and cloudflared (systemd, enabled) running - auto-start on boot verified
- ✅ TLS certificate valid until December 26, 2026
- ✅ Leftover Hiddify services removed, journal capped at 500M ([Issue 5](docs/troubleshooting.md#issue-5-server-frozen-by-a-restart-loop-of-leftover-services))
- ✅ VPN UUID rotated after suspected exposure ([Issue 4](docs/troubleshooting.md#issue-4-full-vpn-outage---expired-certificate--broken-client))
- ⚠️ No certificate expiry alert yet - both 2026 outages involved a certificate that expired without warning

## Current Limitations

This is a learning project and I'm aware of its limitations:

- Single shared UUID (no per-device access control)
- Basic access control (no service-level restrictions)
- Single point of failure (one Docker container)
- Limited traffic analysis protection

## Roadmap

- [x] Auto-start Xray / cloudflared on boot (verified after Issue 5)
- [ ] Certificate expiry alerting (the lesson from Issues 4 and 5)
- [x] Log acme.sh cron output instead of discarding it
- [ ] Rotate Cloudflare Tunnel token
- [ ] Backup route independent of Cloudflare (Xray Reality)
- [ ] Per-device access control
- [ ] Move server credentials into a password manager
- [ ] Basic monitoring (Prometheus + Grafana) - still learning how to use it

## References

- [Xray-core Documentation](https://xtls.github.io/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [GFW Report (Censorship research)](https://gfw.report/)
- [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## License

MIT License - Configuration files and documentation are free to use and modify.

**Disclaimer:** This project is for educational and personal use. Respect local laws regarding VPN usage. Configuration examples use placeholder values - generate your own credentials.

---

**Last updated:** September 27, 2026

---
---

# 日本語版

Xray と Cloudflare Tunnel を使った反検閲機能付き本番稼働 VPN ゲートウェイ。

**運用状況:** 2025年8月から日常的に使用中。常に稼働していたわけではなく、実際の障害は[トラブルシューティング](docs/troubleshooting.md)に記録。  
**目的:** 制限的なファイアウォールを回避するトラフィック難読化機能付き自宅ネットワークへの安全なリモートアクセス

---

## 特徴

- **ポート開放ゼロ** - すべてのトラフィックが Cloudflare Tunnel 経由、自宅ルーターに露出ポートなし
- **トラフィック難読化** - WebSocket over TLS で VPN トラフィックを通常の HTTPS 閲覧に偽装
- **反検閲** - Deep Packet Inspection (DPI) と制限的なネットワークフィルタを回避
- **コンテナ化** - 容易な管理と再現性のための Docker デプロイ
- **ゼロトラスト SSH** - 管理用 SSH は同じトンネル上で Cloudflare Access（Google ログイン + 2FA）を経由、SSH ポートはインターネットに非公開
- **実戦で検証済み** - 2025年8月から稼働、2回の全面停止からの復旧経験あり

## システム構成

```
VPN:  クライアント → Cloudflare CDN → Cloudflare Tunnel → cloudflared → Docker (Xray) → 自宅ネットワーク
管理: ssh → Cloudflare Access（Google + 2FA）→ 同じトンネル → サーバーの SSH
```

すべてのトラフィックは TLS 1.3 で暗号化、WebSocket プロトコルがネットワーク検査から VPN シグネチャを隠蔽。

**[📘 詳細なアーキテクチャドキュメント](docs/architecture.md)**

## 技術スタック

| コンポーネント | 技術 | 用途 |
|--------------|------|------|
| CDN & トンネル | Cloudflare Tunnel | ポート開放不要、DDoS 保護、IP 隠蔽 |
| VPN プロトコル | Xray (VLESS) | WebSocket ベースのトンネル、トラフィック難読化機能付き |
| コンテナ | Docker | サービス分離と管理 |
| ファイアウォール | UFW + Fail2Ban | ホストレベルのセキュリティ |
| 管理アクセス | Cloudflare Access | SSH の前段で Google ログイン + 2FA |

## クイックスタート

```bash
# 1. Cloudflare Tunnel セットアップ
cloudflared tunnel create my-vpn
cloudflared tunnel route dns my-vpn vpn.example.com

# 2. Xray コンテナデプロイ
docker run -d \
  --name xray \
  --restart unless-stopped \
  -p 127.0.0.1:8443:8443 \
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  teddysun/xray

# 3. トンネル起動
cloudflared tunnel run my-vpn
```

**[📗 完全セットアップガイド](docs/setup-guide.md)**

## 設定例

設定テンプレートは [`configs/`](configs/) ディレクトリにあります:

- [`xray-config-example.json`](configs/xray-config-example.json) - Xray VLESS 設定
- [`cloudflared-config.yml`](configs/cloudflared-config.yml) - Cloudflare Tunnel 設定
- [`docker-compose.yml`](configs/docker-compose.yml) - Docker Compose デプロイ

すべての機密値（UUID、ドメイン名、トンネル ID）はプレースホルダーに置き換えられています。

## パフォーマンス

**日常使用で測定:**

- **レイテンシ:** 合計 30-50ms（Cloudflare オーバーヘッド 10-20ms）
- **スループット:** ダウンロード 150-250 Mbps、アップロード 80-120 Mbps
- **信頼性:** 最初の約3ヶ月は安定稼働。2026年に計画外の全面停止が2回発生：証明書期限切れとクライアント破損（[問題4](docs/troubleshooting.md#問題4vpn全面停止---証明書期限切れとクライアント破損)）、残存サービスの再起動ループによるサーバー停止（[問題5](docs/troubleshooting.md#問題5残存サービスの再起動ループでサーバーが停止)）
- **反検閲:** 制限的なネットワーク環境で接続を正常に維持

## ドキュメント

- **[アーキテクチャと設計](docs/architecture.md)** - なぜこの構成か、トラフィック難読化の仕組み
- **[セットアップガイド](docs/setup-guide.md)** - 完全なデプロイ手順
- **[トラブルシューティング](docs/troubleshooting.md)** - 実際に遭遇した問題と解決策
- **[代替アプローチ](docs/alternatives.md)** - なぜ WireGuard、Shadowsocks、Tor、商用 VPN ではないのか
- **[セキュリティ上の考慮事項](docs/security.md)** - 現在の限界と改善計画

## 学んだこと

**技術スキル:**
- 反検閲技術（WebSocket 難読化、CDN ベースのマスキング）
- Cloudflare インフラ（Tunnel、WAF、グローバル CDN）
- Docker コンテナ化と systemd 統合
- ネットワークセキュリティ基礎（多層防御、ファイアウォール設計）

**運用経験:**
- 2025年8月からセルフホストサービスを維持
- 8つ以上の主要問題を解決（2回の全面停止を含む）
- GRUB リカバリーモードで停止したサーバーを復旧し、systemd の再起動ループを原因として特定
- 露出が疑われた認証情報のローテーション
- TLS 証明書とサービス更新の管理
- 様々なネットワーク条件に最適化

**[📙 詳細な学習の振り返り](docs/what-i-learned.md)**

## 現在の状況（2026年9月）

- ✅ Xray（Docker、`restart: unless-stopped`）と cloudflared（systemd、有効）が稼働中 - 起動時の自動起動を確認済み
- ✅ TLS 証明書は2026年12月26日まで有効
- ✅ Hiddify の残存サービスを削除、ジャーナルを500Mに制限（[問題5](docs/troubleshooting.md#問題5残存サービスの再起動ループでサーバーが停止)）
- ✅ 露出が疑われた VPN UUID をローテーション済み（[問題4](docs/troubleshooting.md#問題4vpn全面停止---証明書期限切れとクライアント破損)）
- ⚠️ 証明書期限アラートは未導入 - 2026年の2回の障害はどちらも警告なしで期限切れになった証明書が関係

## 現在の限界

これは学習プロジェクトであり、限界を認識しています:

- 単一共有 UUID（デバイスごとのアクセス制御なし）
- 基本的なアクセス制御（サービスレベルの制限なし）
- 単一障害点（1つの Docker コンテナ）
- 限定的なトラフィック分析保護

## ロードマップ

- [x] Xray / cloudflared の自動起動（問題5の後に確認済み）
- [ ] 証明書期限切れアラート（問題4・5の教訓）
- [x] acme.sh の cron 出力を捨てずにログに残す
- [ ] Cloudflare Tunnel トークンのローテーション
- [ ] Cloudflare に依存しないバックアップ経路（Xray Reality）
- [ ] デバイスごとのアクセス制御
- [ ] サーバー認証情報をパスワードマネージャーに移行
- [ ] 基本的な監視（Prometheus + Grafana）- 使い方を学習中

## 参考資料

- [Xray-core Documentation](https://xtls.github.io/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [GFW Report（検閲研究）](https://gfw.report/)
- [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## ライセンス

MIT License - 設定ファイルと文書は自由に使用・修正可能。

**免責事項:** このプロジェクトは教育と個人使用目的。VPN 使用に関する地域の法律を尊重してください。設定例はプレースホルダー値を使用 - 独自の認証情報を生成してください。

---

**最終更新:** 2026年9月27日
