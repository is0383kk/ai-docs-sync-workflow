---
read_when:
    - メモリ検索プロバイダーまたは埋め込みモデルを設定する場合
    - QMD バックエンドをセットアップする場合
    - ハイブリッド検索、MMR、または時間減衰を有効にしたい場合
    - マルチモーダルメモリのインデックス作成を有効にする場合
sidebarTitle: Memory config
summary: メモリ検索プロバイダー、取得モード、QMD、マルチモーダルインデックス作成
title: メモリ設定リファレンス
x-i18n:
    generated_at: "2026-07-26T09:50:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91f843b1516093c49e18b3d659ab24ea9cb7be32aaaac722205eca8bc3f2ca5b
    source_path: reference/memory-config.md
    workflow: 16
---

このページでは、OpenClaw のメモリ検索に関するすべての設定項目を一覧で説明します。概念の概要については、以下を参照してください。

<CardGroup cols={2}>
  <Card title="メモリの概要" href="/ja-JP/concepts/memory">
    メモリの仕組み。
  </Card>
  <Card title="組み込みエンジン" href="/ja-JP/concepts/memory-builtin">
    デフォルトの SQLite バックエンド。
  </Card>
  <Card title="QMD エンジン" href="/ja-JP/concepts/memory-qmd">
    ローカルファーストのサイドカー。
  </Card>
  <Card title="メモリ検索" href="/ja-JP/concepts/memory-search">
    検索パイプラインとチューニング。
  </Card>
  <Card title="Active Memory" href="/ja-JP/concepts/active-memory">
    対話型セッション用のメモリサブエージェント。
  </Card>
</CardGroup>

共有メモリのすべての設定は、`openclaw.json` の最上位 `memory` にあります。検索のデフォルト設定には `memory.search` を使用し、エージェントごとの検索オーバーライドには `agents.entries.*.memory.search` を使用します。

<Note>
推奨されるパーソナルエージェントのワークフローでは、
`memory.search.rememberAcrossConversations` を使用します。高度な Active Memory の対象指定、
モデル、プロンプト、レイテンシーの制御は `plugins.entries.active-memory` にあります。

両方の有効化方法、トランスクリプトの永続化、安全なロールアウトのガイダンスについては、
[Active Memory](/ja-JP/concepts/active-memory) を参照してください。
</Note>

---

## 会話をまたいで記憶する

| キー                           | 型      | デフォルト                                                    | 説明                                                                    |
| ----------------------------- | --------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `rememberAcrossConversations` | `boolean` | 個人用インストールではオン、DM 分離が設定されている場合はオフ | このエージェントが認識している他の非公開会話から、関連するコンテキストを使用します。 |

信頼できるパーソナルエージェントだけが会話をまたいだトランスクリプトの想起を使用する場合は、
エージェントごとに設定します。

```json5
{
  agents: {
    entries: {
      personal: {
        memory: {
          search: {
            rememberAcrossConversations: true,
          },
        },
      },
    },
  },
}
```

この値は通常の `memory.search` 継承に従い、
エージェントごとにオーバーライドできます。未設定の場合、グローバルな
`session.dmScope` が未設定または `"main"` であり、どのバインディングにも `session.dmScope`
オーバーライドがない場合にのみ、デフォルトでオンになります。DM 分離が設定されている場合、デフォルトはオフになります。明示的な `true` または
`false` が常に優先されます。有効にするとセッショントランスクリプトのインデックス作成も有効になり、
エージェントの解決済みメモリソースに `sessions` が追加されます。QMD では、
そのエージェントのセッションエクスポートも有効になります。このモードでは別途
`memory.qmd.sessions.enabled` を設定する必要はありません。

OpenClaw の組み込みメモリプロバイダーは、組み込みバックエンドと QMD バックエンドの両方で、
この保護されたパスをサポートします。代替メモリプロバイダーは独自の
想起フックと高度な Active Memory ツールを引き続き使用できますが、現在のプロバイダーが
保護された非公開トランスクリプトの想起をサポートしていない限り、この設定はスキップされます。
`openclaw doctor` は、サポートされていないプロバイダー、または `memory_search` を省略した明示的な Active Memory の
`toolsAllow` リストを報告します。

