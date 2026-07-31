---
read_when:
    - プロバイダー認証情報および `auth-profiles.json` 参照用の SecretRefs の設定
    - 本番環境でシークレットの再読み込み、監査、設定、適用を安全に行う
    - 起動時のフェイルファスト、非アクティブなサーフェスのフィルタリング、最終正常状態の動作を理解する
sidebarTitle: Secrets management
summary: シークレット管理：SecretRef コントラクト、ランタイムスナップショットの動作、安全な一方向スクラビング
title: シークレット管理
x-i18n:
    generated_at: "2026-07-26T10:02:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d10989ebbce367c68d28768244d4e3649028af5ab63c9523974352c270a3c55e
    source_path: gateway/secrets.md
    workflow: 16
---

OpenClaw は加算的な SecretRef をサポートしているため、サポート対象の認証情報を設定内に平文で保存する必要はありません。

<Note>
平文も引き続き使用できます。SecretRef は認証情報ごとにオプトインします。
</Note>

<Warning>
平文の認証情報がエージェントから参照可能なファイル（`openclaw.json`、`auth-profiles.json`、`.env`、または生成された `agents/*/agent/models.json` ファイルなど）に存在する場合、引き続きエージェントから読み取れます。SecretRef によってこのローカルでの影響範囲が縮小されるのは、サポート対象のすべての認証情報を移行し、`openclaw secrets audit --check` で平文の残存が報告されなくなった後に限られます。
</Warning>

## ランタイムモデル

- Secrets は、リクエストパスで遅延的にではなく、アクティベーション中に即時解決され、メモリ内のランタイムスナップショットに格納されます。
- Gateway のコールドスタートでは、既知の非 Gateway オーナーが分離をサポートしている場合、再試行可能な SecretRef の失敗をそのオーナーに分離します。マッピング対象のオーナークラスには、モデルプロバイダーと Skills、メディア/TTS/cron プロバイダー、対象となる認証プロファイル、エージェントごとのメモリ、サンドボックス SSH、チャネルアカウント、マニフェストで宣言された Plugin ルートが含まれます。Gateway は起動し、そのオーナーを「設定済みだが利用不可」として記録し、機密情報を除去した機能低下警告を出力します。Gateway の受信認証、構造的に無効な参照または解決値、フェイルクローズのオーナー、およびランタイムオーナーがマッピングされていない参照では、引き続き起動に失敗します。
- リロードでは、マッピング対象の各オーナーを個別に検証した後、1 つのアトミックなスナップショットを公開します。正常なオーナーは更新されます。対象となる失敗したオーナーは、参照 ID、プロバイダー定義、およびシークレット以外の完全なオーナー契約が変更されていない場合に限り、最後に正常だった値を維持して stale 状態になります。変更された、または新規の失敗したオーナーは cold 状態になります。厳格な失敗ではリロードを拒否し、アクティブなスナップショットを維持します。
- ポリシー違反（たとえば、OAuth モードの認証プロファイルと SecretRef 入力の組み合わせ）は、ランタイムの切り替え前にアクティベーションを失敗させます。
- ランタイムリクエストは、アクティブなメモリ内スナップショットのみを読み取ります。モデルプロバイダーの SecretRef 認証情報は、外部送信時までプロセスローカルなセンチネルとして認証ストレージとストリームオプションを通過します。外部配信パス（Discord の返信/スレッド配信、Telegram のアクション送信）もそのスナップショットを読み取り、送信ごとに参照を再解決しません。

これにより、シークレットプロバイダーの障害がホットなリクエストパスから排除されます。

Gateway の受信保護、構造的に無効な設定または解決値、ポリシー違反、不明な所有関係では、引き続きフェイルクローズします。分離されたオーナーが優先順位の低い認証情報ソースへフォールスルーすることはありません。

## 外部送信時の注入（センチネル）

SecretRef によって提供されるモデルプロバイダー認証情報について、OpenClaw はモデル認証の解決中に不透明なプロセスローカルのセンチネルを生成します。そのため、認証ストレージ、ストリームオプション、SDK 設定、ログ、エラーオブジェクト、および大半のランタイムイントロスペクションからは、プロバイダー認証情報ではなく `oc-sent-v1-...` のような値が見えます。保護されたモデル fetch と、管理対象のローカルプロバイダーのヘルスプローブは、各リクエストがプロセスを離れる直前に、URL とヘッダーの値に含まれる既知のセンチネルを置き換えます。

