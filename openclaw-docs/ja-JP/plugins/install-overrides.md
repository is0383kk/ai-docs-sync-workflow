---
read_when:
    - ローカルでパッケージ化したPluginを使用したオンボーディングまたはセットアップフローのテスト
    - 公開前にプラグインパッケージを検証する
    - プラグインの自動インストールをテストアーティファクトに置き換える
sidebarTitle: Install overrides
summary: セットアップ時のインストールフローでパッケージ化されたPluginのオーバーライドをテストする
title: Plugin インストールのオーバーライド
x-i18n:
    generated_at: "2026-07-26T09:42:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: adc823f49ea9f8fa86e6a89933e43fdc309d808ac24397770495dbe81cb4b0d7
    source_path: plugins/install-overrides.md
    workflow: 16
---

Plugin インストールのオーバーライドを使用すると、メンテナーはセットアップ時の Plugin インストールで、カタログ、バンドル、またはデフォルトの npm ソースではなく、特定の npm パッケージやローカルの npm-pack tarball を指定できます。これは E2E とパッケージ検証専用です。通常のユーザーは
[`openclaw plugins install`](/ja-JP/cli/plugins)
を使用して Plugin をインストールします。

<Warning>
オーバーライドは、指定したソースの Plugin コードを実行します。隔離された状態ディレクトリまたは使い捨てのテストマシンでのみ使用してください。
</Warning>

## 環境

次の両方の変数が設定されていない場合、オーバーライドは無効です。

```bash
export OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1
export OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{
  "codex": "npm-pack:/tmp/openclaw-codex-2026.5.8.tgz",
  "openclaw-web-search": "npm:@openclaw/web-search@2026.5.8"
}'
```

オーバーライドマップは、Plugin ID をキーとする JSON です。値では次の形式を使用できます。

| プレフィックス                | ソース                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| `npm:<registry-spec>` | レジストリパッケージ、正確なバージョン、またはタグ                                                       |
| `npm-pack:<path.tgz>` | `npm pack` によって生成されたローカル tarball。相対パスは現在の作業ディレクトリを基準に解決されます |

## 動作

セットアップ時のフローで、マップに ID が含まれる Plugin をインストールすると、OpenClaw はカタログ、バンドル、またはデフォルトの npm ソースではなく、オーバーライドソースを使用します。これはオンボーディングと、共有のセットアップ時 Plugin インストーラーを使用するその他すべてのフローに適用されます。

- オーバーライドでも、想定される Plugin ID は引き続き検証されます。`codex` にマッピングされた tarball は、マニフェスト ID が `codex` である Plugin をインストールする必要があります。
- オーバーライドは、公式の信頼済みソースのステータスを継承しません。通常はカタログエントリが OpenClaw 所有のパッケージを表す場合でも、オーバーライドはオペレーターが指定したテスト入力として扱われます。
- ワークスペースの `.env` ファイルでは、インストールオーバーライドを有効にできません。両方の環境変数は、ワークスペースの dotenv ブロックリストに含まれています。OpenClaw を起動する信頼済みシェル、CI ジョブ、またはリモートテストコマンドで設定してください。

## パッケージ E2E

パッケージのインストールとインストール記録が通常の OpenClaw の状態に影響しないように、隔離された状態ディレクトリを使用してください。

```bash
npm pack extensions/codex --pack-destination /tmp

OPENCLAW_STATE_DIR="$(mktemp -d)" \
OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1 \
OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{"codex":"npm-pack:/tmp/openclaw-codex-2026.5.8.tgz"}' \
pnpm openclaw onboard --mode local
```

状態ディレクトリ内にインストールされたパッケージを確認します。

```bash
find "$OPENCLAW_STATE_DIR/npm/projects" -path '*/node_modules/@openclaw/codex/package.json' -print
grep -R '"@openclaw/codex"' "$OPENCLAW_STATE_DIR/npm/projects"/*/package-lock.json
```

ライブプロバイダーの E2E では、テストコマンドを起動する前に、信頼済みシェルまたは CI シークレットから実際の API キーを読み込んでください。キーは出力せず、ソースとキーが存在したかどうかのみを報告してください。
