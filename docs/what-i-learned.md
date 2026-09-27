# What I Learned

Reflection on 3+ months of building and running a production VPN. This wasn't just a weekend project – it's been a continuous learning experience with plenty of mistakes along the way.

---

## Technical Skills Acquired

### 1. VPN Protocols & Anti-Censorship Techniques

**Before this project:**
- "VPNs encrypt traffic, that's all I need to know"
- Thought WireGuard was the best option for everything
- Didn't really understand how some VPNs get blocked

**After 3 months:**
- Realized encryption alone isn't enough – protocols have fingerprints
- Learned how DPI (Deep Packet Inspection) actually works and its usefulness
- Understand the cat-and-mouse game between censorship and circumvention

**The "aha!" moment:**
Was testing WireGuard in a restrictive network. Got blocked within 24 hours. Spent a whole weekend reading about protocol fingerprinting and discovered WebSocket obfuscation. That's when I understood – it's not about *how* you encrypt, it's about *how you hide* what you're doing. Can't win at a country-level security (ofc).

**Practical knowledge:**
- WebSocket over TLS makes VPN traffic look like a web app
- CDN integration (Cloudflare) provides natural camouflage
- Protocol selection matters more than I thought
- Understanding the adversary's detection methods is key

---

### 2. Cloudflare Infrastructure

**Initial misconception:**
"Cloudflare Tunnel is just fancy port forwarding"