取得の境界は一般的なセッション検索よりも狭くなります。

- 対象となるのは、同じエージェントが認識している非公開会話のみです
- 回答対象の会話は除外されます
- グループとチャンネルは、取得元と取得先の両方から除外されます
- 会話の種類が不明な場合はフェイルクローズします
- サンドボックス化された想起では、会話をまたぐ特別な認可を使用できません

この設定は、`tools.sessions.visibility`、セッションキー、
トランスクリプトの保存、配信ルーティング、または `sessions_list`、
`sessions_history`、`sessions_send` の権限を変更しません。Active Memory は範囲を限定した
読み取り専用の取得処理を実行します。取得が利用できない場合やタイムアウトした場合でも、
応答はブロックされません。

---

## プロバイダーの選択

| キー        | 型      | デフォルト          | 説明                                                                                                                                                                                                                                                                                 |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`  | `boolean` | `true`           | メモリ検索を有効または無効にします                                                                                                                                                                                                                                                             |
| `provider` | `string`  | `"openai"`       | `bedrock`、`deepinfra`、`gemini`、`github-copilot`、`local`、`mistral`、`ollama`、`openai`、`openai-compatible`、`voyage` などの埋め込みアダプター ID。また、`api` がメモリ埋め込みアダプターまたは OpenAI 互換モデル API を指すように設定された `models.providers.<id>` も使用できます |
| `model`    | `string`  | プロバイダーのデフォルト | 埋め込みモデル名                                                                                                                                                                                                                                                                        |
| `fallback` | `string`  | `"none"`         | プライマリが失敗した場合のフォールバックアダプター ID                                                                                                                                                                                                                                                  |

`provider` が設定されていない場合、OpenClaw は OpenAI の埋め込みを使用します。Bedrock、DeepInfra、Gemini、GitHub Copilot、Mistral、Ollama、
Voyage、ローカル GGUF モデル、または OpenAI 互換の `/v1/embeddings` エンドポイントを使用するには、`provider`
を明示的に設定します。
引き続き `provider: "auto"` を使用しているレガシー設定は、`openai` として解決されます。

<Warning>
埋め込みプロバイダー、モデル、プロバイダー設定、ソース、スコープ、
チャンキング、またはトークナイザーを変更すると、既存の SQLite ベクトルインデックスと互換性がなくなる可能性があります。
OpenClaw はすべてを自動的に再埋め込みする代わりに、
ベクトル検索を一時停止し、インデックス識別情報に関する警告を報告します。準備ができたら、
`openclaw memory status --index --agent <id>` または
`openclaw memory index --force --agent <id>` で再構築してください。
</Warning>

`provider` が未設定の場合、レガシーな `provider: "auto"` が存在する場合、または
`provider: "none"` で意図的に FTS のみのモードを選択している場合、埋め込みが利用できなくても、
メモリ想起では字句 FTS ランキングを引き続き使用できます。

明示的に指定された非ローカルプロバイダーはフェイルクローズします。`memory.search.provider` を
Bedrock、DeepInfra、Gemini、GitHub Copilot、LM Studio、Mistral、Ollama、OpenAI、Voyage、OpenAI 互換の
カスタムプロバイダーなど、リモートバックエンドを使用する特定のプロバイダーに設定し、そのプロバイダーが実行時に利用できない場合、
`memory_search` は暗黙的に FTS のみの想起を使用せず、利用不可の結果を返します。
プロバイダーまたは認証の設定を修正するか、到達可能なプロバイダーに切り替えるか、
意図的に FTS のみの想起を使用する場合は `provider: "none"` を設定してください。

### カスタムプロバイダー ID

`memory.search.provider` は、`ollama` などのメモリ専用プロバイダーアダプター、または `openai-responses` / `openai-completions` などの OpenAI 互換モデル API 用のカスタム `models.providers.<id>` エントリを指すことができます。OpenClaw は、エンドポイント、認証、モデルプレフィックスの処理用にカスタムプロバイダー ID を維持しながら、埋め込みアダプター用にそのプロバイダーの `api` 所有者を解決します。これにより、複数 GPU または複数ホストの構成で、メモリ埋め込み専用の特定のローカルエンドポイントを使用できます。

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b", name: "Qwen3 Embedding 0.6B" }],
      },
    },
  },
  memory: {
    search: {
      provider: "ollama-5080",
      model: "qwen3-embedding:0.6b",
    },
  },
}
```

