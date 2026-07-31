---
read_when:
    - Bonjour の検出/アドバタイズの実装または変更
    - リモート接続モード（直接接続と SSH 接続）の調整
    - リモート Node の検出とペアリングの設計
summary: Gateway を検出するための Node ディスカバリーとトランスポート（Bonjour、Tailscale、SSH）
title: 検出とトランスポート
x-i18n:
    generated_at: "2026-07-26T10:00:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw には、関連しているものの異なる 2 つの検出上の問題があります。

1. **オペレーターによるリモート制御**: 別の場所で稼働している Gateway を macOS メニューバーアプリから制御します。
2. **Node のペアリング**: iOS/Android（および将来の Node）が Gateway を検出し、安全にペアリングします。

ネットワーク上の検出とアドバタイズはすべて **Node Gateway**
（`openclaw gateway`）が担い、クライアント（Mac アプリ、iOS）は利用するだけです。

## 用語

- **Gateway**: 状態（セッション、ペアリング、Node レジストリ）を所有し、チャンネルを実行する単一の長時間稼働プロセスです。ほとんどの構成ではホストごとに 1 つ使用しますが、分離された複数 Gateway 構成も可能です。
- **Gateway WS（コントロールプレーン）**: デフォルトでは `127.0.0.1:18789` 上の WebSocket エンドポイントです。`gateway.bind` を使用して LAN/tailnet にバインドします。
- **直接 WS トランスポート**: LAN/tailnet 向けの Gateway WS エンドポイントです（SSH は使用しません）。
- **SSH トランスポート（フォールバック）**: SSH 経由で `127.0.0.1:18789` を転送してリモート制御します。
- **従来の TCP ブリッジ（削除済み）**: 以前の Node トランスポートです（[ブリッジプロトコル](/ja-JP/gateway/bridge-protocol)を参照）。検出用にアドバタイズされなくなり、現在のビルドにも含まれていません。

プロトコルの詳細: [Gateway プロトコル](/ja-JP/gateway/protocol)、
[ブリッジプロトコル（従来版）](/ja-JP/gateway/bridge-protocol)。

## 直接接続と SSH の両方が存在する理由

- **直接 WS** は、同じネットワーク上および tailnet 内で最良のユーザー体験を提供します。Bonjour による LAN 自動検出、Gateway が管理するペアリングトークンと ACL を利用でき、シェルアクセスも必要ありません。
- **SSH** は汎用的なフォールバックです。SSH アクセスさえあれば無関係なネットワーク間でも利用でき、マルチキャスト/mDNS の問題の影響を受けにくく、SSH 以外の新しい受信ポートも必要ありません。

## 検出情報源

### 1) Bonjour / DNS-SD

マルチキャスト Bonjour はベストエフォートであり、ネットワークを越えて動作しません。OpenClaw は、設定された広域 DNS-SD ドメインを介して同じ Gateway ビーコンを参照することにも対応しているため、同じ LAN 上の `local.` と、ネットワークを越えた検出用に設定されたユニキャスト DNS-SD ドメインの両方を検出対象にできます。

バンドルされた `bonjour` Plugin が有効な場合、**Gateway** は Bonjour 経由で WS エンドポイントをアドバタイズします。クライアントはそれを参照して「Gateway を選択」リストを表示し、選択されたエンドポイントを保存します。

トラブルシューティングとビーコンの詳細: [Bonjour](/ja-JP/gateway/bonjour)。

#### サービスビーコンの詳細

- サービスタイプ: `_openclaw-gw._tcp`（Gateway トランスポートビーコン）。
- TXT キー（機密情報ではありません）:

  | キー                         | 注記                                                                                                                                                            |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | 常に存在します。                                                                                                                                                  |
  | `transport=gateway`         | 常に存在します。                                                                                                                                                  |
  | `displayName=<name>`        | オペレーターが設定した表示名。                                                                                                                                |
  | `lanHost=<hostname>.local`  | LAN mDNS アドバタイザーのみ。広域 DNS-SD では書き込まれません。                                                                                                       |
  | `gatewayPort=18789`         | Gateway WS + HTTP ポート。                                                                                                                                          |
  | `gatewayTls=1`              | TLS が有効な場合のみ。                                                                                                                                        |
  | `gatewayTlsSha256=<sha256>` | TLS が有効で、フィンガープリントを利用できる場合のみ。                                                                                                         |
  | `tailnetDns=<magicdns>`     | 任意のヒント。Tailscale が利用可能な場合は自動検出されます。                                                                                                        |
  | `sshPort=<port>`            | `discovery.mdns.mode="full"` の場合のみ存在します。デフォルトの `"minimal"` モードでは省略され（SSH のデフォルトは `22`）、LAN アドバタイザーと広域 DNS-SD の両方で同様です。 |
  | `cliPath=<path>`            | `sshPort` と同じ `discovery.mdns.mode="full"` ゲート。CLI パスに関するリモートインストール用のヒントです。                                                                     |

  将来のキャンバスホストポート用として、Plugin 検出コントラクトには `canvasPort` TXT キーが定義されていますが、現在は値を設定するコードパスがないため、現時点では出力されません。

セキュリティ上の注意:

