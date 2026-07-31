---
read_when:
    - 設定を非対話形式で読み取りまたは編集したい場合
sidebarTitle: Config
summary: '`openclaw config` の CLI リファレンス（get/set/patch/unset/file/schema/validate）'
title: 設定
x-i18n:
    generated_at: "2026-07-26T09:35:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c4f8edb19737070e421c9107f7da8886e5617d9a043d8647666505c7ac9638d
    source_path: cli/config.md
    workflow: 16
---

`openclaw.json` 用の非対話型ヘルパーです。パスを指定して値を取得、設定、パッチ、設定解除するほか、スキーマの出力、検証、またはアクティブなファイルパスの出力を行います。サブコマンドなしで `openclaw config` を実行すると、`openclaw configure` と同じガイド付きウィザードが開きます。

<Note>
`OPENCLAW_NIX_MODE=1` の場合、OpenClaw は `openclaw.json` を不変として扱います。読み取り専用コマンド（`config get`、`config file`、`config schema`、`config validate`）は引き続き動作しますが、設定を書き込むコマンドは拒否されます。代わりに、インストール用の Nix ソースを編集してください。ファーストパーティの nix-openclaw ディストリビューションでは、[nix-openclaw クイックスタート](https://github.com/openclaw/nix-openclaw#quick-start)を使用し、`programs.openclaw.config` または `instances.<name>.config` の配下に値を設定します。
</Note>

## ルートオプション

<ParamField path="--section <section>" type="string">
  サブコマンドなしで `openclaw config` を実行するときに繰り返し指定できる、ガイド付きセットアップのセクションフィルターです。
</ParamField>

ガイド付きセクション：`workspace`、`model`、`web`、`gateway`、`daemon`、`channels`、`plugins`、`skills`、`health`。

## 例

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### パス

ドット記法またはブラケット記法を使用します。zsh が `[0]` をグロブ展開しないよう、シェルの例ではブラケット記法のパスを引用符で囲んでください。

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.entries.main
openclaw config get agents.entries
openclaw config set 'agents.entries.work.tools.exec.node' "node-id-or-name"
```

### `config get`

秘匿化された設定スナップショットから値を読み取ります（シークレットは出力されません）。`--json` は未加工の値を JSON として出力します。それ以外の場合、文字列、数値、ブール値は装飾なしで、オブジェクトと配列は整形済み JSON として出力されます。

パスが存在しない場合、`--json` は `{ "error": "Config path not found: <path>" }` を標準出力に書き込み、ステータス 1 で終了します。`--json` を指定しない場合、診断メッセージは標準エラー出力に残ります。

```bash
openclaw config get browser.executablePath
openclaw config get agents.defaults.model --json
```

### `config file`

`OPENCLAW_CONFIG_PATH` またはデフォルトの場所から解決した、アクティブな設定ファイルのパスを出力します。このパスが指すのはシンボリックリンクではなく通常ファイルです。[書き込みの安全性](#write-safety)を参照してください。

### `config schema`

`openclaw.json` 用に生成された JSON スキーマを標準出力に出力します。

<AccordionGroup>
  <Accordion title="含まれる内容">
    - 現在のルート設定スキーマと、エディターツール用のルート `$schema` 文字列フィールド。
    - Control UI で使用されるフィールド `title` / `description` のドキュメントメタデータ。
    - 対応するフィールドドキュメントが存在する場合、ネストされたオブジェクト、ワイルドカード（`*`）、配列項目（`[]`）の各ノードは、同じ `title` / `description` メタデータを継承します。
    - `anyOf` / `oneOf` / `allOf` の各ブランチも、同じドキュメントメタデータを継承します。
    - ランタイムマニフェストを読み込める場合の、ベストエフォートのライブ Plugin + チャンネルスキーマメタデータ。
    - 現在の設定が無効な場合でも使用できる、クリーンなフォールバックスキーマ。

  </Accordion>
  <Accordion title="関連するランタイム RPC">
    `config.schema.lookup` は、浅いスキーマノード（`title`、`description`、`type`、`enum`、`const`、共通の境界値）、一致する UI ヒントメタデータ、および直下の子要素の概要とともに、正規化された設定パスを 1 つ返します。Control UI またはカスタムクライアントで、パス単位のドリルダウンに使用します。
  </Accordion>
</AccordionGroup>

```bash
openclaw config schema
openclaw config schema > openclaw.schema.json
```

### `config validate`

Gateway を起動せずに、現在の設定をアクティブなスキーマに照らして検証します。

```bash
openclaw config validate
openclaw config validate --json
```

<Note>
検証がすでに失敗している場合は、`openclaw configure` または `openclaw doctor --fix` から始めてください。`openclaw chat` では無効な設定に対するガードを回避できません。
</Note>

## 値

値は可能であれば JSON5 として解析され、それ以外の場合は未加工の文字列として扱われます。文字列へのフォールバックを行わず、標準 JSON を必須にするには `--strict-json` を使用します。この場合、コメント、末尾のカンマ、引用符なしのキーなど、JSON5 のみで使用できる構文は拒否されます。`--json` は、`config set` における `--strict-json` のレガシーエイリアスです。

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` は、ターミナル用に整形されたテキストではなく、未加工の値を JSON として出力します。

書き込みによって `agents.defaults.model` またはエージェントごとの `agents.entries.*.model` が変更される場合、OpenClaw は書き込み前に、変更された各プライマリまたはフォールバックを設定済みのプロバイダーカタログで解決します。不明なモデル参照は、アクティブな設定を変更せずに拒否されます。利用可能なモデルを確認するには `openclaw models list` を実行してください。

<Note>
オブジェクトの代入は、デフォルトで対象パスを置き換えます。ユーザーが追加したエントリを保持することが多い次の保護対象パスでは、`--replace` を渡さない限り、既存エントリを削除する置換が拒否されます：`agents.defaults.models`、`agents.entries`、`models.providers`、`models.providers.<id>`、`models.providers.<id>.models`、`plugins.entries`、`auth.profiles`。
</Note>

これらのマップにエントリを追加するときは、`--merge` を使用します。

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

指定した値を意図的に対象全体の値とする場合にのみ、`--replace` を使用します。

## `config set` モード

<Tabs>
  <Tab title="値モード">
    ```bash
    openclaw config set <path> <value>
    ```
  </Tab>
  <Tab title="SecretRef ビルダーモード">
    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  </Tab>
  <Tab title="プロバイダービルダーモード">
    `secrets.providers.<alias>` パスのみが対象です。

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  </Tab>
  <Tab title="バッチモード">
    ```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

    バッチファイルの上限は 8 MiB です。

  </Tab>
</Tabs>

<Warning>
SecretRef の代入は、サポートされていないランタイム変更可能サーフェス（例：`hooks.token`、`commands.ownerDisplaySecret`、Discord スレッドバインディングの Webhook トークン、WhatsApp の認証情報 JSON）では拒否されます。[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)を参照してください。
</Warning>

バッチ解析では、常にバッチペイロード（`--batch-json`/`--batch-file`）が信頼できる情報源として使用されます。`--strict-json` / `--json` はバッチ解析の動作を変更しません。

JSON のパス/値モードは、SecretRef とプロバイダーを直接指定する場合にも使用できます。

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

### プロバイダービルダーフラグ

プロバイダービルダーの対象では、パスとして `secrets.providers.<alias>` を使用する必要があります。

<AccordionGroup>
  <Accordion title="共通フラグ">
    - `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>`（`file`、`exec`）

  </Accordion>
  <Accordion title="Env プロバイダー（--provider-source env）">
    - `--provider-allowlist <ENV_VAR>`（繰り返し指定可能）

  </Accordion>
  <Accordion title="ファイルプロバイダー（--provider-source file）">
    - `--provider-path <path>`（必須）
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  </Accordion>
  <Accordion title="Exec プロバイダー（--provider-source exec）">
    - `--provider-command <path>`（必須）
    - `--provider-arg <arg>`（繰り返し指定可能）
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env <KEY=VALUE>`（繰り返し指定可能）
    - `--provider-pass-env <ENV_VAR>`（繰り返し指定可能）
    - `--provider-trusted-dir <path>`（繰り返し指定可能）
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  </Accordion>
</AccordionGroup>

強化された exec プロバイダーの例：

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## `config patch`

パスベースの `config set` コマンドを多数実行する代わりに、設定と同じ形状の JSON5 パッチを貼り付けるかパイプで渡します。オブジェクトは再帰的にマージされ、配列とスカラー値は対象を置き換え、`null` は対象パスを削除します。

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

パッチファイルの上限は 8 MiB です。パイプで渡す `--stdin` パッチの上限は 1 MiB です。

リモートセットアップスクリプトでは、標準入力経由でパッチをパイプします。

```bash
ssh user@gateway-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh user@gateway-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

パッチの例：

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

1 つのオブジェクトまたは配列を再帰的にパッチするのではなく、指定した値そのものにする必要がある場合は、`--replace-path <path>` を使用します。

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run` は、書き込みを行わずにスキーマと SecretRef の解決可能性をチェックします。dry-run では、exec を使用する SecretRef はデフォルトでスキップされます。dry-run で意図的にプロバイダーコマンドを実行する場合は、`--allow-exec` を追加してください。

## dry-run

`--dry-run` は、`openclaw.json` に書き込まずに変更を検証します。`config set`、`config patch`、`config unset` で使用できます。

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

<AccordionGroup>
  <Accordion title="dry-run の動作">
    - ビルダーモード: 変更された参照とプロバイダーについて、SecretRef の解決可能性チェックを実行します。
    - JSON モード（`--strict-json`、`--json`、またはバッチモード）: スキーマ検証と SecretRef の解決可能性チェックを実行します。
    - ポリシー検証は変更後の設定全体に対して実行されるため、親オブジェクトへの書き込み（たとえば `hooks` をオブジェクトとして設定する場合）で、サポート対象外のサーフェスに対する検証を回避することはできません。
    - コマンドの副作用を避けるため、exec SecretRef のチェックはデフォルトでスキップされます。明示的に有効にするには `--allow-exec` を渡してください（プロバイダーコマンドが実行される場合があります）。`--allow-exec` は dry-run 専用であり、`--dry-run` がない場合はエラーになります。

  </Accordion>
  <Accordion title="--dry-run --json のフィールド">
    - `ok`: dry-run が成功したかどうか
    - `operations`: 評価された代入の数
    - `checks`: スキーマ／解決可能性チェックが実行されたかどうか
    - `checks.resolvabilityComplete`: 解決可能性チェックが最後まで実行されたかどうか（exec 参照がスキップされた場合は false）
    - `refsChecked`: dry-run 中に実際に解決された参照の数
    - `skippedExecRefs`: `--allow-exec` が設定されていなかったためスキップされた exec 参照の数
    - `errors`: `ok=false` の場合に返される、パス欠落、スキーマ、または解決可能性に関する構造化されたエラー

  </Accordion>
</AccordionGroup>

### JSON 出力形式

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder" | "unset", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "missing-path" | "schema" | "resolvability" | "model",
      message: string,
      ref?: string, // 解決可能性エラーの場合に存在
    },
  ],
}
```

<Tabs>
  <Tab title="成功例">
    ```json
    {
      "ok": true,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0
    }
    ```
  </Tab>
  <Tab title="失敗例">
    ```json
    {
      "ok": false,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0,
      "errors": [
        {
          "kind": "resolvability",
          "message": "エラー: 環境変数 \"MISSING_TEST_SECRET\" が設定されていません。",
          "ref": "env:default:MISSING_TEST_SECRET"
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="dry-run が失敗した場合">
    - `config schema validation failed`: 変更後の設定形式が無効です。パス／値またはプロバイダー／参照オブジェクトの形式を修正してください。
    - `Config policy validation failed: unsupported SecretRef usage`: その認証情報をプレーンテキスト／文字列入力に戻してください。SecretRef はサポート対象のサーフェスでのみ使用してください。
    - `SecretRef assignment(s) could not be resolved`: 参照先のプロバイダー／参照を現在解決できません（環境変数の欠落、無効なファイルポインター、exec プロバイダーの失敗、またはプロバイダーとソースの不一致）。
    - `model reference validation failed`: 変更されたテキストモデルのプライマリまたはフォールバックが不明です。`openclaw models list` を実行し、使用可能なモデルを選択してください。
    - `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: exec の解決可能性を検証する必要がある場合は、`--allow-exec` を指定して再実行してください。
    - バッチモードでは、失敗したエントリを修正し、書き込む前に `--dry-run` を再実行してください。

  </Accordion>
</AccordionGroup>

## 変更の適用

`config set`／`config patch`／`config unset` が正常に完了するたびに、Gateway の再起動が必要かどうかを判断できるよう、CLI は次の 3 つのヒントのいずれかを表示します。

| ヒント                                              | 意味                                   |
| --------------------------------------------------- | -------------------------------------- |
| `Restart the gateway to apply.`                     | 変更されたパスには完全な再起動が必要です。 |
| `Change will apply without restarting the gateway.` | ホットリロードによって自動的に反映されます。 |
| `No gateway restart needed.`                        | ランタイムに関連する変更はありません。 |

CLI はすべての Plugin のリロードメタデータが読み込まれていることを確認できないため、`plugins.entries`（またはその任意のサブパス）への書き込みには常に再起動が必要です。

## 書き込みの安全性

`openclaw config set` および OpenClaw が所有するその他の設定ライターは、ディスクに確定する前に、変更後の設定全体を検証します。新しいペイロードがスキーマ検証に失敗するか、破壊的な上書きと見なされた場合、アクティブな設定は変更されず、拒否されたペイロードが `openclaw.json.rejected.*` としてその隣に保存されます。

OpenClaw が所有する書き込み処理では、JSON5 が標準 JSON として再シリアライズされます。ソースにコメントが含まれている場合、ライターはコメントを削除する直前に警告します。コメントを保持する必要がある場合は、エディターで直接編集してください。

<Warning>
アクティブな設定パスは通常ファイルである必要があります。シンボリックリンクを使用した `openclaw.json` の配置は書き込みではサポートされていません。代わりに `OPENCLAW_CONFIG_PATH` を使用して実ファイルを直接指定してください。
</Warning>

小規模な編集には CLI からの書き込みを推奨します。

```bash
openclaw config set gateway.reload.mode hybrid --dry-run
openclaw config set gateway.reload.mode hybrid
openclaw config validate
```

書き込みが拒否された場合は、保存されたペイロードを確認し、設定全体の形式を修正してください。

```bash
CONFIG="$(openclaw config file)"
ls -lt "$CONFIG".rejected.* 2>/dev/null | head
openclaw config validate
```

エディターによる直接書き込みも引き続き可能ですが、実行中の Gateway は、検証されるまでそれらを信頼できないものとして扱います。無効な直接編集があると、起動に失敗するか、ホットリロードでスキップされます。Gateway は `openclaw.json` を書き換えません。プレフィックス付き／上書きされた設定を修復するか、最後に正常だったコピーを復元するには、`openclaw doctor --fix` を実行してください。[Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting#gateway-rejected-invalid-config)を参照してください。

ファイル全体の復旧は doctor による修復に限定されます。Plugin のスキーマ変更や `minHostVersion` の不整合では、モデル、プロバイダー、認証プロファイル、チャンネル、Gateway の公開範囲、ツール、メモリ、ブラウザー、Cron 設定など、無関係なユーザー設定をロールバックせず、明示的にエラーを報告します。

## 修復ループ

`openclaw config validate` が成功したら、ローカル TUI を使用して、組み込みエージェントにアクティブな設定とドキュメントを比較させながら、同じターミナルから各変更を検証します。

```bash
openclaw chat
```

TUI 内では、行頭の `!` によって、文字どおりのローカルシェルコマンドが実行されます（セッションごとに初回のみ確認プロンプトが表示されます）。

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

<Steps>
  <Step title="ドキュメントとの比較">
    エージェントに、現在の設定を関連するドキュメントページと比較し、最小限の修正を提案するよう依頼します。
  </Step>
  <Step title="対象を絞った編集の適用">
    `openclaw config set` または `openclaw configure` を使用して、対象を絞った編集を適用します。
  </Step>
  <Step title="再検証">
    変更するたびに `openclaw config validate` を再実行します。
  </Step>
  <Step title="ランタイムの問題に doctor を使用">
    検証に成功してもランタイムが正常でない場合は、移行と修復の支援を得るために `openclaw doctor` または `openclaw doctor --fix` を実行します。
  </Step>
</Steps>

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [設定](/ja-JP/gateway/configuration)
