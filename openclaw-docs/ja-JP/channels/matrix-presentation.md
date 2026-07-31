---
read_when:
    - OpenClaw のリッチレスポンスをレンダリングする Matrix クライアントの構築
    - com.openclaw.presentation イベント内容のデバッグ
summary: OpenClaw 対応クライアント向け Matrix MessagePresentation メタデータ
title: Matrix プレゼンテーションメタデータ
x-i18n:
    generated_at: "2026-07-26T09:13:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw は、送信 Matrix `m.room.message` イベントの `com.openclaw.presentation` コンテンツキー配下に、正規化された `MessagePresentation` メタデータを付加します。

標準の Matrix クライアントは、プレーンテキストの `body` を引き続きレンダリングします。OpenClaw 対応クライアントは構造化メタデータを読み取り、ボタン、選択コントロール、コンテキスト行、区切り線などのネイティブ UI をレンダリングできます。

## イベント内容

```json
{
  "msgtype": "m.text",
  "body": "モデルを選択\n\nモデルを選択してください:\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "モデルを選択",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "モデルを選択",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` はメタデータのスキーマバージョンで、現在のバージョンは `1` です。`type` は安定した識別子で、常に `"message.presentation"` です。Matrix アダプターは、このバージョンとタイプに完全に一致するペイロードのみを送信します。同様にクライアントも、安全に解釈できない未知のバージョン、未知の `type` 値、未知のブロックタイプを無視する必要があります。
- `title` と `tone`（`info`、`success`、`warning`、`danger`、`neutral`）は任意のヒントです。
- ボタンと選択肢には、従来の文字列 `value` と併せて、型付きの `action`（`{ "type": "command", "command": "/..." }` または `{ "type": "callback", "value": "..." }`）を指定できます。両方が存在する場合は `action` を優先してください。

## フォールバック動作

OpenClaw は常に、読みやすいプレーンテキストのフォールバックを `body` にレンダリングします。構造化メタデータは付加的なものであり、基本的な Matrix の相互運用性に必須としてはなりません。

フォールバックのレンダリング規則:

- `title`、`text`、`context` のコンテンツはプレーンテキストの行としてレンダリングされます。
- `command` アクションを持つボタンは、コマンドをコピーできるように ``label: `/command` `` としてレンダリングされます。`callback` アクションを持つボタン、または従来の `value` のみを持つボタンは、不透明なコールバック値を非公開に保つため、ラベルのみでレンダリングされます。無効なボタンも常にラベルのみでレンダリングされます。URL ボタンとウェブアプリボタンは `label: URL` としてレンダリングされます。
- 選択ブロックは、プレースホルダー（または `Options:`）を見出しとして、その後にラベルのみの選択肢の行を並べてレンダリングします。
- 区切り線のみの表示など、何もレンダリングされない場合、本文は `---` にフォールバックします。

未対応のクライアントでは、引き続きフォールバックテキストが表示されます。OpenClaw 対応クライアントは、コピー、検索、通知、アクセシビリティのためにフォールバックを維持しながら、表示には構造化メタデータを優先できます。

## 対応ブロック

Matrix 送信アダプターは、以下のネイティブ対応を通知します:

- `buttons`
- `select`
- `context`
- `divider`

`text` ブロックは、フォールバック本文を通じて常にサポートされます。すべてのブロックをベストエフォートの表示ヒントとして扱い、メッセージ全体を失敗させるのではなく、未知のフィールドとブロックタイプを無視してください。

## 操作

このメタデータは、Matrix のコールバックセマンティクスを追加するものではありません。ボタンと選択肢の値はフォールバック用の操作ペイロードで、通常はスラッシュコマンドまたはテキストコマンドです。操作に対応する Matrix クライアントは、コントロールの値（`action.command`、次に `action.value`、その次に `value`）を解決し、通常のメッセージとしてルームに送り返します。

たとえば、値が `/model deepseek/deepseek-chat` のボタンは、その値を同じルームに暗号化された Matrix テキストメッセージとして送信することで処理できます。

## 承認メタデータとの関係

`com.openclaw.presentation` は、一般的なリッチメッセージ表示用です。

承認プロンプトでは、承認に安全性に関わる状態、判断、実行・Plugin の詳細が含まれるため、専用の `com.openclaw.approval` メタデータを使用します。同じイベントに両方のメタデータキーが存在する場合、クライアントは専用の承認レンダラーを優先する必要があります。

## メディアメッセージ

返信に複数のメディア URL が含まれる場合、OpenClaw はメディア URL ごとに 1 つの Matrix イベントを送信します。クライアントがレンダラーの重複なく 1 つの安定した構造化ペイロードを取得できるように、キャプションテキストと表示メタデータは最初のイベントにのみ付加されます。長いテキストが複数のイベントに分割される場合も同じ規則が適用され、メタデータは最初のイベントにのみ付加されます。

表示メタデータはコンパクトに保ってください。ユーザーに表示される長いテキストは `body` に保持し、通常の Matrix テキスト分割処理を使用する必要があります。