不明なセンチネル形式の値は、ネットワーク処理の前にフェイルクローズします。OpenClaw は、未解決のセンチネルをプロバイダーへ転送せず、リクエストの送信を拒否します。多層防御として、解決済みのシークレット値も完全一致によるログ秘匿化の対象として登録されます。

プロバイダーアダプターは、SDK がサポートする最も遅い注入ポイントを使用します。

- カスタム fetch オプションを備えた SDK には OpenClaw の保護された fetch が渡されるため、SDK 内ではセンチネルが維持されます。
- カスタム fetch オプションがない SDK では、クライアントを構築する直前にセンチネルを展開します。Plugin が所有するプロバイダーストリームとエージェントハーネスは、これらのトランスポートが OpenClaw の保護された fetch を共有しないため、コアが所有する最終ハンドオフ時にセンチネルを展開します。

センチネルはモデル呼び出しチェーン全体での平文露出を減らしますが、プロセス分離ではありません。実際の値は同一プロセスのメモリ内に引き続き存在し、最終的なアダプター境界に現れます。SecretRef を介さずに設定された平文の環境認証情報は平文のままであり、この仕組みの対象外です。

インシデント対応または互換性のトラブルシューティング中にセンチネルの生成を無効にするには、`OPENCLAW_SECRET_SENTINELS=off`（`0` または `false` も大文字と小文字を区別せず受け付けます）を設定します。このキルスイッチでは、完全一致による秘匿化登録は無効になりません。

## エージェントアクセス境界

SecretRef は認証情報が設定や生成済みモデルファイルに永続化されることを防ぎますが、プロセス分離の境界ではありません。エージェントが読み取れるディスク上のパスに残された平文の認証情報は、API レベルの秘匿化を迂回し、ファイルツールまたはシェルツールで引き続き読み取れます。

エージェントからアクセス可能なファイルが対象範囲に含まれる本番環境では、次のすべてを満たす場合に限り、移行が完了したものとして扱ってください。

- サポート対象の認証情報で、平文値ではなく SecretRef を使用している。
- `openclaw.json`、`auth-profiles.json`、`.env`、および生成された `models.json` ファイルから、従来の平文の残存が除去されている。
- 移行後の `openclaw secrets audit --check` がクリーンである。
- 残っている未サポートまたはローテーション対象の認証情報が、OS 分離、コンテナ分離、または外部認証情報プロキシによって保護されている。

このため、監査/設定/適用ワークフローは単なる利便性のためのヘルパーではなく、セキュリティ移行のゲートです。

<Warning>
SecretRef を使用しても、任意の読み取り可能なファイルが安全になるわけではありません。バックアップ、コピーされた設定、古い生成済みモデルカタログ、未サポートの認証情報クラスは、削除されるか、エージェントの信頼境界外へ移動されるか、個別に分離されるまで、本番シークレットのままです。
</Warning>

## アクティブサーフェスのフィルタリング

SecretRef は、実質的にアクティブなサーフェスでのみ検証されます。

- **有効なサーフェス**：マッピング済みで分離可能なオーナーにおける再試行可能な失敗は、cold または stale の機能低下状態になります。厳格、フェイルクローズ、Gateway に必須、またはマッピングされていない失敗は、起動/リロードを阻止します。
- **非アクティブなサーフェス**：未解決の参照は起動/リロードを阻止せず、致命的でない `SECRETS_REF_IGNORED_INACTIVE_SURFACE` 診断を出力します。

<Accordion title="非アクティブなサーフェスの例">
- 無効化されたチャネル/アカウントエントリ。
- 有効なアカウントに継承されないトップレベルのチャネル認証情報。
- 無効化されたツール/機能サーフェス。
- `tools.web.search.provider` で選択されていない Web 検索プロバイダー固有のキー。自動モード（プロバイダー未設定）では、いずれか 1 つが解決されるまで、優先順位に従って自動検出用のキーを参照します。選択後、選択されなかったプロバイダーのキーは非アクティブになります。
- サンドボックス SSH 認証情報（`agents.defaults.sandbox.ssh.identityData`、`certificateData`、`knownHostsData`、およびエージェントごとのオーバーライド）は、デフォルトエージェントまたは有効なエージェントについて、有効なサンドボックスバックエンドが `ssh` であり、かつサンドボックスモードが `off` ではない場合にのみアクティブです。
- `gateway.remote.token` / `gateway.remote.password` SecretRef は、次のいずれかを満たす場合にアクティブです。
  - `gateway.mode=remote`
  - `gateway.remote.url` が設定されている
  - `gateway.tailscale.mode` が `serve` または `funnel` である
  - これらのリモートサーフェスがないローカルモードでは、トークン認証が優先される可能性があり、かつ環境/認証トークンが設定されていない場合に `gateway.remote.token` がアクティブになります。パスワード認証が優先される可能性があり、かつ環境/認証パスワードが設定されていない場合にのみ `gateway.remote.password` がアクティブになります。
