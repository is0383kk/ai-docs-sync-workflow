---
read_when:
    - メディアパイプラインまたは添付ファイルの変更
summary: 送信、Gateway、エージェント応答における画像とメディアの処理ルール
title: 画像とメディアのサポート
x-i18n:
    generated_at: "2026-07-26T09:39:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

WhatsApp チャンネルは Baileys Web 上で動作します。このページでは、送信、Gateway、エージェントの返信におけるメディア処理ルールについて説明します。

## 目標

- `openclaw message send --media` を使用して、任意のキャプション付きでメディアを送信する。
- Web 受信トレイからの自動返信に、テキストとともにメディアを含められるようにする。
- 種類ごとの制限を妥当かつ予測可能に保つ。

## CLI サーフェス

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — メディア（画像、音声、動画、ドキュメント）を添付します。ローカルパスまたは URL を指定できます。省略可能です。メディアのみを送信する場合、キャプションは空でもかまいません。
- `--gif-playback` — 動画メディアを GIF として再生します（WhatsApp のみ）。
- `--force-document` — チャンネルによる圧縮を避けるため、メディアをドキュメントとして送信します（Telegram、WhatsApp）。画像、GIF、動画に適用されます。
- `--reply-to <id>`、`--thread-id <id>`、`--pin`、`--silent` — テキストのみの送信と共通の配信およびスレッド化オプションです。
- `--dry-run` — 解決済みのペイロードを表示し、送信をスキップします。
- `--json` — 結果を JSON 形式で表示します：`{ action, channel, dryRun, handledBy, messageId?, payload }`（`payload` には、メディア参照を含むチャンネル固有の送信結果が格納されます）。

## WhatsApp Web チャンネルの動作

- 入力：ローカルファイルパス、または HTTP(S) URL。
- 処理フロー：バッファーに読み込み、メディアの種類を検出してから、種類ごとに送信用ペイロードを構築します。
  - **画像：** `channels.whatsapp.mediaMaxMb`（デフォルト 50MB）未満に収まるよう最適化されます。不透明な画像は JPEG に再圧縮されます（デフォルトの辺長候補は 2048px から始まり、サイズ超過が繰り返されるたびに短くなります）。透過を含む画像は PNG のまま保持されます。ソースがサイズと辺長の上限内に収まる適切な JPEG、PNG、WebP である場合は、再圧縮せず、元のバイト列がそのまま保持されます。アニメーション GIF は再エンコードされず、サイズチェックのみ行われます。
  - **音声／ボイス：** すでにネイティブのボイス音声（`.ogg`/`.opus` または `audio/ogg`/`audio/opus`）でない限り、送信前に `ffmpeg` を介して Opus/OGG（48kHz モノラル、64kbps、最大 20 分）へトランスコードされ、ボイスメモ（`ptt: true`）として送信されます。
  - **動画：** 16MB までは変換せずそのまま送信されます。
  - **ドキュメント：** その他すべてが対象で、上限は 100MB です。ファイル名が取得できる場合は保持されます。
- WhatsApp の GIF 風再生：モバイルクライアントでインラインループ再生されるよう、`gifPlayback: true`（CLI：`--gif-playback`）を指定して MP4 を送信します。
- MIME 検出では、マジックバイトによる判定、ファイル拡張子、レスポンスヘッダーの順に優先されます。判定された汎用コンテナ（`application/octet-stream`、`zip`）が、より具体的な拡張子のマッピング（XLSX と ZIP など）を上書きすることはありません。
- キャプションは `--message` または `reply.text` から取得されます。空のキャプションも許可されます。
- ログ：非詳細モードでは `↩️`/`✅` が表示され、詳細モードではサイズとソースのパス／URL も表示されます。

<Note>
上記の音声／動画の 16MB とドキュメントの 100MB は、明示的なバイト上限が渡されなかった場合に使用される、種類ごとの共通デフォルト値です。WhatsApp の送信では、`channels.whatsapp.mediaMaxMb`（デフォルト 50MB）から明示的な上限が設定され、そのアカウントではすべての種類に一律で適用されます。
</Note>

