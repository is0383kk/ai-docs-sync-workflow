---
read_when:
    - ターミナルからワークスペース内のファイルにある単一の末端要素を確認または編集したい場合
    - ワークスペースの状態をスクリプトで操作しており、種類に依存しない安定したアドレス指定方式が必要です
    - セルフホスト型 Gateway でオプションの `oc-path` Plugin を有効にするかどうかを判断しています
summary: 同梱 `oc-path` Plugin：`oc://` ワークスペースファイル指定スキーム用の `openclaw path` CLI を提供します
title: OC Path Plugin
x-i18n:
    generated_at: "2026-07-26T10:11:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb7bb1aacd37e5cc9c391372b871dc519f4048232d93a0016138ae00a6985a59
    source_path: plugins/oc-path.md
    workflow: 16
---

バンドルされている `oc-path` plugin は、`oc://` ワークスペースファイルアドレス指定スキーム用の [`openclaw path`](/ja-JP/cli/path) CLI を追加します。これは OpenClaw リポジトリの
`extensions/oc-path/` に同梱されていますが、オプトイン方式です。インストールまたはビルドしただけでは休止状態のままであり、有効化するまで動作しません。

`oc://` アドレスは、ワークスペースファイル内の単一のリーフ（またはワイルドカードで指定したリーフの集合）を指します。この plugin は、次の 4 種類のファイルを認識します。

- **markdown**（`.md`）：フロントマター、セクション、項目、フィールド
- **jsonc**（`.jsonc`、`.json`）：コメントと書式を保持
- **jsonl**（`.jsonl`、`.ndjson`）：行指向レコード
- **yaml**（`.yaml`、`.yml`、`.lobster`）：`yaml` パッケージの `Document` API を介したマップ／シーケンス／スカラーノード

セルフホスト環境やエディター拡張機能では、SDK に対するスクリプトを直接記述せずに単一のリーフを読み書きするために CLI を使用します。エージェントやフックでは、バイト忠実度を保つラウンドトリップとリダクションセンチネルガードをすべての種類に一貫して適用できる、決定論的な基盤として扱います。完全な文法、動詞ごとのフラグ一覧、ファイル種類別の実例については
[CLI リファレンス](/ja-JP/cli/path)を参照してください。このページでは、plugin を有効にする理由と方法について説明します。

## 有効にする理由

スクリプト、フック、またはローカルのエージェントツールが、ファイル形式ごとに専用パーサーを用意せずにワークスペース状態の特定箇所を指す必要がある場合は、`oc-path` を有効にします。単一の `oc://` アドレスで、Markdown のフロントマターキー、セクション項目、JSONC 設定のリーフ、JSONL イベントのフィールド、または YAML ワークフローのステップを指定できます。

これは、変更を小さく、監査可能で、再現可能に保つ必要があるメンテナーワークフローで重要です。1 つの値を調査し、一致するレコードを検索し、書き込みをドライランしてから、コメント、改行コード、周辺の書式には手を加えず、対象のリーフだけを適用できます。

有効にする一般的な理由は次のとおりです。

- **ローカル自動化**：シェルスクリプトでは、Markdown、JSONC、
  JSONL、YAML ごとに個別の解析コードを持たず、`openclaw path … --json` を使用してワークスペース内の 1 つの値を解決または更新できます。
- **エージェントから確認できる編集**：エージェントは、アドレス指定された 1 つのリーフについて、書き込む前にドライランの差分を表示します。これは自由形式でファイル全体を書き換えるよりもレビューしやすくなります。
- **エディター統合**：エディターは、見出しテキストから推測することなく、`oc://AGENTS.md/tools/gh` を正確な Markdown ノードと行番号に対応付けます。
- **診断**：`emit` はパーサーとエミッターを通してファイルをラウンドトリップするため、自動編集に依存する前に、そのファイル種類がバイト単位で安定しているかどうかを確認できます。

