---
read_when:
    - 認証情報、デバイス、またはエージェントのデフォルト設定を対話形式で調整したい場合
summary: '`openclaw configure` の CLI リファレンス（対話形式の設定プロンプト）'
title: 設定する
x-i18n:
    generated_at: "2026-07-26T09:55:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5980d06e75a5df9e5269d0ef78431f730d6f5fd050dca74784ef3426fb0433d8
    source_path: cli/configure.md
    workflow: 16
---

# `openclaw configure`

既存のセットアップに対象を絞った変更を行うための対話型プロンプトです。対象には、認証情報、デバイス、エージェントのデフォルト、Gateway、チャンネル、Plugin、Skills、ヘルスチェックが含まれます。

初回実行時の完全なガイド付きフローには `openclaw onboard` または `openclaw setup`、基本設定とワークスペースのみには `openclaw setup --baseline`、チャンネルアカウントのセットアップのみが必要な場合は `openclaw channels add` を使用します。

<Tip>
サブコマンドなしの `openclaw config` でも同じウィザードが開きます。非対話型の編集には `openclaw config get|set|unset` を使用します。
</Tip>

## オプション

`--section <section>`: 繰り返し指定可能なセクションフィルターです。使用可能なセクション:

`workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```

`gateway`、`daemon`、`health` のいずれかを選択すると（または `--section` なしで完全なウィザードを実行すると）、Gateway の実行場所を尋ね、`gateway.mode` を更新します。3 つすべてをスキップするセクションフィルターを指定した場合、Gateway モードを尋ねることなく、要求されたセットアップに直接進みます。リモート Gateway モードを選択すると、リモート設定を書き込んですぐに終了します。Plugin のインストールなど、ローカル専用の手順は実行されません。

<Note>
`openclaw configure` には対話型ターミナルが必要です（stdin と stdout の両方が TTY である必要があります）。対話型ターミナルがない場合は、途中まで実行する代わりに、同等の非対話型 `openclaw config get|set|patch|validate` コマンドを表示し、エラーで終了します。
</Note>

## モデルセクション

<Note>
**モデル**には、明示的な `agents.defaults.modelPolicy.allow` リスト（`/model` とモデルピッカーに表示される項目）の複数選択が含まれます。プロバイダー単位のセットアップで選択したモデルは、設定内にすでに存在する無関係なプロバイダーを置き換えず、既存のリストに統合されます。モデルごとのエイリアスとパラメーターは引き続き `agents.defaults.models` で管理されます。これらのエントリ自体がモデルのオーバーライドを制限することはありません。

configure からプロバイダー認証を再実行した場合、プロバイダーの認証手順が独自の推奨デフォルトモデルを含む設定パッチを返しても、既存の `agents.defaults.model.primary` は維持されます。プロバイダーを追加または再認証すると、そのモデルが使用可能になりますが、現在のプライマリモデルが置き換えられることはありません。デフォルトモデルを意図的に変更するには、`openclaw models auth login --provider <id> --set-default` または `openclaw models set <model>` を使用します。
</Note>

configure をプロバイダー認証の選択肢から開始すると、デフォルトモデルとモデルポリシーのピッカーでは、そのプロバイダーが自動的に優先されます。Volcengine と BytePlus のような対になったプロバイダーでは、同じ優先設定がコーディングプランのバリアント（`volcengine-plan/*`、`byteplus-plan/*`）にも一致します。優先プロバイダーのフィルターによってリストが空になる場合、空のピッカーを表示する代わりに、フィルターされていないカタログへフォールバックします。

## Web セクション

`openclaw configure --section web` では Web 検索プロバイダーを選択し、その認証情報を設定します。一部のプロバイダーでは、プロバイダー固有の追加設定が表示されます。

- **Grok**では、同じ xAI OAuth プロファイルまたは API キーを使用するオプションの `x_search` セットアップが提示され、`x_search` モデルを選択できます。
- **Kimi**では、Moonshot API のリージョン（`api.moonshot.ai` または `api.moonshot.cn`）とデフォルトの Kimi Web 検索モデルを尋ねる場合があります。

## その他の注意事項

- ローカル設定の書き込み後、選択したセットアップパスで必要な場合は、選択されたダウンロード可能な Plugin が configure によってインストールされます。リモート Gateway の設定では、ローカルの Plugin パッケージはインストールされません。
- チャンネル指向のサービス（Slack/Discord/Matrix/Microsoft Teams）では、セットアップ中にチャンネルまたはルームの許可リストを尋ねます。名前または ID を入力できます。可能な場合、ウィザードは名前を ID に解決します。
- デーモンのインストール手順を実行する場合、トークン認証にはトークンが必要です。`gateway.auth.token` が SecretRef で管理されている場合、configure は SecretRef を検証しますが、解決された平文のトークン値をスーパーバイザーサービスの環境メタデータに永続化しません。SecretRef を解決できない場合、configure は、実行可能な修正ガイダンスを提示してデーモンのインストールをブロックします。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、モードを明示的に設定するまで configure はデーモンのインストールをブロックします。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [設定](/ja-JP/gateway/configuration)
- 設定 CLI: [Config](/ja-JP/cli/config)
