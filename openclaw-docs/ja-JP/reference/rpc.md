---
read_when:
    - 外部 CLI 統合の追加または変更
    - RPC アダプター（signal-cli、imsg）のデバッグ
summary: 外部 CLI（signal-cli、imsg）向け RPC アダプターと Gateway パターン
title: RPC アダプター
x-i18n:
    generated_at: "2026-07-26T09:51:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw は JSON-RPC を介して外部 CLI と統合します。現在、2 つのパターンが使用されています。

## パターン A: HTTP デーモン（signal-cli）

- `signal-cli` は、HTTP 経由の JSON-RPC を使用するデーモンとして実行されます。
- イベントストリームは SSE（`/api/v1/events`）です。
- ヘルスプローブ: `/api/v1/check`。
- `channels.signal.transport.kind="managed-native"`（デフォルト）の場合、OpenClaw がライフサイクルを管理します。

セットアップとエンドポイントについては、[Signal](/ja-JP/channels/signal) を参照してください。

## パターン B: stdio 子プロセス（imsg）

- OpenClaw は、[iMessage](/ja-JP/channels/imessage) のために `imsg rpc` を子プロセスとして起動します。
- JSON-RPC は stdin/stdout を介して行区切りで送受信されます（1 行につき 1 つの JSON オブジェクト）。
- TCP ポートもデーモンも不要です。

使用される主要なメソッド:

- `watch.subscribe` → 通知（`method: "message"`）
- `watch.unsubscribe`
- `send`
- `chats.list`（プローブ/診断）

セットアップとアドレス指定については、[iMessage](/ja-JP/channels/imessage) を参照してください（表示文字列より `chat_id` を推奨）。

## アダプターのガイドライン

- Gateway がプロセスを管理します（起動/停止はプロバイダーのライフサイクルに連動）。
- RPC クライアントの耐障害性を維持します。タイムアウトを設定し、終了時に再起動してください。
- 表示文字列よりも安定した ID（例: `chat_id`）を優先してください。

## 関連項目

- [Gateway プロトコル](/ja-JP/gateway/protocol)
