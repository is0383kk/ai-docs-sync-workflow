---
read_when:
    - 音声ウェイクワードの動作またはデフォルトの変更
    - ウェイクワードの同期が必要な新しい Node プラットフォームの追加
summary: グローバル音声ウェイクワード（Gateway 管理）と Node 間での同期方法
title: 音声ウェイク
x-i18n:
    generated_at: "2026-07-26T09:39:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aef2a5bba664ce10fb6ab457bb6d202639dcc6c0a9df61567e7cb402c290bbec
    source_path: nodes/voicewake.md
    workflow: 16
---

ウェイクワードは **Gateway が所有する単一のグローバルリスト**です。Node ごとのカスタムリストはありません。どの Node またはアプリ UI からでもリストを編集できます。Gateway は変更を永続化し、接続されているすべてのクライアントにブロードキャストします。

- **macOS**: ローカルの Voice Wake 有効化/無効化トグル。macOS 26 以降が必要です。ランタイム/PTT の詳細については、[Voice wake（macOS）](/ja-JP/platforms/mac/voicewake)を参照してください。
- **iOS**: Settings にあるローカルの Voice Wake 有効化/無効化トグル。
- **Android**: Settings → Voice にあるローカルの Voice Wake 有効化/無効化トグルとウェイクワードエディター。Android のオンデバイス音声認識が必要です。

## ストレージ

ウェイクワードとルーティングルールは Gateway の状態データベースに保存されます。デフォルトは `~/.openclaw/state/openclaw.sqlite` です（`OPENCLAW_STATE_DIR` でオーバーライド可能）。使用するテーブルは `voicewake_triggers`、`voicewake_routing_config`、`voicewake_routing_routes` です。従来の `settings/voicewake.json` と `settings/voicewake-routing.json` は `openclaw doctor --fix` の移行入力としてのみ使用され、ランタイムが読み取ることはありません。

## プロトコル

### トリガーリスト

| メソッド          | パラメーター                   | 結果                   |
| --------------- | ------------------------ | ------------------------ |
| `voicewake.get` | なし                     | `{ triggers: string[] }` |
| `voicewake.set` | `{ triggers: string[] }` | `{ triggers: string[] }` |

`voicewake.set` は入力を正規化します。空白を除去し、空のエントリを削除し、トリガーを最大 32 件に制限し、サロゲートペアを分割せずに各トリガーを UTF-16 コード単位で 64 までに切り詰めます。結果が空の場合は、組み込みのデフォルト（`openclaw`、`claude`、`computer`）にフォールバックします。

### ルーティング（トリガーからターゲットへ）

| メソッド                  | パラメーター                               | 結果                               |
| ----------------------- | ------------------------------------ | ------------------------------------ |
| `voicewake.routing.get` | なし                                 | `{ config: VoiceWakeRoutingConfig }` |
| `voicewake.routing.set` | `{ config: VoiceWakeRoutingConfig }` | `{ config: VoiceWakeRoutingConfig }` |

```json
{
  "version": 1,
  "defaultTarget": { "mode": "current" },
  "routes": [{ "trigger": "robot wake", "target": { "sessionKey": "agent:main:main" } }],
  "updatedAtMs": 1730000000000
}
```

各ルートの `target` は、次のいずれか 1 つだけをサポートします。

- `{ "mode": "current" }`
- `{ "agentId": "main" }`
- `{ "sessionKey": "agent:main:main" }`

上限: ルートは最大 32 件、トリガーテキストは最大 64 文字です。ルートのトリガーは照合と重複検出のため、小文字化、各単語の先頭と末尾にある句読点の除去、空白の集約によって正規化されます（`"Hey, Bot!!"` と `"hey bot"` は一致し、重複として数えられます）。これは、上記のグローバルトリガーリストで使用される単純なトリミングより厳格な正規化です。

### イベント

| イベント                       | ペイロード                              |
| --------------------------- | ------------------------------------ |
| `voicewake.changed`         | `{ triggers: string[] }`             |
| `voicewake.routing.changed` | `{ config: VoiceWakeRoutingConfig }` |

どちらも、読み取りスコープを持つすべての WebSocket クライアント（macOS アプリ、WebChat など）と、接続されているすべての Node にブロードキャストされます。Node は接続直後に、初期スナップショットのプッシュとして両方を受信します。

## クライアントの動作

- **macOS**: `voicewake.set`/`voicewake.get` を呼び出し、他のクライアントとの同期を維持するために `voicewake.changed` をリッスンします。
- **iOS**: `voicewake.set`/`voicewake.get` を呼び出し、ローカルのウェイクワード検出の応答性を維持するために `voicewake.changed` をリッスンします。
- **Android**: `voicewake.set`/`voicewake.get` を呼び出し、`voicewake.changed` をリッスンし、有効な間は `voiceWake` を通知します。認識はオンデバイスかつフォアグラウンドでのみ動作します。Talk、手動ディクテーション、ボイスメモの録音、またはメッセージ読み上げが音声を使用している間は一時停止します。

## 関連項目

- [Talk モード](/ja-JP/nodes/talk)
- [音声とボイスメモ](/ja-JP/nodes/audio)
- [メディア理解](/ja-JP/nodes/media-understanding)
