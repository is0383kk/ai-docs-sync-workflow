---
read_when:
    - OpenClaw を Upstash Box にデプロイする
    - SSH トンネル経由でダッシュボードにアクセスできる、OpenClaw 用のマネージド Linux 環境が必要な場合
summary: キープアライブと SSH トンネルアクセスを使用して Upstash Box 上で OpenClaw をホストする
title: Upstash Box
x-i18n:
    generated_at: "2026-07-26T09:28:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 29232c43e0e4940b7445ab8896c9ccd3e81d0fdbdd522d7f50cb8c8057ac18f0
    source_path: install/upstash.md
    workflow: 16
---

Upstash Box（キープアライブライフサイクルをサポートするマネージド Linux 環境）上で、永続的な OpenClaw Gateway を実行します。

ダッシュボードへのアクセスには SSH トンネルを使用します。Gateway ポートを公開インターネットに直接公開しないでください。

## 前提条件

- Upstash アカウント
- キープアライブ対応の Upstash Box
- ローカルマシン上の SSH クライアント

## Box を作成する

Upstash Console でキープアライブ対応の Box を作成します。Box ID（例：
`right-flamingo-14486`）と Box API キーを控えておきます。

Upstash は、最新の OpenClaw Box チュートリアルを
[OpenClaw のセットアップ](https://upstash.com/docs/box/guides/openclaw-setup)で提供しています。

## SSH トンネルで接続する

OpenClaw ダッシュボードのポートをローカルマシンに転送します。プロンプトが表示されたら、
Box API キーを SSH パスワードとして使用します。

```bash
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

キープアライブオプションにより、オンボーディング中にアイドル状態のトンネルが切断されにくくなります。

## OpenClaw をインストールする

Box 内で次を実行します。

```bash
sudo npm install -g openclaw
```

## オンボーディングを実行する

```bash
openclaw onboard --install-daemon
```

プロンプトに従います。オンボーディングが完了したら、ダッシュボードの URL とトークンをコピーします。

## Gateway を起動する

Box ネットワーク用に Gateway を設定し、バックグラウンドで起動します。

```bash
openclaw config set gateway.bind lan
nohup openclaw gateway > gateway.log 2>&1 &
```

SSH トンネルが有効な状態で、ダッシュボードの URL をローカルで開きます。

```text
http://127.0.0.1:18789/#token=<your-token>
```

## 自動再起動

Box の起動時に Gateway が再起動するよう、このコマンドを Box の初期化スクリプトとして設定します。

```bash
nohup openclaw gateway > gateway.log 2>&1 &
```

## トラブルシューティング

オンボーディング中に SSH がフリーズする場合は、クリーンな SSH 設定とキープアライブを使用して再接続します。

```bash
ssh -F /dev/null -o ControlMaster=no -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

これにより、古いローカルの `~/.ssh/config` 設定を回避し、ネットワークがアイドル状態の間もトンネルを維持します。

## 関連項目

- [リモートアクセス](/ja-JP/gateway/remote)
- [Gateway のセキュリティ](/ja-JP/gateway/security)
- [OpenClaw の更新](/ja-JP/install/updating)
