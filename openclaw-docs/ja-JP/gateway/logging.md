---
read_when:
    - ログ出力または形式の変更
    - CLI または Gateway の出力のデバッグ
summary: ログ出力先、ファイルログ、WS ログスタイル、コンソール書式設定
title: Gateway のログ記録
x-i18n:
    generated_at: "2026-07-26T09:02:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# ロギング

ユーザー向けの概要（CLI + Control UI + 設定）については、[/logging](/ja-JP/logging)を参照してください。

OpenClaw には 2 つのログ出力先があります。

- **コンソール出力** - ターミナル / Debug UI に表示される内容。
- **ファイルログ** - Gateway ロガーによって書き込まれる JSON 行。

起動時に、Gateway は解決済みのデフォルトエージェントモデルと、新しいセッションに影響するモードのデフォルト値をログに記録します。

```text
エージェントモデル: openai/gpt-5.6-sol (thinking=medium, fast=on)
```

`thinking` はデフォルトエージェント、モデルパラメーター、またはグローバルなエージェントのデフォルト値から取得されます。未設定の場合は `medium` と表示されます。`fast` はデフォルトエージェント、またはモデルの `fastMode` パラメーターから取得されます。

## ファイルベースのロガー

- デフォルトのローテーションログファイルは `/tmp/openclaw/` 配下にあり（1 日につき 1 ファイル）、Gateway ホストのローカルタイムゾーンに基づいて日付が付けられます。デフォルトプロファイルは `openclaw-YYYY-MM-DD.log` を使用し、名前付きプロファイルは `openclaw-<profile>-YYYY-MM-DD.log`（例: `openclaw-dev-YYYY-MM-DD.log`）を使用します。そのディレクトリが安全でないか書き込み不能な場合（所有者が不正、誰でも書き込み可能、シンボリックリンク）、OpenClaw は代わりにユーザースコープの `os.tmpdir()/openclaw-<uid>` パスへフォールバックします。Windows では常に、その OS 一時ディレクトリへのフォールバックを使用します。
- アクティブなログファイルは `logging.maxFileBytes`（デフォルト: 100 MB）でローテーションされ、番号付きアーカイブを最大 5 個（`.1` から `.5`）保持し、新しいアクティブファイルへの書き込みを続けます。
- ログファイルのパスとレベルは `~/.openclaw/openclaw.json` で設定します: `logging.file`、`logging.level`。
- ファイル形式は、1 行につき 1 つの JSON オブジェクトです。

トーク、リアルタイム音声、および管理対象ルームのコードパスは、運用デバッグと OTLP ログエクスポートを目的とした、範囲を限定したライフサイクル記録に共有ファイルロガーを使用します。トランスクリプトのテキスト、音声ペイロード、ターン ID、通話 ID、およびプロバイダー項目 ID がログレコードにコピーされることはありません。

Control UI の Logs タブは、Gateway（`logs.tail`）経由でこのファイルを追尾します。CLI も同様です。

```bash
openclaw logs --follow
```

### 詳細出力とログレベル

