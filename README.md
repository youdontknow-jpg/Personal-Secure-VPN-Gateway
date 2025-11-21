# Personal Secure VPN Gateway

Production VPN gateway with anti-censorship capabilities using Xray and Cloudflare Tunnel.

**Status:** In production since August 2025 (3+ months, 99.8% uptime)  
**Purpose:** Secure remote access to home network with traffic obfuscation to bypass restrictive firewalls

📖 **[日本語版はこちら](#日本語版)**

---

## Features

- **Zero port forwarding** - All traffic through Cloudflare Tunnel, no exposed ports on home router
- **Traffic obfuscation** - WebSocket over TLS disguises VPN traffic as regular HTTPS browsing
- **Anti-censorship** - Bypasses deep packet inspection (DPI) and restrictive network filters
- **Containerized** - Docker deployment for easy management and reproducibility
- **Production-ready** - 3+ months stable operation, 99.8% uptime

## Architecture

```
Client → Cloudflare CDN → Cloudflare Tunnel → cloudflared → Docker (Xray) → Home Network
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

**Measured over 3+ months of production use:**

- **Latency:** 30-50ms total (10-20ms Cloudflare overhead)
- **Throughput:** 150-250 Mbps download, 80-120 Mbps upload
- **Reliability:** 99.8% uptime, only downtime from planned maintenance
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
- Maintained production service for 3+ months
- Troubleshot and resolved 6+ major issues
- Managed TLS certificates and service updates
- Optimized for various network conditions

**[📙 Detailed Learning Reflections](docs/what-i-learned.md)**

## Current Limitations

This is a learning project and I'm aware of its limitations:

- Single shared UUID (no per-device access control)
- Basic access control (no service-level restrictions)
- Single point of failure (one Docker container)
- Limited traffic analysis protection

**Planned improvements:** Phased roadmap (Q1-Q4 2026) for monitoring, access control, redundancy, and advanced anti-censorship features.

**Why not now:** Currently focused on Network+ certification (exam Dec 28, 2025), and current setup has proven stable over 3+ months of testing.

## References

- [Xray-core Documentation](https://xtls.github.io/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [GFW Report (Censorship research)](https://gfw.report/)
- [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## License

MIT License - Configuration files and documentation are free to use and modify.

**Disclaimer:** This project is for educational and personal use. Respect local laws regarding VPN usage. Configuration examples use placeholder values - generate your own credentials.

---

**Last updated:** November 21, 2025

---
---

# 日本語版

Xray と Cloudflare Tunnel を使った反検閲機能付き本番稼働 VPN ゲートウェイ。

**運用状況:** 2025年7月から本番稼働中（6ヶ月以上、稼働率 99.8%）  
**目的:** 制限的なファイアウォールを回避するトラフィック難読化機能付き自宅ネットワークへの安全なリモートアクセス

---

## 特徴

- **ポート開放ゼロ** - すべてのトラフィックが Cloudflare Tunnel 経由、自宅ルーターに露出ポートなし
- **トラフィック難読化** - WebSocket over TLS で VPN トラフィックを通常の HTTPS 閲覧に偽装
- **反検閲** - Deep Packet Inspection (DPI) と制限的なネットワークフィルタを回避
- **コンテナ化** - 容易な管理と再現性のための Docker デプロイ
- **本番環境対応** - 6ヶ月以上安定稼働、稼働率 99.8%

## システム構成

```
クライアント → Cloudflare CDN → Cloudflare Tunnel → cloudflared → Docker (Xray) → 自宅ネットワーク
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

**6ヶ月以上の本番運用で測定:**

- **レイテンシ:** 合計 30-50ms（Cloudflare オーバーヘッド 10-20ms）
- **スループット:** ダウンロード 150-250 Mbps、アップロード 80-120 Mbps
- **信頼性:** 稼働率 99.8%、ダウンタイムは計画メンテナンスのみ
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
- 6ヶ月以上の本番サービス維持
- 6つ以上の主要問題のトラブルシューティングと解決
- TLS 証明書とサービス更新の管理
- 様々なネットワーク条件に最適化

**[📙 詳細な学習の振り返り](docs/what-i-learned.md)**

## 現在の限界

これは学習プロジェクトであり、限界を認識しています:

- 単一共有 UUID（デバイスごとのアクセス制御なし）
- 基本的なアクセス制御（サービスレベルの制限なし）
- 単一障害点（1つの Docker コンテナ）
- 限定的なトラフィック分析保護

**改善計画:** 監視、アクセス制御、冗長性、高度な反検閲機能のための段階的ロードマップ（2026年 Q1-Q4）。

**なぜ今ではないのか:** 現在は Network+ 資格取得に集中中（試験 2025年12月28日）、そして現在の構成は 6ヶ月以上のテストで安定性を証明済み。

## 参考資料

- [Xray-core Documentation](https://xtls.github.io/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [GFW Report（検閲研究）](https://gfw.report/)
- [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## ライセンス

MIT License - 設定ファイルと文書は自由に使用・修正可能。

**免責事項:** このプロジェクトは教育と個人使用目的。VPN 使用に関する地域の法律を尊重してください。設定例はプレースホルダー値を使用 - 独自の認証情報を生成してください。

---

**最終更新:** 2025年11月21日
