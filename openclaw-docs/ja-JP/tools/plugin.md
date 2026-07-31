---
doc-schema-version: 1
read_when:
    - プラグインのインストールまたは設定
    - Plugin の検出と読み込みルールを理解する
    - Codex/Claude 互換 Plugin バンドルの操作
sidebarTitle: Getting Started
summary: OpenClaw Plugin のインストール、設定、管理
title: プラグイン
x-i18n:
    generated_at: "2026-07-26T09:23:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f210dccab059527192eeb0aa2e780dcea243959273938ffaacc867ec96f5085e
    source_path: tools/plugin.md
    workflow: 16
---

Plugin は、チャネル、モデルプロバイダー、エージェントハーネス、ツール、
Skills、音声、リアルタイム文字起こし、音声機能、メディア理解、生成、
Web フェッチ、Web 検索、その他のランタイム機能を OpenClaw に追加します。

このページでは、Plugin のインストール、Gateway の再起動、ランタイムに
読み込まれたことの確認、および一般的なセットアップ失敗への対処方法を説明します。コマンドのみの例については、
[Plugin を管理する](/ja-JP/plugins/manage-plugins)を参照してください。バンドル済み、公式外部、およびソース専用の
Plugin について生成された一覧は、
[Plugin 一覧](/ja-JP/plugins/plugin-inventory)を参照してください。

## 要件

- an OpenClaw checkout or installation with the `openclaw` CLI available
- network access to the selected source (ClawHub, npm, or a git host)
- any plugin-specific credentials, config keys, or OS tools named by that
  plugin's setup docs
- permission for the Gateway that serves your channels to reload or restart

## クイックスタート

<Steps>
  <Step title="Plugin を探す">
    公開 Plugin パッケージを [ClawHub](/ja-JP/clawhub) で検索します。

    ```bash
    openclaw plugins search "calendar"
    ```

    ClawHub は、コミュニティ Plugin を見つけるための主要な場所です。移行開始期間中は、
    通常のプレフィックスなしパッケージ指定は、公式 Plugin ID と一致しない限り、
    引き続き npm からインストールされます。バンドル済み Plugin と一致する未加工の `@openclaw/*` 指定は、
    そのバンドル済みコピーに解決されます。特定のソースを明示的に使用する必要がある場合は、
    ソースプレフィックスを指定してください。

  </Step>

  <Step title="Plugin をインストールする">
    ```bash
    # ClawHub から。
    openclaw plugins install clawhub:<package>

    # npm から。
    openclaw plugins install npm:<package>

    # git から。
    openclaw plugins install git:github.com/<owner>/<repo>@<ref>

    # ローカルの開発用チェックアウトから。
    openclaw plugins install ./my-plugin
    openclaw plugins install --link ./my-plugin
    ```

    Plugin のインストールは、コードの実行と同様に扱ってください。再現可能な本番環境への
    インストールには、固定バージョンを推奨します。ClawHub パッケージと OpenClaw の
    バンドル済み／公式カタログは信頼できるソースです。新しい任意の npm、git、
    ローカルパス／アーカイブ、`npm-pack:`、またはマーケットプレイスのソースでは、
    ソースを確認して信頼した後、非対話型インストール時に
    `--force` が必要です。

  </Step>

  <Step title="設定して有効化する">
    Plugin 固有の設定を `plugins.entries.<id>.config` の下に構成します。
    Plugin がまだ有効でない場合は、有効化します。

    ```bash
    openclaw plugins enable <plugin-id>
    ```

    `plugins.allow` が設定されている場合、Plugin を読み込むには、インストール済み Plugin ID が
    そのリストに含まれている必要があります。`openclaw plugins install` は、既存の
    `plugins.allow` リストにインストール済み ID を追加し、
    `plugins.deny` から同じ ID を削除するため、明示的にインストールした Plugin を再起動後に読み込めます。

  </Step>

  <Step title="Gateway を再読み込みさせる">
    Plugin コードをインストール、更新、またはアンインストールした場合は、Gateway の
    再起動が必要です。設定の再読み込みが有効な管理対象 Gateway は、変更された
    Plugin インストール記録を検出し、自動的に再起動します。それ以外の場合は、
    手動で再起動してください。

    ```bash
    openclaw gateway restart
    ```

    有効化／無効化では、設定とコールドレジストリが更新されます。ライブのランタイムサーフェスを確認するには、
    引き続きランタイム検査が最も明確な証拠となります。

  </Step>

  <Step title="ランタイム登録を確認する">
    ```bash
    openclaw plugins inspect <plugin-id> --runtime --json
    ```

    登録済みのツール、フック、サービス、Gateway メソッド、または Plugin が所有する
    CLI コマンドを確認するには、`--runtime` を使用します。通常の `inspect` は、
    コールドマニフェストとレジストリのみを確認します。

  </Step>