### API キーの解決

リモート埋め込みには API キーが必要です。ただし、Bedrock は代わりに AWS SDK のデフォルト認証情報チェーン（インスタンスロール、SSO、アクセスキー、または Bedrock API キー）を使用します。

| プロバイダー       | 環境変数                                             | 設定キー                          |
| -------------- | --------------------------------------------------- | ----------------------------------- |
| Bedrock        | AWS 認証情報チェーン、または `AWS_BEARER_TOKEN_BEDROCK` | API キーは不要                   |
| DeepInfra      | `DEEPINFRA_API_KEY`                                 | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                    | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、`GITHUB_TOKEN`  | デバイスログイン経由の認証プロファイル       |
| Mistral        | `MISTRAL_API_KEY`                                   | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY`（プレースホルダー）                      | --                                  |
| OpenAI         | `OPENAI_API_KEY`                                    | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                    | `models.providers.voyage.apiKey`    |

<Note>
Codex OAuth はチャットと補完のみを対象とし、埋め込みリクエストには使用できません。
</Note>

---

## リモートエンドポイントの設定

グローバルな OpenAI チャット認証情報を継承しない汎用 OpenAI 互換
`/v1/embeddings` サーバーには、`provider: "openai-compatible"` を使用します。

<ParamField path="remote.baseUrl" type="string">
  カスタム API ベース URL。
</ParamField>
<ParamField path="remote.apiKey" type="string">
  API キーをオーバーライドします。
</ParamField>
<ParamField path="remote.headers" type="object">
  追加の HTTP ヘッダー（プロバイダーのデフォルトとマージされます）。
</ParamField>

```json5
{
  memory: {
    search: {
      provider: "openai-compatible",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_KEY",
      },
    },
  },
}
```

---

## プロバイダー固有の設定

<AccordionGroup>
  <Accordion title="Gemini">
    | キー                    | 型     | デフォルト                | 説明                                |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------- |
    | `model`                | `string` | `gemini-embedding-001` | `gemini-embedding-2-preview` もサポートします |
    | `outputDimensionality` | `number` | `3072`                 | Embedding 2 の場合：768、1536、または 3072        |

    <Warning>
    モデルまたは `outputDimensionality` を変更すると、インデックス識別情報が変わります。OpenClaw は、
    メモリインデックスを明示的に再構築するまでベクトル検索を一時停止します。
    </Warning>

  </Accordion>
  <Accordion title="OpenAI 互換の入力タイプ">
    OpenAI 互換の埋め込みエンドポイントでは、プロバイダー固有の `input_type` リクエストフィールドを任意で使用できます。これは、クエリとドキュメントの埋め込みに異なるラベルを必要とする非対称埋め込みモデルに役立ちます。

    | キー                 | 型     | デフォルト | 説明                                             |
    | ------------------- | -------- | ------- | -------------------------------------------------------- |
    | `inputType`         | `string` | 未設定   | クエリとドキュメントの埋め込みで共有する `input_type`   |
    | `queryInputType`    | `string` | 未設定   | クエリ時の `input_type`。`inputType` をオーバーライドします          |
    | `documentInputType` | `string` | 未設定   | インデックス／ドキュメントの `input_type`。`inputType` をオーバーライドします      |

    ```json5
    {
      memory: {
        search: {
          provider: "openai-compatible",
          remote: {
            baseUrl: "https://embeddings.example/v1",
            apiKey: "${EMBEDDINGS_API_KEY}",
          },
          model: "asymmetric-embedder",
          queryInputType: "query",
          documentInputType: "passage",
        },
      },
    }
    ```

    これらの値を変更すると、プロバイダーのバッチインデックス作成における埋め込みキャッシュの識別情報に影響します。上流モデルがラベルを異なるものとして扱う場合は、変更後にメモリの再インデックスを実行してください。

  </Accordion>
  <Accordion title="Bedrock">
    ### Bedrock 埋め込み設定

    Bedrock は AWS SDK のデフォルト認証情報チェーンと、OpenClaw が確認するベアラートークンを使用するため、設定に API キーは保存されません。OpenClaw が Bedrock 対応のインスタンスロールを持つ EC2 上で動作している場合は、プロバイダーとモデルを設定するだけです。

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0",
        },
      },
    }
    ```

    | キー                    | 型     | デフォルト                        | 説明                     |
    | ---------------------- | -------- | ------------------------------- | -------------------------------- |
    | `model`                | `string` | `amazon.titan-embed-text-v2:0` | 任意の Bedrock 埋め込みモデル ID  |
    | `outputDimensionality` | `number` | モデルのデフォルト                  | Titan V2 の場合：256、512、または 1024 |

    **対応モデル**（ファミリー検出と次元数のデフォルトを含む）：

    | モデル ID                                   | プロバイダー   | デフォルト次元数 | 設定可能な次元数          |
    | ------------------------------------------- | ---------- | ------------- | -------------------------- |
    | `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024             |
    | `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                          |
    | `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072       |
    | `cohere.embed-english-v3`                  | Cohere     | 1024         | --                          |
    | `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                          |
    | `cohere.embed-v4:0`                        | Cohere     | 1536         | 256, 384, 512, 768, 1024, 1536 |
    | `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                          |
    | `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                          |

    スループット接尾辞付きのバリアント（例：`amazon.titan-embed-text-v1:2:8k`）と、リージョン接頭辞付きの推論プロファイル ID（例：`us.amazon.titan-embed-text-v2:0`）は、ベースモデルの設定を継承します。

    **リージョン：** 次の順序で解決されます：`memory.search.remote.baseUrl` のオーバーライド、`models.providers.amazon-bedrock.baseUrl` の設定、`AWS_REGION`、`AWS_DEFAULT_REGION`、最後にデフォルトの `us-east-1`。

    **認証：** OpenClaw は最初に `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` または `AWS_BEARER_TOKEN_BEDROCK` を確認し、その後、標準の AWS SDK デフォルト認証情報プロバイダーチェーンにフォールスルーします。

    1. 環境変数（`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`）。ただし、`AWS_PROFILE` も設定されている場合を除く
    2. SSO（SSO フィールドが設定されている場合のみ）
    3. 共有認証情報ファイルと設定ファイル（`fromIni`、`AWS_PROFILE` を含む）
    4. 認証情報プロセス（AWS 設定ファイル内の `credential_process`）
    5. ウェブアイデンティティトークン認証情報
    6. ECS または EC2 インスタンスメタデータ認証情報

    **IAM 権限：** IAM ロールまたはユーザーには次の権限が必要です。

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    最小権限にするには、`InvokeModel` の範囲を特定のモデルに限定します。

    ```text
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  </Accordion>
  <Accordion title="ローカル（GGUF + llama.cpp）">
    | キー                   | 型               | デフォルト                | 説明                                                                                                                                                                                                                                                                                                          |
    | --------------------- | ------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`     | `string`           | 自動ダウンロード        | GGUF モデルファイルへのパス                                                                                                                                                                                                                                                                                              |
    | `local.modelCacheDir` | `string`           | node-llama-cpp のデフォルト | ダウンロードしたモデルのキャッシュディレクトリ                                                                                                                                                                                                                                                                                      |
    | `local.contextSize`   | `number \| "auto"` | `4096`                 | 埋め込みコンテキストのコンテキストウィンドウサイズ。4096 は一般的なチャンク（128～512 トークン）を処理しながら、重み以外の VRAM 使用量を制限します。リソースに制約のあるホストでは 1024～2048 に下げてください。`"auto"` はモデルが学習した最大値を使用します。8B 以上のモデルには推奨されません（Qwen3-Embedding-8B：最大 40 960 トークンでは VRAM 使用量が約 32 GB に達する可能性があります）。 |

    最初に公式の llama.cpp プロバイダーをインストールします：`openclaw plugins install @openclaw/llama-cpp-provider`。
    デフォルトモデル：`embeddinggemma-300m-qat-Q8_0.gguf`（約 0.6 GB、自動ダウンロード）。ソースチェックアウトでは、引き続きネイティブビルドの承認が必要です：`pnpm approve-builds`、続いて `pnpm rebuild node-llama-cpp`。

    スタンドアロン CLI を使用して、Gateway が使用するものと同じプロバイダーパスを検証します。

    ```bash
    openclaw memory status --deep --agent main
    openclaw memory index --force --agent main
    ```

    数値の `local.contextSize` 値は、モデルの重みと要求された埋め込みコンテキストが同時に収まるように、node-llama-cpp の GPU レイヤー自動配置にも使用されます。`openclaw memory status --deep` は、ランタイムがロードされた後に、最後に確認された llama.cpp のバックエンド、デバイス、オフロード、要求されたコンテキスト、およびタイムスタンプ付きのメモリ情報を報告します。パッシブステータスではモデルをロードしません。

    ローカル GGUF 埋め込みには `provider: "local"` を明示的に設定してください。明示的なローカル設定では、`hf:` および HTTP(S) モデル参照がサポートされています（node-llama-cpp のモデル解決を使用）が、デフォルトプロバイダーは変更されません。

  </Accordion>
