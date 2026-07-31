---
read_when:
    - ローカル AI CLI バックエンド Plugin を構築しています
    - acme-cli/model のようなモデル参照用のバックエンドを登録する場合
    - サードパーティ製 CLI を OpenClaw のテキストフォールバックランナーにマッピングする必要があります
sidebarTitle: CLI backend plugins
summary: ローカル AI CLI バックエンドを登録する Plugin を構築する
title: CLI バックエンド Plugin の構築
x-i18n:
    generated_at: "2026-07-26T09:50:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI バックエンド Plugin を使用すると、OpenClaw はローカルの AI CLI をテキスト推論バックエンドとして呼び出せます。バックエンドは、モデル参照内でプロバイダープレフィックスとして表されます。

```text
acme-cli/acme-large
```

アップストリーム連携がすでにローカルコマンドとして公開されている場合、CLI がローカルのログイン状態を管理する場合、または API プロバイダーが利用できない場合のフォールバックとして、CLI バックエンドを使用します。

<Info>
  アップストリームサービスが通常の HTTP モデル API を公開している場合は、代わりに
  [プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins) を作成します。アップストリーム
  ランタイムが完全なエージェントセッション、ツールイベント、Compaction、またはバックグラウンド
  タスクの状態を管理する場合は、[エージェントハーネス](/ja-JP/plugins/sdk-agent-harness) を使用します。
</Info>

## Plugin が管理するもの

CLI バックエンド Plugin には、3 つのコントラクトがあります。

| コントラクト         | ファイル               | 目的                                                      |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| パッケージエントリ   | `package.json`         | OpenClaw に Plugin ランタイムモジュールを指定する          |
| マニフェスト所有権   | `openclaw.plugin.json` | ランタイムのロード前にバックエンド ID を宣言する          |
| ランタイム登録       | `index.ts`             | コマンドのデフォルト値を指定して `api.registerCliBackend(...)` を呼び出す |

マニフェストは検出用メタデータです。CLI を実行したり、ランタイム動作を登録したりするものではありません。ランタイム動作は、Plugin エントリが
`api.registerCliBackend(...)` を呼び出した時点で開始されます。

## 最小構成のバックエンド Plugin

