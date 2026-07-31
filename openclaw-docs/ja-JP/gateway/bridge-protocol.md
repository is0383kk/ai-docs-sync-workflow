---
read_when:
    - 古い Node クライアントコードまたはアーカイブされたペアリングログの調査
    - 従来の Node サーフェスが以前公開していた内容の監査
summary: 過去のブリッジプロトコル（レガシー Node）：TCP JSONL、ペアリング、スコープ付き RPC
title: ブリッジプロトコル
x-i18n:
    generated_at: "2026-07-26T09:01:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e8b69c59f2170439f0e7b139bf5bbdb429d7c9d8dde7b36cd64aab63939c95d
    source_path: gateway/bridge-protocol.md
    workflow: 16
---

<Warning>
TCP ブリッジは**削除されました**。現在の OpenClaw ビルドにはブリッジリスナーが含まれておらず、`bridge.*` 設定キーもスキーマから削除されています。このページは過去の参照資料としてのみ提供されています。すべての Node/オペレータークライアントでは、[Gateway プロトコル](/ja-JP/gateway/protocol)を使用してください。
</Warning>

## 存在していた理由

- **セキュリティ境界**：Gateway API の全サーフェスではなく、小規模な許可リストを公開していました。
- **ペアリングと Node ID**：Node の受け入れは Gateway が管理し、Node ごとのトークンに関連付けられていました。
- **検出 UX**：Node は LAN 上の Bonjour を介して Gateway を検出するか、tailnet 経由で直接接続できました。
- **ループバック WS**：SSH でトンネリングしない限り、完全な WS コントロールプレーンはローカルに維持されていました。

## トランスポート

- TCP。1 行につき 1 つの JSON オブジェクト（JSONL）。
- 任意の TLS（`bridge.tls.enabled: true`）。
- デフォルトのリスナーポートは `18790` でした。

TLS が有効な場合、検出 TXT レコードには `bridgeTls=1` に加えて、秘密ではないヒントとして `bridgeTlsSha256` が含まれていました。Bonjour/mDNS TXT レコードは認証されていないため、クライアントは他の帯域外検証なしに、通知されたフィンガープリントを信頼できるピンとして扱うことはできませんでした。

## ハンドシェイクとペアリング

1. クライアントは、Node のメタデータとトークン（すでにペアリング済みの場合）を含む `hello` を送信します。
2. ペアリングされていない場合、Gateway は `error`（`NOT_PAIRED` / `UNAUTHORIZED`）を返します。
3. クライアントは `pair-request` を送信します。
4. Gateway は承認を待ってから、`pair-ok` と `hello-ok` を送信します。

`hello-ok` は以前 `serverName` を返していました。現在、ホストされている Plugin サーフェスは、現行の Gateway プロトコルの `pluginSurfaceUrls` を介して通知されます（Canvas/A2UI は `pluginSurfaceUrls.canvas` を使用します）。

## フレーム

クライアントから Gateway：

- `req` / `res`：スコープ付き Gateway RPC（チャット、セッション、設定、ヘルス、音声ウェイク、skills.bins）。
- `event`：Node シグナル（音声文字起こし、エージェントリクエスト、チャット購読、exec ライフサイクル）。

Gateway からクライアント：

- `invoke` / `invoke-res`：Node コマンド（`canvas.*`、`camera.*`、`screen.record`、`location.get`、`sms.send`）。
- `event`：購読中のセッションに対するチャット更新。
- `ping` / `pong`：キープアライブ。

許可リストの適用は `src/gateway/server-bridge.ts` に実装されていました（削除済み）。

## Exec ライフサイクルイベント

Node は、完了した `system.run` アクティビティを通知するために `exec.finished` を発行し、Gateway によってシステムイベントにマッピングされていました（旧式の Node は `exec.started` を発行することもできました）。`exec.denied` は、拒否された `system.run` 試行を、システムイベントをキューに追加したりエージェント処理を起動したりせず、終端の拒否としてマークしていました。

ペイロードフィールド（明記されているものを除き、すべて任意）：

| フィールド                            | 注記                                                                                          |
| -------------------------------- | ---------------------------------------------------------------------------------------------- |
| `sessionKey`                     | 必須。イベントの関連付けに使用するエージェントセッション。`exec.finished` の場合は、システムイベントの配信にも使用します。 |
| `runId`                          | グループ化に使用する一意の exec ID。                                                                   |
| `command`                        | 未加工または整形済みのコマンド文字列。                                                               |
| `exitCode`、`timedOut`、`output` | 完了の詳細（完了時のみ）。                                                            |
| `reason`                         | 拒否理由（拒否時のみ）。                                                                   |

## 過去の tailnet 使用方法

- ブリッジを tailnet IP にバインド：`~/.openclaw/openclaw.json` で `bridge.bind: "tailnet"` を指定します（過去の情報のみ。`bridge.*` は有効な設定ではなくなりました）。
- クライアントは MagicDNS 名または tailnet IP を介して接続していました。
- Bonjour はネットワークをまたがないため、それ以外の場合は広域 DNS-SD または手動で指定したホスト/ポートが必要でした。

## バージョニング

ブリッジは暗黙的な v1 で、最小/最大バージョンのネゴシエーションはありませんでした。現在の Node/オペレータークライアントは、プロトコルのバージョン範囲をネゴシエートする WebSocket [Gateway プロトコル](/ja-JP/gateway/protocol)を使用します。

## 関連項目

- [Gateway プロトコル](/ja-JP/gateway/protocol)
- [Node](/ja-JP/nodes)
