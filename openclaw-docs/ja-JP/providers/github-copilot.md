---
read_when:
    - GitHub Copilot をモデルプロバイダーとして使用する場合
    - '`openclaw models auth login-github-copilot` フローが必要です'
    - 組み込みの Copilot プロバイダー、Copilot SDK ハーネス、Copilot Proxy のいずれかを選択します
summary: デバイスフローまたは非対話型トークンインポートを使用して、OpenClaw から GitHub Copilot にサインインする
title: GitHub Copilot
x-i18n:
    generated_at: "2026-07-26T09:48:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e839e6c72e7e7cb106a2f98c62c4994b4f3d6f34a2e76b549f2f6ccfdac91fe6
    source_path: providers/github-copilot.md
    workflow: 16
---

GitHub Copilot は GitHub の AI コーディングアシスタントです。GitHub アカウントとプランで利用可能な Copilot
モデルへのアクセスを提供します。OpenClaw では、Copilot をモデル
プロバイダーまたはエージェントランタイムとして、3 つの異なる方法で使用できます。

## OpenClaw で Copilot を使用する 3 つの方法

<Tabs>
  <Tab title="組み込みプロバイダー（github-copilot）">
    ネイティブのデバイスログインフローを使用して GitHub トークンを取得し、OpenClaw の実行時に
    Copilot API トークンと交換します。VS Code を必要としないため、これは**デフォルト**かつ最も簡単な方法です。

    <Steps>
      <Step title="ログインコマンドを実行する">
        ```bash
        openclaw models auth login-github-copilot
        ```

        URL にアクセスしてワンタイムコードを入力するよう求められます。完了するまで
        ターミナルを開いたままにしてください。
      </Step>
      <Step title="デフォルトモデルを設定する">
        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        または設定ファイルで指定します。

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Copilot SDK ハーネス Plugin（copilot）">
    選択した `github-copilot/*` モデルについて、GitHub の
    Copilot CLI と SDK に低レベルのエージェントループを管理させる場合は、外部の `@openclaw/copilot` Plugin をインストールします。

    ```bash
    openclaw plugins install @openclaw/copilot
    ```

    次に、モデルまたはプロバイダーでこのランタイムの使用を明示的に有効にします。

    ```json5
    {
      agents: {
        defaults: {
          model: "github-copilot/gpt-5.5",
          models: {
            "github-copilot/gpt-5.5": {
              agentRuntime: { id: "copilot" },
            },
          },
        },
      },
    }
    ```

    これらのエージェントターンで、ネイティブの Copilot CLI セッション、SDK が管理するスレッド
    状態、および Copilot が管理する Compaction を使用する場合に選択してください。明示的な
    `agentRuntime` オプトインがなければ、`github-copilot/*` モデルは引き続き
    組み込みプロバイダーを使用します。ランタイムの完全な契約については、[Copilot SDK ハーネス](/ja-JP/plugins/copilot)を参照してください。

  </Tab>

  <Tab title="Copilot Proxy Plugin（copilot-proxy）">
    **Copilot Proxy** VS Code 拡張機能をローカルブリッジとして使用します。OpenClaw は
    プロキシの `/v1` エンドポイント（デフォルトは `http://localhost:3000/v1`）と通信し、設定した
    モデルリストを使用します。

    `copilot-proxy` Plugin は OpenClaw に同梱されており、デフォルトで有効です。
    次のコマンドでベース URL とモデル ID を設定します。

    ```bash
    openclaw models auth login --provider copilot-proxy --set-default
    ```

    <Note>
    すでに VS Code で Copilot Proxy を実行している場合や、Copilot Proxy 経由で
    ルーティングする必要がある場合に選択してください。VS Code 拡張機能を実行したままにする必要があります。
    </Note>

  </Tab>
</Tabs>

## GitHub Enterprise（データ所在地）

組織がデータ所在地対応の GitHub Enterprise テナント（
`your-org.ghe.com` などの `*.ghe.com` ホスト）を使用している場合、Copilot は公開
`github.com` ではなく、テナントローカルのエンドポイント上にあります。OpenClaw ではこれを
正式な認証選択肢として提供しているため、URL を手動で編集する必要はありません。

