---
read_when:
    - ペアリング済みの Node（カメラ、画面、キャンバス）を管理している場合
    - リクエストを承認するか、Node コマンドを実行する必要があります
summary: '`openclaw nodes` の CLI リファレンス（ステータス、ペアリング、呼び出し、カメラ／キャンバス／画面／位置情報／通知）'
title: Nodeたち
x-i18n:
    generated_at: "2026-07-26T08:58:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

ペアリング済みの Node（デバイス）を管理し、Node の機能を呼び出します。

関連項目: [Node の概要](/ja-JP/nodes) - [アクティブなコンピューターのプレゼンス](/ja-JP/nodes/presence) - [カメラ Node](/ja-JP/nodes/camera) - [画像 Node](/ja-JP/nodes/images)

すべてのサブコマンドに共通するオプション: `--url <url>`、`--token <token>`、`--timeout <ms>`（デフォルト `10000`）、`--json`。

## ステータス

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

`status` と `list` はどちらも、`--connected`（接続中の Node のみ）と `--last-connected <duration>`（例: `24h`、`7d`。指定期間内に接続した Node のみ）を受け付けます。`list` は保留中とペアリング済みの Node を別々の表に表示し、ペアリング済みの行には直近の接続からの経過時間（Last Connect）が含まれます。`status` は、Node ごとの機能、バージョン、最終入力の詳細を統合した 1 つの表に表示します。接続中の macOS Node は、ユーザーが **アクティブなコンピューターの検出**を有効にしてアクセシビリティ権限を付与した後にのみ、最終入力を報告します。最新の行には `active` が付けられます。[アクティブなコンピューターのプレゼンス](/ja-JP/nodes/presence)を参照してください。`describe` は、1 つの Node の機能、権限、アクティビティ、および有効な呼び出しコマンドと保留中の呼び出しコマンドを出力します。

## ペアリング

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

これらのコマンドは、Node の WS `connect` ハンドシェイクを制御するデバイスペアリング（`openclaw devices approve`）とは別の、Gateway が所有する `node.pair.*` ストアを操作します。両者の関係については、[Node](/ja-JP/nodes)を参照してください。

- `remove` は、Node のペアリング済みロールエントリを取り消します。デバイスを基盤とする Node の場合、デバイスペアリングストア内の `node` ロールを取り消し、その Node ロールのセッションを切断します。複数ロールを持つデバイスは行が維持され、`node` ロールのみを失います。Node 専用デバイスの行は削除されます。また、一致する従来の Gateway 所有 Node ペアリングレコードも消去されます。
- `pending` に必要なのは `operator.pairing` スコープだけです。
- `gateway.nodes.pairing.autoApproveCidrs` は、明示的に信頼された初回の `role: node` デバイスペアリングについて、保留手順を省略できます。デフォルトでは無効で、ロールのアップグレードは承認しません。
- `gateway.nodes.pairing.sshVerify`（デフォルトで有効）は、Gateway が SSH 経由で Node ホストのデバイスキーを検証できる場合、初回の `role: node` デバイスペアリングを自動承認します。最初の機能サーフェスも同じ手順で承認されます。[Node のペアリング](/ja-JP/gateway/pairing#ssh-verified-device-auto-approval-default)を参照してください。
- `approve` のスコープ要件は、保留中のリクエストで宣言されたコマンドに従います。
  - コマンドなしのリクエスト: `operator.pairing`
  - 通常の Node コマンド: `operator.pairing` + `operator.write`
  - 管理者権限が関係するコマンド（`system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`fs.listDir`、`system.execApprovals.get/set`）: `operator.pairing` + `operator.admin`
- `remove` のスコープ: `operator.pairing` はオペレーター以外の Node の行を削除できます。複数ロールを持つデバイスで、自身の Node ロールを取り消すデバイストークン呼び出し元には、追加で `operator.admin` が必要です。

## 呼び出し

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

フラグ:

- `--command <command>`（必須）: 例: `canvas.eval`。
- `--params <json>`: JSON オブジェクト文字列（デフォルト `{}`）。
- `--invoke-timeout <ms>`: Node 呼び出しのタイムアウト（デフォルト `15000`）。
- `--idempotency-key <key>`: オプションの冪等性キー。

ここでは `system.run` と `system.run.prepare` はブロックされます。代わりに、シェル実行には `host=node` を指定した `exec` ツールを使用してください。`system.which` は `invoke` 経由で許可されます。

## 通知、プッシュ、位置情報、画面

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify` は、`system.notify` を宣言している Node にローカル通知を送信します。対象には macOS、iOS、Android、および直接接続された watchOS Node が含まれます。watchOS への直接配信には、OpenClaw がアクティブである必要があります。`--title` または `--body` が必要です。オプション: `--sound <name>`、`--priority <passive|active|timeSensitive>`、`--delivery <system|overlay|auto>`（デフォルト `system`）、`--invoke-timeout <ms>`（デフォルト `15000`）。
- `push` は、iOS Node に APNs テストプッシュを送信します。オプション: `--title <text>`（デフォルト `OpenClaw`）、`--body <text>`、検出された APNs 環境を上書きする `--environment <sandbox|production>`。
- `location get` は、Node の現在位置を取得します。オプション: `--max-age <ms>`（キャッシュ済みの測位結果を再利用）、`--accuracy <coarse|balanced|precise>`、`--location-timeout <ms>`（デフォルト `10000`）、`--invoke-timeout <ms>`（デフォルト `20000`）。
- `screen record` は短いクリップをキャプチャし、保存先のパスを出力します（または `--json` を指定して JSON を書き込みます）。オプション: `--screen <index>`（デフォルト `0`）、`--duration <ms|10s>`（デフォルト `10000`）、`--fps <fps>`（デフォルト `10`）、`--no-audio`、`--out <path>`、`--invoke-timeout <ms>`（デフォルト `120000`）。

カメラと Canvas のコマンドには、それぞれ専用のドキュメントがあります: [カメラ Node](/ja-JP/nodes/camera)、[Canvas](/ja-JP/platforms/mac/canvas)。Canvas は、同梱されている実験的な Canvas Plugin によって実装されています。コアは互換性のためのマウントポイントとして `openclaw nodes canvas` を維持します。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Node](/ja-JP/nodes)
