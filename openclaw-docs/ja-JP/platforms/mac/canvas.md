---
read_when:
    - macOS Canvas パネルの実装
    - ビジュアルワークスペースにエージェント制御を追加する
    - WKWebView の canvas 読み込みをデバッグする
summary: WKWebView とカスタム URL スキームを介して埋め込まれる、エージェント制御の Canvas パネル
title: キャンバス
x-i18n:
    generated_at: "2026-07-26T09:49:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 56532246bc06601aa753a59f85f33bfa8d6599deecade591a03972e8b9b16fc2
    source_path: platforms/mac/canvas.md
    workflow: 16
---

macOS アプリは、`WKWebView` を使用したエージェント制御の **Canvas パネル**を組み込んでいます。これは、
HTML/CSS/JS、A2UI、小規模なインタラクティブ UI
サーフェス向けの軽量ビジュアルワークスペースです。

## Canvas の保存場所

Canvas の状態は Application Support 配下に保存されます。

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

Canvas パネルは、カスタム URL スキーム
`openclaw-canvas://<session>/<path>` を介してこれらのファイルを提供します。

- `openclaw-canvas://main/` -> `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` -> `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` -> `<canvasRoot>/main/widgets/todo/index.html`

ルートに `index.html` が存在しない場合、アプリは組み込みのスキャフォールドページを表示します。

## パネルの動作

- メニューバー（またはマウスカーソル）の近くに固定される、枠なしでサイズ変更可能なパネル。
- Canvas を表示しても、アプリは切り替わらず、キーボードフォーカスも奪いません。
- セッションごとにサイズと位置を記憶します。
- ローカルの Canvas ファイルが変更されると自動的に再読み込みします。
- 一度に表示される Canvas パネルは 1 つだけです（必要に応じてセッションを切り替えます）。

Canvas は Settings -> **Allow Canvas** から無効にできます。無効にすると、
Canvas の Node コマンドは `CANVAS_DISABLED` を返します。

## エージェント API サーフェス

Canvas は Gateway WebSocket を介して公開されるため、エージェントは
パネルの表示と非表示、パスまたは URL への移動、JavaScript の評価、
スナップショット画像のキャプチャを実行できます。

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

`eval` と `a2ui.*` は、パネルを開いたり表示したりせずにコンテンツを更新します。表示するのは
`present`、`navigate`、またはユーザー操作だけです。非表示にした後も、コンテンツの更新は
非表示のパネルに引き続き適用されます。`snapshot` には表示中のパネルが必要で、
それ以外の場合は `CANVAS_HIDDEN` を返します。最初に `present` を実行してください。

`canvas.navigate` は、ローカル Canvas パス、`http(s)` URL、`file://`
URL を受け付けます。`"/"` を渡すと、ローカルのスキャフォールドまたは `index.html` が表示されます。

`/__openclaw__/canvas/` および
`/__openclaw__/a2ui/` 配下の Gateway ホスト対象は、Node セッションの現在のスコープ付き
Canvas URL を介して解決されます。アプリは移動前にその短時間有効なケイパビリティを更新します。
ケイパビリティ URL を自分で構築またはコピーする必要はありません。

## Canvas の A2UI

A2UI は Gateway Canvas ホストによってホストされ、Canvas
パネル内にレンダリングされます。Gateway が Canvas ホストを通知すると、macOS アプリは
初回表示時に A2UI ホストページへ自動的に移動します。

通知される URL はケイパビリティでスコープされます。例:
`http://<gateway-host>:18789/__openclaw__/cap/<token>/__openclaw__/a2ui/?platform=macos`。
安定したリンクではなく、一時的な認証情報として扱ってください。

### A2UI コマンド（v0.8）

Canvas は、A2UI v0.8 のサーバーからクライアントへのメッセージ `beginRendering`、
`surfaceUpdate`、`dataModelUpdate`、`deleteSurface` を受け付けます。`createSurface`（v0.9）は
まだサポートされていません。

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"Canvas（A2UI v0.8）"},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"これを読める場合、A2UI プッシュは機能しています。"},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

簡単なスモークテスト:

```bash
openclaw nodes canvas a2ui push --node <id> --text "A2UI からこんにちは"
```

## Canvas からエージェント実行をトリガーする

Canvas は `openclaw://agent?...` ディープリンクを介して、新しいエージェント実行をトリガーできます。

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

サポートされているクエリパラメーター:

| パラメーター                  | 意味                                               |
| -------------------------- | ----------------------------------------------------- |
| `message`                  | 事前入力されたエージェントプロンプト。                               |
| `sessionKey`               | 安定したセッション識別子。                            |
| `thinking`                 | 任意の思考プロファイル。                            |
| `deliver`, `to`, `channel` | 配信先。                                      |
| `timeoutSeconds`           | 任意の実行タイムアウト。                                 |
| `key`                      | 信頼されたローカル呼び出し元向けにアプリが生成した安全トークン。 |

有効なキーが指定されていない限り、アプリは確認を求めます。キーなしの
リンクは承認前にメッセージと URL を表示し、配信ルーティング
フィールドを無視します。キー付きリンクは通常の Gateway 実行パスを使用します。

## セキュリティに関する注意事項

- Canvas スキームはディレクトリトラバーサルをブロックします。ファイルはセッションルート配下に存在する必要があります。
- ローカル Canvas コンテンツはカスタムスキームを使用します（local loopback サーバーは不要です）。
- 外部の `http(s)` URL は、明示的に移動した場合にのみ許可されます。
- 通常のウェブページはレンダリング専用です。エージェント操作は、
  アプリ所有の Canvas スキーム、またはアプリが選択した、ケイパビリティでスコープされた正確な Gateway A2UI ドキュメントからのみ
  受け付けられます。サブフレーム、リダイレクト、期限切れのケイパビリティ、変更された
  クエリからは操作をディスパッチできません。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [WebChat](/ja-JP/web/webchat)
