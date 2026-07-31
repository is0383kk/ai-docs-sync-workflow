---
read_when:
    - ターミナルからワークスペースファイル内の末端要素を読み書きしたい場合
    - ワークスペースの状態を対象にスクリプトを作成しており、種類に依存しない安定したアドレス指定方式が必要な場合
    - '`oc://` パスをデバッグしている（構文を検証し、何に解決されるかを確認する）'
summary: '`openclaw path` の CLI リファレンス（`oc://` アドレス指定方式を使用してワークスペースファイルを検査・編集）'
title: パス
x-i18n:
    generated_at: "2026-07-26T09:56:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

`oc://` アドレス指定方式へのシェルアクセス：アドレス指定可能なワークスペースファイル（markdown、jsonc、jsonl、yaml/yml/lobster）を検査および編集するための、種類に応じてディスパッチされる単一のパス構文です。セルフホスト運用者、Plugin 作成者、エディター拡張機能は、ファイルごとのパーサーを独自実装することなく、限定された位置の読み取り、検索、更新にこれを使用できます。

`path` は、同梱のオプション `oc-path` Plugin によって提供されます。初めて使用する前に有効化してください：

```bash
openclaw plugins enable oc-path
```

CLI の動詞はアドレス指定モデルに対応しています：

- `resolve` は具体的で、単一一致です。
- `find` は、ワイルドカード、ユニオン、述語、位置展開に対応する複数一致用の動詞です。
- `set` は具体的なパスまたは挿入マーカーのみを受け付けます。ワイルドカードパターンは書き込み前に拒否されます。
- `validate` は、ファイルシステムにアクセスせずにパスを解析します。
- `emit` は、解析と出力を通じてファイルをラウンドトリップします（バイト忠実性の診断）。

## 使用する理由

OpenClaw の状態は、人が編集する Markdown、コメント付き JSONC 設定、追記専用 JSONL ログ、YAML ワークフロー／仕様ファイルに分散しています。スクリプト、フック、エージェントがこれらのファイルから必要とするのは、多くの場合、frontmatter のキー、Plugin 設定、ログレコードのフィールド、YAML のステップ、名前付きセクション下の箇条書き項目など、1 つの小さな値です。

`openclaw path` は、ファイル種類ごとに一度限りの grep、正規表現、パーサーを用意する代わりに、これらの呼び出し元へ安定したアドレスを提供します。同じ `oc://` パスを端末から検証、解決、検索、ドライラン、書き込みできるため、限定的な自動化のレビューと再実行が容易になります。ファイルの残りの部分は保持されるため、1 つの末端を書き込んでも、コメント、改行コード、周辺の書式は乱れません。

必要な対象に論理的なアドレスがある一方で、ファイル形式が異なる場合に使用します：

- フックがコメント付き JSONC から 1 つの設定を読み取り、値を書き戻す際にもコメントを失いません。
- 保守スクリプトが、JSONL ログ全体を独自パーサーへ読み込むことなく、一致するすべてのイベントフィールドを検索します。
- エディターが、Markdown のセクションまたは箇条書き項目へスラッグで移動し、解決した正確な行を表示します。
- エージェントが小規模なワークスペース編集を適用前にドライランし、変更されるバイトをレビューで確認できるようにします。

通常のファイル全体の編集、高度な設定移行、メモリ固有の書き込みには `openclaw path` を使用しないでください。これらには所有者のコマンドまたは Plugin を使用する必要があります。`path` は、別の専用パーサーを作るよりも再実行可能な端末コマンドが適している、小規模でアドレス指定可能なファイル操作向けです。

## 使用方法

人が編集する設定ファイルから 1 つの値を読み取ります：

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

ディスクに触れずに書き込みをプレビューします：

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

追記専用 JSONL ログから一致するレコードを検索します：

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Markdown 内の指示を、行番号ではなくセクションと項目で指定します：

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

スクリプトが読み取りまたは書き込みを行う前に、CI または事前確認スクリプトでパスを検証します：

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

