---
read_when:
    - Ollama 経由でクラウドモデルまたはローカルモデルを使用して OpenClaw を実行したい場合
    - Ollama のセットアップと設定に関するガイダンスが必要です
    - 画像理解に Ollama のビジョンモデルを使用したい場合
summary: Ollama（クラウドモデルとローカルモデル）で OpenClaw を実行する
title: Ollama
x-i18n:
    generated_at: "2026-07-26T09:48:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw は、OpenAI 互換の
`/v1` エンドポイントではなく、Ollama のネイティブ API（`/api/chat`）と通信します。次の 3 つのモードがサポートされています。

| モード          | 使用するもの                                                                     |
| ------------- | -------------------------------------------------------------------------------- |
| クラウド + ローカル | 到達可能な Ollama ホスト。ローカルモデルと、（サインイン済みの場合）`:cloud` モデルを提供 |
| クラウドのみ    | `https://ollama.com` を直接使用。ローカルデーモンは不要                                   |
| ローカルのみ    | 到達可能な Ollama ホスト。ローカルモデルのみ                                       |

専用の `ollama-cloud` プロバイダー ID を使用するクラウドのみのセットアップについては、
[Ollama Cloud](/ja-JP/providers/ollama-cloud) を参照してください。クラウドルーティングをローカルの `ollama` プロバイダーと
分離しておきたい場合は、`ollama-cloud/<model>` 参照を使用してください。

<Warning>
`/v1` OpenAI 互換 URL（`http://host:11434/v1`）は使用しないでください。ツール呼び出しが機能しなくなり、モデルが未処理のツール呼び出し JSON をプレーンテキストとして出力する可能性があります。ネイティブ URL（`baseUrl: "http://host:11434"`、`/v1` なし）を使用してください。
</Warning>

正規の設定キーは `baseUrl` です。OpenAI SDK 形式の例では `baseURL` も使用できますが、
新しい設定では `baseUrl` を使用してください。

## 認証ルール

<AccordionGroup>
  <Accordion title="ローカルおよび LAN ホスト">
    ループバック、プライベートネットワーク、`.local`、およびホスト名のみの Ollama URL には、実際のベアラートークンは必要ありません。これらには OpenClaw が `ollama-local` マーカーを使用します。
  </Accordion>
  <Accordion title="リモートおよび Ollama Cloud ホスト">
    パブリックなリモートホストと `https://ollama.com` には、実際の認証情報（`OLLAMA_API_KEY`、認証プロファイル、またはプロバイダーの `apiKey`）が必要です。ホスト型サービスを直接使用する場合は、`ollama-cloud` プロバイダーを推奨します。
  </Accordion>
  <Accordion title="カスタムプロバイダー ID">
    `api: "ollama"` を持つカスタムプロバイダーにも同じルールが適用されます。たとえば、プライベート LAN ホストを参照する `ollama-remote` プロバイダーは `apiKey: "ollama-local"` を使用できます。サブエージェントはこれを認証情報の欠如として扱わず、Ollama プロバイダーフックを介してそのマーカーを解決します。埋め込みでその Ollama エンドポイントを使用するために、`memory.search.provider` でカスタムプロバイダー ID を指定することもできます。
  </Accordion>
  <Accordion title="認証プロファイル">
    `auth-profiles.json` にはプロバイダー ID の認証情報を保存し、エンドポイント設定（`baseUrl`、`api`、モデル、ヘッダー、タイムアウト）は `models.providers.<id>` に配置します。`{ "ollama-windows": { "apiKey": "ollama-local" } }` などの古いフラットファイルはランタイム形式ではありません。`openclaw doctor --fix` はバックアップを作成し、それらを正規の `ollama-windows:default` API キープロファイルに書き換えます。そのレガシーファイル内の `baseUrl` 値は不要な情報であり、プロバイダー設定に移す必要があります。
  </Accordion>
  <Accordion title="メモリ埋め込みのスコープ">
    Ollama のメモリ埋め込みに使用するベアラー認証のスコープは、宣言時に指定されたホストに限定されます。

    - プロバイダーレベルのキーは、そのプロバイダーのホストにのみ送信されます。
    - `memory.search.remote.apiKey` およびエージェント単位のオーバーライドは、それぞれのリモート埋め込みホストにのみ送信されます。
    - `OLLAMA_API_KEY` 環境変数の値のみが設定されている場合、それは Ollama Cloud の規約として扱われ、デフォルトではローカルまたはセルフホスト型ホストには送信されません。

  </Accordion>
</AccordionGroup>

## はじめに