- `OPENCLAW_GATEWAY_TOKEN` が設定されている場合、そのランタイムでは環境トークン入力が優先されるため、起動時の認証解決では `gateway.auth.token` SecretRef は非アクティブです。

</Accordion>

## Gateway 認証サーフェスの診断

`gateway.auth.token`、`gateway.auth.password`、`gateway.remote.token`、または `gateway.remote.password` に SecretRef が設定されている場合、Gateway の起動/リロード時に、コード `SECRETS_GATEWAY_AUTH_SURFACE` でサーフェスの状態がログに記録されます。

- `active`：SecretRef は有効な認証サーフェスの一部であり、解決する必要があります。
- `inactive`：別の認証サーフェスが優先されるか、リモート認証が無効または非アクティブです。

ログエントリには、アクティブサーフェスポリシーが使用した理由が含まれます。

## オンボーディングでの参照の事前検証

対話型オンボーディングで SecretRef ストレージを選択すると、保存前に事前検証が実行されます。

- 環境参照：環境変数名を検証し、セットアップ中に空でない値が参照可能であることを確認します。
- プロバイダー参照（`file` または `exec`）：プロバイダーの選択を検証し、`id` を解決して、解決値の型を確認します。
- クイックスタートフロー：`gateway.auth.token` がすでに SecretRef である場合、同じ即時失敗ゲートを使用し、プローブ/ダッシュボードのブートストラップ前に（`env`、`file`、および `exec` 参照について）解決します。

検証に失敗するとエラーが表示され、再試行できます。

## SecretRef の契約

すべての場所で使用するオブジェクト形式は 1 つです。

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

<Tabs>
  <Tab title="env">
    ```json5
    { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    ```

    SecretInput フィールドでは短縮文字列も使用できます。

    ```json5
    "${OPENAI_API_KEY}"
    "$OPENAI_API_KEY"
    ```

    検証：

    - `provider` は `^[a-z][a-z0-9_-]{0,63}$` に一致する必要があります
    - `id` は `^[A-Z][A-Z0-9_]{0,127}$` に一致する必要があります

  </Tab>
  <Tab title="file">
    ```json5
    { source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
    ```

    検証：

    - `provider` は `^[a-z][a-z0-9_-]{0,63}$` に一致する必要があります
    - `id` は絶対 JSON ポインター（`/...`）であるか、`singleValue` プロバイダーの場合はリテラル `value` である必要があります
    - セグメント内の RFC 6901 エスケープ：`~` は `~0` に、`/` は `~1` になります

  </Tab>
  <Tab title="exec">
    ```json5
    { source: "exec", provider: "vault", id: "providers/openai/apiKey#value" }
    ```

    検証：

    - `provider` は `^[a-z][a-z0-9_-]{0,63}$` に一致する必要があります
    - `id` は `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` に一致する必要があります（`secret#json_key` などのセレクターをサポート）
    - `id` には、スラッシュで区切られたパスセグメントとして `.` または `..` を含めてはなりません（たとえば `a/../b` は拒否されます）

  </Tab>
</Tabs>

## プロバイダー設定

