---
read_when:
    - メッセージ CLI アクションの追加または変更
    - 送信チャネルの動作を変更する
summary: '`openclaw message` の CLI リファレンス（送信 + チャンネルアクション）'
title: メッセージ
x-i18n:
    generated_at: "2026-07-26T09:36:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2d1cca9be7cfa7625cac3e440ecb5847d9fab9c545c9267a41a2f99c26c514b
    source_path: cli/message.md
    workflow: 16
---

# `openclaw message`

Discord、Google Chat、iMessage、Matrix、Mattermost（Plugin）、Microsoft Teams、
Signal、Slack、Telegram、WhatsApp でメッセージやチャンネルアクションを送信するための
単一の送信コマンドです。

```bash
openclaw message <subcommand> [flags]
```

## チャンネルの選択

- 複数のチャンネルが設定されている場合、`--channel <name>` は必須です。
  設定されているチャンネルが 1 つだけの場合は、そのチャンネルがデフォルトになります。
- 値: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp`
  （Mattermost には Plugin が必要です）。
- チャンネル接頭辞付きのターゲット（例: `discord:channel:123`）は、明示的な
  `--channel` がなくても、そのターゲットを所有する Plugin に解決されます。

## ターゲット形式（`-t, --target`）

| チャンネル          | 形式                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| Discord             | `channel:<id>`、`user:<id>`、`<@id>` メンション、または数値のみの ID（チャンネル ID として扱われます） |
| Google Chat         | `spaces/<spaceId>` または `users/<userId>`                                                               |
| iMessage            | ハンドル、`chat_id:<id>`、`chat_guid:<guid>`、または `chat_identifier:<id>`                                |
| Mattermost（Plugin） | `channel:<id>`、`user:<id>`、`@username`、または ID のみ（チャンネルとして扱われます）    |
| Matrix              | `@user:server`、`!room:server`、または `#alias:server`                                          |
| Microsoft Teams     | `conversation:<id>`（`19:...@thread.tacv2`）、会話 ID のみ、または `user:<aad-object-id>`                           |
| Signal              | `+E.164`、`group:<id>`、`uuid:<id>`、`username:<name>`/`u:<name>`、またはこれらのいずれかに `signal:` を付けたもの |
| Slack               | `channel:<id>` または `user:<id>`（ID のみの場合はチャンネルとして扱われます）                   |
| Telegram            | チャット ID、`@username`、またはフォーラムトピックのターゲット: `<chatId>:topic:<topicId>`（または `--thread-id <topicId>`） |
| WhatsApp            | E.164、グループ JID（`...@g.us`）、またはチャンネル／ニュースレター JID（`...@newsletter`）       |

チャンネル名の検索: ディレクトリを持つプロバイダー（Discord、Slack など）では、
`Help` や `#help` のような名前はディレクトリキャッシュを介して解決されます。キャッシュにない場合は、
プロバイダーが対応していればライブのディレクトリ検索にフォールバックします。

## 共通フラグ

すべてのアクションで `--channel <name>`、`--account <id>`、`--json`、
`--dry-run`、`--verbose` を使用できます。宛先を取るアクションでは、
`-t, --target <dest>` も使用できます。

## SecretRef の解決

`openclaw message` はアクションの実行前に、可能な限り狭いスコープで
チャンネルの SecretRef を解決します。

- `--channel` が設定されている場合（または接頭辞付きターゲットから推測された場合）はチャンネルスコープ
- `--account` も設定されている場合はアカウントスコープ
- どちらも設定されていない場合は、設定済みのすべてのチャンネル

無関係なチャンネルで未解決の SecretRef があっても、対象を指定したアクションがブロックされることはありません。一方、
選択したチャンネルまたはアカウントに未解決の SecretRef がある場合、アクションは安全側に失敗します。

## アクション

### コア

