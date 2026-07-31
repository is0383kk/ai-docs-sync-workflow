---
read_when:
    - モデルのフォールバック動作または選択 UX の変更
    - 「model is not allowed」または古いデフォルトプロバイダーへのフォールバックをデバッグする
    - models.json のマージ／シークレット動作に取り組む
sidebarTitle: Models CLI
summary: OpenClaw がプロバイダー／モデル参照、設定キー、`/model` チャットコマンドを解決する仕組み
title: モデル CLI
x-i18n:
    generated_at: "2026-07-26T09:33:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cd13a2aae6575bdfeefb477b7fe8be740b77c66cb76454b07d82481f6612152
    source_path: concepts/models.md
    workflow: 16
---

<CardGroup cols={2}>
  <Card title="モデルのフェイルオーバー" href="/ja-JP/concepts/model-failover">
    認証プロファイルのローテーション、クールダウン、およびフォールバックとの連動。
  </Card>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers">
    プロバイダーの概要と例。
  </Card>
  <Card title="モデル CLI リファレンス" href="/ja-JP/cli/models">
    `openclaw models` のコマンドとフラグの完全なリファレンス。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/config-agents#agent-defaults">
    モデル設定キー、デフォルト、および例。
  </Card>
</CardGroup>

モデル参照（`provider/model`）は、低レベルのエージェントランタイムではなく、プロバイダーとモデルを選択します。ランタイムポリシーが未設定または `auto` の場合、OpenAI のプロバイダー所有ルートポリシーが Codex を選択できるのは、作成者によるリクエストオーバーライドがなく、公式 HTTPS Platform Responses または ChatGPT Responses のルートと完全に一致する場合のみです。`openai/*` プレフィックスだけでは、Codex が選択されることはありません。Completions アダプター、カスタムエンドポイント、および作成者が指定したリクエスト動作では、OpenClaw が引き続き使用されます。公式の平文 HTTP エンドポイントは拒否されます。[OpenAI の暗黙的エージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)を参照してください。

サブスクリプション版 Copilot の参照（`github-copilot/*`）では、外部の GitHub Copilot エージェントランタイム Plugin をオプトインできますが、この経路は常に明示的です（`auto` によって選択されることはありません）。ランタイムオーバーライドは、エージェントまたはセッション全体ではなく、プロバイダー／モデルポリシーに設定します。ランタイムの選択によって課金方法が決まるわけではありません。OpenAI API キーの認証情報と ChatGPT/Codex サブスクリプションの認証情報は、引き続き別個です。[エージェントランタイム](/ja-JP/concepts/agent-runtimes)および [GitHub Copilot エージェントランタイム](/ja-JP/plugins/copilot)を参照してください。

## 選択順序

<Steps>
  <Step title="プライマリモデル">
    `agents.defaults.model.primary`（またはプレーン文字列としての `agents.defaults.model`）。
  </Step>
  <Step title="フォールバック">
    `agents.defaults.model.fallbacks`。順番に試行されます。
  </Step>
  <Step title="認証フェイルオーバー">
    OpenClaw が次のフォールバックモデルへ移る前に、プロバイダー内で認証プロファイルのローテーションが行われます。
  </Step>
</Steps>

関連するモデル設定サーフェス：