これらのコマンドは、シェルスクリプトへコピーできるように設計されています。呼び出し元が構造化出力を必要とする場合は `--json` を、人が結果を確認する場合は `--human` を使用します。

## 仕組み

1. `oc://` アドレスを、ファイル、セクション、項目、フィールド、オプションのセッションクエリというスロットに解析します。
2. 対象の拡張子（`.md`、`.jsonc`、`.json`、`.jsonl`、`.ndjson`、`.yaml`、`.yml`、`.lobster`）からファイル種類のアダプターを選択します。
3. そのファイル種類の構造（Markdown の見出し／項目、JSONC のオブジェクトキー／配列インデックス、JSONL の行レコード、YAML のマップ／シーケンスノード）に対してスロットを解決します。
4. `set` では、同じアダプターを通じて編集後のバイトを出力するため、その種類が対応している場合、ファイル内の変更されていない部分のコメント、改行コード、周辺の書式が保持されます。

`resolve` と `set` には、具体的な対象が 1 つ必要です。`find` は探索用の動詞です。ワイルドカード、ユニオン、述語、序数を具体的な一致へ展開し、書き込む対象を 1 つ選ぶ前に確認できます。

## サブコマンド

| サブコマンド              | 目的                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | パスにある具体的な一致（または「見つかりません」）を出力します。                      |
| `find <pattern>`        | ワイルドカード／ユニオン／述語パスの一致を列挙します。                  |
| `set <oc-path> <value>` | 具体的なパスにある末端または挿入対象へ書き込みます。`--dry-run` に対応しています。  |
| `validate <oc-path>`    | 解析のみを行い、構造の内訳（ファイル／セクション／項目／フィールド）を出力します。 |
| `emit <file>`           | 解析と出力を通じてファイルをラウンドトリップします（バイト忠実性の診断）。          |

## グローバルフラグ

| フラグ            | 適用対象                       | 目的                                                                  |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`、`find`、`set`、`emit` | このディレクトリを基準にファイルスロットを解決します（デフォルト：`process.cwd()`）。 |
| `--file <path>` | `resolve`、`find`、`set`、`emit` | ファイルスロットの解決済みパスを上書きします（絶対パスアクセス）。                |
| `--json`        | すべて                              | JSON 出力を強制します（stdout が TTY でない場合のデフォルト）。                    |
| `--human`       | すべて                              | 人間向け出力を強制します（stdout が TTY の場合のデフォルト）。                       |
| `--value-json`  | `set`                            | JSON／JSONC／JSONL の末端置換用に `<value>` を JSON として解析します。           |
| `--dry-run`     | `set`                            | 実際には書き込まず、書き込まれるバイトを出力します。                   |
| `--diff`        | `set`（`--dry-run` が必要）     | 完全なバイト列ではなく unified diff を出力します。                          |

`validate` は `--json`／`--human` のみを受け取ります。ファイルシステムへアクセスしないため、`--cwd` と `--file` は適用されません。

## `oc://` の構文

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

スロットの規則：`field` には `item` が必要で、`item` には `section` が必要です。4 つのすべてのスロットに共通する規則：

- **引用符付きセグメント** — `"a/b.c"` は、`/` および `.` の区切りを越えて保持されます。内容はバイト単位のリテラルです。引用符内では `"` と `\` は使用できません。ファイルスロットも引用符を認識し、`oc://"skills/email-drafter"/Tools/$last` は `skills/email-drafter` を単一のファイルパスとして扱います。
- **述語** — `[k=v]`、`[k!=v]`、`[k<v]`、`[k<=v]`、`[k>v]`、`[k>=v]`。数値演算子では、両辺を有限数へ変換できる必要があります。
- **ユニオン** — `{a,b,c}` は、いずれかの選択肢に一致します。
- **ワイルドカード** — `*`（単一のサブセグメント）と `**`（0 個以上、再帰的）。`find` はこれらを受け付けますが、`resolve` と `set` は曖昧であるため拒否します。
- **位置指定** — `$first`／`$last` は、最初／最後のインデックスまたは宣言済みキーへ解決されます。
- **序数** — `#N` は、ドキュメント順で N 番目の一致を表します。
- **挿入マーカー** — `+`、`+key`、`+nnn` は、キー指定／インデックス指定の挿入に使用します（`set` とともに使用）。
- **セッションスコープ** — `?session=cron-daily` など。スロットの入れ子とは独立しています。セッション値は生の値であり、パーセントデコードされません。制御文字または予約済みクエリ区切り文字（`?`、`&`、`%`）を含めることはできません。

