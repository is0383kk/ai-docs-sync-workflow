---
read_when:
    - グローバルなログレベルを上げずに、対象を絞ったデバッグログが必要な場合
    - サポートのためにサブシステム固有のログを収集する必要があります
summary: 対象を絞ったデバッグログ用の診断フラグ
title: 診断フラグ
x-i18n:
    generated_at: "2026-07-26T09:01:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

診断フラグを使用すると、`logging.level` をグローバルに引き上げることなく、1 つのサブシステムについて追加のログ記録を有効にできます。サブシステム側でフラグを確認しない限り、そのフラグは何の効果もありません。

## 仕組み

- フラグは大文字と小文字を区別しない文字列です。設定内の `diagnostics.flags` と、環境変数による `OPENCLAW_DIAGNOSTICS` のオーバーライドから解決され、重複が除去されて小文字に変換されます。
- `name.*` は `name` 自体と、`name.` 配下のすべてに一致します（たとえば、`telegram.*` は `telegram.http` に一致します）。
- `*` または `all` を指定すると、すべてのフラグが有効になります。
- 設定内の `diagnostics.flags` を変更した後は Gateway を再起動してください。ホットリロードはされません。

## 既知のフラグ

| フラグ                  | 有効になる機能                                                   |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Telegram Bot API の HTTP エラーログ                       |
| `brave.http`          | Brave Search のリクエスト、レスポンス、キャッシュのログ               |
| `profiler`            | 応答ステージのプロファイラーと Codex app-server のプロファイラー（両方） |
| `reply.profiler`      | 応答ステージのプロファイラーのみ                                 |
| `codex.profiler`      | Codex app-server のプロファイラーのみ                            |
| `health`              | Gateway のヘルスプローブ、アカウント、バインディングのデバッグ詳細        |
| `ingress.timing`      | セッションの読み込み、モデル選択、モデルカタログの所要時間  |
| `plugin.load-profile` | 同期的な Plugin モジュール読み込みの所要時間                    |
| `timeline`            | 構造化 JSONL タイムラインアーティファクト（以下を参照）            |

## 設定で有効にする

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

複数のフラグ：

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## 環境変数によるオーバーライド（単発）

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

値はカンマまたは空白で分割されます。特殊な値：

| 値                       | 効果                                   |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | 設定によるものも含め、すべてのフラグを無効化 |
| `1`, `true`, `all`, `*`     | すべてのフラグを有効化                        |

`OPENCLAW_DIAGNOSTICS=0` は、そのプロセスについて環境変数と設定の両方からのフラグを無効にします。ファイルを編集せずに、設定で有効なままになっているプロファイラーフラグを一時的に無効化する場合に便利です。

## プロファイラーフラグ

プロファイラーフラグは軽量なタイミングスパンを制御します。無効な場合、オーバーヘッドは発生しません。

Gateway の 1 回の実行について、プロファイラーで制御されるすべてのスパンを有効にします：

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

応答ディスパッチのプロファイラースパンのみを有効にします：

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

Codex app-server の起動、ツール、スレッドのプロファイラースパンのみを有効にします：

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler` は応答プロファイラーと Codex プロファイラーの両方を有効にします。一方のみを有効にするには、スコープ付きのフラグ名を使用してください。

または、設定で指定します：

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

設定フラグを変更した後は Gateway を再起動してください。プロファイラーフラグを無効にするには、`diagnostics.flags` から削除して再起動するか、`OPENCLAW_DIAGNOSTICS=0` を指定してプロセスを起動し、その実行についてすべての診断フラグをオーバーライドします。

## タイムラインアーティファクト

`timeline` フラグ（エイリアス：`diagnostics.timeline`）は、外部の QA ハーネス向けに、構造化された起動時および実行時のタイミングイベントを JSONL として書き込みます：

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

または、設定で有効にします：

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

出力パスは、フラグ自体が設定で指定されている場合でも、常に `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` から取得されます。パス用の設定キーはありません。`timeline` を設定のみで有効にした場合、OpenClaw がまだ設定を読み込んでいないため、最初期の設定読み込みスパンは記録されません。それ以降の起動スパンは通常どおり記録されます。

`OPENCLAW_DIAGNOSTICS=1`、`=all`、`=*` も、すべてのフラグを有効にするため、タイムラインを有効にします。JSONL アーティファクトのみが必要で、ほかのすべての診断フラグは不要な場合、スコープ付きの `timeline` フラグを使用してください。

タイムライン内のイベントループ遅延サンプルには、`timeline` に加えて、もう 1 つオプトインが必要です。タイムラインを有効にしたうえで、`OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1`（または `on`/`true`/`yes`）を設定してください。

タイムラインレコードは `openclaw.diagnostics.v1` エンベロープを使用し、プロセス ID、フェーズ名、スパン名、所要時間、Plugin ID、依存関係数、イベントループ遅延サンプル、プロバイダー操作名、子プロセスの終了状態、起動エラーの名前やメッセージを含む場合があります。タイムラインファイルはローカルの診断アーティファクトとして扱い、使用中のマシン外部へ共有する前に内容を確認してください。

## ログの出力先

フラグによるログは、標準の診断ログファイルに出力されます。デフォルト：

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

名前付きプロファイルでは `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` が使用されます。たとえば、`--dev` では `openclaw-dev-YYYY-MM-DD.log` が使用されます。

`logging.file` を設定した場合は、代わりにそのパスが使用されます。ログは JSONL 形式です（1 行につき 1 つの JSON オブジェクト）。`logging.redactSensitive` に基づく秘匿化も引き続き適用されます。ログパスの完全な解決方法、ローテーション、秘匿化モデルについては、[ログ](/ja-JP/logging)を参照してください。

## ログを抽出する

アクティブなプロファイルの最新ログファイルを読み取ります：

```bash
openclaw logs --plain
# 名前付きプロファイルの例：
openclaw --profile work logs --plain
```

Telegram の HTTP 診断を絞り込みます：

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

Brave Search の HTTP 診断を絞り込みます：

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

または、問題を再現しながら追跡します：

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

リモート Gateway の場合は、代わりに `openclaw logs --follow` を使用してください（[/cli/logs](/ja-JP/cli/logs)を参照）。

## 注意事項

- `logging.level` が `warn` より高く設定されている場合、フラグで制御されるログが抑制されることがあります。デフォルトの `info` で問題ありません。
- `brave.http` は、Brave Search のリクエスト URL とクエリパラメーター、レスポンスのステータスと所要時間、キャッシュのヒット、ミス、書き込みイベントをログに記録します。API キー（リクエストヘッダーとして送信）やレスポンス本文は記録しませんが、検索クエリには機密情報が含まれる可能性があります。
- フラグは有効のままでも安全です。特定のサブシステムのログ量にのみ影響します。
- ログの出力先、レベル、秘匿化を変更するには、[/logging](/ja-JP/logging)を使用してください。

## 関連項目

- [Gateway の診断](/ja-JP/gateway/diagnostics)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)