```bash
# この設定で GitHub plugin は有効になっているか？
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --json

# このセッションログにどのツール呼び出し名が含まれているか？
openclaw path find 'oc://session.jsonl/[event=tool_call]/name' --json

# この小さな設定編集では、どのバイトが書き込まれるか？
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

`oc-path` は、意図的に上位レベルのセマンティクスを所有しません。メモリへの書き込みは引き続きメモリ plugin が所有し、設定コマンドは引き続き設定全体の管理を所有し、最終正常状態（LKG）の設定復旧は引き続き復元／昇格を所有します。`oc-path` は、それらの上位レベルのツールが利用できる、範囲を限定したアドレス指定およびバイト保持ファイル操作レイヤーです。

## 実行場所

この plugin は、コマンドを実行したホスト上で **`openclaw` CLI のプロセス内**で実行されます。実行中の Gateway は不要で、ネットワークソケットも開きません。すべての動詞は、指定されたファイルに対する純粋な変換です。

Plugin のメタデータは `extensions/oc-path/openclaw.plugin.json` にあります。

```json
{
  "id": "oc-path",
  "name": "OC Path",
  "activation": {
    "onStartup": false,
    "onCommands": ["path"]
  },
  "commandAliases": [{ "name": "path", "kind": "cli" }]
}
```

`onStartup: false` により、この plugin は Gateway の起動パスから除外されます。
`commandAliases` と `activation.onCommands` は、`openclaw path …` を初めて実行したときに plugin を遅延読み込みするよう CLI に指示します。そのため、この動詞を一度も使用しないインストール環境ではコストが発生しません。

## 有効化

```bash
openclaw plugins enable oc-path
```

Gateway を実行している場合は、マニフェストのスナップショットに新しい状態が反映されるよう、Gateway を再起動します。同じホストでは、単独の `openclaw path` 呼び出しが直ちに機能します。CLI は必要に応じて plugin を読み込みます。

無効にするには、次を実行します。

```bash
openclaw plugins disable oc-path
```

## 依存関係

すべてのパーサー依存関係は plugin 内に限定されています。`oc-path` を有効にしても、コアランタイムに新しいパッケージは追加されません。

| 依存関係       | 用途                                                                   |
| -------------- | ---------------------------------------------------------------------- |
| `commander`    | `resolve`、`find`、`set`、`validate`、`emit` のサブコマンド接続。    |
| `jsonc-parser` | コメントと末尾のカンマを保持した JSONC の解析およびリーフ編集。     |
| `markdown-it`  | セクション／項目／フィールドモデル用の Markdown トークン化。            |
| `yaml`         | コメントとフロースタイルを保持した YAML `Document` の解析／出力／編集。 |

JSONL は独自実装のままです。行指向の解析はどの依存関係を使用するよりも単純であり、行ごとの解析はすでに `jsonc-parser` を経由しています。

## 提供される機能

| サーフェス                     | 提供元                                                  |
| ------------------------------ | ------------------------------------------------------- |
| `openclaw path` CLI            | `extensions/oc-path/cli-registration.ts`                |
| `oc://` パーサー／フォーマッター | `extensions/oc-path/src/oc-path/oc-path.ts`             |
| 種類ごとの解析／出力／編集     | `extensions/oc-path/src/oc-path/{md,jsonc,jsonl,yaml}`  |
| 汎用的な解決／検索／設定       | `extensions/oc-path/src/oc-path/{resolve,find,edit}.ts` |
| リダクションセンチネルガード   | `extensions/oc-path/src/oc-path/sentinel.ts`            |

現在、公開されているサーフェスは CLI のみです。基盤の動詞は plugin の非公開機能です。利用側は CLI を使用するか、SDK に対して独自の plugin を構築します。

## 他の plugin との関係

- **`memory-*`**：メモリへの書き込みは、`oc-path` ではなくメモリ plugin を経由します。`oc-path` は汎用的なファイル基盤であり、メモリ plugin はその上に独自のセマンティクスを重ねます。
- **LKG**：`path` は最終正常状態の設定復元を認識しません。`path` を通して編集したファイルが LKG の追跡対象でもある場合、次の設定監視サイクルで、そのファイルを昇格するか復旧するかが決まります。`path` による編集は、そのファイルに対する他の直接書き込みと同様に扱ってください。

## 安全性

`set` は、基盤の出力パスを通じて生バイトを書き込みます。この出力パスでは、リダクションセンチネルガードが自動的に適用されます。`__OPENCLAW_REDACTED__` をそのまま、または部分文字列として含むリーフは、書き込み時に `OC_EMIT_SENTINEL` で拒否されます。また CLI は、出力する人間向けまたは JSON のすべての出力からリテラルのセンチネルを除去し、`[REDACTED]` に置き換えます。そのため、ターミナルのキャプチャやパイプラインにマーカーが漏れることはありません。

## 関連項目

- [`openclaw path` CLI リファレンス](/ja-JP/cli/path)
- [Plugin の管理](/ja-JP/plugins/manage-plugins)
- [Plugin の構築](/ja-JP/plugins/building-plugins)