引用符付き、述語、ユニオンの各セグメント外にある予約文字（`?`、`&`、`%`）は拒否されます。制御文字（U+0000-U+001F、U+007F）は、`session` クエリ値を含むあらゆる場所で拒否されます。

正規パスでは `formatOcPath(parseOcPath(path)) === path` が保証されます。非正規のクエリパラメーターは、最初の空でない `session=` 値を除いて無視されます。

ハードリミット：パスは最大 4096 バイト、最大 4 スロット（ファイル／セクション／項目／フィールド）、各スロットで最大 64 個のドット区切りサブセグメント、深い JSON パスでは最大 256 レベルの入れ子走査に制限されます。これとは別に、16 MiB を超える JSONC／JSON ファイル入力は、そのファイルを読み込むすべての動詞において、解析される代わりに解析診断とともに拒否されます。

## ファイル種類別のアドレス指定

| 種類          | ファイル拡張子             | アドレス指定モデル                                                                                    |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | H2 セクションはスラッグ、箇条書き項目はスラッグまたは `#N`、frontmatter は `[frontmatter]` で指定します。                 |
| JSONC/JSON    | `.jsonc`、`.json`           | オブジェクトキーと配列インデックス。引用符で囲まれていない限り、ドットで入れ子のサブセグメントを分割します。                        |
| JSONL         | `.jsonl`、`.ndjson`         | トップレベルの行アドレス（`L1`、`L2`、`$first`、`$last`）から、行内を JSONC 形式で下位へたどります。 |
| YAML/.lobster | `.yaml`、`.yml`、`.lobster` | マップキーとシーケンスインデックス。コメントとフロースタイルは YAML ドキュメント API によって処理されます。        |

`resolve` は、1 始まりの行番号を含む構造化された一致（`root`、`node`、`leaf`、または `insertion-point`）を返します。Plugin 作成者が種類ごとの AST 形状に依存せずプレビューを表示できるよう、末端値はテキストと `leafType` として公開されます。

## 変更の契約

`set` は、具体的な対象を 1 つ書き込みます：

- Markdown の frontmatter 値と `- key: value` 項目フィールドは文字列の
  リーフです。Markdown の挿入では、セクション、frontmatter キー、またはセクション
  項目を追加し、変更対象ファイルを正規化された Markdown 形式でレンダリングします。セクション
  本文全体を `set` で書き込むことはできません。
- JSONC のリーフ書き込みでは、文字列値を既存のリーフ型
  （`string`、有限の `number`、`true`/`false`、または `null`）に型変換します。JSONC/JSON/JSONL のリーフ置換で `<value>` を JSON として解析し、
  文字列によるシークレット参照の省略形をオブジェクトに置き換える場合など、形状を変更できるようにするには `--value-json`
  を使用します。JSONC のオブジェクトおよび配列への挿入では、`<value>` を JSON として解析し、
  通常のリーフ書き込みには `jsonc-parser` 編集パスを使用して、コメントと
  周辺の書式を保持します。
- JSONL のリーフ書き込みでは、行内で JSONC と同様に型変換します。行全体の置換
  および追加では、`<value>` を JSON として解析します。レンダリングされた JSONL は、ファイルで
  優勢な LF/CRLF 改行規則を保持します（ファイル内の
  改行の多数決により、ほとんどが CRLF のファイルは、少数の LF が混在していても CRLF のままです）。