<Steps>
  <Step title="パッケージメタデータを作成する">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    公開するパッケージには、ビルド済みの JavaScript ランタイムファイルを含める必要があります。ソース
    エントリが `./src/index.ts` の場合は、ビルド済みの
    JavaScript 側を指す `openclaw.runtimeExtensions` を追加します。[エントリポイント](/ja-JP/plugins/sdk-entrypoints)を参照してください。

  </Step>

  <Step title="バックエンドの所有権を宣言する">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "OpenClaw を通じて Acme のローカル AI CLI を実行する",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` はランタイムの所有権リストです。これにより、モデル選択または `agentRuntime.id` で `acme-cli` が指定された場合に、OpenClaw が
    Plugin を自動ロードできます。

    `setup.cliBackends` は、記述子を優先するセットアップサーフェスです。モデル検出、オンボーディング、またはステータスで、Plugin ランタイムをロードせずにバックエンドを認識する必要がある場合に追加します。
    セットアップにこれらの静的記述子だけで十分な場合にのみ、`requiresRuntime: false` を使用します。

  </Step>

  <Step title="バックエンドを登録する">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    バックエンド ID は、マニフェストの `cliBackends` エントリと一致する必要があります。登録された
    アダプターが正式な Plugin コードです。OpenClaw の設定はバックエンドを選択しますが、そのコマンドコントラクトを書き換えることはありません。

  </Step>
</Steps>

## 設定の構造

`CliBackendConfig` は、OpenClaw が CLI を起動して解析する方法を定義します。上記の実例では、バンドルされている
`google-gemini-cli` アダプターと同じコマンド、再開、JSONL、モデルエイリアス、セッション、画像、ウォッチドッグの各フィールドを意図的に使用しています。

| フィールド                                                | 用途                                                                              |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | バイナリ名またはコマンドの絶対パス                                               |
| `args`                                                    | 新規実行用の基本 argv                                                             |
| `resumeArgs`                                              | 再開セッション用の代替 argv。`{sessionId}` をサポート                       |
| `output` / `resumeOutput`                                 | パーサー：`json`、`jsonl`、または `text`                                                |
| `jsonlDialect`                                            | JSONL イベント方言：`claude-stream-json` または `gemini-stream-json`                 |
| `liveSession`                                             | 長時間実行される CLI プロセスモード（`claude-stdio`）                                      |
| `input`                                                   | プロンプトの転送方法：`arg` または `stdin`                                                |
| `maxPromptArgChars`                                       | `arg` モードで stdin にフォールバックするまでの最大プロンプト長                     |
| `env` / `clearEnv`                                        | 注入する追加環境変数、または起動前に削除する環境変数名                         |
| `modelArg`                                                | モデル ID の前に使用するフラグ                                                    |
| `modelAliases`                                            | OpenClaw のモデル ID を CLI ネイティブ ID にマッピングする                       |
| `sessionArgs`                                             | `{sessionId}` を使用してセッション ID を渡す方法                                      |
| `sessionMode`                                             | `always`、`existing`、または `none`                                                   |
| `sessionIdFields`                                         | OpenClaw が CLI 出力から読み取る JSON フィールド                                 |
| `systemPromptArg` / `systemPromptFileArg`                 | システムプロンプトの転送方法                                                      |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | システムプロンプトファイルを設定で上書きするための転送方法（例：`-c`）             |
| `systemPromptMode`                                        | `append` または `replace`                                                             |
| `systemPromptWhen`                                        | `first`、`always`、または `never`                                                     |
| `imageArg` / `imageMode`                                  | 画像パスのフラグと複数画像の渡し方（`repeat` または `list`）              |
| `imagePathScope`                                          | 引き渡し前にステージングされた画像ファイルを置く場所：`temp` または `workspace`               |
| `serialize`                                               | 同じバックエンドの実行順序を維持する                                             |
| `reseedFromRawTranscriptWhenUncompacted`                  | 安全なセッションリセットのため、Compaction 前に制限付きの生トランスクリプト再シードを有効にする |
| `reliability.watchdog`                                    | 出力なしタイムアウトの調整。新規実行と再開実行で個別に設定                       |

CLI に合致する最小限の静的設定を推奨します。本当にバックエンドが担うべき動作にのみ、Plugin コールバックを追加してください。

## 高度なバックエンドフック

`CliBackendPlugin` では、次の項目も定義できます。

| フック                             | 用途                                                                        |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | 登録済みの静的アダプターをランタイムコンテキストで正規化する               |
| `resolveExecutionArgs(ctx)`        | 思考労力や補助質問の分離など、リクエスト単位のフラグを追加する             |
| `prepareExecution(ctx)`            | 起動前に一時的な認証、設定、または環境のブリッジを作成する                 |
| `transformSystemPrompt(ctx)`       | CLI 固有のシステムプロンプト変換を最後に適用する                            |
| `textTransforms`                   | プロンプトと出力を双方向に置換する                                          |
| `defaultAuthProfileId`             | 特定の OpenClaw 認証プロファイルを優先する                                  |
| `authEpochMode`                    | 認証の変更によって保存済み CLI セッションを無効化する方法を決定する        |
| `nativeToolMode`                   | ネイティブツールが存在しない、常に有効、またはホストで選択可能かを宣言する |
| `toolAvailabilityEnforcement`      | 正確なツール上限を argv または実行ステージングで適用するかを宣言する       |
| `sideQuestionToolMode`             | `/btw` の補助質問で無効にするネイティブツールを宣言する                     |
| `bundleMcp` / `bundleMcpMode`      | OpenClaw のループバック MCP ツールブリッジを有効にする                      |
| `ownsNativeCompaction`             | バックエンドが独自の Compaction を管理し、OpenClaw は処理を委ねる           |
| `subscriptionAuthDispatch`         | オプトインされたサブスクリプション認証情報による組み込み実行を、このバックエンド経由で実行する |
| `runtimeArtifact`                  | スクリプトランチャーを、バンドルされた完全なパッケージツリーに限定する     |

これらのフックはプロバイダー側で管理してください。バックエンドフックで動作を表現できる場合は、コアに CLI 固有の分岐を追加しないでください。

`prepareExecution(ctx)` は、実行用に選択された有効なトークン上限である `ctx.contextTokenBudget` を受け取ります。ネイティブ Compaction を所有するバックエンドは、その予算を各 CLI 固有の起動契約にマッピングできます。

`runtimeArtifact` は Plugin が所有します。これは、ライブ推論ターンが検証済みセットアップ権限を新規発行または再検証する場合にのみ参照されます。通常の CLI 実行では必要ありません。この宣言がないバックエンドは、検証済み CLI セットアップ権限を発行できません。`bundled-package-tree` 宣言では、正確な `package.json` 所有者を指定し、パッケージのエントリポイントがコマンドであることを必須とします。OpenClaw は、ネストされた依存関係を含む、制限された完全なインストール済みパッケージツリーをハッシュし、リダイレクトするシンボリックリンク、宣言されたパッケージ外のランチャー、必須の外部依存関係宣言、サイズ超過のツリー、不明なスクリプトがある場合はフェイルクローズします。そのツリーに完全な推論実装が含まれる場合にのみ、これを宣言してください。オプションのツール統合があっても、外部実装グラフが安全になるわけではありません。

同じバックエンドが自己完結型のネイティブ実行可能ファイルも提供する場合は、その正規ベース名を `nativeExecutableNames` に列挙します。その他のネイティブコマンドは未検証のままです。

`ctx.executionMode` は、通常のターンでは `"agent"`、一時的な `/btw` 呼び出しでは `"side-question"` です。BTW のためにネイティブツール、セッションの永続化、再開動作を無効にする場合など、CLI に異なるワンショットフラグが必要なときに使用します。通常は `nativeToolMode: "always-on"` を持つバックエンドでも、補足質問用の argv によってそれらのツールが確実に無効化される場合は、`sideQuestionToolMode: "disabled"` も設定してください。そうでない場合、BTW がツールなしの CLI 実行を必要とすると、OpenClaw はフェイルクローズします。

バックエンドが実行ごとにすべてのバックエンドネイティブツールを無効化できる場合にのみ、`nativeToolMode: "selectable"` を設定してください。制限付き実行には正規契約が渡されます。`ctx.toolAvailability.native` はバックエンドネイティブツールの正確なリスト、`ctx.toolAvailability.openClaw` は OpenClaw ツール名の正確なリストです。ホストは、生成される MCP 設定と付与も、その OpenClaw リストに独立して制限します。Plugin はこれをコア内で変換したり、トランスポートプレフィックスを追加したりしてはなりません。

バックエンドがその契約をどのように適用するかを宣言します。

- `toolAvailabilityEnforcement: "execution-args"` には
  `resolveExecutionArgs` が必要です。フックは競合するツールフラグを置換し、選択されたツール以外を実行できるカスタマイズ面を無効化して、新規実行と再開実行の両方に適用 argv を返す必要があります。
- `toolAvailabilityEnforcement: "prepare-execution"` には
  `prepareExecution` が必要です。フックは実行ごとの正確なポリシーをステージングして `toolAvailabilityEnforced: true` を返す必要があります。確認応答がない場合はフェイルクローズし、OpenClaw は起動前にステージング済みリソースをクリーンアップします。

Cron の `toolsAllow` などのランタイム上限は、この契約が構築される前に OpenClaw によって正規化され、グループ展開されます。ネイティブツールは無効化され、完全に宣言された適用経路を持たないバックエンドは実行前に失敗します。

`v2026.7.2-beta.1` から `v2026.7.2-beta.3` を対象として構築された Plugin は、非推奨の `ctx.toolAvailability.mcp` トランスポート名プロジェクションを引き続き読み取れる場合があり、選択可能なバックエンドが `resolveExecutionArgs` を実装している場合は `toolAvailabilityEnforcement` を省略できる場合があります。OpenClaw は、Plugin パッケージに必須の `openclaw.build.openclawVersion` メタデータから、そのリリース済みベータ経路を認識し、`2026.8.x` 系列まで維持します。新規および更新済みの Plugin では、正規の `ctx.toolAvailability.openClaw` 名を使用し、`toolAvailabilityEnforcement: "execution-args"` を明示的に宣言してください。ベータ互換経路は、その期間の終了後に削除される予定です。

### `ownsNativeCompaction`: OpenClaw Compaction のオプトアウト

バックエンドが**独自の**トランスクリプトを Compaction するエージェントを実行する場合は、`ownsNativeCompaction: true` を設定し、OpenClaw のセーフガード要約機能がそのセッションに対して決して実行されないようにします。CLI Compaction のライフサイクルは何もせずに戻り、ターンが続行されます。Claude Code はハーネスエンドポイントを使用せずに内部で Compaction を行うため、`claude-cli` を宣言します。一方、Codex などのネイティブハーネスセッションは、引き続きハーネスの Compaction エンドポイントにルーティングされます。

**以下のすべてを満たす場合にのみ宣言してください**。そうでない場合、遅延された予算超過セッションが予算超過のままになったり、古くなったりする可能性があります（OpenClaw はそのセッションを救済しなくなります）。

- バックエンドは、ウィンドウ上限に近づくと、独自のトランスクリプトを確実に Compaction または制限する。
- Compaction 済みの状態がターンをまたいで保持されるよう、再開可能なセッションを永続化する（例: `--resume` / `--session-id`）。
- ネイティブハーネスの Compaction セッションではない。`agentHarnessId` に一致するセッションは、代わりにハーネスエンドポイントへルーティングされる。

## MCP ツールブリッジ

CLI バックエンドは、デフォルトでは OpenClaw ツールを受け取りません。CLI が MCP 設定を利用できる場合は、明示的にオプトインします。

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

サポートされるブリッジモード:

| モード                     | 用途                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | MCP 設定ファイルを受け取る CLI                              |
| `codex-config-overrides` | argv で設定オーバーライドを受け取る CLI                        |
| `gemini-system-settings` | システム設定ディレクトリから MCP 設定を読み取る CLI |

CLI が実際にブリッジを利用できる場合にのみ有効化してください。CLI に無効化できない独自の組み込みツールレイヤーがある場合は、`nativeToolMode:
"always-on"` を設定します。これにより、呼び出し元がネイティブツールなしを要求したときに OpenClaw がフェイルクローズできます。実行ごとにすべてのネイティブツールを無効化できる場合は、上記の `resolveExecutionArgs` 契約とともに `"selectable"` を使用します。

## バックエンドの選択

ユーザーは、モデル参照プレフィックスを使用してスタンドアロンバックエンドを選択します。正規の `modelProvider` を宣言するバックエンドは、代わりにそのプロバイダーモデルの `agentRuntime.id` を通じて選択できます。アダプターの仕組みは Plugin 内に保持されます。

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

認証情報は、OpenClaw の認証プロファイルまたは Plugin 所有の設定に配置します。登録済みコマンドが Gateway サービスの `PATH` 上にあることを確認してください。異なるパスまたは argv を必要とするデプロイでは、Plugin 登録を変更するかラップする必要があります。

## 検証

バンドルされた Plugin の場合は、ビルダーとセットアップ登録に焦点を絞ったテストを追加してから、Plugin の対象テストレーンを実行します。

```bash
pnpm test extensions/acme-cli
```

ローカルまたはインストール済みの Plugin の場合は、検出と実際のモデル実行を 1 回検証します。

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "正確に返信してください: backend ok" --model acme-cli/acme-large
```

