---
read_when:
    - Linux サーバーまたはクラウド VPS で Gateway を実行したい場合
    - ホスティングガイドの概要をすぐに確認したい場合
    - OpenClaw 向けの汎用的な Linux サーバーチューニングを行いたい場合
sidebarTitle: Linux Server
summary: Linux サーバーまたはクラウド VPS で OpenClaw を実行する — プロバイダーの選択、アーキテクチャ、チューニング
title: Linux サーバー
x-i18n:
    generated_at: "2026-07-26T09:57:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

任意の Linux サーバーまたはクラウド VPS で OpenClaw Gateway を実行します。このページでは、
プロバイダーの選択方法、クラウドデプロイの仕組み、およびあらゆる環境に適用できる
一般的な Linux チューニングについて説明します。

## プロバイダーを選ぶ

<CardGroup cols={2}>
  <Card title="Azure" href="/ja-JP/install/azure">Linux VM</Card>
  <Card title="DigitalOcean" href="/ja-JP/install/digitalocean">シンプルな有料 VPS</Card>
  <Card title="exe.dev" href="/ja-JP/install/exe-dev">HTTPS プロキシ付き VM</Card>
  <Card title="Fly.io" href="/ja-JP/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/ja-JP/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/ja-JP/install/hetzner">Hetzner VPS 上の Docker</Card>
  <Card title="Hostinger" href="/ja-JP/install/hostinger">ワンクリックセットアップ対応 VPS</Card>
  <Card title="Northflank" href="/ja-JP/install/northflank">ワンクリックのブラウザーセットアップ</Card>
  <Card title="Oracle Cloud" href="/ja-JP/install/oracle">Always Free ARM ティア</Card>
  <Card title="Railway" href="/ja-JP/install/railway">ワンクリックのブラウザーセットアップ</Card>
  <Card title="Raspberry Pi" href="/ja-JP/install/raspberry-pi">ARM セルフホスト</Card>
</CardGroup>

**AWS（EC2 / Lightsail / 無料利用枠）**も適しています。
コミュニティによる動画ガイドは
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
で視聴できます（コミュニティリソースのため、利用できなくなる可能性があります）。

## クラウドセットアップの仕組み

- **Gateway は VPS 上で実行され**、状態とワークスペースを管理します。
- ノート PC またはスマートフォンから **Control UI** または **Tailscale/SSH** 経由で接続します。
- VPS を信頼できる唯一の情報源として扱い、状態とワークスペースを定期的に**バックアップ**してください。
- 安全なデフォルト設定：Gateway をループバックに維持し、SSH トンネルまたは Tailscale Serve 経由でアクセスします。
  `lan` または `tailnet` にバインドする場合、認証が信頼できる
  プロキシに委任されていない限り、Gateway には共有シークレット
  （`gateway.auth.token` または `gateway.auth.password`）が必要です。

関連ページ：[Gateway のリモートアクセス](/ja-JP/gateway/remote)、[プラットフォームハブ](/ja-JP/platforms)。

## 最初に管理アクセスを強化する

公開 VPS に OpenClaw をインストールする前に、そのマシン自体をどのように管理するかを
決めてください。

- Tailnet のみによる管理アクセスの場合：まず Tailscale をインストールし、VPS を
  tailnet に参加させ、Tailscale IP または MagicDNS 名を使用した別の SSH セッションを確認してから、
  公開 SSH を制限します。
- Tailscale を使用しない場合：追加のサービスを公開する前に、
  SSH 経路に同等の強化を適用します。
- これは Gateway へのアクセスとは別です。OpenClaw を引き続き
  ループバックにバインドし、ダッシュボードには SSH トンネルまたは Tailscale Serve を使用できます。

Tailscale 固有の Gateway オプションについては、[Tailscale](/ja-JP/gateway/tailscale)を参照してください。

## VPS 上の社内共有エージェント

すべてのユーザーが同じ信頼境界内にいて、エージェントを業務専用とする場合、
チームで単一のエージェントを実行する構成は有効です。

- 専用ランタイム（VPS/VM/コンテナ + 専用の OS ユーザー/アカウント）で実行してください。
- そのランタイムを個人の Apple/Google アカウントや、個人用のブラウザー/パスワードマネージャープロファイルにサインインさせないでください。
- ユーザー同士が敵対的である場合は、Gateway、ホスト、OS ユーザーごとに分離してください。

セキュリティモデルの詳細：[セキュリティ](/ja-JP/gateway/security)。

## VPS で Node を使用する

Gateway をクラウドに置いたまま、ローカルデバイス
（Mac/iOS/Android/ヘッドレス）の **Node** をペアリングできます。Gateway をクラウドに維持しながら、
Node によってローカルの画面、カメラ、キャンバス、および `system.run`
機能を利用できます。

ドキュメント：[Node](/ja-JP/nodes)、[Node CLI](/ja-JP/cli/nodes)。

## 小規模 VM と ARM ホスト向けの起動チューニング

低性能の VM（または ARM ホスト）で CLI コマンドが遅く感じられる場合は、Node のモジュールコンパイルキャッシュを有効にします。

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` により、繰り返し実行するコマンドの起動時間が短縮されます。初回実行時にキャッシュがウォームアップされます。
- `OPENCLAW_NO_RESPAWN=1` により、通常の Gateway 再起動が同一プロセス内で維持されるため、余分なプロセス間の引き渡しを回避し、小規模ホストでの PID 追跡を簡素に保てます。
- Raspberry Pi 固有の情報については、[Raspberry Pi](/ja-JP/install/raspberry-pi) を参照してください。

### systemd チューニングチェックリスト（任意）

`systemd` を使用する VM ホストでは、以下を検討してください。

- 安定した起動経路のためのサービス環境変数：`OPENCLAW_NO_RESPAWN=1` および
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- 明示的な再起動動作：`Restart=always`、`RestartSec=2`、`TimeoutStartSec=90`
- ランダム I/O によるコールドスタートのペナルティを軽減するため、状態/キャッシュパスには SSD ベースのディスクを使用します。

標準の `openclaw onboard --install-daemon` 経路では systemd ユーザー
ユニットがインストールされます。編集するには、次を実行します。

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

意図的にシステムユニットをインストールした場合は、
`sudo systemctl edit openclaw-gateway.service` で編集してください。

`Restart=` ポリシーが自動復旧に役立つ仕組み：
[systemd でサービス復旧を自動化できます](https://www.redhat.com/en/blog/systemd-automate-recovery)。

Linux の OOM 動作、子プロセスの強制終了対象の選択、および `exit 137`
の診断については、[Linux のメモリプレッシャーと OOM キル](/ja-JP/platforms/linux#memory-pressure-and-oom-kills)を参照してください。

## 関連情報

- [インストール概要](/ja-JP/install)
- [DigitalOcean](/ja-JP/install/digitalocean)
- [Fly.io](/ja-JP/install/fly)
- [Hetzner](/ja-JP/install/hetzner)