## 自動返信パイプライン

- `getReplyFromConfig` は、`text?`、`mediaUrl?`、`mediaUrls?` などのフィールドを含む返信ペイロード（またはペイロードの配列）を返します。
- メディアが存在する場合、Web 送信側は `openclaw message send` と同じパイプラインを使用してローカルパスまたは URL を解決します。
- 複数のメディア項目が指定されている場合は、順番に送信されます。

## 受信メディアからコマンドへの引き渡し

- Web からの受信メッセージにメディアが含まれている場合、OpenClaw はそれを一時ファイルにダウンロードし、次のテンプレート変数を公開します。
  - `{{AttachmentUrl}}` — 現在の添付ファイルの元の URL またはプロバイダー参照。
  - `{{AttachmentPath}}` — コマンド実行前に書き込まれたローカル一時パス。
  - `{{AttachmentContentType}}` — MIME コンテンツタイプ。
  - `{{AttachmentDir}}` — ローカルパスを含むディレクトリ。
  - `{{AttachmentIndex}}` — 0 始まりのソースファクトインデックス。
- セッションごとの Docker サンドボックスが有効な場合、受信メディアはサンドボックスのワークスペースにコピーされ、添付ファイルのパス／参照は `media/inbound/<filename>` のようなサンドボックス相対パスに書き換えられます。
- `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}`、`{{MediaDir}}` は、Plugin SDK の移行期間中、非推奨の互換性エイリアスとして維持されます。
- メディア理解（`tools.media.*` または共有の `tools.media.models` で設定）はテンプレート処理の前に実行され、`[Image]`、`[Audio]`、`[Video]` ブロックを `Body` に挿入できます。
  - 音声では `{{Transcript}}` が設定され、コマンド解析に文字起こしが使用されるため、スラッシュコマンドも引き続き機能します。
  - 動画と画像の説明では、コマンド解析用にキャプションテキストが保持されます。
  - アクティブなプライマリモデルがすでにネイティブで画像認識をサポートしている場合、OpenClaw は `[Image]` 要約ブロックをスキップし、代わりに元の画像をモデルへ渡します。
- デフォルトでは、最初に一致した画像、音声、動画の添付ファイルだけが処理されます。複数の添付ファイルを選択するには `tools.media.<capability>.attachments` を使用します。

## 制限とエラー

**送信時の上限（WhatsApp Web 送信）**

- 画像：最適化後に `channels.whatsapp.mediaMaxMb`（デフォルト 50MB）まで。
- 音声／動画：上限 16MB（共通デフォルト。WhatsApp 経由で送信する場合は `mediaMaxMb` で上書きされます）。
- ドキュメント：上限 100MB（共通デフォルト。WhatsApp 経由で送信する場合は `mediaMaxMb` で上書きされます）。
- サイズ超過または読み取り不能なメディアについては、ログに明確なエラーが記録され、返信はスキップされます。

**メディア理解の上限（文字起こし／説明）**

- 画像のデフォルト：10MB（`tools.media.image.maxBytes`、または各
  `tools.media.models[]` エントリの `maxBytes` で上書きできます）。
- 音声のデフォルト：20MB（`tools.media.audio.maxBytes`、またはエントリごとに上書きできます）。
- 動画のデフォルト：50MB（`tools.media.video.maxBytes`、またはエントリごとに上書きできます）。
- サイズ超過のメディアでは理解処理がスキップされますが、元の本文を含む返信は引き続き実行されます。

## テストに関する注意事項

- 画像、音声、ドキュメントの各ケースについて、送信と返信のフローを網羅します。
- 画像最適化後のサイズ上限と、音声のボイスメモフラグを検証します。
- 複数メディアの返信が順次送信として展開されることを確認します。

## 関連項目

- [カメラ撮影](/ja-JP/nodes/camera)
- [メディア理解](/ja-JP/nodes/media-understanding)
- [音声とボイスメモ](/ja-JP/nodes/audio)
