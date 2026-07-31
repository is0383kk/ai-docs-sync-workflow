---
read_when:
    - ネットワークアーキテクチャとセキュリティの概要が必要です
    - local と tailnet のアクセスまたはペアリングをデバッグしている場合
    - ネットワーク関連ドキュメントの正式な一覧を確認したい場合
summary: ネットワークハブ：Gateway の公開インターフェース、ペアリング、検出、セキュリティ
title: ネットワーク
x-i18n:
    generated_at: "2026-07-26T10:18:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

このハブでは、OpenClaw が localhost、LAN、tailnet 全体でデバイスに接続し、ペアリングし、保護する仕組みに関する主要ドキュメントを案内します。

## コアモデル

ほとんどの操作は、チャネル接続と WebSocket コントロールプレーンを管理する単一の長時間稼働プロセスである Gateway（`openclaw gateway`）を経由します。

- **ループバックを優先**：Gateway WS のデフォルトは `ws://127.0.0.1:18789` です。
  ループバック以外へのバインドは、有効な Gateway 認証パスがなければ起動を拒否します。
  共有シークレットによるトークン／パスワード認証、または正しく構成されたループバック以外の
  `trusted-proxy` デプロイが必要です。
- **ホストごとに 1 つの Gateway**を推奨します。分離が必要な場合は、プロファイルとポートを分離して複数の Gateway を実行します（[複数の Gateway](/ja-JP/gateway/multiple-gateways)）。
- **Canvas ホスト**は Gateway と同じポート（`/__openclaw__/canvas/`、`/__openclaw__/a2ui/`）で提供され、ループバック以外にバインドする場合は Gateway 認証によって保護されます。
- **リモートアクセス**には通常、SSH トンネルまたは Tailscale VPN を使用します（[リモートアクセス](/ja-JP/gateway/remote)）。

主要な参照資料：

- [Gateway アーキテクチャ](/ja-JP/concepts/architecture)
- [Gateway プロトコル](/ja-JP/gateway/protocol)
- [Gateway ランブック](/ja-JP/gateway)
- [Web サーフェスとバインドモード](/ja-JP/web)

## ペアリングとアイデンティティ

- [ペアリングの概要（DM と Node）](/ja-JP/channels/pairing)
- [Gateway が管理する Node のペアリング](/ja-JP/gateway/pairing)
- [デバイス CLI（ペアリングとトークンローテーション）](/ja-JP/cli/devices)
- [ペアリング CLI（DM の承認）](/ja-JP/cli/pairing)

ローカルでの信頼：

- 直接の local loopback 接続（転送／プロキシヘッダーなし）は、同一ホストでの UX を円滑に保つため、
  ペアリングを自動承認できます。
- OpenClaw には、信頼された共有シークレットのヘルパーフロー向けに限定された、
  バックエンド／コンテナローカルの自己接続パスもあります。
- 同一ホストの tailnet バインドを含む tailnet および LAN クライアントには、引き続き
  明示的なペアリング承認が必要です。

## 検出とトランスポート

- [検出とトランスポート](/ja-JP/gateway/discovery)
- [Bonjour／mDNS](/ja-JP/gateway/bonjour)
- [リモートアクセス（SSH）](/ja-JP/gateway/remote)
- [Tailscale](/ja-JP/gateway/tailscale)

## Node とトランスポート

- [Node の概要](/ja-JP/nodes)
- [ブリッジプロトコル（旧式 Node、歴史的資料）](/ja-JP/gateway/bridge-protocol)
- [Node ランブック：iOS](/ja-JP/platforms/ios)
- [Node ランブック：Android](/ja-JP/platforms/android)

## セキュリティ

- [セキュリティの概要](/ja-JP/gateway/security)
- [Gateway 設定リファレンス](/ja-JP/gateway/configuration)
- [トラブルシューティング](/ja-JP/gateway/troubleshooting)
- [Doctor](/ja-JP/gateway/doctor)

## 関連項目

- [Gateway ランブック](/ja-JP/gateway)
- [リモートアクセス](/ja-JP/gateway/remote)