</AccordionGroup>

## インデックス作成の動作

メモリエンジンは、同期、バッチ処理、監視、および Compaction 後の
インデックス作成ヒューリスティクスを管理します。OpenClaw はインストールごとの
タイミング切り替えを公開するのではなく、管理されたデフォルトでこれらの動作を有効に保ちます。

## ハイブリッド検索の設定

すべて `memory.search.query` 配下です。

| キー          | 型     | デフォルト | 説明                               |
| ------------ | -------- | ------- | ----------------------------------------- |
| `maxResults` | `number` | `6`     | 注入前に返されるメモリヒットの最大数 |
| `minScore`   | `number` | `0.35`  | ヒットを含めるための最小関連度スコア  |

ハイブリッド検索は引き続き有効です。MMR と時間減衰は、
組み込みエンジンのポリシーによって無効のままです。

### 完全な例

```json5
{
  memory: {
    search: {
      query: {
        maxResults: 6,
        minScore: 0.35,
      },
    },
  },
}
```

---

## 追加のメモリパス

| キー          | 型       | 説明                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | インデックスを作成する追加のディレクトリまたはファイル |

```json5
{
  memory: {
    search: {
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },
}
```

パスには絶対パスまたはワークスペース相対パスを指定できます。ディレクトリでは `.md` ファイルが再帰的にスキャンされます。シンボリックリンクの処理はアクティブなバックエンドによって異なります。組み込みエンジンはシンボリックリンクをスキップしますが、QMD は基盤となる QMD スキャナーの動作に従います。