- `agents.defaults.models` は、エイリアスとモデルごとの設定を保存します。エントリを追加しても、モデルオーバーライドは制限されません。
- `agents.defaults.modelPolicy.allow` は、任意のオーバーライド許可リストです。完全一致の参照、または `provider/*` や `provider/namespace/*` のような末尾プレフィックスワイルドカードを使用します。任意のモデルを許可するには、省略するか `[]` に設定します。エージェントごとの `agents.entries.*.modelPolicy.allow` は、そのエージェントのデフォルトポリシーを置き換えます。
- `agents.defaults.utilityModel` は、生成されるダッシュボードセッションタイトル、対応チャネルのスレッド／トピックタイトル、進捗説明など、短い内部タスクに使用する任意の低コストモデルです。エージェントごとの `agents.entries.*.utilityModel` でオーバーライドできます。未設定の場合、OpenClaw はプライマリプロバイダーが宣言した小規模モデルのデフォルトがあればそれを使用し（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）、なければエージェントのプライマリモデルを使用します。ユーティリティルーティングを無効にするには、空文字列に設定します。生成タイトルでは、別個のユーティリティモデルが失敗した場合、プライマリモデルで 1 回再試行します。ダッシュボードタイトルでは、自動ユーティリティ導出と通常のフォールバックが、有効なセッションプロバイダーと認証プロファイルに従います。明示的なユーティリティモデルでは、設定されたプロバイダー／認証が維持されます。ユーティリティモデルが空でも、代替の小規模モデル経路だけが省略され、ダッシュボードタイトルの生成自体は省略されません。ユーティリティタスクは別個のモデル呼び出しであり、制限されたタスク内容が選択したモデルプロバイダーへ送信される場合があります。
- `agents.defaults.imageModel` は、プライマリモデルが画像を受け付けられない場合にのみ使用されます。
- `agents.defaults.pdfModel` は、`pdf` ツールで使用されます。未設定の場合、ツールは `imageModel`、次に解決済みのセッション／デフォルトモデルへフォールバックします。
- `agents.defaults.mediaModels.{image,music,video}` は、共有メディア生成ツールを支えます。未設定の場合、各ツールは認証済みのプロバイダーデフォルトを推測します。現在のデフォルトプロバイダーを最初に使用し、次にその機能について登録されている残りのプロバイダーをプロバイダー ID 順に使用します。プロバイダーをまたぐフォールバックは固定のデフォルト動作です。
- エージェントごとの `agents.entries.*.model`（およびバインディング）は `agents.defaults.model` をオーバーライドします。[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)を参照してください。

キーの完全なリファレンス、デフォルト、および JSON5 の例：[設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults)。

## 選択元とフォールバックの厳格性

同じ `provider/model` でも、その取得元によって動作が異なります。

| 取得元                                                                  | 動作                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 設定済みのデフォルト（`agents.defaults.model.primary`、エージェントごとのプライマリ） | 通常の開始点。`agents.defaults.model.fallbacks` を使用します。                                                                                                                                                                                                 |
| 自動フォールバック                                                           | 一時的な復旧状態で、`modelOverrideSource: "auto"` として保存されます。OpenClaw は元のプライマリを定期的に再プローブし、復旧すると自動選択を解除し、フォールバック／復旧の遷移を状態変更ごとに 1 回通知します。                              |
| ユーザーによるセッション選択                                                  | 完全一致かつ厳格です。`/model`、モデルピッカー、`session_status(model=...)`、および `sessions.patch` は `modelOverrideSource: "user"` を保存します。そのプロバイダー／モデルに到達できなくなった場合、別の設定済みモデルへ移行せず、実行は明示的なエラーで失敗します。 |
| Cron `--model`／ペイロード `model`                                        | ジョブごとのプライマリです。ジョブが独自のペイロード `fallbacks` を指定しない限り、設定済みのフォールバックも使用します（`fallbacks: []` は厳格な実行を強制します）。                                                                                                                    |

その他の選択ルール：

- `agents.defaults.model.primary` を変更しても、既存のセッション固定は書き換えられません。ステータスに `This session is pinned to X; config primary Y will apply to new/unpinned sessions.` と表示される場合は、`/model default` を実行して固定を解除します。
- CLI のデフォルトモデルおよび許可リストのピッカーは、完全な組み込みカタログではなく `models.providers.*.models` のみを一覧表示することで、`models.mode: "replace"` に従います。
- Control UI のモデルピッカーは、設定済みモデルビューを Gateway に要求します。明示的な `modelPolicy.allow` がある場合、末尾プレフィックスワイルドカードのエントリも含め、それに基づいてフィルタリングします。それ以外の場合は、設定済みモデルと、使用可能な認証を持つプロバイダーが表示されます。完全な組み込みカタログは、明示的な参照ビュー（`models.list` と `view: "all"`、または `openclaw models list --all`）専用です。
- プロバイダーインベントリ UI は、ピッカーの許可リストを適用せず、ソースで定義された `models.providers.*.models` 行を表示するために、`models.list` と `view: "provider-config"` を使用します。