- **ファイルログ**は `logging.level` のみによって制御されます。
- `--verbose` は**コンソールの詳細度**（および WS ログの形式）にのみ影響し、ファイルログレベルは引き上げません。
- 詳細出力でのみ得られる情報をファイルログに記録するには、`logging.level` を `debug` または `trace` に設定します。
- トレースログには、Plugin ツールファクトリの準備など、選択されたホットパスの診断タイミング概要も含まれます。[/tools/plugin#slow-plugin-tool-setup](/ja-JP/tools/plugin#slow-plugin-tool-setup)を参照してください。

## コンソールのキャプチャ

CLI は `console.log/info/warn/error/debug/trace` をキャプチャしてファイルログに書き込み、同時に stdout/stderr にも出力します。

コンソールの詳細度は個別に調整できます。

- `logging.consoleLevel`（デフォルト `info`）
- `logging.consoleStyle`（`pretty` | `compact` | `json`。デフォルトは TTY 上では `pretty`、それ以外では `compact`）

## マスキング

OpenClaw は、ログまたはトランスクリプト出力がプロセス外へ出る前に、機密トークンをマスキングします。このマスキングポリシーは、コンソール、ファイルログ、OTLP ログレコード、およびセッショントランスクリプトのテキスト出力先に適用されるため、一致したシークレット値は、JSONL 行またはメッセージがディスクに書き込まれる前にマスキングされます。

- 機密値のマスキングは常に有効です。
- `logging.redactPatterns`: 正規表現文字列の配列（デフォルトを上書き）
  - 生の正規表現文字列（自動 `gi`）、またはカスタムフラグ用の `/pattern/flags` を使用します。
  - 一致箇所は、先頭 6 文字と末尾 4 文字を残してマスキングされます（18 文字以上の値）。それより短い値は `***` になります。
  - デフォルトでは、一般的なキー代入、CLI フラグ、JSON フィールド、Bearer ヘッダー、PEM ブロック、主要ベンダーのトークンプレフィックス、および決済認証情報のフィールド名（カード番号、CVC/CVV、共有決済トークン、決済認証情報）を対象とします。

Control UI のツール呼び出しイベント、`sessions_history` の出力、診断エクスポート、プロバイダーエラー、exec 承認表示、Gateway WebSocket ログなどの安全境界では、常にマスキングが行われます。`logging.redactPatterns` を使用すると、デプロイ固有のパターンを追加できます。

## Gateway WebSocket ログ

Gateway は WebSocket プロトコルログを 2 つのモードで出力します。

- **通常モード（`--verbose` なし）**: 「注目すべき」RPC 結果のみを出力します。エラー（`ok=false`）、遅い呼び出し（デフォルトしきい値: `>= 50ms`）、および解析エラーです。
- **詳細モード（`--verbose`）**: すべての WS リクエスト/レスポンストラフィックを出力します。

### WS ログ形式

`openclaw gateway` は Gateway ごとの形式切り替えをサポートします。

- `--ws-log auto`（デフォルト）: 通常モードは最適化され、詳細モードではコンパクトな出力を使用します。
- `--ws-log compact`: 詳細モードでのコンパクトな出力（リクエスト/レスポンスのペア）。
- `--ws-log full`: 詳細モードでのフレームごとの完全な出力。
- `--compact`: `--ws-log compact` のエイリアス。

```bash
# 最適化（エラー/遅い呼び出しのみ）
openclaw gateway

# すべての WS トラフィックを表示（ペア形式）
openclaw gateway --verbose --ws-log compact

# すべての WS トラフィックを表示（完全なメタデータ）
openclaw gateway --verbose --ws-log full
```

## コンソールの書式設定（サブシステムロギング）

コンソールフォーマッターは **TTY を認識**し、一貫したプレフィックス付きの行を出力します。サブシステムロガーにより、出力はグループ化され、確認しやすく保たれます。

- すべての行に付く**サブシステムプレフィックス**（例: `[gateway]`、`[canvas]`、`[tailscale]`）。
- **サブシステムの色**（名前からハッシュ化され、サブシステムごとに固定）とレベル別の色。
- **出力先が TTY の場合**、または環境がリッチターミナルのように見える場合（`TERM`/`COLORTERM`/`TERM_PROGRAM`）に色を使用します。`NO_COLOR` と `FORCE_COLOR` が尊重されます。
- **短縮されたサブシステムプレフィックス**: 先頭の `gateway/`、`channels/`、または `providers/` セグメントを削除し、残りのセグメントのうち末尾最大 2 つを保持します（例: `channels/turn/kernel` は `turn/kernel` と表示されます）。既知のチャンネルサブシステム（`telegram`、`whatsapp`、`slack` など）は、常にチャンネル名だけに短縮されます。
- **サブシステムごとのサブロガー**（自動プレフィックス + 構造化フィールド `{ subsystem }`）。
- QR/UX 出力用の **`logRaw()`**（プレフィックスなし、書式設定なし）。
- **コンソール形式**: `pretty` | `compact` | `json`。
- **コンソールログレベル**はファイルログレベルとは別です（`logging.level` が `debug`/`trace` の場合も、ファイルには完全な詳細が保持されます）。
- **WhatsApp のメッセージ本文**は `debug` レベルでログに記録されます（表示するには `--verbose` を使用します）。

これにより、ファイルログの安定性を維持しながら、対話型出力を確認しやすくできます。

## 関連項目

- [ロギング](/ja-JP/logging)
- [OpenTelemetry エクスポート](/ja-JP/gateway/opentelemetry)
- [診断エクスポート](/ja-JP/gateway/diagnostics)