エージェントスコープのエージェント間トランスクリプト検索には、`memory.qmd.paths` ではなく `agents.entries.*.memory.search.qmd.extraCollections` を使用してください。これらの追加コレクションは同じ `{ path, name, pattern? }` 形式に従いますが、エージェントごとにマージされ、パスが現在のワークスペース外を指す場合は明示的な共有名を維持できます。同じ解決済みパスが `memory.qmd.paths` と `memory.search.qmd.extraCollections` の両方に現れる場合、QMD は最初のエントリを保持し、重複をスキップします。

---

## マルチモーダルメモリ（Gemini）

Gemini Embedding 2 を使用して、Markdown とともに画像と音声のインデックスを作成します。

| キー                       | 型       | デフォルト    | 説明                            |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | マルチモーダルインデックス作成を有効化             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`、`["audio"]`、または `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10485760` | インデックス作成対象の最大ファイルサイズ（10 MiB）    |

<Note>
`extraPaths` 内のファイルにのみ適用されます。デフォルトのメモリルートは Markdown のみのままです。`gemini-embedding-2-preview` が必要です。`fallback` は `"none"` でなければなりません。
</Note>

対応形式：`.jpg`、`.jpeg`、`.png`、`.webp`、`.gif`、`.heic`、`.heif`（画像）、`.mp3`、`.wav`、`.ogg`、`.opus`、`.m4a`、`.aac`、`.flac`（音声）。

---

## 埋め込みキャッシュ

| キー             | 型      | デフォルト | 説明                      |
| --------------- | --------- | ------- | -------------------------------- |
| `cache.enabled` | `boolean` | `true`  | チャンク埋め込みを SQLite にキャッシュ |

再インデックスまたはトランスクリプトの更新時に、変更されていないテキストが再度埋め込み処理されるのを防ぎます。

---

## バッチインデックス作成

| キー                          | 型      | デフォルト | 説明                |
| ---------------------------- | --------- | ------- | -------------------------- |
| `remote.nonBatchConcurrency` | `number`  | `4`     | 並列インライン埋め込み |
| `remote.batch.enabled`       | `boolean` | `false` | バッチ埋め込み API を有効化 |

`gemini`、`openai`、および `voyage` で利用できます。大規模なバックフィルでは、通常 OpenAI のバッチが最も高速かつ低コストです。

並行処理、ポーリング、およびタイムアウトの動作はプロバイダーが管理します。

---

## セッションメモリ検索

セッショントランスクリプトのインデックスを作成し、`memory_search` を介して提示します。

| キー                           | 型       | デフォルト      | 説明                              |
| ----------------------------- | ---------- | ------------ | ---------------------------------------- |
| `rememberAcrossConversations` | `boolean`  | `false`      | 非公開の会話間想起を許可 |
| `sources`                     | `string[]` | `["memory"]` | トランスクリプトを含めるには `"sessions"` を追加  |

<Warning>
セッションのインデックス作成はオプトインで、非同期に実行されます。結果が若干古い場合があります。セッションログはディスク上に保存されるため、ファイルシステムへのアクセスを信頼境界として扱ってください。
</Warning>

通常のモデルから呼び出されるセッショントランスクリプト検索は、
[`tools.sessions.visibility`](/ja-JP/gateway/config-tools#toolssessions) に従います。デフォルトの
`tree` 可視性では、現在のセッション、そのセッションが生成したセッション、および
周辺のグループ認識を通じて監視される同一エージェントのグループセッションが公開されます。その他の
無関係なセッションには `agent` 可視性が必要です（エージェントをまたぐ
想起も必要で、かつエージェント間ポリシーで許可されている場合に限り `all`）。

`rememberAcrossConversations` はこの設定を拡張しません。これは、範囲が限定された Active Memory パス中の
同一エージェントのプライベートトランスクリプトに限定された、
実行時専用の個別の認可を提供します。

以下の例では、これらの設定をトップレベルの `memory.search` 配下に配置しています。1 つの
エージェントのみがセッショントランスクリプトをインデックス化して検索する必要がある場合は、
エージェントごとの `memory.search` オーバーライドで同等の設定を適用することもできます。

同一エージェントの Gateway から DM への想起の場合：

<Tabs>
  <Tab title="組み込みバックエンド">
    ```json5
    {
      memory: {
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
  <Tab title="QMD バックエンド">
    ```json5
    {
      memory: {
        backend: "qmd",
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
        qmd: {
          sessions: { enabled: true },
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
</Tabs>

QMD を使用する場合、`sources: ["sessions"]` だけではトランスクリプトは QMD にエクスポートされません。
`memory.qmd.sessions.enabled: true` も設定してください。より上位の
`rememberAcrossConversations: true` 設定は例外で、そのエージェントに必要な
QMD セッションエクスポートを暗黙的に有効にします。暗黙的なエクスポートは非公開のままです。
常にデフォルトの内部エクスポート先を使用し（設定された
`sessions.exportDir` は明示的なエクスポートにのみ適用されます）、そのエージェントによる会話横断の
想起中にのみ検索され、通常の `memory_get` からは読み取れません。明示的な
`memory.qmd.sessions.enabled: true` は従来の動作を維持し、
エクスポートされたトランスクリプトを通常のメモリコーパスに含めます。

---

## SQLite ベクトル高速化（sqlite-vec）

| キー                          | 型      | デフォルト | 説明                       |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | ベクトルクエリに sqlite-vec を使用 |
| `store.vector.extensionPath` | `string`  | バンドル版 | sqlite-vec のパスをオーバーライド          |

sqlite-vec が利用できない場合、OpenClaw は自動的にプロセス内のコサイン類似度へフォールバックします。

---

## インデックスストレージ

組み込みメモリインデックスは、各エージェントの OpenClaw SQLite データベース内の
`agents/<agentId>/agent/openclaw-agent.sqlite` に保存されます。

| キー                   | 型     | デフォルト     | 説明                               |
| --------------------- | -------- | ----------- | ----------------------------------------- |
| `store.fts.tokenizer` | `string` | `unicode61` | FTS5 トークナイザー（`unicode61` または `trigram`） |

---

## QMD バックエンド設定

有効にするには `memory.backend = "qmd"` を設定します。すべての QMD 設定は `memory.qmd` 配下にあります。

| キー                      | 型      | デフォルト  | 説明                                                                           |
| ------------------------ | --------- | -------- | ------------------------------------------------------------------------------------- |
| `command`                | `string`  | `qmd`    | QMD 実行ファイルのパス。サービスの `PATH` がシェルと異なる場合は絶対パスを設定 |
| `searchMode`             | `string`  | `search` | 検索コマンド：`search`、`vsearch`、`query`                                          |
| `rerank`                 | `boolean` | --       | QMD の再ランキングをスキップするには、`searchMode: "query"` および QMD 2.1+ とともに `false` に設定          |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` を自動インデックス化                                             |
| `paths[]`                | `array`   | --       | 追加パス：`{ name, path, pattern? }`                                               |
| `sessions.enabled`       | `boolean` | `false`  | セッショントランスクリプトを QMD にエクスポート                                                   |
| `sessions.retentionDays` | `number`  | --       | トランスクリプトの保持期間                                                                  |
| `sessions.exportDir`     | `string`  | --       | エクスポートディレクトリ                                                                      |

`searchMode: "search"` は字句/BM25 専用です。OpenClaw はこのモードでは、`memory status --deep` 中を含め、
セマンティックベクトルの準備状態プローブや QMD 埋め込みのメンテナンスを実行しません。
`vsearch` と `query` では、引き続き QMD ベクトルの準備完了と埋め込みが必要です。

`rerank: false` は QMD の `query` モードのみを変更し、QMD 2.1 以降が必要です。直接 CLI モードでは
OpenClaw は `--no-rerank` を渡し、mcporter 経由の MCP モードでは QMD の統合クエリツールに
`rerank: false` を渡します。QMD のデフォルトのクエリ再ランキング動作を使用するには、未設定のままにしてください。

OpenClaw は現在の QMD コレクションおよび MCP クエリ形式を優先しますが、必要に応じて互換性のある
コレクションパターンフラグや旧 MCP ツール名を試すことで、古い QMD リリースも動作するよう維持します。
QMD が複数のコレクションフィルターへの対応を通知する場合、同一ソースのコレクションは 1 つの QMD
プロセスで検索されます。古い QMD ビルドでは、コレクションごとの互換パスが維持されます。同一ソースとは、
永続メモリコレクション（デフォルトのメモリファイルとカスタムパス）をまとめてグループ化することを意味します。
一方、セッショントランスクリプトのコレクションは別のグループのままとなり、ソースの多様化に引き続き両方の入力が含まれます。

<Note>
QMD モデルのオーバーライドは OpenClaw の設定ではなく、QMD 側に保持されます。QMD のモデルをグローバルに
オーバーライドする必要がある場合は、Gateway の実行環境で `QMD_EMBED_MODEL`、`QMD_RERANK_MODEL`、
`QMD_GENERATE_MODEL` などの環境変数を設定してください。
</Note>

<AccordionGroup>
  <Accordion title="制限">
    | キー                       | 型     | デフォルト | 説明                |
    | --------------------------- | -------- | ------- | ------------------------------ |
    | `limits.maxResults`       | `number` | `4`     | 検索結果の最大数         |
    | `limits.maxSnippetChars`  | `number` | `450`   | スニペット長を制限       |
    | `limits.maxInjectedChars` | `number` | `2200`  | 挿入される合計文字数を制限 |
    | `limits.timeoutMs`        | `number` | `4000`  | `memory_search` を含む QMD バックエンド検索中の QMD コマンドタイムアウト。セットアップ、同期、組み込みフォールバック、補足処理にはデフォルトのツール期限が維持される |
  </Accordion>
  <Accordion title="スコープ">
    QMD 検索結果を受け取れるセッションを制御します。[`session.sendPolicy`](/ja-JP/gateway/config-agents#session) と同じスキーマです。

    ```json5
    {
      memory: {
        qmd: {
          scope: {
            default: "deny",
            rules: [{ action: "allow", match: { chatType: "direct" } }],
          },
        },
      },
    }
    ```

    出荷時のデフォルトでは DM/直接チャットのみが許可され、グループおよびその他のチャンネル種別は拒否されます。`match.keyPrefix` は正規化されたセッションキーに一致し、`match.rawKeyPrefix` は `agent:<id>:` を含む未加工のキーに一致します。

  </Accordion>
  <Accordion title="引用">
    `memory.citations` はすべてのバックエンドに適用されます。

    | 値            | 動作                                            |
    | ------------------ | ------------------------------------------------------ |
    | `auto`（デフォルト） | スニペットに `Source: <path#line>` フッターを含める    |
    | `on`             | 常にフッターを含める                               |
    | `off`            | フッターを省略（パスは引き続き内部的にエージェントへ渡される） |

  </Accordion>
</AccordionGroup>

QMD はメモリが初めて使用されたときに遅延初期化され、そのアダプターが更新と埋め込みのスケジュールを管理します。

### QMD の完全な例

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 4, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming

Dreaming は `memory.search` 配下ではなく、`plugins.entries.memory-core.config.dreaming` 配下で設定します。

Dreaming は 1 回のスケジュールされたスイープとして実行され、内部の light/deep/REM フェーズを実装の詳細として使用します。

概念的な動作とスラッシュコマンドについては、[Dreaming](/ja-JP/concepts/dreaming) を参照してください。

### ユーザー設定

| キー                                    | 型      | デフォルト       | 説明                                                                                                                      |
| -------------------------------------- | --------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                              | `boolean` | `false`       | Dreaming 全体を有効または無効にする                                                                                              |
| `frequency`                            | `string`  | `0 3 * * *`   | Dreaming の完全なスイープに使用する任意の Cron 実行間隔                                                                                |
| `model`                                | `string`  | デフォルトモデル | 任意の Dream Diary サブエージェントモデルのオーバーライド                                                                                     |
| `phases.deep.maxPromotedSnippetTokens` | `number`  | `160`         | `MEMORY.md` に昇格される各短期想起スニペットから保持する推定トークンの最大数。出所メタデータは引き続き表示される |

### 例

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        subagent: {
          allowModelOverride: true,
          allowedModels: ["anthropic/claude-sonnet-4-6"],
        },
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
            model: "anthropic/claude-sonnet-4-6",
          },
        },
      },
    },
  },
}
```

<Note>
- Dreaming はマシン状態を `memory/.dreams/` に書き込みます。
- Dreaming は人間が読めるナラティブ出力を `DREAMS.md`（または既存の `dreams.md`）に書き込みます。
- `dreaming.model` は既存の Plugin サブエージェント信頼ゲートを使用します。有効にする前に `plugins.entries.memory-core.subagent.allowModelOverride: true` を設定してください。
- 設定されたモデルが利用できない場合、Dream Diary はセッションのデフォルトモデルで 1 回再試行します。信頼または許可リストの失敗はログに記録され、暗黙には再試行されません。
- light/deep/REM フェーズのポリシーとしきい値は内部動作であり、ユーザー向けの設定ではありません。

</Note>

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference)
- [メモリの概要](/ja-JP/concepts/memory)
- [メモリ検索](/ja-JP/concepts/memory-search)