詳細な仕組み：[モデルのフェイルオーバー](/ja-JP/concepts/model-failover)。

## クイックモデルポリシー

- 利用できる最新世代の最も強力なモデルをプライマリに設定します。
- コスト／レイテンシー重視のタスクや重要度の低いチャットには、フォールバックを使用します。
- ツール対応エージェントや信頼できない入力では、旧世代または性能の低いモデル階層を避けます。

## オンボーディング

```bash
openclaw onboard
```

OpenAI Codex サブスクリプション OAuth や Anthropic（API キーまたは Claude CLI の再利用）を含む一般的なプロバイダーのモデルと認証を、設定を手動編集せずにセットアップします。

プライマリモデルが未設定の場合、新規の OpenAI API キーセットアップでは `openai/gpt-5.6` が選択されます。修飾なしの直接 API ID は Sol 階層に解決されます。新規の ChatGPT/Codex OAuth セットアップでは、完全一致する `openai/gpt-5.6-sol` カタログ参照が選択されます。再認証では、`openai/gpt-5.5` を含む既存の明示的なプライマリモデルが維持されます。アカウントで GPT-5.6 を利用できない場合は、`openai/gpt-5.5` を明示的に選択してください。OpenClaw が暗黙的にダウングレードすることはありません。

## 「モデルが許可されていません」（および応答が停止する理由）

`agents.defaults.modelPolicy.allow` が空でない場合、`/model`、セッションオーバーライド、および `--model` の許可リストになります。許可リスト外のモデルを選択すると、通常の応答が生成される前に処理が終了します。エージェントごとの `agents.entries.*.modelPolicy.allow` は、そのエージェントのデフォルトポリシーを置き換えます。

```text
モデルオーバーライド「provider/model」は agents.defaults.modelPolicy.allow で許可されていません。
「provider/model」、「provider/*」、またはより限定的な「provider/namespace/*」プレフィックスを agents.defaults.modelPolicy.allow に追加するか、任意のモデルを許可するためにリストを削除または空にしてください。
```

修正するには、指定された `modelPolicy.allow` キーにモデルまたはプロバイダーのワイルドカードを追加するか、そのリストを削除または空にするか、`/model list` からモデルを選択します。拒否されたコマンドに `/model openai/gpt-5.5 --runtime codex` などのランタイムオーバーライドが含まれていた場合は、まず許可リストを修正してから、同じコマンドを再試行します。

ローカル／GGUF モデルの場合、許可リストにはプロバイダーのプレフィックスを含む完全な参照が必要です。たとえば `ollama/gemma4:26b` や `lmstudio/Gemma4-26b-a4-it-gguf` です。正確な文字列は `openclaw models list --provider <provider>` で確認してください。許可リストが有効になると、ファイル名だけ、または表示名だけでは不十分です。

すべてのモデルを列挙せずにプロバイダーを制限するには、末尾プレフィックスワイルドカードのエントリを使用します。プロバイダー全体を対象とする `provider/*` は、そのプロバイダー配下のすべてのモデルに一致します。`clawrouter/anthropic/*` のようなより限定的なプレフィックスは、その名前空間のみに一致します。

```json5
{
  agents: {
    defaults: {
      modelPolicy: {
        allow: ["openai/*", "vllm/*"],
      },
    },
  },
}
```