- Bonjour/mDNS TXT レコードは**認証されていません**。クライアントは TXT 値をユーザー体験上のヒントとしてのみ扱う必要があります。
- ルーティング（ホスト/ポート）では、TXT が提供する `lanHost`、`tailnetDns`、`gatewayPort` よりも、**解決済みのサービスエンドポイント**（SRV + A/AAAA）を優先する必要があります。
- TLS ピンニングでは、アドバタイズされた `gatewayTlsSha256` によって、以前に保存されたピンが上書きされることがあってはなりません。
- 選択した経路が安全な TLS ベースである場合、iOS/Android Node は初回のピンを保存する前に、明示的な「このフィンガープリントを信頼する」という確認（帯域外検証）を必須にする必要があります。

有効化、無効化、上書き:

- `openclaw plugins enable bonjour` は LAN マルチキャストアドバタイズを有効にします。
- `openclaw.json` 内の `discovery.mdns.mode` は mDNS ブロードキャストを制御します。`"minimal"`（デフォルト）、`"full"`（LAN ビーコンとすべての広域 DNS-SD ゾーンの両方に `cliPath`/`sshPort` を追加）、または `"off"`（mDNS を無効化）を指定できます。
- `OPENCLAW_DISABLE_BONJOUR=1` はアドバタイズを強制的に無効にし、`discovery.mdns.mode="off"` はそれとは独立して無効にします。`OPENCLAW_DISABLE_BONJOUR=0` は、検出されたコンテナ（Docker、containerd、Kubernetes、LXC）内での Plugin の自動無効化を上書きする明示的なオプトインです。ただし、`discovery.mdns.mode="off"` は上書きしません。バンドルされた `bonjour` Plugin は macOS ホスト（`enabledByDefaultOnPlatforms: ["darwin"]`）では自動起動し、検出されたコンテナ内では自動的に無効になります。Linux、Windows、およびその他のコンテナ化されたデプロイでは、明示的な `plugins enable bonjour` が必要です。
- `~/.openclaw/openclaw.json` 内の `gateway.bind` は Gateway のバインドモードを制御します。
- `OPENCLAW_SSH_PORT` は、アドバタイズされる SSH ポートを上書きします（`discovery.mdns.mode="full"` の場合のみ有効です）。
- `OPENCLAW_TAILNET_DNS` は `tailnetDns` ヒント（MagicDNS）を公開します。
- `OPENCLAW_CLI_PATH` は、アドバタイズされる CLI パスを上書きします。

### 2) Tailnet（ネットワーク間）

異なる物理ネットワーク上にある Gateway の検出には、Bonjour は役立ちません。直接接続先として推奨されるのは、Tailscale MagicDNS 名（推奨）または安定した tailnet IP です。

Gateway が Tailscale 環境で稼働していることを検出すると、クライアント向けの任意のヒントとして `tailnetDns` を公開します（広域ビーコンを含む）。macOS アプリは Gateway の検出時に、生の Tailscale IP よりも MagicDNS 名を優先します。MagicDNS は現在の IP を自動的に解決するため、tailnet IP が変更された場合（Node の再起動、CGNAT の再割り当て）でも信頼性を維持できます。

モバイル Node のペアリングでは、検出ヒントによって tailnet/パブリック経路上のトランスポートセキュリティ要件が緩和されることはありません。

- iOS/Android では、初回の tailnet/パブリック接続時に安全な接続経路（`wss://` または Tailscale Serve/Funnel）が引き続き必要です。
- 検出された生の tailnet IP はルーティングのヒントであり、平文のリモート `ws://` の使用を許可するものではありません。
- プライベート LAN への直接接続 `ws://` は引き続きサポートされます。
- モバイル Node で最も簡単に Tailscale を利用するには、Tailscale Serve を使用し、検出とセットアップの両方が同じ安全な MagicDNS エンドポイントを解決するようにします。

### 3) 手動 / SSH 接続先

直接経路がない場合（または直接接続が無効な場合）、クライアントは loopback Gateway ポートを転送することで、いつでも SSH 経由で接続できます。[リモートアクセス](/ja-JP/gateway/remote)を参照してください。

## トランスポートの選択（クライアントポリシー）

1. ペアリング済みの直接エンドポイントが設定され、到達可能な場合は、それを使用します。
2. それ以外の場合、検出によって `local.` または設定された広域ドメイン上で Gateway が見つかった場合は、ワンタップの「この Gateway を使用」選択肢を提示し、直接エンドポイントとして保存します。
3. それ以外の場合、tailnet DNS/IP が設定されていれば直接接続を試みます。tailnet/パブリック経路上のモバイル Node では、直接接続とは安全なエンドポイントを意味し、平文のリモート `ws://` を意味しません。
4. それ以外の場合は、SSH にフォールバックします。

## ペアリングと認証（直接トランスポート）

Gateway は Node/クライアントの受け入れに関する信頼できる唯一の情報源です。

- ペアリング要求は Gateway 内で作成、承認、拒否されます（[Gateway のペアリング](/ja-JP/gateway/pairing)を参照）。
- Gateway は認証（トークン/キーペア）、スコープ/ACL（すべてのメソッドに対する単なる生のプロキシではありません）、およびレート制限を適用します。

## コンポーネント別の責務

- **Gateway**: 検出ビーコンをアドバタイズし、ペアリングの判断を管理し、WS エンドポイントをホストします。
- **macOS アプリ**: Gateway の選択を支援し、ペアリングプロンプトを表示し、フォールバックとしてのみ SSH を使用します。
- **iOS/Android Node**: 利便性のために Bonjour を参照し、ペアリング済みの Gateway WS に接続します。

## 関連項目

- [リモートアクセス](/ja-JP/gateway/remote)
- [Tailscale](/ja-JP/gateway/tailscale)
- [Bonjour による検出](/ja-JP/gateway/bonjour)