<Tabs>
  <Tab title="オンボーディング（推奨）">
    <Steps>
      <Step title="オンボーディングを実行">
        ```bash
        openclaw onboard
        ```

        **Ollama** を選択し、次にモード（**Cloud + Local**、**Cloud only**、または **Local only**）を選択します。

        新規のガイド付きセットアップでは、OpenClaw は最初にデフォルトまたは設定済みの
        Ollama ホストを確認します。`/api/show` によってツールサポートと
        16K 以上のコンテキストウィンドウが確認された場合に限り、インストール済みモデルが自動的に提示されます。
        コンテキストメタデータがない場合やそれより小さい場合は、手動セットアップの経路が維持されます。
        共通の CLI/macOS セットアップ手順では、保存前に実際の補完を使用して、選択されたルートを引き続き検証します。
        この自動確認によってモデルがプルされることはありません。
        適切なインストール済みモデルが存在しない場合、オンボーディングは通常の
        Ollama モデル選択画面に進みます。
      </Step>
      <Step title="モデルを選択">
        `Cloud only` は `OLLAMA_API_KEY` の入力を求め、ホスト型クラウドのデフォルトを提案します。`Cloud + Local` と `Local only` は Ollama のベース URL の入力を求め、利用可能なモデルを検出し、選択されたローカルモデルがない場合は自動的にプルします。`gemma4:latest` のようなインストール済みの `:latest` タグは、`gemma4` と重複して表示されず、一度だけ表示されます。`Cloud + Local` は、クラウドアクセス用にホストへサインイン済みかどうかも確認します。
      </Step>
      <Step title="検証">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    非対話形式：

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` と `--custom-model-id` は省略可能です。省略すると、ローカルのデフォルトホストと `gemma4` の推奨モデルが使用されます。

  </Tab>

  <Tab title="手動セットアップ">
    <Steps>
      <Step title="Ollama をインストールして起動">
        [ollama.com/download](https://ollama.com/download) から入手し、モデルをプルします。

        ```bash
        ollama pull gemma4
        ```

        ハイブリッドクラウドアクセスの場合は、同じホストで `ollama signin` を実行します。
      </Step>
      <Step title="認証情報を設定">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # ローカル/LAN ホストでは任意の値を使用可能
        export OLLAMA_API_KEY="your-real-key"   # https://ollama.com のみ
        ```

        または設定で指定します：`openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`。
      </Step>
      <Step title="モデルを選択">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        または設定で指定します。

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## ローカルホスト経由のクラウドモデル

`Cloud + Local` は、ローカルモデルと `:cloud` モデルの両方を、到達可能な 1 つの
Ollama ホスト経由でルーティングします。これは Ollama のハイブリッドフローであり、
両方を使用する場合にセットアップ中に選択するモードです。

OpenClaw はベース URL の入力を求め、ローカルモデルを検出し、
`ollama signin` の状態を確認します。サインイン済みの場合は、ホスト型サービスのデフォルト
（`kimi-k2.5:cloud`、`minimax-m2.7:cloud`、`glm-5.1:cloud`、`glm-5.2:cloud`）を提案します。
サインインしていない場合、`ollama signin` を実行するまでセットアップはローカルのみのままです。

ローカルデーモンを使用せずにクラウドのみにアクセスする場合は、`openclaw onboard --auth-choice ollama-cloud` を使用し、[Ollama Cloud](/ja-JP/providers/ollama-cloud) を参照してください。この経路では `ollama signin` も実行中のサーバーも必要ありません。

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

`openclaw onboard` 中に表示されるクラウドモデルの一覧は
`https://ollama.com/api/tags` からリアルタイムで取得され、最大 500 件に制限されるため、
選択画面には現在のホスト型カタログが反映されます。セットアップ時に `ollama.com` に到達できない場合やモデルが返されない場合、
OpenClaw はハードコードされた推奨リストにフォールバックするため、
オンボーディングは引き続き完了できます。

## モデル検出（暗黙的プロバイダー）

`OLLAMA_API_KEY`（または認証プロファイル）が設定されており、
`models.providers.ollama` も、`api: "ollama"` を持つ別のカスタムプロバイダーも
定義されていない場合、OpenClaw は `http://127.0.0.1:11434` からモデルを検出します。

| 動作             | 詳細                                                                                                                                                                                                                                                                                        |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| カタログクエリ        | `/api/tags`                                                                                                                                                                                                                                                                                   |
| 機能検出 | ベストエフォートの `/api/show` が、`contextWindow`、`num_ctx` の Modelfile パラメーター、および機能（ビジョン、ツール、思考）を読み取ります                                                                                                                                                                       |
| ビジョンモデル        | `/api/show` の `vision` 機能により、モデルが画像対応（`input: ["text", "image"]`）としてマークされます                                                                                                                                                                                             |
| 推論検出  | 利用可能な場合は `/api/show` の `thinking` 機能を使用します。Ollama が機能を省略した場合は、名前のヒューリスティック（`r1`、`reason`、`reasoning`、`think`）にフォールバックします。報告された機能にかかわらず、`glm-5.2:cloud` と `deepseek-v4-flash\|pro:cloud` は常に推論モデルとして扱われます。 |
| トークン制限         | `maxTokens` のデフォルトは OpenClaw の Ollama 最大トークン上限です                                                                                                                                                                                                                                       |
| コスト                | すべてのコストは `0` です                                                                                                                                                                                                                                                                             |