これにより、`/model`、`/models`、およびモデルピッカーには、それらのプロバイダーで検出されたカタログのみが表示され、新しいモデルも許可リストを編集せずに表示できます。完全一致の `provider/model` エントリと `provider/*` エントリを組み合わせることで、別のプロバイダーから特定のモデルを 1 つ追加できます。

エイリアスとモデルごとの設定を含む許可リストの例：

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      modelPolicy: {
        allow: ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
}
```

<Accordion title="許可リストを明示的に編集">
完全なリストを直接設定します。

```bash
openclaw config set agents.defaults.modelPolicy.allow '["openai/gpt-5.4","anthropic/*"]' --strict-json
```

`openclaw models set`、プロバイダーのセットアップ、および `openclaw models aliases add` は、`agents.defaults.models` 配下にエントリを追加できますが、`modelPolicy.allow` を変更することはありません。これにより、モデルのメタデータとエイリアスがオーバーライドポリシーから独立して維持されます。
</Accordion>

## チャットでの `/model`

```text
/model
/model list
/model 3
/model openai/gpt-5.4
/model default
/model status
```

- `/model` と `/model list` は、コンパクトな番号付きピッカー（モデルファミリー + 利用可能なプロバイダー）を表示し、`/model <#>` はそこから選択します。Discord では、Submit ステップを伴うプロバイダー／モデルのドロップダウンが開きます。Telegram では、ピッカーでの選択はセッションスコープであり、`openclaw.json` にあるエージェントの永続的なデフォルトを書き換えることはありません。`/models add` は非推奨であり、チャットからモデルを登録する代わりにメッセージを返します。
- `/model` は、新しいセッション選択を即座に永続化します。エージェントがアイドル状態の場合、次の実行ですぐに使用されます。実行がすでにアクティブな場合、切り替えは次のクリーンな再試行ポイント（ツールの動作または応答出力がすでに開始されている場合は、それ以降のポイント）までキューに入れられます。
- `/model default` はセッション選択をクリアし、設定済みのプライマリを再び継承するようにします。
- ユーザーが選択した `/model` 参照は、そのセッションでは厳格に適用されます。到達不能になった場合、`agents.defaults.model.fallbacks` を通じて暗黙的にフォールバックするのではなく、応答が明示的に失敗します。設定済みのデフォルトと cron ジョブのプライマリでは、引き続きフォールバックチェーンが使用されます。
- `/model status` は詳細ビューです。プロバイダーごとの認証候補に加え、設定されている場合はプロバイダーエンドポイント `baseUrl` と `api` モードが表示されます。
- モデル参照は、最初の `/` で分割して解析されます。`provider/model` と入力してください。モデル ID 自体に `/` が含まれる場合（OpenRouter 形式）は、プロバイダーのプレフィックスを含めます（例: `/model openrouter/moonshotai/kimi-k2`）。プロバイダーを省略すると、OpenClaw は（1）エイリアスの一致、（2）そのプレフィックスなしの正確なモデル ID に対する一意の設定済みプロバイダーの一致、（3）設定済みのデフォルトプロバイダー（非推奨のフォールバック）の順で試行します。そのプロバイダーが設定済みのデフォルトモデルを提供しなくなっている場合は、削除済みプロバイダーの古いデフォルトが表面化するのを避けるため、代わりに最初の設定済みプロバイダー／モデルを使用します。
- モデル参照は小文字に正規化されます。それ以外ではプロバイダー ID は厳密に扱われるため、Plugin が提示する ID を使用してください。

コマンドの完全な動作と設定: [スラッシュコマンド](/ja-JP/tools/slash-commands)。

## CLI

```bash
openclaw models status
openclaw models list
openclaw models set <provider/model>
openclaw models set-image <provider/model>
openclaw models scan
openclaw models aliases list|add|remove
openclaw models fallbacks list|add|remove|clear
openclaw models image-fallbacks list|add|remove|clear
openclaw models auth list|add|login|paste-api-key|paste-token|setup-token|order
```