- YAML のリーフ書き込みでは、既存のスカラー型（`string`、有限の
  `number`、`true`/`false`、または `null`）に型変換します。YAML の挿入では、バンドルされた
  `yaml` パッケージのドキュメント API を使用してマップ／シーケンスを更新します。パーサーエラーを含む不正な YAML
  ドキュメントは、変更前に
  `parse-error` で拒否されます。

正確なバイト列が重要な場合は、ユーザーに見える書き込みの前に `--dry-run` を使用します。JSONC
と YAML の編集では、既存のドキュメントにパッチを適用するため（`jsonc-parser` または `yaml`
ドキュメント API を使用）、通常、変更していないバイト列は保持されます。一方、Markdown はどの編集でも
解析済み構造からファイルを再構築するため、変更したリーフ以外の付随的な
書式が正規化されることがあります。レンダリングされたファイル全体ではなく、変更前後に絞ったパッチとして
プレビューするには、`--diff` を追加します。

## 例

```bash
# パスを検証（ファイルシステムにはアクセスしない）
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# リーフを読み取る
openclaw path resolve 'oc://gateway.jsonc/version'

# ワイルドカード検索
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# 書き込みをドライラン
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# 書き込みを unified diff としてドライラン
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# 書き込みを適用
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# バイト忠実性を確認するラウンドトリップ（診断用）
openclaw path emit ./AGENTS.md
```

その他の文法例：

```bash
# / または . を含むキーを引用符で囲む
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# 深い JSON/JSONC パスではスラッシュ区切りを使用でき、ドット区切りのサブセグメントに正規化される
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# JSONC のリーフを解析済みオブジェクトで置換
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# JSONC の子要素を述語で検索
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# JSONC 配列に挿入
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# JSONC オブジェクトのキーを挿入
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# JSONL イベントを追加
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# JSONL の最後の値の行を解決
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# YAML ワークフローのステップを解決
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# YAML スカラーを更新
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# Markdown の frontmatter を指定
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# Markdown の frontmatter に挿入
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# Markdown の項目フィールドを検索
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# セッションスコープのパスを検証
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## ファイル種別別のレシピ

同じ 5 つの動詞がすべての種別で機能し、アドレス指定方式は
ファイル拡張子に応じて処理を振り分けます。

### Markdown

```text
<!-- frontmatter.md -->
---
name: drafter
description: メール下書きエージェント
tier: core
---
## ツール
- gh: GitHub CLI
- curl: HTTP クライアント
- send_email: 有効
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
リーフ @ L4: "core"（文字列）

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
リーフ @ L9: "GitHub CLI"（文字列）

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
oc://x.md/tools/* に 3 件一致：
  oc://x.md/tools/gh           →  ノード @ L9 [md-item]
  oc://x.md/tools/curl         →  ノード @ L10 [md-item]
  oc://x.md/tools/send-email   →  ノード @ L11 [md-item]