```bash
ollama list
openclaw models list
```

明示的な `models` 配列を持つ `models.providers.ollama`、または
`api: "ollama"` と非ループバックの `baseUrl` を持つカスタムプロバイダーを設定すると、
自動検出は無効になります。その場合、モデルを手動で定義する必要があります
（[設定](#configuration)を参照）。ホスト型の `https://ollama.com` を参照する
`models.providers.ollama` エントリでも、Ollama Cloud モデルはプロバイダーによって管理されるため、
検出がスキップされます。`http://127.0.0.2:11434` などのループバックのカスタムプロバイダーは
引き続きローカルとして扱われ、自動検出が維持されます。

手動で `models.json` エントリを作成せずに、`ollama/<pulled-model>:latest` のような
完全な参照を使用できます。OpenClaw はそれをリアルタイムで解決します。サインイン済みの
ホストでは、一覧にない `ollama/<model>:cloud` 参照を選択すると、`/api/show` を使用して
その正確なモデルを検証し、Ollama がメタデータを確認した場合に限りランタイムカタログへ追加します。入力ミスは引き続き不明なモデルとして失敗します。

### スモークテスト

エージェントのツールサーフェス全体を省略する限定的なテキストプローブ：

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Reply with exactly: pong" \
    --json
```

軽量なビジョンモデルプローブには、画像とともに `--file` を追加します（PNG/JPEG/WebP に対応。
画像以外のファイルは Ollama が呼び出される前に拒否されます。音声には
`openclaw infer audio transcribe` を使用してください）。

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Describe this image in one sentence." \
    --file ./photo.jpg \
    --json
```

どちらの経路も、チャットツール、メモリ、セッションコンテキストを読み込みません。これが成功する一方で
通常のエージェント応答が失敗する場合、問題はエンドポイントではなく、モデルのツールまたはエージェント処理能力にある可能性が高いです。

`/model ollama/<model>` でのモデル選択は、ユーザーによる厳密な選択です。設定された
`baseUrl` に到達できない場合、別の設定済みモデルへ暗黙的にフォールバックせず、
次の応答はプロバイダーエラーで失敗します。

分離された cron ジョブは、エージェントターンを開始する前にローカルの安全性チェックを 1 つ追加します。
選択したモデルが local/private-network/`.local` の Ollama
プロバイダーに解決され、`/api/tags` に到達できない場合、OpenClaw はその実行を
`skipped` として記録し、エラーテキストにモデルを含めます。このエンドポイントチェックは
ホストごとに 5 分間キャッシュされるため、停止したデーモンに対する cron ジョブが繰り返されても、
すべてが失敗するリクエストを起動することはありません。

ライブ検証:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Ollama Cloud の場合は、同じライブテストをホスト型エンドポイントに向けます（デフォルトでは
埋め込みをスキップします。クラウドキーでは `/api/embed` が
許可されていない場合があるため、`OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` で強制します）。

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

モデルを追加するには、それを pull すると自動的に検出されます。

```bash
ollama pull mistral
```

## Node ローカル推論

エージェントは、ペアリングされたデスクトップまたはサーバー Node 上の Ollama モデルに
短いタスクを委任できます。プロンプトと応答は既存の認証済み
Gateway/Node 接続を通過し、リクエストは Node 自身の loopback Ollama
エンドポイント（`http://127.0.0.1:11434`）で実行されます。

<Steps>
  <Step title="Node で Ollama を起動する">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="Node ホストを接続する">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "Local inference"
    ```

    Gateway ホストでデバイスとその Node コマンドを承認してから、確認します。

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    初回接続、または Ollama コマンドを追加するアップグレードによって、
    Node コマンドの承認がトリガーされることがあります。Node が
    `ollama.models` と `ollama.chat` を公開せずに接続された場合は、
    `openclaw nodes pending` を再度確認してください。

  </Step>
  <Step title="エージェントから使用する">
    バンドルされた Ollama Plugin は `node_inference` ツールを公開します。エージェントは
    まず `action: "discover"` を呼び出し、次にその結果に含まれる Node とモデルを指定して
    `action: "run"` を呼び出します（対応可能な Node が 1 つだけ接続されている場合、
    `run` では Node を省略できます）。例: 「Node 上の Ollama モデルを検出してから、
    読み込み済みの最速モデルを使用してこのテキストを要約してください。」
  </Step>
</Steps>

検出では `/api/tags` を読み取り、`/api/show` の機能を確認し、
利用可能な場合は `/api/ps` を使用して、すでに読み込まれているモデルを優先的にランク付けします。
Ollama がチャット対応（`completion` 機能）として報告するローカルモデルのみを返します。
Ollama Cloud の行と埋め込み専用モデルは除外されます。各実行ではモデルの思考を無効にし、
ツール呼び出しで別の `maxTokens` が要求されない限り、出力はデフォルトで 512 トークン
（上限 8192）になります。一部のモデル（GPT-OSS など）は思考の無効化をサポートしておらず、
推論トークンを出力する場合があります。

Ollama をエージェントに公開せず、Node 上で実行し続けるには、次を実行します。

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

Node を再起動します（`openclaw node restart`、またはフォアグラウンドセッションの場合は
`openclaw node run` を停止して再実行します）。Node は `ollama.models` と
`ollama.chat` の公開を停止しますが、Ollama 自体と Gateway の Ollama プロバイダーには影響しません。
値を `true` に戻して再起動すると、再度有効になります。コマンドサーフェスが変更された場合は、
再接続後に `openclaw nodes pending` の承認が再び必要になることがあります。

エージェントターンを使用せず、Node コマンドを直接確認します。

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout` は、Node がコマンドを実行できる時間を制限します。
`--timeout` は Gateway 呼び出し全体を制限するため、より大きな値にする必要があります。

Node ローカル推論では、常に Node 自身の loopback エンドポイントが使用されます。
設定済みのリモート/クラウド `models.providers.ollama.baseUrl` は再利用されません。
Node コマンドは、macOS、Linux、Windows の Node ホストでデフォルトで利用でき、
通常の Node ペアリング/コマンドポリシーが引き続き適用されます。

## ビジョンと画像の説明

バンドルされた Ollama Plugin は、Ollama を画像対応の
メディア理解プロバイダーとして登録します。そのため OpenClaw は、明示的な画像説明リクエストと、
設定済みの画像モデルのデフォルトを、ローカルまたはホスト型の Ollama ビジョンモデルにルーティングできます。

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` は完全な `<provider/model>` 参照でなければなりません。設定されている場合、
`infer image
describe` は、ネイティブビジョンをすでにサポートするモデルで説明をスキップする代わりに、
最初にそのモデルを試します。呼び出しが失敗した場合、OpenClaw は
`agents.defaults.imageModel.fallbacks` を通じて続行できます。ファイル/URL の準備エラーは、
フォールバックが試行される前に失敗します。OpenClaw の画像理解フローと設定済みの
`imageModel` には `infer image describe` を使用し、カスタムプロンプトによる
生のマルチモーダルプローブには `infer model run
--file` を使用します。

Ollama を受信メディアのデフォルトの画像理解プロバイダーにするには、次のようにします。

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

完全な `ollama/<model>` 参照を推奨します。`qwen2.5vl:7b` のような
ベアな `imageModel` 参照は、その正確なモデルが
`input: ["text", "image"]` を伴って `models.providers.ollama.models` の下に記載され、
同じベア ID を公開する設定済みの画像プロバイダーが他にない場合に限り、
`ollama/qwen2.5vl:7b` に正規化されます。それ以外の場合は、プロバイダープレフィックスを明示的に使用してください。

低速なローカルビジョンモデルでは、クラウドモデルよりも長い画像理解タイムアウトが必要になることがあります。
また、Ollama がモデルで公開されているビジョンコンテキスト全体を割り当てようとすると、
リソースが制約されたハードウェアでクラッシュする可能性があります。機能のタイムアウトを設定し、
`num_ctx` に上限を設定します。

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

このタイムアウトは、受信画像の理解と明示的な
`image` ツールに適用されます。通常のモデル呼び出しでは、
`models.providers.ollama.timeoutSeconds` が引き続き基盤となる Ollama HTTP リクエストのガードを制御します。

ライブ検証:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

`models.providers.ollama.models` を手動で定義する場合は、ビジョンモデルを
明示的にマークします。

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw は、画像対応としてマークされていないモデルへの画像説明リクエストを拒否します。
暗黙的な検出では、これは `/api/show` のビジョン機能から取得されます。

## 設定

<Tabs>
  <Tab title="基本（暗黙的な検出）">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    `OLLAMA_API_KEY` が設定されている場合、プロバイダーエントリで `apiKey` を省略できます。OpenClaw が可用性チェック用に補完します。
    </Tip>

  </Tab>

  <Tab title="明示的（手動モデル）">
    ホスト型クラウドのセットアップ、デフォルト以外のホスト/ポート、コンテキストウィンドウの強制、
    または完全に手動のモデル一覧には、明示的な設定を使用します。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="カスタムベース URL">
    明示的な設定では自動検出が無効になるため、モデルを列挙する必要があります。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // /v1 なし - ネイティブ Ollama API URL
            api: "ollama", // 明示的: ネイティブのツール呼び出し動作を保証
            timeoutSeconds: 300, // 任意: コールド状態のローカルモデル向けに接続/ストリームの時間枠を延長
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // 任意: ターン間でモデルを読み込み済みのまま維持
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    `/v1` を追加しないでください。このパスでは OpenAI 互換モードが選択されますが、ツール呼び出しは信頼できません。
    </Warning>

  </Tab>
</Tabs>

## 一般的なレシピ

モデル ID は、`ollama list` または
`openclaw models list --provider ollama` の正確な名前に置き換えてください。

<AccordionGroup>
  <Accordion title="自動検出を使用するローカルモデル">
    Gateway と同じマシン上の Ollama が自動的に検出されます。

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    手動モデルが必要でない限り、`models.providers.ollama` ブロックを追加しないでください。

  </Accordion>

  <Accordion title="手動モデルを使用する LAN Ollama ホスト">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow` は OpenClaw のコンテキスト上限で、`params.num_ctx` は
    Ollama に送信されます。ハードウェアでモデルの公開済みコンテキスト全体を実行できない場合は、
    両者を一致させてください。

  </Accordion>

  <Accordion title="Ollama Cloud のみ">
    ローカルデーモンを使用せず、ホスト型モデルを直接使用します。

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    この形式の代わりに専用の `ollama-cloud` プロバイダー ID を使用する場合は、
    [Ollama Cloud](/ja-JP/providers/ollama-cloud)を参照してください。

  </Accordion>

  <Accordion title="サインイン済みデーモンを介したクラウドとローカルの併用">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="複数の Ollama ホスト">
    複数の Ollama サーバーを実行する場合は、カスタムプロバイダー ID を使用します。それぞれに
    独自のホスト、モデル、認証、タイムアウトが設定されます。

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw は Ollama を呼び出す前に、アクティブなプロバイダーのプレフィックス
    （プレーンな `ollama/` プレフィックスにフォールバック）を削除するため、
    `ollama-large/qwen3.5:27b` は `qwen3.5:27b` として Ollama に渡されます。

  </Accordion>

  <Accordion title="軽量なローカルモデルプロファイル">
    一部のローカルモデルは単純なプロンプトには対応できますが、エージェントの
    ツールサーフェス全体を扱うのは困難です。グローバルなランタイム設定を変更する前に、
    ツールとコンテキストを制限してください。

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    `compat.supportsTools: false` は、モデルまたはサーバーがツールスキーマで確実に
    失敗する場合にのみ使用してください。安定性と引き換えにエージェントの能力が低下します。
    `localModelLean` は、明示的に必要とされない限り、負荷の高いブラウザー、Cron、メッセージ、メディア生成、
    音声、PDF の各ツールをエージェントの直接的なサーフェスから除外し、
    より大きなカタログを Tool Search の背後に配置します。Ollama の
    ランタイムコンテキストや思考モードは変更しません。ループする、または
    隠れた推論に予算を費やす小規模な Qwen 系思考モデルでは、`params.num_ctx` および
    `params.thinking: false` と組み合わせてください。

  </Accordion>
</AccordionGroup>

### モデルの選択

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

カスタムプロバイダー ID も同じように機能します。`ollama-spark/qwen3:32b` のように
アクティブなプロバイダーのプレフィックスを使用する参照の場合、OpenClaw は
Ollama を呼び出す前にそのプレフィックスを削除し、`qwen3:32b` を送信します。

低速なローカルモデルでは、エージェントランタイム全体のタイムアウトを延長する前に、
プロバイダー単位の調整を優先してください。

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds` は、接続の確立、ヘッダー、本文のストリーミング、
保護された fetch の中止までを含む、モデルへの HTTP リクエスト全体を対象とします。`params.keep_alive` は
ネイティブの `/api/chat` リクエストでトップレベルの `keep_alive` として転送されます。初回ターンの
読み込み時間がボトルネックの場合は、モデルごとに設定してください。

### クイック検証

```bash
# このマシンから Ollama デーモンが見えることを確認
curl http://127.0.0.1:11434/api/tags

# OpenClaw のカタログと選択されたモデル
openclaw models list --provider ollama
openclaw models status

# モデルを直接スモークテスト
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "Reply with exactly: ok"
```

リモートホストでは、`127.0.0.1` を `baseUrl` ホストに置き換えてください。`curl` は
動作するのに OpenClaw が動作しない場合は、Gateway が別の
マシン、コンテナ、またはサービスアカウントで実行されていないか確認してください。

## Ollama Web Search

OpenClaw には、`web_search` プロバイダーとして **Ollama Web Search** がバンドルされています。

| プロパティー | 詳細                                                                                                                                                     |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ホスト      | 設定されている場合は `models.providers.ollama.baseUrl`、それ以外は `http://127.0.0.1:11434`。`https://ollama.com` はホスト型 API を直接使用します                          |
| 認証        | サインイン済みのローカルホストではキー不要。`https://ollama.com` を直接検索する場合または認証で保護されたホストでは、`OLLAMA_API_KEY` または設定済みのプロバイダー認証を使用します           |
| 要件        | ローカルまたはセルフホストのホストが実行中であり、`ollama signin` でサインイン済みである必要があります。ホスト型検索を直接使用するには、`baseUrl: "https://ollama.com"` と実際の API キーが必要です |

`openclaw onboard` または `openclaw configure --section web` の実行中に選択するか、次のように設定します。

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Ollama Cloud 経由でホスト型検索を直接使用する場合は、次のように設定します。

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

セルフホストのホストでは、OpenClaw は最初にローカルの `/api/experimental/web_search`
プロキシを試し、その後、同じホスト上のホスト型 `/api/web_search` パスにフォールバックします。
通常、サインイン済みのローカルデーモンはローカルプロキシ経由で応答します。`https://ollama.com` を
直接呼び出す場合は、常にホスト型の `/api/web_search` エンドポイントを使用します。

<Note>
完全なセットアップと動作については、[Ollama Web Search](/ja-JP/tools/ollama-search)を参照してください。
</Note>

## 高度な設定

<AccordionGroup>
  <Accordion title="従来の OpenAI 互換モード">
    <Warning>
    **このモードではツール呼び出しの信頼性がありません。** プロキシで OpenAI 形式が必要で、ネイティブのツール呼び出しに依存しない場合にのみ使用してください。
    </Warning>

    `/v1/chat/completions` の背後にあるプロキシでは、`api: "openai-completions"` を
    明示的に設定してください。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // デフォルト: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    このモードでは、ストリーミングとツール呼び出しの同時使用がサポートされない場合があります。
    モデルで `params: { streaming: false }` が必要になることがあります。

    このモードでは、Ollama が暗黙的に 4096 トークンのコンテキストへ
    フォールバックしないように、OpenClaw はデフォルトで `options.num_ctx` を挿入します。
    プロキシが未知の `options` フィールドを拒否する場合は、無効にしてください。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="コンテキストウィンドウ">
    自動検出されたモデルでは、OpenClaw は `/api/show` が報告するコンテキストウィンドウを使用します。
    これには、カスタム Modelfile のより大きな `PARAMETER num_ctx` 値も含まれます。
    それ以外の場合は、OpenClaw のデフォルトの Ollama コンテキストウィンドウにフォールバックします。

    プロバイダーレベルの `contextWindow`、`contextTokens`、`maxTokens` は、
    そのプロバイダー配下のすべてのモデルにデフォルト値を設定し、モデルごとに
    上書きできます。`contextWindow` は OpenClaw 独自のプロンプトおよび Compaction の予算です。ネイティブの
    `/api/chat` リクエストでは、`params.num_ctx` を明示的に設定しない限り
    `options.num_ctx` は未設定のままになり、Ollama 独自のモデル、
    `OLLAMA_CONTEXT_LENGTH`、または VRAM に基づくデフォルトが適用されます。無効、ゼロ、負、
    または有限でない `params.num_ctx` 値は無視されます。古い設定でネイティブリクエストの
    コンテキストを強制するために `contextWindow`/`maxTokens` のみを使用していた場合は、
    `openclaw doctor --fix` を実行して、それらを `params.num_ctx` にコピーしてください。
    OpenAI 互換アダプターは引き続き、設定された `params.num_ctx` または
    `contextWindow` からデフォルトで `options.num_ctx` を挿入します。アップストリームが
    `options` を拒否する場合は、`injectNumCtxForOpenAICompat: false` で無効にしてください。

    ネイティブモデルのエントリーでは、`params` 配下に一般的な Ollama ランタイムオプションも指定でき、
    ネイティブの `/api/chat` `options` として転送されます: `num_keep`、`seed`、
    `num_predict`、`top_k`、`top_p`、`min_p`、`typical_p`、`repeat_last_n`、
    `temperature`、`repeat_penalty`、`presence_penalty`、`frequency_penalty`、
    `stop`、`num_batch`、`num_gpu`、`main_gpu`、`use_mmap`、`num_thread`。
    一部のキー（`format`、`keep_alive`、`truncate`、`shift`）は、
    `options` にネストされず、トップレベルのリクエストフィールドとして転送されます。OpenClaw が
    転送するのはこれらの Ollama リクエストキーのみであるため、`streaming` のような
    ランタイム専用のパラメーターが Ollama に送信されることはありません。トップレベルの
    `think` を設定するには、`params.think`（または
    `params.thinking`）を使用してください。`false` は、Qwen 系思考モデルの
    API レベルの思考を無効にします。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    モデルごとの `agents.defaults.models["ollama/<model>"].params.num_ctx` も
    機能します。両方が設定されている場合は、明示的なプロバイダーモデルエントリが優先されます。

  </Accordion>

  <Accordion title="Thinking の制御">
    OpenClaw は Ollama が期待する形式で Thinking を転送します。つまり、`options.think` ではなく
    トップレベルの `think` です。自動検出されたモデルのうち、`/api/show` が
    `thinking` 機能を報告するものでは、`/think low`、`/think medium`、`/think high`、
    `/think max` が公開されます。Thinking 非対応モデルでは `/think off` のみが公開されます。

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    または、モデルのデフォルトを設定します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    モデルごとの `params.think`/`params.thinking` により、特定モデルの API
    Thinking を無効化または強制できます。アクティブな実行に暗黙の `off`
    デフォルトしかない場合、OpenClaw はその明示的な設定を保持します。ただし、`/think medium` のような
    off 以外のランタイムコマンドは、引き続きその設定を上書きします。明示的に
    `reasoning: false` と指定されたモデルには、真と評価される Thinking リクエストは送信されません。
    `think: false` リクエストは常に送信されます。

  </Accordion>

  <Accordion title="推論モデル">
    `deepseek-r1`、`reasoning`、`reason`、`think` という名前のモデルは、
    デフォルトで推論対応として扱われます。追加設定は不要です。

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="モデルのコスト">
    Ollama はローカルで実行され、無料であるため、自動検出されたモデルと手動定義されたモデルの
    どちらでも、すべてのモデルコストは `0` です。
  </Accordion>

  <Accordion title="メモリ埋め込み">
    同梱の Ollama Plugin は、[メモリ検索](/ja-JP/concepts/memory)用のメモリ埋め込みプロバイダーを
    登録します。設定済みの Ollama ベース URL と API キーを使用して
    `/api/embed` を呼び出し、可能な場合は複数のメモリチャンクを 1 回の
    `input` リクエストにまとめます。

    `proxy.enabled=true` の場合、設定された `baseUrl` から導出される、正確にホストローカルの
    loopback オリジンへの埋め込みリクエストでは、管理対象の転送プロキシではなく
    OpenClaw の保護された直接パスが使用されます。設定するホスト名自体が
    `localhost` または loopback IP リテラルでなければなりません。単に DNS 解決の結果が
    loopback になる名前では、引き続き管理対象プロキシパスが使用されます。LAN、
    tailnet、プライベートネットワーク、およびパブリックの Ollama ホストでは常に
    管理対象プロキシパスが使用され、別のホストやポートへのリダイレクトには
    信頼が引き継がれません。`proxy.loopbackMode: "proxy"` では loopback トラフィックも
    プロキシ経由になります。`proxy.loopbackMode: "block"` では接続前に拒否されます。
    [管理対象プロキシ](/ja-JP/security/network-proxy#gateway-loopback-mode)を参照してください。

    | プロパティ | 値 |
    | --- | --- |
    | デフォルトモデル | `nomic-embed-text` |
    | 自動 pull | ローカルに存在しない場合は実行 |
    | デフォルトのインライン並行数 | 1（ほかのプロバイダーのデフォルトはより高い値です。ホストに余力がある場合は `nonBatchConcurrency` で増やしてください） |

    クエリ時の埋め込みでは、取得用プレフィックスが必須または推奨されるモデル、
    `nomic-embed-text`、`qwen3-embedding`、`mxbai-embed-large` に対して
    それらを使用します。ドキュメントのバッチは未加工のままなので、既存のインデックスに
    形式の移行は不要です。

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // Ollama のデフォルトです。再インデックスが遅すぎる場合は、大規模なホストで増やしてください。
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    リモートの埋め込みホストでは、認証のスコープをそのホストに限定してください。

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="ストリーミング設定">
    Ollama はデフォルトで **ネイティブ API**（`/api/chat`）を使用します。これは
    ストリーミングとツール呼び出しの併用をサポートしており、特別な設定は不要です。

    ネイティブリクエストでは、Thinking の制御が直接転送されます。明示的な
    `params.think`/`params.thinking` が設定されていない限り、`/think off`
    と `openclaw agent --thinking off` はトップレベルの `think: false` を送信します。`/think
    low|medium|high` は対応する
    effort 文字列を送信します。`/think max` は Ollama の最大 effort である
    `think: "high"` に対応します。

    <Tip>
    代わりに OpenAI 互換エンドポイントを使用する場合は、上記の「従来の OpenAI 互換モード」を参照してください。このモードでは、ストリーミングとツール呼び出しが併用できない場合があります。
    </Tip>

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="WSL2 のクラッシュループ（再起動の繰り返し）">
    NVIDIA/CUDA を使用する WSL2 では、公式の Ollama Linux インストーラーが
    `Restart=always` を指定した `ollama.service` systemd ユニットを作成します。そのサービスが
    自動起動し、WSL2 の起動中に GPU を使用するモデルを読み込むと、Ollama が読み込み中に
    ホストメモリを固定することがあります。Hyper-V のメモリ回収ではそのページを必ずしも
    回収できないため、Windows が WSL2 VM を終了し、systemd が
    Ollama を再起動して、ループが繰り返されることがあります。

    判断材料としては、WSL2 の再起動や終了が繰り返されること、WSL2 起動直後に
    `app.slice` または `ollama.service` の CPU 使用率が高くなること、Linux の OOM killer ではなく
    systemd から SIGTERM が送られることが挙げられます。

    OpenClaw は、WSL2、`Restart=always` を指定して有効化された `ollama.service`、
    および可視の CUDA マーカーを検出すると、起動時に警告を記録します。

    対処方法：

    ```bash
    sudo systemctl disable ollama
    ```

    Windows 側では、次の内容を `%USERPROFILE%\.wslconfig` に追加してから
    `wsl --shutdown` を実行します。

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    または、keep-alive を短くするか、必要なときだけ Ollama を手動で起動します。

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317)を参照してください。

  </Accordion>

  <Accordion title="Ollama が検出されない">
    Ollama が実行中であること、`OLLAMA_API_KEY`（または認証プロファイル）が設定されていること、
    および `models.providers.ollama` が明示的に定義されて**いない**ことを確認してください。

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="利用可能なモデルがない">
    モデルをローカルに pull するか、
    `models.providers.ollama` で明示的に定義します。

    ```bash
    ollama list  # インストール済みのモデルを確認
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # または別のモデル
    ```

  </Accordion>

  <Accordion title="接続が拒否される">
    ```bash
    # Ollama が実行中か確認
    ps aux | grep ollama

    # または Ollama を再起動
    ollama serve
    ```

  </Accordion>

  <Accordion title="リモートホストは curl では動作するが OpenClaw では動作しない">
    Gateway を実行しているものと同じマシンおよびランタイムから確認してください。

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    一般的な原因：

    - `baseUrl` が `localhost` を指しているが、Gateway は Docker 内または別のホストで実行されている。
    - URL が `/v1` を使用しているため、ネイティブ Ollama ではなく OpenAI 互換の動作が選択されている。
    - リモートホストでファイアウォールまたは LAN バインドの変更が必要。
    - モデルがノート PC のデーモンには存在するが、リモート側には存在しない。

  </Accordion>

  <Accordion title="モデルがツール JSON をテキストとして出力する">
    通常、プロバイダーが OpenAI 互換モードになっているか、モデルが
    ツールスキーマを処理できないことが原因です。ネイティブモードを推奨します。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    小規模なローカルモデルが引き続きツールスキーマで失敗する場合は、そのモデルエントリに
    `compat.supportsTools: false` を設定して再テストしてください。

  </Accordion>

  <Accordion title="Kimi または GLM が文字化けした記号を返す">
    ホスト型 Kimi/GLM の応答が、言語として意味をなさない記号の長い連続である場合、
    成功した応答ではなく、失敗したプロバイダー呼び出しとして扱われます。そのため、
    破損したテキストをセッションに保存する代わりに、通常の再試行、フォールバック、
    エラー処理が実行されます。

    再発した場合は、モデル名、現在のセッションファイル、および実行で
    `Cloud + Local` と `Cloud only` のどちらを使用したかを記録し、新しい
    セッションとフォールバックモデルを試してください。

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Reply with exactly: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="コールド状態のローカルモデルがタイムアウトする">
    大規模なローカルモデルは、初回読み込みに時間がかかることがあります。タイムアウトを
    Ollama プロバイダーに限定し、必要に応じてターン間もモデルを読み込み済みの状態に保ちます。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    ホスト自体が接続を受け付けるまでに時間がかかる場合は、`timeoutSeconds` により
    このプロバイダーの保護された接続タイムアウトも延長されます。

  </Accordion>

  <Accordion title="大規模コンテキストモデルが遅すぎる、またはメモリ不足になる">
    多くのモデルは、ハードウェアで無理なく実行できる容量を超えるコンテキストを
    公称値として示します。ネイティブ Ollama は、`params.num_ctx` が設定されていない限り、
    独自のランタイムデフォルトを使用します。最初のトークンが返るまでのレイテンシーを予測可能にするには、
    OpenClaw の割り当てと Ollama のリクエストコンテキストの両方に上限を設定します。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw が送信するプロンプトが多すぎる場合は、`contextWindow` を下げます。
    Ollama のランタイムコンテキストがマシンに対して大きすぎる場合は、`params.num_ctx` を下げます。
    生成に時間がかかりすぎる場合は、`maxTokens` を下げます。

  </Accordion>
</AccordionGroup>

<Note>
詳細なヘルプについては、[トラブルシューティング](/ja-JP/help/troubleshooting)と[よくある質問](/ja-JP/help/faq)を参照してください。
</Note>

## 関連項目

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/ja-JP/providers/ollama-cloud" icon="cloud">
    専用の `ollama-cloud` プロバイダーを使用する、クラウド専用のセットアップです。
  </Card>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    すべてのプロバイダー、モデル参照、フェイルオーバー動作の概要です。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/models" icon="brain">
    モデルを選択して設定する方法です。
  </Card>
  <Card title="Ollama Web 検索" href="/ja-JP/tools/ollama-search" icon="magnifying-glass">
    Ollama を利用した Web 検索の完全なセットアップと動作の詳細です。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    完全な設定リファレンスです。
  </Card>
</CardGroup>