サブコマンドなしの `openclaw models` は `models status` のショートカットです。これは認証ストアのプロファイルについて OAuth の有効期限も表示します（デフォルトでは残り 24 時間以内になると警告します）。すべてのフラグ、JSON 形式、認証プロファイルのサブコマンドについては、[Models CLI リファレンス](/ja-JP/cli/models)を参照してください。

<AccordionGroup>
  <Accordion title="スキャン（OpenRouter の無料モデル）">
    `openclaw models scan` は OpenRouter の公開無料モデルカタログを調査し、ツールと画像のサポートについて候補をライブでプローブできます。カタログ自体は公開されているため、メタデータのみのスキャン（`--no-probe`）にはキーは不要です。ライブプローブと `--set-default`/`--set-image` には OpenRouter API キー（認証プロファイルまたは `OPENROUTER_API_KEY`）が必要で、キーがない場合はメタデータのみの出力にフェイルクローズします。

    結果は、画像サポート、ツールのレイテンシー、コンテキストサイズ、パラメーター数の順でランク付けされます。TTY では、プローブされた結果に対して対話形式でフォールバックの選択を求めます。非対話モードでデフォルトを受け入れるには `--yes` が必要です。

  </Accordion>
</AccordionGroup>

## モデルレジストリ（`models.json`）

`models.providers` の下に設定されたカスタムプロバイダーは、エージェントディレクトリ（デフォルトは `~/.openclaw/agents/<agentId>/agent/models.json`）内の `models.json` に書き込まれます。プロバイダー Plugin のカタログは、生成された Plugin 所有のカタログシャードとして別途保存され、自動的に読み込まれます。このファイルはデフォルトで設定とマージされます。設定したプロバイダーのみを使用するには、`models.mode: "replace"` を設定してください。

<AccordionGroup>
  <Accordion title="マージモードの優先順位">
    一致するプロバイダー ID について:

    - エージェントの `models.json` にすでに存在する空でない `baseUrl` が優先されます。
    - `models.json` 内の空でない `apiKey` は、そのプロバイダーが現在の設定／認証プロファイルのコンテキストで SecretRef によって管理されていない場合にのみ優先されます。
    - SecretRef によって管理される `apiKey` の値は、解決済みのシークレットを永続化する代わりに、ソースマーカーから更新されます。環境変数参照では環境変数名、ファイル／exec 参照では `secretref-managed` を使用します。
    - SecretRef によって管理されるヘッダー値も同じ方法で更新され、環境変数参照には `secretref-env:ENV_VAR_NAME` を使用します。
    - `models.json` 内の空または欠落している `apiKey`/`baseUrl` は、設定の `models.providers` にフォールバックします。
    - その他のプロバイダーフィールドは、設定および正規化されたカタログデータから更新されます。

  </Accordion>
</AccordionGroup>

マーカーの永続化ではソースが正とされます。OpenClaw は、`models.json` を再生成するたびに、解決済みのランタイムシークレット値ではなく、アクティブなソース設定のスナップショット（解決前）からマーカーを書き込みます。これには `openclaw agent` のようなコマンド駆動のパスも含まれます。

## 関連項目

- [エージェントランタイム](/ja-JP/concepts/agent-runtimes) — OpenClaw、Codex、およびその他のエージェントループランタイム
- [設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults) — モデル設定キー
- [画像生成](/ja-JP/tools/image-generation) — 画像モデルの設定
- [モデルフェイルオーバー](/ja-JP/concepts/model-failover) — フォールバックチェーン
- [モデルプロバイダー](/ja-JP/concepts/model-providers) — プロバイダーのルーティングと認証
- [Models CLI リファレンス](/ja-JP/cli/models) — コマンドとフラグの完全なリファレンス
- [音楽生成](/ja-JP/tools/music-generation) — 音楽モデルの設定
- [動画生成](/ja-JP/tools/video-generation) — 動画モデルの設定
