---
read_when:
    - 入力中インジケーターの動作またはデフォルトの変更
summary: OpenClaw が入力中インジケーターを表示するタイミングとその調整方法
title: 入力中インジケーター
x-i18n:
    generated_at: "2026-07-26T09:40:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

実行中は、チャットチャンネルに入力中インジケーターが送信されます。入力中表示を**いつ**開始するかは `agents.defaults.typingMode` で、**どの頻度で**更新するかは `typingIntervalSeconds` で制御します（キープアライブ間隔、デフォルトは 6 秒）。

## デフォルト

`agents.defaults.typingMode` が**未設定**の場合：

- **ダイレクトチャット**：モデルループが開始すると、入力中表示が即座に開始されます。
- **メンションありのグループチャット**：入力中表示が即座に開始されます。
- **メンションなしのグループチャット**：許可された実行で、ハーネスの実行アクティビティやメッセージテキストなど、ユーザーに見えるアクティビティが発生すると入力中表示が開始されます。
- **Heartbeat 実行**：解決された Heartbeat の送信先が入力中表示に対応するチャットであり、入力中表示が無効化されていない場合、Heartbeat 実行の開始時に入力中表示が開始されます。

## モード

`agents.defaults.typingMode` を次のいずれかに設定します：

- `never` - 入力中インジケーターを一切表示しません。
- `instant` - 後で実行がサイレント応答トークンのみを返す場合でも、**モデルループが開始するとすぐに**入力中表示を開始します。
- `thinking` - **最初の推論差分**、またはターンが受理された後のアクティブなハーネス実行時に入力中表示を開始します。
- `message` - アクティブなハーネス実行やサイレントではないテキスト差分など、**ユーザーに見える最初の応答アクティビティ**で入力中表示を開始します。`NO_REPLY` などのサイレント応答トークンは、テキストアクティビティとしてカウントされません。

「どの程度早く作動するか」の順序：`never` -> `message`/`thinking` -> `instant`。

## 設定

エージェントレベルのデフォルトを設定します：

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

1 つのエージェントについてポリシーを上書きします：

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## 注意事項

- `message` モードはサイレント応答トークンでは開始されませんが、アクティブな実行により、アシスタントのテキストが利用可能になる前でも入力中表示が行われることがあります。
- `thinking` はストリーミングされた推論（`reasoningLevel: "stream"`）にも反応し、推論差分が到着する前にアクティブな実行によって開始されることもあります。
- Heartbeat の入力中表示は、解決された送信先に対する稼働中シグナルです。`message` または `thinking` のストリームタイミングに従うのではなく、Heartbeat 実行の開始時に開始されます。無効にするには `typingMode: "never"` を設定します。
- Heartbeat の送信先が `"none"` の場合、送信先を解決できない場合、Heartbeat のチャット送信が無効な場合、またはチャンネルが入力中表示をサポートしていない場合、Heartbeat は入力中表示を行いません。
- `agents.defaults.typingIntervalSeconds` は、開始時刻ではなく、すべてのエージェントの**更新間隔**を制御します。デフォルト：6 秒。

## 関連項目

<CardGroup cols={2}>
  <Card title="プレゼンス" href="/ja-JP/concepts/presence" icon="signal">
    Gateway が Control UI の Devices ページと macOS の Instances タブのために、接続中のクライアントを追跡する仕組み。
  </Card>
  <Card title="ストリーミングとチャンク分割" href="/ja-JP/concepts/streaming" icon="bars-staggered">
    送信ストリーミングの動作、チャンク境界、チャンネル固有の配信。
  </Card>
</CardGroup>