</Steps>

## 設定

### インストールソースを選択する

| ソース      | 使用する場合                                                                       | 例                                                        |
| ----------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| ClawHub     | OpenClaw ネイティブの検索、スキャン、バージョンメタデータ、インストールヒントが必要な場合 | `openclaw plugins install clawhub:<package>`                   |
| npm         | npm レジストリまたは dist-tag のワークフローを直接使用する必要がある場合                             | `openclaw plugins install npm:<package>`                       |
| git         | リポジトリのブランチ、タグ、またはコミットが必要な場合                            | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| ローカルパス  | 同じマシン上で Plugin を開発またはテストしている場合                     | `openclaw plugins install --link ./my-plugin`                  |
| マーケットプレイス | Claude 互換のマーケットプレイス Plugin をインストールする場合                      | `openclaw plugins install <plugin> --marketplace <source>`     |

プレフィックスなしパッケージ指定には、特別な互換動作があります。バンドル済み Plugin ID と
一致するプレフィックスなしの名前は、そのバンドル済みソースを使用します。公式外部 Plugin ID と一致する
プレフィックスなしの名前は、公式パッケージカタログを使用します。その他のプレフィックスなし指定は、
移行開始期間中は npm 経由でインストールされます。バンドル済み Plugin と一致する未加工の `@openclaw/*`
指定も、npm にフォールバックする前にバンドル済みコピーに解決されます。バンドル済みコピーではなく
外部 npm パッケージを意図的にインストールするには、`npm:@openclaw/<plugin>@<version>` を使用します。
ソースを確定的に選択するには、`clawhub:`、`npm:`、
`git:`、または `npm-pack:` を使用します。コマンドの完全な仕様については、
[`openclaw plugins`](/ja-JP/cli/plugins#install)を参照してください。

npm インストールでは、固定されていない指定と `@latest` は、この OpenClaw ビルドとの
互換性を示す最新の安定版パッケージを選択します。npm の現在の最新リリースで、このビルドがサポートするものより
新しい `openclaw.compat.pluginApi` または `openclaw.install.minHostVersion` が宣言されている場合、
OpenClaw は過去の安定版を走査し、適合する最新バージョンをインストールします。正確なバージョンと、
`@beta` などの明示的なチャネルタグは、選択したパッケージに固定されたままとなり、
互換性がない場合は失敗します。

### オペレーターのインストールポリシー

Plugin のインストールまたは更新を続行する前に、信頼できるローカルポリシーコマンドを実行するよう
`security.installPolicy` を設定します。このポリシーは、メタデータと
ステージング済みソースパスを受け取り、インストールを許可またはブロックできます。CLI と
Gateway 経由の両方のインストール／更新パスが対象です。Plugin の `before_install` フックは
その後に実行され、Plugin フックが読み込まれている OpenClaw プロセス内でのみ動作するため、
オペレーターが管理するインストール判断には、代わりに `security.installPolicy` を使用してください。
非推奨の `--dangerously-force-unsafe-install` フラグは互換性のために受け付けられますが、
何も行いません。インストールポリシーや OpenClaw 組み込みの Plugin 依存関係拒否リストを
回避するものではありません。

Skills と Plugin の両方で使用される共通の `security.installPolicy` exec スキーマについては、
[Skills の設定](/ja-JP/tools/skills-config#operator-install-policy-securityinstallpolicy)を参照してください。

### Plugin ポリシーを設定する

一般的な Plugin 設定の形式は次のとおりです。

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-plugin"] },
    slots: { memory: "memory-core" },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

主なポリシールール:

- `plugins.enabled: false` disables all plugins and skips discovery/load
  work. Stale plugin references stay inert while this is active; re-enable
  plugins before running doctor cleanup if you want stale ids removed.
- `plugins.deny` wins over allow and per-plugin enablement.
- `plugins.allow` is an exclusive allowlist. Plugin-owned tools outside the
  allowlist stay unavailable even when `tools.allow` includes `"*"`.
- `plugins.entries.<id>.enabled: false` disables one plugin while keeping its
  config.
- `plugins.load.paths` adds explicit local plugin files or directories.
  Managed `plugins install` local paths must be plugin directories or
  archives; use `plugins.load.paths` for standalone plugin files.
- Workspace-origin plugins are disabled by default; explicitly enable or
  allowlist them before using local workspace code.
- Bundled plugins follow their built-in default-on/default-off metadata
  unless config explicitly overrides it.
- `plugins.slots.<slot>` (`memory` or `contextEngine`) picks one plugin for an
  exclusive category. Slot selection counts as explicit activation and
  force-enables the selected plugin for that slot, even if it would otherwise
  be opt-in. `plugins.deny` and `plugins.entries.<id>.enabled: false` still
  block it.
- Bundled opt-in plugins can auto-activate when config names one of their
  owned surfaces, such as a provider/model ref, channel config, CLI backend,
  or agent harness runtime.
- OpenAI-family Codex routing keeps provider and runtime plugin boundaries
  separate: legacy Codex model refs are legacy config that doctor repairs,
  while the bundled `codex` plugin owns Codex app-server runtime for
  canonical `openai/*` agent refs, explicit `agentRuntime.id: "codex"`, and
  legacy `codex/*` refs.

`plugins.allow` が未設定で、バンドルされていない Plugin がワークスペースまたは
グローバル Plugin ルートから自動検出された場合、起動ログに
`plugins.allow is empty; discovered non-bundled plugins may auto-load: ...`
と、リストが短い場合は最小限の `plugins.allow`
スニペットが記録されます。信頼できる Plugin を `openclaw.json` にコピーする前に、
一覧に示された Plugin ID に対して [`openclaw plugins list --enabled --verbose`](/ja-JP/cli/plugins#list)
または [`openclaw plugins inspect <id>`](/ja-JP/cli/plugins#inspect) を実行してください。診断で Plugin が
`without install/load-path provenance` を読み込んだと表示された場合も、同じように信頼を固定します。
その Plugin ID を検査してから、`plugins.allow` に固定するか、
OpenClaw がインストール元情報を記録できるよう、信頼できるソースから再インストールしてください。

設定検証で古い Plugin ID、許可リストとツールの不一致、または従来のバンドル済み Plugin
パスが報告された場合は、`openclaw doctor` または `openclaw doctor --fix` を実行してください。

## Plugin 形式を理解する

OpenClaw は、次の 2 つの Plugin 形式を認識します。

| 形式                 | 読み込み方法                                                                 | 使用する場合                                                               |
| ---------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| ネイティブ OpenClaw Plugin | `openclaw.plugin.json` と、プロセス内に読み込まれるランタイムモジュール               | OpenClaw 固有のランタイム機能をインストールまたは構築する場合  |
| 互換バンドル      | OpenClaw の Plugin 一覧にマッピングされた Codex、Claude、または Cursor の Plugin レイアウト | 互換性のある Skills、コマンド、フック、またはバンドルメタデータを再利用する場合 |

どちらの形式も `openclaw plugins list`、`openclaw plugins inspect`、
`openclaw plugins enable`、および `openclaw plugins disable` に表示されます。バンドルの互換性境界については
[Plugin バンドル](/ja-JP/plugins/bundles)を、ネイティブ Plugin の作成については
[Plugin の構築](/ja-JP/plugins/building-plugins)を参照してください。

## Plugin フック

Plugin は、2 つの異なる API を通じて実行時にフックを登録できます。

- `api.on(...)` typed hooks for runtime lifecycle events. This is the
  preferred surface for middleware, policy, message rewriting, prompt
  shaping, and tool control.
- `api.registerHook(...)` for the internal hook system described in
  [Hooks](/ja-JP/automation/hooks). This is mainly for coarse command/lifecycle side
  effects and compatibility with existing HOOK-style automation.

簡単な判断基準: ハンドラーに優先順位、マージセマンティクス、または
ブロック／キャンセル動作が必要な場合は、型付きフックを使用します。`command:new`、
`command:reset`、`message:sent`、または同様の大まかなイベントに反応するだけなら、
`api.registerHook` で問題ありません。

Plugin が管理する内部フックは、`openclaw hooks list` に
`plugin:<id>` とともに表示されます。`openclaw hooks` を通じて有効化または無効化することはできません。
代わりに Plugin を有効化または無効化してください。

## アクティブな Gateway を確認する

`openclaw plugins list` と通常の `openclaw plugins inspect` は、コールド状態の設定、
マニフェスト、レジストリの状態を読み取ります。すでに実行中の
Gateway が同じプラグインコードをインポート済みであることは証明しません。

プラグインがインストール済みと表示されるものの、ライブチャットのトラフィックで使用されない場合:

```bash
openclaw gateway status --deep --require-rpc
openclaw plugins inspect <plugin-id> --runtime --json
openclaw gateway restart
```

管理対象の Gateway は、プラグインのソースを変更するインストール、更新、
アンインストールの後に自動的に再起動します。VPS またはコンテナへのインストールでは、
手動再起動の対象がラッパーやスーパーバイザーだけではなく、チャンネルを提供する実際の
`openclaw gateway run` 子プロセスであることを確認してください。

## トラブルシューティング

| 症状                                                        | 確認事項                                                                                                                                      | 修正方法                                                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| プラグインが `plugins list` に表示されるが、ランタイムフックが実行されない  | `openclaw plugins inspect <id> --runtime --json` を使用し、`gateway status --deep --require-rpc` でアクティブな Gateway を確認する             | インストール、更新、設定、またはソースの変更後に稼働中の Gateway を再起動する                               |
| チャンネルまたはツールの所有権重複に関する診断が表示される         | `openclaw plugins list --enabled --verbose` を実行し、疑わしい各プラグインを `--runtime --json` で調査して、チャンネル／ツールの所有権を比較する | 一方の所有者を無効化するか、古いインストールを削除する。意図的に置き換える場合はマニフェストの `preferOver` を使用する      |
| 設定でプラグインが見つからないと表示される                                | [プラグイン一覧](/ja-JP/plugins/plugin-inventory)で、同梱、公式外部、ソース限定のいずれであるかを確認する                           | 外部パッケージをインストールするか、同梱プラグインを有効化するか、古い設定を削除する                         |
| インストール中に設定が無効になる                               | 検証メッセージを読み、古いプラグイン状態が示されている場合は `openclaw doctor --fix` を実行する                                             | Doctor は、エントリを無効化して無効なペイロードを削除することにより、無効なプラグイン設定を隔離できる     |
| 不審な所有権または権限のためプラグインパスがブロックされる | 設定エラーの前に表示される診断を確認する                                                                                             | ファイルシステムの所有権／権限を修正してから、`openclaw plugins registry --refresh` を実行する                    |
| `OPENCLAW_NIX_MODE=1` がライフサイクルコマンドをブロックする                | インストールが Nix によって管理されていることを確認する                                                                                                      | プラグイン変更コマンドを使用せず、Nix ソース内でプラグインの選択を変更する                      |
| ランタイムで依存関係のインポートに失敗する                             | プラグインが npm/git/ClawHub 経由でインストールされたか、ローカルパスから読み込まれたかを確認する                                                 | `openclaw plugins update <id>` を実行するか、ソースを再インストールするか、ローカルプラグインの依存関係を自身でインストールする |

有効化された管理対象プラグインが Gateway の起動中にペイロード検証に失敗すると、
OpenClaw はその起動中、該当するインストール済みプラグインのルートのみを隔離し、
他のプラグインの提供を継続します。`openclaw status --all`、`openclaw health`、
`openclaw doctor` はこれを `configured-unavailable` として報告します。プラグインを
修正または再インストールしてから、Gateway を再起動してください。同じプラグイン ID を持つ
正常で明示的な `plugins.load.paths` オーバーライドは、古い破損したインストールによって隔離されません。

古いプラグイン設定が、すでに検出できないチャンネルプラグインを引き続き指定している場合、
設定検証ではそのチャンネルキーを重大なエラーではなく警告に格下げするため、
Gateway の起動時にも他のすべてのチャンネルを提供できます。
`openclaw doctor --fix` を実行して、古いプラグインおよびチャンネルのエントリを削除してください。
古いプラグインの証拠がない不明なチャンネルキーは引き続き検証に失敗するため、
入力ミスを認識できます。

意図的にチャンネルを置き換える場合、優先するプラグインでは、従来のプラグイン ID または
優先度の低いプラグイン ID を指定して `channelConfigs.<channel-id>.preferOver` を宣言する必要があります。
両方のプラグインが明示的に有効化されている場合、OpenClaw は一方の所有者を暗黙に選択せず、
その要求を維持してチャンネル／ツールの所有権重複に関する診断を報告します。

インストール済みパッケージで `requires compiled runtime output for
TypeScript entry ...` と報告される場合、そのパッケージは
OpenClaw がランタイムで必要とする JavaScript ファイルを含めずに公開されています。
公開者がコンパイル済み JavaScript をリリースした後に更新または再インストールするか、
それまではプラグインを無効化またはアンインストールしてください。

### ブロックされたプラグインパスの所有権

診断に
`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`
と表示され、その後の検証で `plugin present but blocked` と表示される場合、OpenClaw は
プラグインファイルが、それを読み込むプロセスとは異なる Unix ユーザーによって
所有されていることを検出しています。プラグイン設定はそのまま維持し、
ファイルシステムの所有権を修正するか、状態ディレクトリを所有するユーザーと
同じユーザーで OpenClaw を実行してください。

Docker へのインストールでは、公式イメージは `node`（uid `1000`）として実行されるため、
ホストからバインドマウントされた OpenClaw の設定ディレクトリとワークスペースディレクトリは、
通常 uid `1000` によって所有されている必要があります。

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

OpenClaw を意図的に root として実行する場合は、代わりに管理対象プラグインのルートを
root 所有に修正してください。

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
```

所有権を修正した後、`openclaw doctor --fix` または
`openclaw plugins registry --refresh` を再実行し、永続化されたプラグインレジストリを
修正済みのファイルと一致させてください。

### プラグインツールのセットアップが遅い場合

ツールの準備中にエージェントのターンが停止しているように見える場合は、
トレースログを有効にし、プラグインツールファクトリの所要時間を示す行を確認してください。

```bash
openclaw config set logging.level trace
openclaw logs --follow
```

次の行を探します。

```text
[trace:plugin-tools] factory timings ...
```

概要には、ファクトリの合計所要時間と、最も遅いプラグインツールファクトリが一覧表示されます。
これにはプラグイン ID、宣言されたツール名、結果の形状、およびツールがオプションかどうかが含まれます。
単一のファクトリに少なくとも 1s かかる場合、またはプラグインツールファクトリの準備全体に
少なくとも 5s かかる場合、遅延を示す行は警告に昇格します。

OpenClaw は、同じ実効リクエストコンテキストで解決を繰り返す場合に備えて、
成功したプラグインツールファクトリの結果をキャッシュします。キャッシュキーには、
実効ランタイム設定、ワークスペースおよびエージェント ID、サンドボックスポリシー、
ブラウザ設定、配信コンテキスト、リクエスト元のアイデンティティ、所有権の状態が含まれるため、
これらの信頼されたフィールドに依存するファクトリはコンテキストが変わると再実行されます。
所要時間が長いままの場合、プラグインがツール定義を返す前に
コストの高い処理を行っている可能性があります。

1 つのプラグインが所要時間の大半を占める場合は、そのランタイム登録を調査します。

```bash
openclaw plugins inspect <plugin-id> --runtime --json
```

次に、そのプラグインを更新、再インストール、または無効化します。プラグインの作成者は、
コストの高い依存関係の読み込みをツールファクトリ内で行わず、
ツール実行パスの後段に移す必要があります。

依存関係のルート、パッケージメタデータの検証、レジストリレコード、起動時の再読み込み動作、
従来データのクリーンアップについては、
[プラグインの依存関係解決](/ja-JP/plugins/dependency-resolution)を参照してください。

## 関連項目

- [プラグインの管理](/ja-JP/plugins/manage-plugins) - 一覧表示、インストール、更新、アンインストール、公開のコマンド例
- [`openclaw plugins`](/ja-JP/cli/plugins) - CLI の完全なリファレンス
- [プラグイン一覧](/ja-JP/plugins/plugin-inventory) - 生成された同梱および外部プラグインの一覧
- [プラグインリファレンス](/ja-JP/plugins/reference) - 生成されたプラグインごとのリファレンスページ
- [コミュニティプラグイン](/ja-JP/plugins/community) - ClawHub での検索とドキュメント PR のポリシー
- [プラグインの依存関係解決](/ja-JP/plugins/dependency-resolution) - インストールルート、レジストリレコード、ランタイム境界
- [プラグインの構築](/ja-JP/plugins/building-plugins) - ネイティブプラグイン作成ガイド
- [プラグイン SDK の概要](/ja-JP/plugins/sdk-overview) - ランタイム登録、フック、API フィールド
- [プラグインマニフェスト](/ja-JP/plugins/manifest) - マニフェストとパッケージメタデータ