<Steps>
  <Step title="Enterprise の認証選択肢を選ぶ">
    オンボーディングまたは `openclaw models auth` で、
    **GitHub Copilot (Enterprise / data residency)** を選択します。Enterprise ドメイン
    （例: `your-org.ghe.com`）の入力を求められ、その後、そのテナントに対してデバイス
    ログインが実行されます。

    テナントルート（`your-org.ghe.com`）のみを入力してください。`api.your-org.ghe.com` や
    `copilot-api.your-org.ghe.com` などの派生サービスホストは使用できません。
    OpenClaw がテナントルートからこれらのエンドポイントを自動的に導出します。

    ```bash
    openclaw models auth login --provider github-copilot --method device-enterprise
    ```

  </Step>
  <Step title="ドメインを設定に永続化する">
    選択したホストはプロバイダーのパラメーターに保存されるため、以降のトークン更新と
    補完は自動的にそのテナントを対象とします。

    ```json5
    {
      models: {
        providers: {
          "github-copilot": { params: { githubDomain: "your-org.ghe.com" } },
        },
      },
    }
    ```

  </Step>
</Steps>

デバイスフロー、トークン交換、補完は、それぞれ
`https://your-org.ghe.com/login/device/code`、
`https://api.your-org.ghe.com/copilot_internal/v2/token`、
`https://copilot-api.your-org.ghe.com` に解決されます。データ所在地対応トークンには
テナントスタンプが含まれ、プロキシヒントは含まれないため、補完のベース URL は公開エンドポイントではなく
テナントの Copilot ホストにフォールバックします。

<Note>
ドメインを切り替えると、必ずデバイスログインが再実行されます。Copilot トークンがすでに保存されている状態で
異なるドメイン（公開 `github.com` ↔ `*.ghe.com`
テナント、またはテナント間）を選択した場合、OpenClaw は既存のトークンを再利用しません。
設定に書き込むドメインにトークンのスコープを限定するため、新しいログインを強制します。
*同じ*ドメインに再ログインする場合は、引き続き現在のトークンを再利用するかどうか確認されます。
公開 `github.com` に戻すと、永続化された
`githubDomain` が消去され、設定がデフォルトに戻ります。
</Note>

<Note>
`COPILOT_GITHUB_DOMAIN` 環境変数は、ドメインを解決するすべての Copilot パスで
解決済みドメインを上書きします。これには、Enterprise デバイスログイン
（`--method device-enterprise`）、スタンドアロンの
`openclaw models auth login-github-copilot` ショートカット、トークン更新、埋め込み、
補完が含まれます。完全なヘッドレス環境または CI 環境では、これを `*.ghe.com` ホストに設定してください。
公開 `github.com` を使用する場合は、環境変数を未設定のままにし、設定パラメーターも指定しないでください。
ログイン時には、トークンを発行したドメインが永続化されます（公開
`github.com` に対するログイン時には消去されます）。そのため、環境変数を未設定に戻した後も
ルーティングは正しく維持されます。
</Note>

## オプションのフラグ

| コマンド                                                               | フラグ          | 説明                                                   |
| ---------------------------------------------------------------------- | --------------- | ------------------------------------------------------ |
| `openclaw models auth login-github-copilot`                                                    | `--yes` | 確認せずに既存の認証プロファイルを上書きする           |
| `openclaw models auth login --provider github-copilot --method device`                                                    | `--set-default` | プロバイダーが推奨するデフォルトモデルも適用する       |

```bash
# 再ログインの確認をスキップ
openclaw models auth login-github-copilot --yes

# 1 回の操作でログインしてデフォルトモデルを設定
openclaw models auth login --provider github-copilot --method device --set-default
```

## 非対話型オンボーディング

デバイスログインフローには対話型 TTY が必要です。ヘッドレス環境でセットアップするには、
既存の GitHub OAuth アクセストークンを `openclaw onboard --non-interactive` でインポートします。

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