バックエンドが画像または MCP をサポートする場合は、実際の CLI でそれらの経路を実証するライブスモークテストを追加します。プロンプト、画像、MCP、セッション再開の動作について、静的検査だけに依存しないでください。

## チェックリスト

<Check>`package.json` に `openclaw.extensions` があり、公開パッケージ用にビルドされたランタイムエントリがある</Check>
<Check>`openclaw.plugin.json` が `cliBackends` と意図した `activation.onStartup` を宣言している</Check>
<Check>セットアップまたはモデル検出がコールド状態でバックエンドを認識すべき場合、`setup.cliBackends` が存在する</Check>
<Check>`api.registerCliBackend(...)` がマニフェストと同じバックエンド ID を使用している</Check>
<Check>バックエンドのモデルプレフィックスまたはモデルスコープの `agentRuntime.id` が登録を選択する</Check>
<Check>セッション、システムプロンプト、画像、出力パーサーの設定が実際の CLI 契約と一致する</Check>
<Check>対象テストと少なくとも 1 回のライブ CLI スモークテストでバックエンド経路を実証する</Check>

## 関連項目

- [CLI バックエンド](/ja-JP/gateway/cli-backends) - ランタイムの選択と動作
- [Plugin の構築](/ja-JP/plugins/building-plugins) - パッケージとマニフェストの基本
- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview) - 登録 API リファレンス
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - `cliBackends` とセットアップ記述子
- [エージェントハーネス](/ja-JP/plugins/sdk-agent-harness) - 完全な外部エージェントランタイム
