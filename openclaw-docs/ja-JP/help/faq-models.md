---
read_when:
    - モデルの選択または切り替え、エイリアスの設定
    - モデルのフェイルオーバー / 「すべてのモデルが失敗しました」のデバッグ
    - 認証プロファイルの仕組みと管理方法を理解する
sidebarTitle: Models FAQ
summary: FAQ：モデルのデフォルト、選択、エイリアス、切り替え、フェイルオーバー、認証プロファイル
title: FAQ：モデルと認証
x-i18n:
    generated_at: "2026-07-26T10:05:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

モデルと認証プロファイルに関する Q&A。セットアップ、セッション、Gateway、チャネル、
トラブルシューティングについては、メインの [FAQ](/ja-JP/help/faq) を参照してください。

## モデル：デフォルト、選択、エイリアス、切り替え

<AccordionGroup>
  <Accordion title='「デフォルトモデル」とは何ですか？'>
    次の項目で設定します。

    ```text
    agents.defaults.model.primary
    ```

    モデルは `provider/model` 参照です（例：`openai/gpt-5.5`、
    `anthropic/claude-sonnet-4-6`）。必ず `provider/model` を明示的に設定してください。
    プロバイダーを省略すると、OpenClaw はまずエイリアスとの一致を試し、次にそのモデル ID に対して
    一意に設定されたプロバイダーとの一致を試し、その後、設定済みのデフォルトプロバイダーへ
    フォールバックします（非推奨の互換パス）。そのプロバイダーに設定済みのデフォルトモデルが
    存在しなくなった場合、OpenClaw は古いデフォルトではなく、最初に設定された
    プロバイダー／モデルへフォールバックします。

  </Accordion>

  <Accordion title="どのモデルを推奨しますか？">
    プロバイダースタックが提供する最新世代の中で最も強力なモデルを使用してください。
    特に、ツールを使用するエージェントや信頼できない入力を扱うエージェントでは重要です。
    性能が低いモデルや過度に量子化されたモデルは、プロンプトインジェクションや安全でない
    動作に対して脆弱です（[セキュリティ](/ja-JP/gateway/security)を参照）。
    日常的でリスクの低いチャットには、エージェントの役割に応じて安価なモデルを割り当ててください。

    エージェントごとにモデルを割り当て、長時間のタスクはサブエージェントで並列化してください
    （各サブエージェントはそれぞれ独自にトークンを消費します）。
    [モデル](/ja-JP/concepts/models)、[サブエージェント](/ja-JP/tools/subagents)、
    [MiniMax](/ja-JP/providers/minimax)、[ローカルモデル](/ja-JP/gateway/local-models)を参照してください。

  </Accordion>

  <Accordion title="設定を消去せずにモデルを切り替えるにはどうすればよいですか？">
    モデルのフィールドだけを変更し、設定全体の置き換えは避けてください。

    - チャットで `/model` を使用（セッション単位。[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照）
    - `openclaw models set ...`（モデル設定のみを更新）
    - `openclaw configure --section model`（対話形式）
    - `~/.openclaw/openclaw.json` 内の `agents.defaults.model` を直接編集

    RPC で編集する場合は、まず `config.schema.lookup` で確認し
    （正規化されたパス、簡易スキーマドキュメント、子項目の概要）、
    部分オブジェクトを指定した `config.apply` よりも `config.patch` を優先してください。
    設定を上書きしてしまった場合は、バックアップから復元するか、
    `openclaw doctor` を実行して修復してください。

    ドキュメント：[モデル](/ja-JP/concepts/models)、[設定](/ja-JP/cli/configure)、
    [構成](/ja-JP/cli/config)、[Doctor](/ja-JP/gateway/doctor)。

  </Accordion>

  <Accordion title="セルフホストモデル（llama.cpp、vLLM、Ollama）は使用できますか？">
    はい。Ollama が最も簡単な方法です。簡単なセットアップ手順は次のとおりです。

    1. `https://ollama.com/download` から Ollama をインストール
    2. ローカルモデルを取得（例：`ollama pull gemma4`）
    3. クラウドモデルも使用する場合は、`ollama signin` を実行
    4. `openclaw onboard` を実行し、`Ollama`、続いて `Local` または `Cloud + Local` を選択

    `Cloud + Local` を使用すると、クラウドモデルとローカルの Ollama モデルの両方を利用できます。
    `kimi-k2.5:cloud` のようなクラウドモデルは、ローカルで取得する必要がありません。
    手動で切り替えるには、`openclaw models list`、続いて `openclaw models set ollama/<model>` を使用します。

    小型モデルや高度に量子化されたモデルは、プロンプトインジェクションに対してより脆弱です。
    ツールへアクセスできるボットには大規模モデルを使用してください。小型モデルを使用する場合でも、
    サンドボックス化と厳格なツール許可リストを有効にしてください。

    ドキュメント：[Ollama](/ja-JP/providers/ollama)、[ローカルモデル](/ja-JP/gateway/local-models)、
    [モデルプロバイダー](/ja-JP/concepts/model-providers)、[セキュリティ](/ja-JP/gateway/security)、
    [サンドボックス化](/ja-JP/gateway/sandboxing)。

  </Accordion>

  <Accordion title="再起動せずに、その場でモデルを切り替えるにはどうすればよいですか？">
    `/model <name>` を単独のメッセージとして送信します。
    番号付き選択メニュー（`/model`、`/model
    list`、`/model 3`）、
    セッションの上書きを解除する `/model default`、
    エンドポイント／API モードの詳細を表示する `/model status` を含む
    完全なコマンド一覧については、[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

    `@profile` を使用すると、セッションごとに特定の認証プロファイルを強制できます。

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    `@profile` で設定したプロファイルの固定を解除するには、
    サフィックスなしで `/model` を再実行するか
    （例：`/model anthropic/claude-opus-4-6`）、`/model` からデフォルトを選択します。
    有効な認証プロファイルを確認するには、`/model status` を使用します。

  </Accordion>

  <Accordion title="2 つのプロバイダーが同じモデル ID を公開している場合、/model はどちらを使用しますか？">
    `/model provider/model` は、そのプロバイダールートを厳密に選択します。
    たとえば、モデル ID が一致していても、`qianfan/deepseek-v4-flash` と
    `deepseek/deepseek-v4-flash` は異なる参照です。OpenClaw がモデル ID だけの一致を理由に、
    暗黙的にプロバイダーを切り替えることはありません。

    ユーザーが選択した `/model` 参照では、フォールバックが厳格に扱われます。
    そのプロバイダー／モデルが利用できなくなると、`agents.defaults.model.fallbacks` へ
    フォールバックする代わりに、応答が明示的に失敗します。設定済みのフォールバックチェーンは、
    設定済みのデフォルト、Cron ジョブのプライマリ、自動選択されたフォールバック状態には
    引き続き適用されます。セッション上書きではない実行でフォールバックが許可されている場合、
    OpenClaw はまず要求されたプロバイダー／モデルを試し、次に設定済みのフォールバック、
    その後に設定済みのプライマリを試します。このため、重複するプロバイダーなしの
    モデル ID から、デフォルトプロバイダーへ直接戻ることはありません。

    [モデル](/ja-JP/concepts/models)と[モデルのフェイルオーバー](/ja-JP/concepts/model-failover)を参照してください。

  </Accordion>

  <Accordion title="日常のタスクには GPT 5.5、コーディングには Codex 5.5 を使用できますか？">
    はい。モデルの選択とランタイムの選択は別々です。

    - **ネイティブ Codex コーディングエージェント：** `agents.defaults.model.primary` を
      `openai/gpt-5.5` に設定します。ChatGPT／Codex サブスクリプション認証には、`openclaw models auth login --provider
      openai` でサインインします。
    - **エージェントループ外の直接的な OpenAI API タスク：**
      画像、埋め込み、音声、リアルタイム、その他のエージェント以外の OpenAI API サーフェス向けに
      `OPENAI_API_KEY` を設定します。
    - **OpenAI エージェントの API キー認証：** 順序付きの
      `openai` API キープロファイルとともに `/model openai/gpt-5.5` を使用します。
    - **サブエージェント：** 独自の `openai/gpt-5.5` モデルを持つ
      Codex 向けエージェントにコーディングタスクを割り当てます。

    [モデル](/ja-JP/concepts/models)と[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

  </Accordion>

  <Accordion title="GPT 5.5 の高速モードを設定するにはどうすればよいですか？">
    - **セッション単位：** `openai/gpt-5.5` の使用中に `/fast on` を送信します。
    - **モデルごとのデフォルト：**
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` を `true` に設定します。
    - **自動終了：** `/fast auto` または `params.fastMode: "auto"` を使用すると、
      終了時点までは新しいモデル呼び出しを高速で実行し、それ以降の再試行、フォールバック、
      ツール結果、継続呼び出しは高速モードなしで実行します。終了時点のデフォルトは
      60 秒です。モデルの `params.fastAutoOnSeconds` で上書きできます。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    高速モードは、ネイティブ OpenAI Responses リクエストの `service_tier = "priority"` に対応します。
    既存の `service_tier` 値は保持され、高速モードによって
    `reasoning` または `text.verbosity` が書き換えられることはありません。
    セッションの `/fast` による上書きは、設定のデフォルトより優先されます。

    [思考と高速モード](/ja-JP/tools/thinking)、および
    [OpenAI](/ja-JP/providers/openai) プロバイダーページの「高度な設定」にある
    「高速モード」セクションを参照してください。

  </Accordion>

  <Accordion title='「Model ... is not allowed」と表示された後、応答がないのはなぜですか？'>
    `agents.defaults.modelPolicy.allow` が空でない場合、これは
    `/model`、セッション上書き、`--model` の
    **許可リスト**になります。そのリストに含まれないモデルを選択すると、
    通常の応答の代わりに次の内容が返されます。

    ```text
    モデルの上書き「provider/model」は agents.defaults.modelPolicy.allow によって許可されていません。
    ```

    修正方法：指定された `modelPolicy.allow` リストに正確なモデルまたは
    `"provider/*"` のようなプロバイダーのワイルドカードを追加するか、
    そのリストを削除／空にするか、`/model list` からモデルを選択します。
    コマンドに `--runtime codex` も含まれていた場合は、まず許可リストを更新してから、
    同じ `/model provider/model --runtime codex` コマンドを再試行してください。

  </Accordion>

  <Accordion title='「Unknown model: minimax/MiniMax-M3」と表示されるのはなぜですか？'>
    古い OpenClaw リリースを使用している場合は、まずアップグレードするか
    （またはソースから `main` を実行し）、Gateway を再起動してください。
    `MiniMax-M3` は、インストール済みリリースのカタログにまだ含まれていない可能性があります。
    それ以外の場合は、MiniMax プロバイダーが設定されていないため
    （プロバイダーエントリまたは認証プロファイルが見つからないため）、モデルを解決できません。
    完全な修正チェックリスト、プロバイダー／モデル ID の表、設定ブロックの例については、
    [MiniMax](/ja-JP/providers/minimax) プロバイダーページの「トラブルシューティング」セクションを
    参照してください。

  </Accordion>

  <Accordion title="MiniMax をデフォルトにし、複雑なタスクには OpenAI を使用できますか？">
    はい。MiniMax をデフォルトとして使用し、セッションごとにモデルを切り替えてください。
    フォールバックはエラー用であり、「難しいタスク」用ではないため、
    `/model` または別のエージェントを使用します。

    **選択肢 A：セッションごとに切り替える**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    次に `/model gpt` を使用します。

    **選択肢 B：別々のエージェント** — エージェント A のデフォルトを MiniMax、
    エージェント B のデフォルトを OpenAI にします。エージェント別に割り当てるか、
    `/agent` を使用して切り替えます。

    ドキュメント：[モデル](/ja-JP/concepts/models)、[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)、
    [MiniMax](/ja-JP/providers/minimax)、[OpenAI](/ja-JP/providers/openai)。

  </Accordion>

  <Accordion title="opus／sonnet／gpt は組み込みのショートカットですか？">
    はい。これらは組み込みの短縮名で、対象モデルが `agents.defaults.models` に存在する場合にのみ
    適用されます。

    | エイリアス | 解決先 |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    同じ名前の独自エイリアスを設定すると、組み込みのものを上書きします。

  </Accordion>

  <Accordion title="モデルのショートカット（エイリアス）を定義／上書きするにはどうすればよいですか？">
    エイリアスは `agents.defaults.models.<modelId>.alias` に設定します。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    これにより、`/model sonnet`（対応している場合は `/<alias>`）が
    そのモデル ID に解決されます。

  </Accordion>

  <Accordion title="OpenRouter や Z.AI など、ほかのプロバイダーのモデルを追加するにはどうすればよいですか？">
    OpenRouter（トークン単位の従量課金、多数のモデル）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI（GLM モデル）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    参照されたプロバイダー／モデルに必要なプロバイダーキーがない場合、
    実行時の認証エラーが発生します（例：`No API key found for provider "zai"`）。

    **新しいエージェントを追加した後に「No API key found for provider」と表示される場合**

    新しいエージェントの認証ストアは空です。認証はエージェントごとに管理され、
    次の場所に保存されます。

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    修正: `openclaw agents add <id>` を実行してウィザードで認証を設定するか、
    メインエージェントのストアから、移植可能な静的 `api_key`/`token` プロファイルのみを
    コピーします。OAuth の場合、新しいエージェントが独自のアカウントを必要とするときに、
    そのエージェントからサインインします。`agentDir` の再利用と認証情報共有に関する
    完全なルールについては、[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)を参照してください。
    エージェント間で `agentDir` を再利用しないでください。

  </Accordion>
</AccordionGroup>

## モデルのフェイルオーバーと「すべてのモデルが失敗しました」

<AccordionGroup>
  <Accordion title="フェイルオーバーはどのように機能しますか？">
    2 段階あります:

    1. 同じプロバイダー内での**認証プロファイルのローテーション**。
    2. `agents.defaults.model.fallbacks` 内の次のモデルへの**モデルフォールバック**。

    失敗したプロファイルにはクールダウン（指数バックオフ）が適用されるため、
    プロバイダーがレート制限中または一時的に失敗している場合でも、OpenClaw は
    応答を継続します。

    レート制限の分類には、単純な `429` だけでなく、`Too many concurrent
    requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai
    ... quota limit exceeded`、`resource exhausted`、
    および定期的な使用量ウィンドウ制限（`weekly/monthly limit reached`）も含まれ、
    すべてフェイルオーバーが必要なレート制限として扱われます。

    課金に関する応答は必ずしも `402` ではなく、一部の `402` は
    課金系ではなく一時的障害／レート制限の分類に残ります。`401`/`403` に
    明示的な課金テキストがある場合は課金系に振り分けられますが、プロバイダー固有の
    テキストマッチャー（例: OpenRouter `Key limit exceeded`）は、そのプロバイダー内に
    限定されます。再試行可能な使用量ウィンドウまたは組織／ワークスペースの支出上限
    （`daily limit reached, resets tomorrow`、
    `organization spending limit exceeded`）のように見える `402` は、
    長期間の課金無効化ではなく `rate_limit` として扱われます。

    コンテキストオーバーフローエラーはフォールバック経路から完全に除外されます。
    `request_too_large`、`input exceeds the maximum number of tokens`、
    `input token count exceeds the maximum number of input tokens`、`input is
    too long for the model`、`ollama error: context length exceeded` のようなシグネチャは、
    モデルフォールバックを進める代わりに Compaction／再試行へ送られます。

    汎用的なサーバーエラーテキストの判定範囲は、「unknown/error を含むものすべて」より
    限定的です。フェイルオーバーシグナルとして扱われるプロバイダー限定の一時的な形式には、
    Anthropic の単独の `An unknown error occurred`、OpenRouter の単独の
    `Provider returned error`、`Unhandled stop reason:
    error` のような停止理由エラー、一時的なサーバーテキスト
    （`internal
    server error`、`unknown error, 520`、`upstream error`、`backend error`）を含む JSON `api_error` ペイロード、
    およびプロバイダーのコンテキストが一致する場合の `ModelNotReadyException` のような
    プロバイダー混雑エラーがあります。`LLM request failed
    with an unknown error.` のような汎用的な内部フォールバックテキストは
    保守的に扱われ、それだけではフォールバックをトリガーしません。

  </Accordion>

  <Accordion title='「No credentials found for profile anthropic:default」とはどういう意味ですか？'>
    認証プロファイル ID `anthropic:default` に、想定される認証ストア内の
    認証情報がありません。

    **修正チェックリスト:**

    - プロファイルの保存場所を確認します — 現在:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`; 旧形式:
      `~/.openclaw/agent/*`（`openclaw doctor` により移行）。
    - Gateway が環境変数を読み込んでいることを確認します。シェル内でのみ設定した
      `ANTHROPIC_API_KEY` は、systemd/launchd 経由で実行される Gateway には届きません。
      `~/.openclaw/.env` に配置するか、`env.shellEnv` を有効にします。
    - 正しいエージェントを編集していることを確認します。マルチエージェント構成には
      複数の `auth-profiles.json` ファイルがあります。
    - 設定済みモデルとプロバイダーの認証状態を確認するには、
      `openclaw models status` を実行します。

    **「No credentials found for profile anthropic」（メールアドレスのサフィックスなし）の場合:**

    実行が、Gateway で見つけられない Anthropic プロファイルに固定されています。

    - Claude CLI を使用します: Gateway ホスト上で `openclaw models auth login --provider anthropic
      --method cli --set-default` を実行します。
    - 代わりに API キーを使用することを推奨します: Gateway ホスト上の
      `~/.openclaw/.env` に `ANTHROPIC_API_KEY` を配置してから、
      存在しないプロファイルを強制する固定順序を解除します:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - リモートモード: 認証プロファイルはノート PC ではなく Gateway マシン上にあります。
      そこでコマンドを実行していることを確認します。

  </Accordion>

  <Accordion title="Google Gemini も試行され、失敗したのはなぜですか？">
    モデル設定に Google Gemini がフォールバックとして含まれている場合（または
    Gemini の短縮名に切り替えた場合）、OpenClaw はフォールバック中にそれを試行します。
    Google の認証情報が設定されていないと `No API key found for provider
    "google"` が発生します。修正方法:
    Google 認証を追加するか、`agents.defaults.model.fallbacks`/エイリアスから Google モデルを削除します。

    **LLM リクエストが拒否されました: 思考シグネチャが必要です（Google Antigravity）**

    原因: セッション履歴にシグネチャのない思考ブロックがあります（多くの場合、
    中止または不完全なストリームが原因です）。Google Antigravity では思考ブロックに
    シグネチャが必要です。OpenClaw は Google Antigravity Claude 向けに署名されていない
    思考ブロックを除去します。それでも表示される場合は、新しいセッションを開始するか、
    そのエージェントに `/thinking off` を設定します。

  </Accordion>
</AccordionGroup>

## 認証プロファイル: 概要と管理方法

関連: [/concepts/oauth](/ja-JP/concepts/oauth)（OAuth フロー、トークン保存、複数アカウントのパターン）

<AccordionGroup>
  <Accordion title="認証プロファイルとは何ですか？">
    プロバイダーに関連付けられた名前付きの認証情報レコード（OAuth または API キー）で、
    次の場所に保存されます:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    シークレットを出力せずに保存済みプロファイルを確認するには、`openclaw models auth
    list`
    （必要に応じて `--provider <id>` または `--json`）を使用します。
    [Models CLI](/ja-JP/cli/models#auth-profiles)を参照してください。

  </Accordion>

  <Accordion title="一般的なプロファイル ID は何ですか？">
    プロバイダーのプレフィックス付きです: `anthropic:default`（メールアドレスの ID が
    存在しない場合に一般的）、OAuth ID の `anthropic:<email>`、または任意に選択した
    カスタム ID（例: `anthropic:work`）。

  </Accordion>

  <Accordion title="最初に試行する認証プロファイルを制御できますか？">
    はい。`auth.order.<provider>` 設定で、プロバイダーごとのローテーション順序を指定します
    （メタデータのみで、シークレットは保存されません）。

    OpenClaw は、短い**クールダウン**（レート制限、タイムアウト、認証失敗）中の
    プロファイルや、より長い**無効**状態（課金／クレジット不足）のプロファイルを
    スキップすることがあります。`openclaw models status
    --json` で確認し、`auth.unusableProfiles` を
    チェックしてください。レート制限のクールダウンはモデル単位の場合があります。
    あるモデルでクールダウン中のプロファイルでも、同じプロバイダーの兄弟モデルには
    使用できます。一方、課金／無効化ウィンドウはプロファイル全体をブロックします。

    エージェントごとの順序オーバーライドを設定します（そのエージェントの `auth-state.json` に保存）:

    ```bash
    # 設定済みのデフォルトエージェントが既定値（--agent は省略）
    openclaw models auth order get --provider anthropic

    # ローテーションを単一のプロファイルに固定
    openclaw models auth order set --provider anthropic anthropic:default

    # または明示的な順序を設定（プロバイダー内でフォールバック）
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # オーバーライドを解除（設定の auth.order／ラウンドロビンにフォールバック）
    openclaw models auth order clear --provider anthropic

    # 特定のエージェントを対象にする
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    実際に何が試行されるかを確認するには、`openclaw models status --probe` を使用します。
    明示的な順序から除外された保存済みプロファイルは、暗黙に試行される代わりに
    `excluded_by_auth_order` と報告されます。

  </Accordion>

  <Accordion title="OAuth と API キーの違いは何ですか？">
    - **OAuth／CLI ログイン**では、プロバイダーが対応している場合、
      多くの場合サブスクリプションアクセスを使用します。Anthropic では、OpenClaw の
      Claude CLI バックエンドが Claude Code `claude -p` を使用します。Anthropic は現在、
      これをサブスクリプションの使用量上限を消費する Agent SDK／プログラムによる使用として
      扱っています。現在の課金停止状況と情報源へのリンクについては、
      [Anthropic](/ja-JP/providers/anthropic)を参照してください。
    - **API キー**では、トークン単位の従量課金を使用します。

    ウィザードは、Anthropic Claude CLI、OpenAI Codex OAuth、API キーに対応しています。

  </Accordion>
</AccordionGroup>

## 関連項目

- [よくある質問](/ja-JP/help/faq) — メインのよくある質問
- [よくある質問 — クイックスタートと初回実行時のセットアップ](/ja-JP/help/faq-first-run)
- [モデルの選択](/ja-JP/concepts/model-providers)
- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover)