`--auth-choice` を省略することもできます。`--github-copilot-token` を渡すと、
GitHub Copilot プロバイダーの認証選択肢が推論されます。このフラグを省略すると、オンボーディングは
`COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、`GITHUB_TOKEN` の順にフォールバックします。
`COPILOT_GITHUB_TOKEN` を設定した状態で `--secret-input-mode ref` を使用すると、
`auth-profiles.json` にプレーンテキストで保存する代わりに、環境変数を参照する
`tokenRef` を保存できます。

<AccordionGroup>
  <Accordion title="対話型 TTY が必要">
    デバイスログインフローには対話型 TTY が必要です。非対話型スクリプトや CI パイプラインではなく、
    ターミナルで直接実行してください。
  </Accordion>

  <Accordion title="利用可能なモデルはプランに依存">
    利用可能な Copilot モデルは GitHub プランによって異なります。モデルが
    拒否された場合は、別の ID（例: `github-copilot/gpt-5.5`）を試してください。現在のモデル一覧については、
    GitHub の [Copilot プランごとの対応モデル](https://docs.github.com/en/copilot/reference/ai-models/supported-models#supported-ai-models-per-copilot-plan)
    を参照してください。
  </Accordion>

  <Accordion title="Copilot API からライブカタログを更新">
    デバイスログイン（または環境変数）による認証パスで GitHub トークンが解決されると、
    OpenClaw は必要に応じて `${baseUrl}/models`（VS Code Copilot が使用するものと同じエンドポイント）から
    モデルカタログを更新します。これにより、マニフェストを頻繁に変更することなく、
    ランタイムがアカウントごとの利用資格と正確なコンテキストウィンドウを追跡できます。
    新しく公開された Copilot モデルは OpenClaw をアップグレードしなくても表示されるようになり、
    コンテキストウィンドウにはモデルごとの実際の上限（例: gpt-5.x シリーズでは 400k、
    内部の `claude-opus-*-1m` バリアントでは 1M）が反映されます。

    検出が無効な場合、ユーザーに GitHub 認証プロファイルがない場合、トークン交換に
    失敗した場合、または `/models` HTTPS 呼び出しでエラーが発生した場合は、
    同梱の静的カタログが表示用のフォールバックとして維持されます。オプトアウトして、
    静的なマニフェストカタログのみを使用するには（オフライン環境やエアギャップ環境）、
    次のように設定します。

    ```json5
    {
      plugins: {
        entries: {
          "github-copilot": {
            config: { discovery: { enabled: false } },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="トランスポートの選択">
    Claude モデル ID では Anthropic Messages トランスポートが自動的に使用されます。
    Gemini モデルでは OpenAI Chat Completions トランスポートが使用され、GPT および o-series
    モデルでは引き続き OpenAI Responses トランスポートが使用されます。OpenClaw はモデル参照に基づいて
    適切なトランスポートを選択します。
  </Accordion>

  <Accordion title="リクエストの互換性">
    OpenClaw は Copilot トランスポートで Copilot IDE 形式のリクエストヘッダー
    （VS Code のエディター／Plugin バージョンと `vscode-chat` 統合 ID）を送信し、
    ツール結果に続くターンをエージェント開始としてマークし、ターンに画像入力が含まれる場合は Copilot
    ビジョンヘッダーを設定します。
  </Accordion>

  <Accordion title="環境変数の解決順序">
    OpenClaw は次の優先順位で環境変数から Copilot 認証を解決します。

    | 優先順位 | 変数                  | 備考                                  |
    | -------- | --------------------- | ------------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN`    | 最高優先度、Copilot 専用              |
    | 2        | `GH_TOKEN`    | GitHub CLI トークン（フォールバック） |
    | 3        | `GITHUB_TOKEN`    | 標準 GitHub トークン（最低優先度）    |

    複数の変数が設定されている場合、OpenClaw は最も優先順位の高いものを使用します。
    デバイスログインフロー（`openclaw models auth login-github-copilot`）では
    トークンが認証プロファイルストアに保存され、すべての環境変数より優先されます。

  </Accordion>

  <Accordion title="トークンの保存">
    ログインによって GitHub トークンが認証プロファイルストア（プロファイル ID
    `github-copilot:github`）に保存され、OpenClaw の実行時に有効期間の短い Copilot API
    トークンと交換されます。トークンを手動で管理する必要はありません。
  </Accordion>
</AccordionGroup>

## メモリ検索の埋め込み

GitHub Copilot は、[メモリ検索](/ja-JP/concepts/memory-search)の埋め込みプロバイダーとしても使用できます。
Copilot サブスクリプションがあり、ログイン済みであれば、OpenClaw は別の API キーなしで
Copilot を埋め込みに使用できます。

### 設定

GitHub Copilot の埋め込みを使用するには、`memory.search.provider` を明示的に設定します。
GitHub トークンを利用できる場合、OpenClaw は Copilot API から利用可能な埋め込みモデルを検出し、
最適なモデルを自動的に選択します。

```json5
{
  memory: {
    search: {
      provider: "github-copilot",
      // オプション: 自動検出されたモデルを上書き
      model: "text-embedding-3-small",
    },
  },
}
```

### 仕組み

1. OpenClaw が GitHub トークンを解決します（環境変数または認証プロファイルから）。
2. 有効期間の短い Copilot API トークンと交換します。
3. Copilot の `/models` エンドポイントに問い合わせて、利用可能な埋め込みモデルを検出します。
4. 最適なモデルを選択します（優先順位: `text-embedding-3-small`、
   `text-embedding-3-large`、`text-embedding-ada-002`）。
5. Copilot の `/embeddings` エンドポイントに埋め込みリクエストを送信します。

利用可能なモデルは GitHub プランによって異なります。利用可能な埋め込みモデルがない場合、
OpenClaw は Copilot をスキップして次のプロバイダーを試します。

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="OAuth と認証" href="/ja-JP/gateway/authentication" icon="key">
    認証の詳細と認証情報の再利用ルール。
  </Card>
</CardGroup>