```

`[frontmatter]` 述語は YAML frontmatter ブロックを指定します。`tools`
はスラッグを介して `## Tools` 見出しに一致し、ソースでアンダースコアが使われていても
項目のリーフはスラッグ形式を維持します（`send_email` は `send-email` になります）。

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
リーフ @ L4: "true"（真偽値）

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: /…/config.jsonc に 142 バイトを書き込む予定
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC の編集は `jsonc-parser` を経由するため、
`set` の実行後もコメントと空白が保持されます。確定する前にバイト列を確認するには、まず `--dry-run` を付けて実行します。
`.json` ファイルは、`.jsonc` と同じアダプターおよび編集パスを使用します。

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
oc://session.jsonl/[event=action]/userId に 1 件一致：
  oc://session.jsonl/L2/userId  →  リーフ @ L2: "u1"（文字列）

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
リーフ @ L2: "2"（数値）
```

各行が 1 つのレコードです。行番号が不明な場合は述語（`[event=action]`）で、
判明している場合は正規の `LN` セグメントで指定します。
`.ndjson` ファイルは、`.jsonl` と同じアダプターを使用します。

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
リーフ @ L3: "fetch"（文字列）

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: /…/workflow.yaml に 99 バイトを書き込む予定
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML は独自実装のパーサーではなく、`yaml` パッケージの `Document` API を使用します。
そのため、通常の解析／出力ラウンドトリップではコメントと記述形式が保持され、
解決済みパスでは JSONC と同じマップキー／シーケンスインデックスモデルが使用されます。
同じアダプターが `.yaml`、`.yml`、および `.lobster` ファイルを処理します。

## サブコマンドリファレンス

### `resolve <oc-path>`

単一のリーフまたはノードを読み取ります。ワイルドカードは拒否されます。その場合は `find` を使用してください。
一致した場合は `0`、正常に一致しなかった場合は `1`、解析エラーまたは拒否された
パターンの場合は `2` で終了します。

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

ワイルドカード／述語／ユニオンパターンに一致するすべての項目を列挙します。1 件以上一致した場合は `0`、
0 件の場合は `1` で終了します。ファイルスロットのワイルドカードは
`OC_PATH_FILE_WILDCARD_UNSUPPORTED` で拒否されます。具体的なファイルを渡してください（複数ファイルの
glob は今後追加予定の機能です）。

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

リーフを書き込みます。ファイルに触れずに書き込まれるバイト列をプレビューするには、
`--dry-run` と組み合わせます。unified diff のプレビューには `--diff` を追加します。
書き込みに成功した場合は `0`、基盤が拒否した場合（たとえば、
センチネルガードに該当した場合）は `1`、解析エラーの場合は `2` で終了します。

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

`+key` 挿入マーカーは、指定された子がまだ存在しない場合に作成します。
`+nnn` と単独の `+` は、それぞれインデックス指定挿入と末尾への追加に使用できます。

### `validate <oc-path>`

解析のみを行うチェックです。ファイルシステムにはアクセスしません。変数を置換する前に
テンプレートパスの形式が正しいことを確認したい場合や、デバッグ用に
構造の内訳を確認したい場合に便利です：

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
有効: oc://AGENTS.md/tools/gh
  ファイル:      AGENTS.md
  セクション:    tools
  項目:          gh
```

有効な場合は `0`、無効な場合は `1`（構造化された `code` と
`message` を伴う）、引数エラーの場合は `2` で終了します。

### `emit <file>`

ファイル種別ごとのパーサーとエミッターを通してファイルをラウンドトリップします。正常なファイルでは、
出力が入力とバイト単位で同一になる必要があります。差異がある場合は、
パーサーのバグまたはセンチネルへの該当を示します。実環境の入力に対する
基盤の動作をデバッグする際に便利です。

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## 終了コード

| コード | 意味                                                                       |
| ---- | -------------------------------------------------------------------------- |
| `0`  | 成功。（`resolve` / `find`：1 件以上一致。`set`：書き込み成功。） |
| `1`  | 一致なし、または `set` が基盤によって拒否された（システムレベルのエラーなし）。 |
| `2`  | 引数エラーまたは解析エラー。                                               |

## 出力モード

`openclaw path` は TTY を認識し、端末では人間が読みやすい出力を使用し、標準出力が
パイプまたはリダイレクトされている場合は JSON を使用します。`--json` と `--human` は
自動検出を上書きします。

## 注記

- `set` は substrate の emit パスを通じてバイトを書き込み、このパスでは
  redaction-sentinel ガードが自動的に適用されます。
  `__OPENCLAW_REDACTED__` を含むリーフ（そのまま、または部分文字列として）は、書き込み
  時に拒否されます。
- JSONC の解析とリーフの編集には Plugin ローカルの `jsonc-parser`
  依存関係が使用されるため、通常のリーフ
  書き込みでは、独自実装のパーサー／再レンダリングパスを経由せず、コメントと書式が保持されます。
- `path` は last-known-good（LKG）設定の追跡や復旧を認識しません。
  そのライフサイクルは別の場所が所有します。`path` を通じて編集するファイルが
  LKG でも追跡されている場合、次回の設定読み取り時に、それを昇格するか
  復旧するかが決定されます。`path` による編集は、そのファイルへの他の直接書き込みと
  同様に扱ってください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
