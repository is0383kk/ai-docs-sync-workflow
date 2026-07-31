---
read_when:
    - セルフホスト型 Synapse または Tuwunel 向け Matrix サイレントストリーミングの設定
    - ユーザーは、プレビューが編集されるたびではなく、ブロックが完了したときだけ通知を受け取ることを望んでいます
summary: 静かに確定されたプレビュー編集に対する受信者ごとの Matrix プッシュルール
title: 静かなプレビューのための Matrix プッシュルール
x-i18n:
    generated_at: "2026-07-26T09:27:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

`channels.matrix.streaming.mode` が `"quiet"` の場合、OpenClaw は単一のプレビューイベントをその場で編集して応答をストリーミングします。プレビューは通知を行わない `m.notice` イベントとして送信され、確定した編集には `content["com.openclaw.finalized_preview"] = true` が付けられます。Matrix クライアントは、ユーザーごとのプッシュルールがこのマーカーに一致する場合に限り、その最終編集時に通知します。このページは、Matrix をセルフホストし、受信者アカウントごとにそのルールをインストールする運用者向けです。

`streaming.mode: "progress"` は同じ経路で下書きを確定するため、同じルールが進捗モードで確定された編集にも適用されます。

Matrix の標準の通知動作のみを使用する場合は、`streaming.mode: "partial"` を使用するか、ストリーミングを無効のままにしてください。[Matrix チャンネルのセットアップ](/ja-JP/channels/matrix#streaming-previews)を参照してください。

## 前提条件

- 受信ユーザー = 通知を受け取るユーザー
- bot ユーザー = 応答を送信する OpenClaw Matrix アカウント
- 以下の API 呼び出しには受信ユーザーのアクセストークンを使用する
- プッシュルール内の `sender` を bot ユーザーの完全な MXID と照合する
- 受信アカウントには、動作するプッシャーがすでに必要です。静かなプレビュールールは、通常の Matrix プッシュ配信が正常な場合にのみ機能します

## 手順

<Steps>
  <Step title="静かなプレビューを設定する">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="受信者のアクセストークンを取得する">
    可能な場合は、既存のクライアントセッショントークンを再利用します。新しいトークンを発行するには、次のコマンドを実行します。

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="プッシャーが存在することを確認する">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

プッシャーが返されない場合は、続行する前にこのアカウントの通常の Matrix プッシュ配信を修正してください。

  </Step>

  <Step title="オーバーライドプッシュルールをインストールする">
    確定済みプレビューのマーカーと、送信者である bot の MXID に一致するルールをインストールします。

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    実行前に次の値を置き換えます。

    - `https://matrix.example.org`: ホームサーバーのベース URL
    - `$USER_ACCESS_TOKEN`: 受信ユーザーのアクセストークン
    - `openclaw-finalized-preview-botname`: 受信者ごと、bot ごとに一意のルール ID（パターン: `openclaw-finalized-preview-<botname>`）
    - `@bot:example.org`: 受信者ではなく、OpenClaw bot の MXID

  </Step>

  <Step title="確認する">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

次に、ストリーミング応答をテストします。quiet モードでは、ルームに通知を行わない下書きプレビューが表示され、ブロックまたはターンが完了すると通知が 1 回送信されます。

  </Step>
</Steps>

後でルールを削除するには、受信者のトークンを使用して、同じルール URL に `DELETE` を実行します。

## 複数 bot に関する注意事項

プッシュルールは `ruleId` をキーとします。同じ ID に対して `PUT` を再実行すると、単一のルールが更新されます。複数の OpenClaw bot から同じ受信者に通知する場合は、送信者の一致条件が異なるルールを bot ごとに 1 つ作成してください。

新しいユーザー定義の `override` ルールはサーバーのデフォルト抑制ルールより前に挿入されるため、追加の順序指定パラメーターは不要です。このルールは、その場で確定可能なテキストのみのプレビュー編集にだけ影響します。メディア応答、古いプレビューへのフォールバック、および Matrix のメンションを有効化する最終テキストは、代わりに通常の通知メッセージとして配信されます。

## ホームサーバーに関する注意事項

<AccordionGroup>
  <Accordion title="Synapse">
    特別な `homeserver.yaml` の変更は必要ありません。通常の Matrix 通知がすでにこのユーザーに届いている場合、主なセットアップ手順は、受信者のトークンを使用した上記の `pushrules` 呼び出しです。

    Synapse をリバースプロキシまたはワーカーの背後で実行している場合は、`/_matrix/client/.../pushrules/` が Synapse に正しく到達することを確認してください。プッシュ配信はメインプロセス、または `synapse.app.pusher` / 設定済みのプッシャーワーカーによって処理されます。これらが正常であることを確認してください。

    このルールは、2023 年に Synapse に追加された `event_property_is` プッシュルール条件（MSC3758、プッシュルール v1.10）を使用します。古い Synapse リリースでは `PUT pushrules/...` 呼び出しが受け入れられても、条件には暗黙的に一致しません。確定済みプレビューの編集時に通知が届かない場合は、Synapse をアップグレードしてください。

  </Accordion>

  <Accordion title="Tuwunel">
    Synapse と同じ手順です。確定済みプレビューのマーカーに Tuwunel 固有の設定は必要ありません。

    ユーザーが別のデバイスでアクティブな間に通知が届かなくなる場合は、`suppress_push_when_active` が有効になっているかどうかを確認してください。Tuwunel はこのオプションを 1.4.2（2025 年 9 月）で追加しました。このオプションは、1 台のデバイスがアクティブな間、他のデバイスへのプッシュを意図的に抑制できます。

  </Accordion>
</AccordionGroup>

## 関連項目

- [Matrix チャンネルのセットアップ](/ja-JP/channels/matrix)
- [ストリーミングの概念](/ja-JP/concepts/streaming)