`secrets.providers` の下にプロバイダーを定義します。

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // または "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
      "team-secrets": {
        source: "exec",
        pluginIntegration: {
          pluginId: "acme-secrets",
          integrationId: "secret-store",
        },
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

<Accordion title="環境プロバイダー">
- `allowlist` による、完全一致名のオプションの許可リスト。
- 環境値が欠落しているか空の場合、解決に失敗します。

</Accordion>

<Accordion title="ファイルプロバイダー">
- `path` にあるローカルファイルを読み取ります。
- `mode: "json"`（デフォルト）は JSON オブジェクトのペイロードを想定し、`id` を JSON ポインターとして解決します。
- `mode: "singleValue"` は参照 ID `"value"` を想定し、ファイルの生の内容を返します（末尾の改行は削除されます）。
- パスは所有権/権限チェックに合格する必要があります。読み取りは `timeoutMs`（デフォルト 5000）と `maxBytes`（デフォルト 1 MiB）によって制限されます。
- Windows ではフェイルクローズします。パスの ACL 検証を利用できない場合、解決に失敗します。信頼できるパスに限り、そのプロバイダーで `allowInsecurePath: true` を設定すると、このチェックを迂回できます。

</Accordion>

<Accordion title="Exec プロバイダー">
- 設定された絶対バイナリパスをシェルなしで直接実行します。
- デフォルトでは、`command` はシンボリックリンクではなく通常ファイルである必要があります。シンボリックリンクのコマンドパス（Homebrew の shim など）を許可するには `allowSymlinkCommand: true` を設定し、パッケージマネージャーのパスのみが対象となるように `trustedDirs`（`["/opt/homebrew"]` など）と組み合わせてください。
- `timeoutMs`（デフォルトは 5000）、`noOutputTimeoutMs`（デフォルトは `timeoutMs` と同じ）、`maxOutputBytes`（デフォルトは 1 MiB）、`env`/`passEnv` 許可リスト、および `trustedDirs` をサポートします。
- `jsonOnly` のデフォルトは `true` です。`jsonOnly: false` が設定され、要求された id が 1 つだけの場合、JSON ではないプレーンな stdout がその id の値として受け入れられます。
- Windows ではフェイルクローズします。コマンドパスの ACL 検証を利用できない場合、解決は失敗します。信頼できるパスに限り、そのプロバイダーに `allowInsecurePath: true` を設定してチェックを回避できます。
- Plugin が管理する Exec プロバイダーでは、コピーした `command`/`args` の代わりに `pluginIntegration` を使用できます。OpenClaw は起動時または再読み込み時に、インストール済み Plugin のマニフェストから現在のコマンド詳細を解決します。Plugin が無効化、削除、信頼解除された場合、または統合を宣言しなくなった場合、そのプロバイダー上のアクティブな SecretRef はフェイルクローズします。

リクエストペイロード（stdin）：

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

レスポンスペイロード（stdout）：

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

id ごとの省略可能なエラー：

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "code": "NOT_FOUND" } }
}
```

`code` は省略可能な機械可読診断情報です。OpenClaw は認識される
コード `NOT_FOUND` および `AMBIGUOUS_DUPLICATE_KEY` を、プロバイダーと参照 id とともに表示します。その他の
コードや `message` などの自由形式フィールドは、protocol-v1 との互換性のために受け入れられますが、
リゾルバーの出力に認証情報が含まれる可能性があるため表示されません。

</Accordion>

## ファイルベースの API キー

設定の `env` ブロックに `file:...` 文字列を配置しないでください。このブロックはリテラルであり上書きされないため、そこで `file:...` が解決されることはありません。

代わりに、サポートされている認証情報フィールドでファイル SecretRef を使用してください。

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

`mode: "singleValue"` の場合、SecretRef `id` は `"value"` です。`mode: "json"` の場合は、`"/providers/xai/apiKey"` などの絶対 JSON ポインターを使用してください。

SecretRef を受け入れるフィールドについては、[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)を参照してください。

## Exec 統合の例

サービスアカウント、同梱のエージェント Skills、トラブルシューティングを扱う 1Password 専用ガイドについては、[1Password](/ja-JP/gateway/1password)を参照してください。

<AccordionGroup>
  <Accordion title="1Password CLI">
    ```json5
    {
      secrets: {
        providers: {
          onepassword_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/op",
            allowSymlinkCommand: true, // Homebrew のシンボリックリンクされたバイナリに必要
            trustedDirs: ["/opt/homebrew"],
            args: ["read", "op://Personal/OpenClaw QA API Key/password"],
            passEnv: ["HOME"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="Bitwarden Secrets Manager (`bws`)">
    リゾルバーラッパーを使用して、SecretRef の id を Bitwarden Secrets Manager の項目キーにマッピングします。リポジトリには `scripts/secrets/openclaw-bws-resolver.mjs` が含まれています。Gateway を実行するホスト上の信頼できる絶対パスにインストールまたはコピーしてください。

    要件：

    - Bitwarden Secrets Manager CLI（`bws`）が Gateway ホストにインストールされていること。
    - `BWS_ACCESS_TOKEN` が Gateway サービスから利用できること。
    - `PATH` がリゾルバーに渡されていること、または `BWS_BIN` に `bws` バイナリの絶対パスが設定されていること。
    - セルフホスト型 Bitwarden インスタンスを使用する場合、環境に `BWS_SERVER_URL` が設定されていること。

    ```json5
    {
      secrets: {
        providers: {
          bws: {
            source: "exec",
            command: "/usr/local/bin/openclaw-bws-resolver.mjs",
            passEnv: ["BWS_ACCESS_TOKEN", "BWS_SERVER_URL", "PATH", "BWS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "bws",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    リゾルバーは要求された id をバッチ処理し、`bws secret list` を実行して、一致するシークレットの `key` フィールドの値を返します。`openclaw/providers/openai/apiKey` など、Exec SecretRef の id 契約を満たすキーを使用してください。アンダースコアを含む環境変数形式のキーは、リゾルバーが実行される前に拒否されます。表示可能な複数の Bitwarden シークレットが要求されたキーを共有している場合、リゾルバーは推測せず、その id を曖昧として失敗させます。設定を更新した後、リゾルバーのパスを検証してください。

    ```bash
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="HashiCorp Vault CLI">
    ```json5
    {
      secrets: {
        providers: {
          vault_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/vault",
            allowSymlinkCommand: true, // Homebrew のシンボリックリンクされたバイナリに必要
            trustedDirs: ["/opt/homebrew"],
            args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
            passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "vault_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="password-store (`pass`)">
    小さなリゾルバーラッパーを使用して、SecretRef の id を `pass` エントリに直接マッピングします。Exec プロバイダーのパスチェックを通過する絶対パス（`/usr/local/bin/openclaw-pass-resolver` など）に、実行可能ファイルとして保存してください。`#!/usr/bin/env node` シバンは、リゾルバープロセスの `PATH` から `node` を解決するため、`passEnv` に `PATH` を含めてください。`pass` がその `PATH` 上にない場合は、親環境に `PASS_BIN` を設定し、`passEnv` にも含めてください。

    ```js
    #!/usr/bin/env node
    const { spawnSync } = require("node:child_process");

    let stdin = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("data", (chunk) => {
      stdin += chunk;
    });
    process.stdin.on("error", (err) => {
      process.stderr.write(`${err.message}\n`);
      process.exit(1);
    });
    process.stdin.on("end", () => {
      let request;
      try {
        request = JSON.parse(stdin || "{}");
      } catch (err) {
        process.stderr.write(`リクエストの解析に失敗しました: ${err.message}\n`);
        process.exit(1);
      }

      const passBin = process.env.PASS_BIN || "pass";
      const values = {};
      const errors = {};

      for (const id of request.ids ?? []) {
        const result = spawnSync(passBin, ["show", id], { encoding: "utf8" });
        if (result.status === 0) {
          values[id] = result.stdout.split(/\r?\n/, 1)[0] ?? "";
        } else {
          errors[id] = { message: (result.stderr || `pass はステータス ${result.status} で終了しました`).trim() };
        }
      }

      process.stdout.write(JSON.stringify({ protocolVersion: 1, values, errors }));
    });
    ```

    次に Exec プロバイダーを設定し、`apiKey` が `pass` エントリのパスを指すようにします。

    ```json5
    {
      secrets: {
        providers: {
          pass_store: {
            source: "exec",
            command: "/usr/local/bin/openclaw-pass-resolver",
            passEnv: ["PATH", "HOME", "GNUPGHOME", "GPG_TTY", "PASSWORD_STORE_DIR", "PASS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "pass_store",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    シークレットは `pass` エントリの 1 行目に保持するか、代わりに `pass show` の出力全体を返すようにラッパーをカスタマイズしてください。設定を更新した後、静的監査と Exec リゾルバーのパスの両方を検証してください。

    ```bash
    openclaw secrets audit --check
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="sops">
    ```json5
    {
      secrets: {
        providers: {
          sops_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/sops",
            allowSymlinkCommand: true, // Homebrew のシンボリックリンクされたバイナリに必要
            trustedDirs: ["/opt/homebrew"],
            args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
            passEnv: ["SOPS_AGE_KEY_FILE"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "sops_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## MCP サーバーの環境変数

`plugins.entries.acpx.config.mcpServers` で設定された MCP サーバーの環境変数は SecretInput を受け入れ、API キーやトークンがプレーンテキスト設定に含まれないようにします。

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

プレーンテキストの文字列値も引き続き機能します。`${MCP_SERVER_API_KEY}` のような環境テンプレート参照と SecretRef オブジェクトは、MCP サーバープロセスが生成される前の Gateway アクティベーション時に解決されます。他の SecretRef サーフェスと同様に、未解決の参照によってアクティベーションがブロックされるのは、`acpx` Plugin が実質的にアクティブな場合だけです。

## サンドボックスの SSH 認証マテリアル

コアの `ssh` サンドボックスバックエンドは、SSH 認証マテリアルの SecretRef もサポートします。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

ランタイムの動作:

- OpenClaw は、SSH 呼び出しのたびに遅延解決するのではなく、サンドボックスの有効化時にこれらの参照を解決します。
- 解決された値は、制限付きのファイル権限 (`0o600`) で一時ディレクトリに書き込まれ、生成される SSH 設定で使用されます。
- 有効なサンドボックスバックエンドが `ssh` でない場合（またはサンドボックスモードが `off` の場合）、これらの参照は非アクティブのままとなり、起動を妨げません。

## サポートされる認証情報の範囲

正式にサポートされる認証情報とサポートされない認証情報は、[SecretRef 認証情報の範囲](/ja-JP/reference/secretref-credential-surface)に記載されています。

<Note>
ランタイムで発行される認証情報、ローテーションされる認証情報、および OAuth 更新用の情報は、読み取り専用の SecretRef 解決から意図的に除外されています。
</Note>

## 必須の動作と優先順位

- 参照のないフィールド: 変更されません。
- 参照のあるフィールド: アクティブな対象では、有効化時に必須です。
- 平文と参照の両方が存在する場合、サポートされる優先順位パスでは参照が優先されます。
- 秘匿化センチネル `__OPENCLAW_REDACTED__` は、内部での設定の秘匿化と復元のために予約されており、送信された設定データ内のリテラル値としては拒否されます。

警告と監査シグナル:

- `SECRETS_REF_OVERRIDES_PLAINTEXT`（ランタイム警告）
- `REF_SHADOWED`（`auth-profiles.json` 認証情報が `openclaw.json` 参照より優先される場合の監査検出事項）

Google Chat の `serviceAccount` は、インライン JSON または SecretRef を受け付けます。このフィールドが未設定の場合、Doctor は廃止された同階層の `serviceAccountRef` をこの正式なフィールドへ移動します。

## 有効化のトリガー

シークレットの有効化は、次の場合に実行されます:

- 起動時（事前検証と最終有効化）
- 設定リロードのホット適用パス
- 設定リロードの再起動チェックパス
- `secrets.reload` による手動リロード
- Gateway 設定書き込み RPC の事前検証（`config.set` / `config.apply` / `config.patch`）。編集を永続化する前に、送信された設定ペイロード内のアクティブ対象の SecretRef を検証します

有効化の契約:

- 成功すると、スナップショットがアトミックに置き換えられます。
- 厳格な起動失敗が発生すると、Gateway の起動は中止されます。
- コールドスタート中に、マッピング済みで分離可能な Gateway 以外の所有者について再試行可能な解決失敗が発生した場合、その所有者のみを設定済み利用不可としてスナップショットを公開することがあります。その所有者へのリクエストは `SECRET_SURFACE_UNAVAILABLE` で失敗します。明示的な参照が失敗した後、モデルプロバイダーの所有者が環境変数や認証プロファイルの認証情報へフォールバックすることはありません。
- リロードと再起動チェックでは、対象となるマッピング済み所有者を分離します。参照 ID、プロバイダー定義、および完全な非シークレット所有者契約がいずれも変更されていない場合、その所有者は最新の正常値を stale 状態としてそのまま保持します。変更された参照や新たに設定された未解決の参照は、その所有者のみを cold 状態で公開します。厳格なリロード失敗の場合、以前のアクティブなスナップショットが保持されます。
- `config.set`、`config.apply`、および `config.patch` は、分離可能な所有者について構文的に有効な未解決の参照を受け付け、秘匿化された `degradedSecretOwners` レポートを返します。Gateway の受信認証、構造的に無効な設定や解決値、ポリシー違反、および不明な所有者は、引き続きディスク変更前に拒否されます。
- 別の所有者が cold または stale 状態でも、正常な同階層の所有者は通常どおり解決され、公開されます。
- 送信ヘルパーやツールの呼び出しに、呼び出し単位の明示的なチャンネルトークンを指定しても、SecretRef の有効化はトリガーされません。有効化ポイントは、起動、リロード、および明示的な `secrets.reload` のままです。

## 機能低下と復旧のシグナル

正常な状態の後にリロード時の有効化が失敗すると、OpenClaw はシークレットの機能低下状態に入り、1 回限りのシステムイベントとログコードを出力します:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

動作:

- 機能低下: 正常な所有者は更新され、stale 状態の所有者は最新の正常値を維持し、cold 状態の所有者は引き続き利用できません。
- 復旧: 次回の有効化が成功した後に 1 回だけ出力されます。
- すでに機能低下状態にある間に失敗が繰り返された場合、警告はログに記録されますが、イベントは再出力されません。
- 厳格な起動失敗ではランタイムが一度もアクティブにならないため、機能低下イベントは出力されません。cold 状態の所有者を伴って起動に成功した場合は、所有者の機能低下がログに記録されますが、リローダーイベントは出力されません。
- 参照単位の起動およびリロード失敗では、影響を受ける所有者ごとに構造化された `SECRETS_DEGRADED` 警告が出力されます。プロバイダー単位の障害では、所有者ごとにプロバイダー障害を繰り返す代わりに、プロバイダーと影響を受ける所有者の完全な一覧を含む `SECRETS_PROVIDER_DEGRADED` 警告が 1 つ出力されます。警告には、秘匿化された理由、所有者の状態を示す `cold` または `stale`、および `openclaw secrets reload` の再試行ヒントが含まれます。解決された値や SecretRef ID が含まれることはありません。
- `openclaw doctor` は、cold および stale 状態の所有者について、影響を受ける設定パス、秘匿化された理由、および再試行ガイダンスを一覧表示します。

## コマンドパスでの解決

コマンドパスは、Gateway スナップショット RPC を介して、サポートされる SecretRef 解決を使用できます。大きく分けて 2 つの動作があります:

<Tabs>
  <Tab title="厳格なコマンドパス">
    たとえば、`openclaw memory` のリモートメモリパスや、リモート共有シークレット参照が必要な場合の `openclaw qr --remote` です。これらはアクティブなスナップショットから読み取り、必要な SecretRef が利用できない場合は即座に失敗します。
  </Tab>
  <Tab title="読み取り専用コマンドパス">
    たとえば、`openclaw status`、`openclaw status --all`、`openclaw channels status`、`openclaw channels resolve`、`openclaw security audit`、および読み取り専用の Doctor/設定修復フローです。これらもアクティブなスナップショットを優先しますが、対象の SecretRef が利用できない場合は中止せず、機能低下状態で処理を継続します。

    読み取り専用の動作:

    - Gateway が実行中の場合、これらのコマンドはまずアクティブなスナップショットから読み取ります。
    - Gateway での解決が不完全な場合、または Gateway が利用できない場合、そのコマンドの対象に限定したローカルフォールバックを試みます。
    - 対象の SecretRef が引き続き利用できない場合、コマンドは機能低下した読み取り専用出力と、その参照は設定済みだがこのコマンドパスでは利用できないことを示す明示的な診断を出力して処理を継続します。
    - この機能低下時の動作はそのコマンド内に限定され、ランタイムの起動、リロード、送信、または認証パスの要件を緩和するものではありません。

  </Tab>
</Tabs>

その他の注意事項:

- バックエンドのシークレットローテーション後のスナップショット更新は、`openclaw secrets reload` によって処理されます。
- これらのコマンドパスで使用される Gateway RPC メソッド: `secrets.resolve`。

## 監査と設定のワークフロー

オペレーターのデフォルトフロー:

<Steps>
  <Step title="現在の状態を監査">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
  <Step title="SecretRef を設定して適用">
    ```bash
    openclaw secrets configure --apply
    ```
  </Step>
  <Step title="再監査">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
</Steps>

再監査で問題がなくなるまで、移行が完了したと見なさないでください。監査で保存時の平文値が引き続き報告される場合、ランタイム API が秘匿化された値を返していても、エージェントによるアクセスのリスクは残ります。

`configure` の実行中に適用せずプランを保存した場合は、再監査の前に `openclaw secrets apply --from <plan-path>` を使用して保存済みプランを適用してください。

<AccordionGroup>
  <Accordion title="secrets audit">
    検出事項には次が含まれます:

    - 保存時の平文値（`openclaw.json`、`auth-profiles.json`、`.env`、および生成された `agents/*/agent/models.json`）。
    - 生成された `models.json` エントリ内に残存する、機密性の高いプロバイダーヘッダーの平文。
    - 未解決の参照。
    - 優先順位による隠蔽（`auth-profiles.json` が `openclaw.json` 参照より優先される状態）。
    - レガシーな残存データ（`auth.json`、OAuth に関する注意事項）。

    exec に関する注意: コマンドによる副作用を避けるため、監査ではデフォルトで exec SecretRef の解決可能性チェックを省略します。監査中に exec プロバイダーを実行するには、`openclaw secrets audit --allow-exec` を使用してください。

    ヘッダーの残存データに関する注意: 機密性の高いプロバイダーヘッダーの検出は、名前に基づくヒューリスティックを使用します（一般的な認証情報ヘッダー名、および `authorization`、`x-api-key`、`token`、`secret`、`password`、`credential` などの断片）。

  </Accordion>
  <Accordion title="secrets configure">
    次の処理を行う対話型ヘルパーです:

    - 最初に `secrets.providers` を設定します（`env`/`file`/`exec`、追加/編集/削除）。
    - 1 つのエージェントスコープについて、`openclaw.json` 内のサポート対象のシークレット保持フィールドと `auth-profiles.json` を選択できます。
    - 対象選択画面から新しい `auth-profiles.json` マッピングを直接作成できます。
    - SecretRef の詳細（`source`、`provider`、`id`）を取得します。
    - 事前解決を実行し、すぐに適用することもできます。

    exec に関する注意: `--allow-exec` が設定されていない限り、事前検証では exec SecretRef のチェックを省略します。`configure --apply` から直接適用し、プランに exec 参照またはプロバイダーが含まれる場合は、適用手順でも `--allow-exec` を設定したままにしてください。

    便利なモード:

    - `openclaw secrets configure --providers-only`
    - `openclaw secrets configure --skip-provider-setup`
    - `openclaw secrets configure --agent <id>`

    `configure` 適用時のデフォルト:

    - 対象のプロバイダーについて、`auth-profiles.json` から一致する静的認証情報を削除します。
    - `auth.json` からレガシーな静的 `api_key` エントリを削除します。
    - 有効な状態およびアクティブな設定の `.env` ファイルから、一致する既知のシークレット行を削除します（両方のパスが一致する場合は重複を排除します）。

  </Accordion>
  <Accordion title="secrets apply">
    保存済みプランを適用します:

    ```bash
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
    ```

    exec に関する注意: `--allow-exec` が設定されていない限り、ドライランでは exec チェックを省略します。書き込みモードでは、`--allow-exec` が設定されていない場合、exec SecretRef またはプロバイダーを含むプランを拒否します。

    厳格な対象/パス契約の詳細と正確な拒否規則については、[シークレット適用プランの契約](/ja-JP/gateway/secrets-plan-contract)を参照してください。

  </Accordion>
</AccordionGroup>

## 一方向の安全ポリシー

<Warning>
OpenClaw は、過去の平文シークレット値を含むロールバック用バックアップを意図的に書き込みません。
</Warning>

安全モデル:

- 書き込みモードの前に事前検証が成功する必要があります。
- コミット前にランタイムの有効化が検証されます。
- 適用処理では、アトミックなファイル置換を使用してファイルを更新し、失敗時にはベストエフォートで復元します。

## レガシー認証の互換性に関する注意事項

静的認証情報について、ランタイムは平文のレガシー認証ストレージに依存しなくなりました。

- ランタイムの認証情報ソースは、解決済みのメモリ内スナップショットです。
- レガシーな静的 `api_key` エントリは、検出時に削除されます。
- OAuth 関連の互換動作は別に維持されます。

## Web UI に関する注意

一部の SecretInput ユニオンは、フォームモードよりも raw エディターモードの方が簡単に設定できます。

## 関連項目

- [認証](/ja-JP/gateway/authentication) - 認証のセットアップ
- [CLI：シークレット](/ja-JP/cli/secrets) - CLI コマンド
- [Vault SecretRefs](/ja-JP/plugins/vault) - HashiCorp Vault プロバイダーのセットアップ
- [環境変数](/ja-JP/help/environment) - 環境変数の優先順位
- [SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface) - 認証情報サーフェス
- [シークレット適用プランの契約](/ja-JP/gateway/secrets-plan-contract) - プラン契約の詳細
- [セキュリティ](/ja-JP/gateway/security) - セキュリティ体制