| アクション      | チャンネル                                                                                                      | 必須                                                           | 注記                                                                                                                                                                                                                                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `send`          | Discord、Google Chat、iMessage、Matrix、Mattermost（Plugin）、Microsoft Teams、Signal、Slack、Telegram、WhatsApp | `--target` と、`--message`/`--media`/`--presentation` のいずれか | 下記の[送信](#send)を参照してください。                                                                                                                                                                                                                                                                |
| `poll`          | Discord、Matrix、Microsoft Teams、Telegram、WhatsApp                                                            | `--target`、`--poll-question`、`--poll-option`（繰り返し） | 下記の[投票](#poll)を参照してください。                                                                                                                                                                                                                                                                |
| `react`          | Discord、Matrix、Nextcloud Talk、Signal、Slack、Telegram、WhatsApp                                              | `--message-id`、`--target`                         | `--emoji`、`--remove`（`--emoji` が必要です。対応している場合、自分のリアクションを消去するには省略します。[リアクション](/ja-JP/tools/reactions)を参照してください）。WhatsApp: `--participant`、`--from-me`。Signal のグループリアクションには `--target-author` または `--target-author-uuid` が必要です。Nextcloud Talk はリアクションの追加のみに対応しており、`--remove` はエラーになります。 |
| `reactions`          | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--message-id`、`--target`                         | `--limit`。                                                                                                                                                                                                                                                                                   |
| `read`          | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--target`                                             | `--limit`、`--message-id`、`--before`、`--after`。Discord: `--around`、`--include-thread`。Slack: `--message-id` は特定のタイムスタンプを読み取ります。スレッド内の特定の返信には `--thread-id` と組み合わせます。 |
| `edit`          | Discord、Matrix、Microsoft Teams、Slack、Telegram                                                               | `--message-id`、`--message`、`--target`     | Telegram のフォーラムスレッドでは `--thread-id` を使用します。                                                                                                                                                                                                                                    |
| `delete`          | Discord、Matrix、Microsoft Teams、Slack、Telegram                                                               | `--message-id`、`--target`                         |                                                                                                                                                                                                                                                                                                        |
| `pin` / `unpin` | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--message-id`、`--target`                         | `unpin` では `--pinned-message-id` も使用できます（Microsoft Teams: チャットメッセージ ID ではなく、ピン留め／ピン留め一覧のリソース ID）。                                                                                                                                                      |
| `pins`（一覧）  | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--target`                                             | `--limit`。                                                                                                                                                                                                                                                                                   |
| `permissions`          | Discord、Matrix                                                                                                 | `--target`                                             | Matrix: 暗号化が有効で、検証アクションが許可されている場合のみ使用できます。                                                                                                                                                                                                                           |
| `search`          | Discord                                                                                                         | `--guild-id`、`--query`                         | `--channel-id`、`--channel-ids`（繰り返し）、`--author-id`、`--author-ids`（繰り返し）、`--limit`。                                                                                                                                                                            |
| `member info`          | Discord、Matrix、Microsoft Teams、Slack                                                                         | `--user-id`                                             | `--guild-id`（Discord）。                                                                                                                                                                                                                                                                        |

### 送信

```bash
openclaw message send --channel discord \
  --target channel:123 --message "hi" --reply-to 456
```

- `--media <path-or-url>`: 画像／音声／動画／ドキュメント（ローカルパスまたは
  URL）を添付します。
- `--presentation <json>`: `text`、`context`、`divider`、
  `chart`、`table`、`buttons`、`select` ブロックを含む共有ペイロードです。各チャンネルの
  機能に応じてレンダリングされます。[メッセージプレゼンテーション](/ja-JP/plugins/message-presentation)を参照してください。
- `--delivery <json>`: 一般的な配信設定です（例: `{"pin":
true}`）。`--pin` は、チャンネルが対応している場合に
  ピン留め配信を指定する短縮形です。
- `--reply-to <id>`、`--thread-id <id>`（Telegram のフォーラムトピック、Slack のスレッド
  タイムスタンプ。`--reply-to` と同じフィールドです）。
- `--force-document`（Telegram、WhatsApp）: チャンネルによる圧縮を避けるため、画像／GIF／動画を
  ドキュメントとして送信します。
- `--silent`（Telegram、Discord）: 通知なしで送信します。
- `--gif-playback`（WhatsApp のみ）: 動画メディアを GIF として再生します。

```bash
openclaw message send --channel discord \
  --target channel:123 --message "選択してください:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"承認","value":"approve","style":"success"},{"label":"拒否","value":"decline","style":"danger"}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat --message "選択してください:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"はい","value":"cmd:yes"},{"label":"いいえ","value":"cmd:no"}]}]}'
```

Slack は、サポートされているチャートブロックをネイティブにレンダリングします。他のチャンネルでは、同じ
データを読みやすいテキストとして受信します:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"blocks":[{"type":"chart","chartType":"bar","title":"四半期売上高","categories":["第1四半期","第2四半期"],"series":[{"name":"売上高","values":[120,145]}],"xLabel":"四半期"}]}'
```

Slack は明示的なテーブルブロックもネイティブにレンダリングします。他のチャンネルでは、
キャプションとすべての行を決定論的なテキストとして受信します:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"title":"パイプラインレポート","blocks":[{"type":"table","caption":"進行中のパイプライン","headers":["アカウント","ステージ","ARR"],"rows":[["Acme","Won",125000],["Globex","Review",82000]],"rowHeaderColumnIndex":0}]}'
```

Telegram Mini App ボタンは `webApp` を使用し（レガシー
JSON 用の `web_app` も引き続き解析されます）、ユーザーとボット間のプライベートチャットでのみレンダリングされます:

```bash
openclaw message send --channel telegram --target 123456789 --message "アプリを開く:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"起動","webApp":{"url":"https://example.com/app"}}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --presentation '{"title":"ステータス更新","blocks":[{"type":"text","text":"ビルドが完了しました"}]}'
```

### 投票

```bash
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "軽食は？" \
  --poll-option ピザ --poll-option 寿司 \
  --poll-multi --poll-duration-hours 48
```

- `--poll-option <choice>`: 2～12回繰り返します。
- `--poll-multi`: 複数選択を許可します。
- Discord: `--poll-duration-hours`、`--silent`、`--message`。
- Telegram: `--poll-duration-seconds <n>`（5～600）、`--silent`、
  `--poll-anonymous` / `--poll-public`、`--thread-id`。

```bash
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "昼食は？" \
  --poll-option ピザ --poll-option 寿司 \
  --poll-duration-seconds 120 --silent
```

```bash
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "昼食は？" \
  --poll-option ピザ --poll-option 寿司
```

### スレッド

- `thread create`: 対応チャンネルは Discord。必須: `--thread-name`、`--target`
  （チャンネル ID）。任意: `--message-id`、`--message`、`--auto-archive-min`。
- `thread list`: 対応チャンネルは Discord。必須: `--guild-id`。任意:
  `--channel-id`、`--include-archived`、`--before`、`--limit`。
- `thread reply`: 対応チャンネルは Discord。必須: `--target`（スレッド ID）、
  `--message`。任意: `--media`、`--reply-to`。

### 絵文字

- `emoji list`: Discord（`--guild-id`）、Slack（追加フラグなし）。
- `emoji upload`: Discord。必須: `--guild-id`、`--emoji-name`、`--media`。
  任意: `--role-ids`（繰り返し）。

### ステッカー

- `sticker send`: Discord。必須: `--target`、`--sticker-id`（繰り返し）。
  任意: `--message`。
- `sticker upload`: Discord。必須: `--guild-id`、`--sticker-name`、
  `--sticker-desc`、`--sticker-tags`、`--media`。

### ロール、チャンネル、ボイス、イベント（Discord）

- `role info`: `--guild-id`。
- `role add` / `role remove`: `--guild-id`、`--user-id`、`--role-id`。
- `channel info`: `--target`。
- `channel list`: `--guild-id`。
- `voice status`: `--guild-id`、`--user-id`。
- `event list`: `--guild-id`。
- `event create`: 必須 `--guild-id`、`--event-name`、`--start-time`。
  任意 `--end-time`、`--desc`、`--channel-id`、`--location`、
  `--event-type`、`--image <url-or-path>`。

### モデレーション（Discord）

- `timeout`: `--guild-id`、`--user-id`。任意で `--duration-min` または
  `--until`（タイムアウトを解除する場合は両方とも省略）、`--reason`。
- `kick`: `--guild-id`、`--user-id`、`--reason`。
- `ban`: `--guild-id`、`--user-id`、`--delete-days`、`--reason`。

### ブロードキャスト

```bash
openclaw message broadcast --targets <target...> [--channel all] [--message <text>] [--media <url>] [--dry-run]
```

1つのペイロードを複数のターゲットに送信します。`--targets` は、スペース区切りの
リストを受け取ります。設定済みのすべてのプロバイダーを対象にするには、`--channel all` を使用します。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [エージェント送信](/ja-JP/tools/agent-send)
- [メッセージプレゼンテーション](/ja-JP/plugins/message-presentation)