**Reality check:**
It's way more complex than that. Spent hours reading docs to understand:
- How Anycast routing works (still don't fully remember honestly)
- Why tunnel connections are outbound-only (security genius)
- How edge termination differs from traditional proxy

**Frustrating learning curve:**
The WAF (Web Application Firewall) blocked my own VPN connections for 2 hours. Thought my config was broken. Turns out Cloudflare's AI thought I was attacking myself. Had to lower security level – felt wrong but necessary.

**What I know now:**
- Cloudflare's free tier is surprisingly powerful
- PoP (Point of Presence) selection affects latency
- WAF needs tuning for WebSocket applications
- Reading firewall logs is an acquired skill

**Still learning:**
- BGP routing and why sometimes it picks bad paths
- Advanced Cloudflare features I haven't explored yet

---

### 3. Docker Containerization

**Started with:**
"Docker is like a lightweight VM, right?"

**Learned the hard way:**
No. It's very different. Made every beginner mistake:
- First container ran as root (security disaster)
- Didn't use restart policies (manual start after every reboot)
- Exposed ports to 0.0.0.0 instead of 127.0.0.1 (opened to internet accidentally)
- Forgot volume mounts were ephemeral (lost configs)

**The 3 AM debugging session:**
Container kept crashing with "connection refused". Spent 4 hours checking Xray config, firewall rules, everything. Problem? Port mapping was wrong. `-p 8443:8443` without `127.0.0.1:` prefix exposed it publicly. Felt stupid but learned a valuable lesson.

**What I understand now:**
- Container networking is complex (bridge vs host vs overlay)
- Read-only mounts are essential for security
- Resource limits prevent one container from killing the host
- Logs need rotation or they eat all disk space

**Practical skills:**
- Can debug container networking issues
- Understand when to use containers vs bare metal
- Know how to properly isolate services
- Can read Docker logs without crying

---

### 4. Network Security Fundamentals

**Defense in depth – learned by necessity:**

Started with just a firewall. Then:
- Week 2: Added Fail2Ban after seeing 200+ SSH brute-force attempts
- Month 1: Realized certificate expiration is a thing (service went down)
- Month 3: Learned about log monitoring (disk was full)
- Month 5: Finally understood why defense in depth matters

**Mistakes that taught me:**
1. **Locked myself out twice**
   - Enabled UFW before allowing SSH port
   - Spent 30 minutes panicking before realizing I could login from console
   - Now I *always* test firewall rules before enabling

2. **Fail2Ban blocked me**
   - Kept fat-fingering SSH key path
   - Got banned from my own server
   - Had to wait 10 minutes for unban
   - Added my IP to whitelist (should have done this first)

3. **Certificate expired at 2 AM**
   - Woke up to "VPN not working" message
   - Auto-renewal had failed silently for weeks
   - Took 2 hours to fix in a panic
   - Set up monitoring and alerts after this

**Current understanding:**
- Security is layers, not a single solution
- Every layer can fail – need monitoring
- Automation is great until it fails silently
- Testing disaster recovery is important (and boring)

---

## Operational Experience

### Uptime in the First 3 Months

**The numbers (Aug–Nov 2025):**
- 3 months × 30 days × 24 hours = 2,160 hours
- Downtime: ~8 hours (all planned maintenance)
- Unplanned outages: 0 (got lucky)

That luck didn't last: in 2026 an expired TLS certificate caused a full outage. See [Troubleshooting – Issue 4](troubleshooting.md#issue-4-full-vpn-outage---expired-certificate--broken-client).

**What "maintenance" actually means:**
- Not just "run apt update and pray"
- Need to plan timing (late night when no one uses it)
- Test changes in development first
- Have rollback plan ready
- Monitor for 30 minutes after changes

**Close calls:**
- Ubuntu update tried to restart Docker automatically (caught it in time)
- Cloudflare had brief outage (tunnel reconnected automatically, phew)
- Router firmware update changed firewall rules (good thing VPN doesn't expose ports)

---

### Real-World Troubleshooting

**Issue #1: DNS not resolving (2 hours)**

*What I thought:* "Cloudflare tunnel is broken"
*What it was:* Domain nameservers still pointing to Namecheap
*How I found out:* Ran `dig` after 90 minutes of checking everything else
*Lesson:* Check DNS first, not last

**Issue #2: WebSocket connections failing (3 hours)**

*Symptoms:* TLS handshake OK, then immediate disconnect
*Debugging process:*
- Checked Xray logs (nothing useful)
- Checked Docker logs (no errors)
- Checked Cloudflare logs (nothing)
- Finally checked UFW logs – port was blocked
*Facepalm moment:* I had created a "deny" rule by mistake
*Lesson:* Check firewall logs earlier in debugging process

**Issue #3: Slow performance from Shanghai (4 hours + research)**

*Problem:* Friend in Shanghai had 500ms+ latency
*Investigation:*
- Ran traceroute – routing through US (why??)
- Researched Cloudflare PoP selection
- Learned about Anycast and BGP
- Discovered this is mostly outside my control
*Solution:* Not really solved, just understood it's Cloudflare's routing decision
*Lesson:* Some problems don't have solutions, only explanations

---

### Unexpected Lessons

**Time management:**
- "Quick fix" usually takes 3x longer than planned
- Weekend projects turn into month-long commitments
- Documentation takes longer than coding

**When to stop tweaking:**
- Spent 2 weeks trying to optimize latency by 5ms
- Realized diminishing returns are real
- "Good enough" is actually good enough for personal use
- Production doesn't mean perfect, it means stable

**Community knowledge:**
- Someone on Reddit had exactly my problem
- GitHub issues are treasure troves
- Cloudflare community forums saved me twice
- Don't reinvent the wheel – search first

---

## Mistakes That Still Make Me Cringe

### 1. Committed credentials to GitHub (briefly)

Created a test script with my actual UUID. Pushed to GitHub. Realized 10 minutes later. Deleted commit. Force pushed. Rotated credentials anyway. Learned about `.gitignore` properly.

**Lesson:** Never trust yourself with secrets. Use environment variables or separate config files.

---

### 2. Didn't backup before major changes

Changed Xray config. Syntax error. Container wouldn't start. No backup. Spent an hour reconstructing config from memory and old screenshots. Now I backup before every change.

**Lesson:** `cp config.json config.json.backup` takes 2 seconds. Reconstruction takes hours.

---

### 3. Ignored certificate warnings for weeks

Xray logs showed "certificate will expire in 30 days" warnings. Thought "I'll deal with it later." Certificate expired. Service down for 2 hours. Users angry. Me embarrassed.

**Lesson:** Warnings are not suggestions. They're countdown timers to problems.

---

### 4. Overcomplicated initial design

First version had Nginx → Xray → Cloudflared. Nginx was completely unnecessary. Added complexity for zero benefit. Removed it after 2 months when troubleshooting was painful.

**Lesson:** Start simple. Add complexity only when needed. Every component is a potential failure point.

---

### 5. Didn't test disaster recovery

Had backups. Never tested restoring them. When I finally did (for practice), discovered backup script had been silently failing for 2 months. Empty backup files.

**Lesson:** Backups you haven't tested are wishes, not backups.

---

## How This Applies to Career Goals

### Network Security Position (2027 Target)

**What this project proves:**
- I can build and maintain production systems (not just lab environments)
- I understand practical security (not just theory)
- I can troubleshoot real problems (not just textbook exercises)
- I document my work (important for team environments)

**What interviews might ask:**
- "Tell me about a technical challenge you overcame"
  → Have 6 real troubleshooting stories
- "How do you approach learning new technologies?"
  → This project shows self-directed learning
- "Describe your problem-solving process"
  → Can walk through actual debugging sessions

**Areas I'm still weak in:**
- Enterprise-scale systems (this is home lab)
- Team collaboration (solo project)
- Formal change management (just me doing things)
- Advanced monitoring (basic health checks only)

### Certifications Alignment

**Network+ (Dec 2025):**
This project is practical application of:
- OSI model layers
- TCP/IP protocols
- Network troubleshooting methodology
- Security best practices

**Security+ (planned Q1 2026):**
Directly relevant experience with:
- Defense in depth
- Cryptography (TLS)
- Access control
- Incident response (certificate expiration)

---

## What's Next

### Short-term (keeping it real)

**Before Network+ exam:**
- No major changes to VPN (stability > features)
- Focus on exam preparation
- Maybe add basic monitoring (if time permits)

**Honest assessment:**
I want to add Prometheus, Grafana, better access control, redundancy... but realistically, exam prep comes first. This project taught me that trying to do everything at once leads to nothing getting done properly.

---

### Long-term (after certifications)

**Q2 2026 (after Network+):**
- Phase 1: Monitoring (Prometheus + Grafana)
- Actually learn how to read metrics properly
- Set up alerts that aren't annoying

**Q3 2026 (after Security+):**
- Phase 2: Better access control
- Per-device authentication
- Maybe Cloudflare Access integration

**Q4 2026 (after CCNA):**
- Phase 3: Redundancy and failover
- Second instance on different hardware
- Load balancing (if I can figure it out)

**Realistic timeline:**
These phases might take longer. Or I might discover they're not necessary. Or I might get a job and be too busy. Planning is easy, executing is hard.

---

## Advice to Future Me

**Things I keep forgetting:**

1. **Document as you go, not after**
   - You will not remember why you did something
   - Screenshots and command history are your friends
   - Comments in config files save time later

2. **Test in development first**
   - Production is not a testing environment
   - Your past self was not as smart as you remember
   - Breaking things teaches you, but annoying users doesn't

3. **Automate carefully**
   - Automation that fails silently is worse than manual work
   - Test automated tasks regularly
   - Monitor automation success/failure

4. **Know when to stop**
   - Perfect is the enemy of done
   - Optimizing 5ms latency isn't worth a week
   - Stability beats features

5. **Ask for help sooner**
   - Reddit/forums have seen your problem before
   - 4 hours of debugging vs 10 minutes of searching
   - Pride is expensive

**Most important lesson:**
This wasn't just about building a VPN. It was about learning how to learn, how to persist through frustration, and how to maintain something over time. Those skills matter more than the technical details.

---

**Last updated:** November 21, 2025  
**Total time invested:** ~80 hours over 3 months  
**Would I do it again?** Absolutely. Mistakes and all.

---
---

# 日本語版

3ヶ月以上の本番VPN構築と運用から学んだこと。これは週末プロジェクトではなく、多くの失敗を伴う継続的な学習体験でした。

---

## 習得した技術スキル

### 1. VPN プロトコルと反検閲技術

**このプロジェクト前:**
- 「VPNはトラフィックを暗号化する、それだけ知っていればいい」
- WireGuardがすべてに最適だと思っていた
- なぜ一部のVPNがブロックされるか理解していなかった

**3ヶ月後:**
- 暗号化だけでは不十分 – プロトコルにはフィンガープリントがある
- DPI（Deep Packet Inspection）の実際の動作を学んだ
- 検閲と回避のいたちごっこを理解した

**「なるほど！」の瞬間:**
制限的なネットワークでWireGuardをテスト。24時間以内にブロック。週末丸ごと費やしてプロトコルフィンガープリンティングについて読み、WebSocket難読化を発見。その時理解した – *どう*暗号化するかではなく、*どう隠す*かが重要だと。

**実践的知識:**
- WebSocket over TLSでVPNトラフィックをウェブアプリのように見せる
- CDN統合（Cloudflare）が自然なカモフラージュを提供
- プロトコル選択は思っていたより重要
- 敵対者の検出方法を理解することが鍵

---

### 2. Cloudflareインフラ

**最初の誤解:**
「Cloudflare Tunnelは派手なポートフォワーディング」

**現実:**
それよりはるかに複雑。ドキュメントを何時間も読んで理解したこと:
- Anycastルーティングの仕組み（正直、まだ完全には理解していない）
- なぜトンネル接続はアウトバウンドのみか（セキュリティの天才的発想）
- エッジ終端が従来のプロキシとどう違うか

**イライラした学習曲線:**
WAF（Webアプリケーションファイアウォール）が自分のVPN接続を2時間ブロック。設定が壊れていると思った。実はCloudflareのAIが自分を攻撃していると判断。セキュリティレベルを下げる必要があった – 間違っている気がしたが必要だった。

**今わかっていること:**
- Cloudflareの無料プランは驚くほど強力
- PoP（Point of Presence）選択がレイテンシに影響
- WAFはWebSocketアプリケーション用にチューニングが必要
- ファイアウォールログを読むのは習得すべきスキル

**まだ学習中:**
- BGPルーティングと時々悪いパスを選ぶ理由
- まだ探索していない高度なCloudflare機能

---

### 3. Dockerコンテナ化

**最初:**
「Dockerは軽量VMみたいなもの？」

**痛い目に遭って学んだ:**
いいえ。全く違う。初心者のミスをすべて犯した:
- 最初のコンテナをrootで実行（セキュリティ災害）
- 再起動ポリシーを使わなかった（毎回リブート後に手動起動）
- 127.0.0.1の代わりに0.0.0.0にポート公開（誤ってインターネットに公開）
- ボリュームマウントが一時的だと忘れた（設定を失った）

**午前3時のデバッグセッション:**
コンテナが"connection refused"でクラッシュし続けた。4時間かけてXray設定、ファイアウォールルール、すべてをチェック。問題？ポートマッピングが間違っていた。`127.0.0.1:`プレフィックスなしの`-p 8443:8443`で公開。馬鹿だと感じたが貴重な教訓。

**今理解していること:**
- コンテナネットワーキングは複雑（bridge vs host vs overlay）
- 読み取り専用マウントはセキュリティに不可欠
- リソース制限が1つのコンテナがホストを殺すのを防ぐ
- ログはローテーションしないとディスクを食い尽くす

**実践スキル:**
- コンテナネットワーキング問題をデバッグできる
- コンテナとベアメタルをいつ使うか理解
- サービスを適切に分離する方法を知っている
- 泣かずにDockerログを読める

---

### 4. ネットワークセキュリティ基礎

**多層防御 – 必要に迫られて学習:**

最初はファイアウォールだけ。その後:
- 2週目: 200+のSSHブルートフォース試行を見てFail2Banを追加
- 1ヶ月目: 証明書期限切れがあることを認識（サービスダウン）
- 3ヶ月目: ログ監視について学習（ディスクがいっぱい）
- 3ヶ月目: やっと多層防御の重要性を理解

**教えてくれたミス:**
1. **2回自分をロックアウト**
   - SSHポートを許可する前にUFWを有効化
   - 30分パニックしてからコンソールからログインできることに気づいた
   - 今は*必ず*有効化前にファイアウォールルールをテスト

2. **Fail2Banに自分をブロックされた**
   - SSHキーパスを何度も間違えた
   - 自分のサーバーから禁止された
   - 禁止解除まで10分待つ必要があった
   - 自分のIPをホワイトリストに追加（最初にすべきだった）

3. **午前2時に証明書期限切れ**
   - 「VPNが動いていない」メッセージで起きた
   - 自動更新が数週間静かに失敗していた
   - パニック状態で修正に2時間かかった
   - この後、監視とアラートを設定

**現在の理解:**
- セキュリティは層、単一ソリューションではない
- すべての層が失敗する可能性 – 監視が必要
- 自動化は静かに失敗するまで素晴らしい
- 災害復旧のテストは重要（そして退屈）

---

## 運用経験

### 最初の3ヶ月の稼働状況

**数字（2025年8月〜11月）:**
- 3ヶ月 × 30日 × 24時間 = 2,160時間
- ダウンタイム: 約8時間（すべて計画メンテナンス）
- 計画外停止: 0（運が良かった）

この幸運は続かなかった：2026年、TLS証明書の期限切れにより全面停止が発生。[トラブルシューティング – 問題4](troubleshooting.md#問題4vpn全面停止---証明書期限切れとクライアント破損)を参照。

**「メンテナンス」の実際の意味:**
- 単に「apt updateして祈る」ではない
- タイミングを計画する必要（誰も使わない深夜）
- まず開発環境で変更をテスト
- ロールバック計画を準備
- 変更後30分間監視

**危機一髪:**
- Ubuntu更新がDockerを自動再起動しようとした（間に合った）
- Cloudflareが短時間停止（トンネルが自動再接続、ほっ）
- ルーターファームウェア更新がファイアウォールルールを変更（VPNがポートを公開しないので良かった）

---

## まだ開発中のスキル

### 改善が必要な分野

**監視と可観測性:**
- 現在: 手動ログチェック
- 目標: PrometheusとGrafanaで自動監視
- 学ぶ必要: メトリクス収集、アラート、ダッシュボード

**ネットワークアーキテクチャ:**
- 現在: 基本的な理解
- 目標: 高度なネットワーキング（VLAN、高度なルーティング）
- 学ぶ必要: ネットワークセグメンテーション、Zero-Trust原則

---

## キャリア目標への適用

### ネットワークセキュリティ職（2027年目標）

**このプロジェクトが証明すること:**
- 本番システムを構築・維持できる（ラボ環境だけでなく）
- 実践的セキュリティを理解している（理論だけでなく）
- 実際の問題をトラブルシューティングできる（教科書の演習だけでなく）
- 作業を文書化する（チーム環境で重要）

**面接で聞かれるかもしれないこと:**
- 「克服した技術的課題を教えてください」
  → 6つの実際のトラブルシューティングストーリーがある
- 「新しい技術をどう学びますか？」
  → このプロジェクトが自主学習を示す
- 「問題解決プロセスを説明してください」
  → 実際のデバッグセッションを説明できる

**まだ弱い分野:**
- エンタープライズスケールシステム（これはホームラボ）
- チームコラボレーション（ソロプロジェクト）
- 正式な変更管理（自分だけでやっている）
- 高度な監視（基本的なヘルスチェックのみ）

---

## 次のステップ

### 短期（現実的に）

**Network+試験前:**
- VPNへの大きな変更なし（安定性 > 機能）
- 試験準備に集中
- 時間があれば基本的な監視を追加

**正直な評価:**
Prometheus、Grafana、より良いアクセス制御、冗長性を追加したい...しかし現実的には、試験準備が優先。このプロジェクトは、一度にすべてをやろうとすると何も適切に完了しないことを教えてくれた。

---

### 長期（資格取得後）

**2026年Q2（Network+後）:**
- フェーズ1: 監視（Prometheus + Grafana）
- メトリクスの適切な読み方を実際に学ぶ
- 煩わしくないアラートを設定

**2026年Q3（Security+後）:**
- フェーズ2: より良いアクセス制御
- デバイスごとの認証
- Cloudflare Access統合かも

**2026年Q4（CCNA後）:**
- フェーズ3: 冗長性とフェイルオーバー
- 別ハードウェアに2つ目のインスタンス
- ロードバランシング（理解できれば）

**現実的なタイムライン:**
これらのフェーズは長くかかるかもしれない。または不要だと気づくかもしれない。または就職して忙しくなるかもしれない。計画は簡単、実行は難しい。

---

## 未来の自分へのアドバイス

**覚えておくべきこと:**

1. **進めながら文書化、後ではない**
   - なぜそうしたか覚えていないでしょう
   - スクリーンショットとコマンド履歴が友達
   - 設定ファイルのコメントが後で時間を節約

2. **まず開発環境でテスト**
   - 本番はテスト環境ではない
   - 過去の自分は覚えているほど賢くない
   - 物を壊すことは学びだが、ユーザーを困らせることは違う

3. **慎重に自動化**
   - 静かに失敗する自動化は手作業より悪い
   - 自動化タスクを定期的にテスト
   - 自動化の成功/失敗を監視

4. **いつ止めるか知る**
   - 完璧は完了の敵
   - 5msのレイテンシ最適化は1週間の価値がない
   - 安定性は機能に勝る

5. **早めに助けを求める**
   - Reddit/フォーラムはあなたの問題を見たことがある
   - 4時間のデバッグ vs 10分の検索
   - プライドは高くつく

**最も重要な教訓:**
これは単にVPNを構築することだけではなかった。学び方、フラストレーションを通じて粘り強く続ける方法、時間をかけて何かを維持する方法を学ぶことだった。これらのスキルは技術的詳細よりも重要。

---

**最終更新:** 2025年11月21日  
**投資した総時間:** 3ヶ月で約80時間  
**また同じことをするか?** 絶対に。ミスも含めて。
