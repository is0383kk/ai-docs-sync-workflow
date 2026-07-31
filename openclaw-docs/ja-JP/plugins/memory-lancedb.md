---
read_when:
    - memory-lancedb Pluginを設定しています
    - 自動想起または自動取り込み機能を備えた LanceDB ベースの長期記憶が必要な場合
    - Ollama などのローカルな OpenAI 互換埋め込みを使用しています
sidebarTitle: Memory LanceDB
summary: ローカルの Ollama 互換埋め込みを含む、公式の外部 LanceDB メモリ Plugin を設定する
title: メモリ LanceDB
x-i18n:
    generated_at: "2026-07-26T09:33:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdb7208925ac6c76430ee36dfcd9733041530e0f2ee175950b3cdb8010d67b24
    source_path: plugins/memory-lancedb.md
    workflow: 16
---

`memory-lancedb` は、ベクトル検索を備えた LanceDB に長期記憶を保存する公式の外部 plugin です。モデルのターン前に関連する記憶を自動的に呼び出し、応答後に重要な事実を自動的に取り込むことができます。

ローカルのベクトルデータベース、OpenAI 互換の埋め込みエンドポイント、またはデフォルトの組み込みメモリバックエンド以外のメモリストアとして使用します。

## インストール

```bash
openclaw plugins install @openclaw/memory-lancedb
```

この plugin は npm で公開されており、OpenClaw ランタイムイメージには同梱されていません。インストールすると plugin エントリが書き込まれて有効になり、`plugins.slots.memory` が `memory-lancedb` に切り替わります。現在別の plugin がメモリスロットを所有している場合、その plugin は警告とともに無効になります。

<Note>
`memory-wiki` などのコンパニオン plugin は `memory-lancedb` と併用できますが、アクティブなメモリスロットを同時に所有できる plugin は 1 つだけです。
</Note>

<Note>
LanceDB の `memory_recall` には、`memory.search.rememberAcrossConversations` で使用される、保護された非公開トランスクリプトの認可は付与されません。[高度な Active Memory](/ja-JP/concepts/active-memory#lancedb-memory)を介して、LanceDB の `autoRecall` またはその `memory_recall` ツールを使用してください。現在のメモリプロバイダーで「会話をまたいで記憶」が利用できない場合、`openclaw doctor` が報告します。
</Note>

## クイックスタート

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

plugin 設定を変更した後は Gateway を再起動し、読み込まれたことを確認します。

```bash
openclaw gateway restart
openclaw plugins list
```

## 埋め込み設定

`embedding` は必須で、少なくとも 1 つのフィールドを含める必要があります。`provider` のデフォルトは `openai`、`model` のデフォルトは `text-embedding-3-small` です。

| フィールド                  | 型          | 注記                                                                    |
| ---------------------- | ------------- | ------------------------------------------------------------------------ |
| `embedding.provider`   | 文字列        | アダプター ID（例: `openai`、`github-copilot`、`ollama`）。デフォルトは `openai`。 |
| `embedding.model`      | 文字列        | デフォルトは `text-embedding-3-small`。                                        |
| `embedding.apiKey`     | 文字列        | 省略可能。`${ENV_VAR}` の展開に対応します。                               |
| `embedding.baseUrl`    | 文字列        | 省略可能。`${ENV_VAR}` の展開に対応します。                               |
| `embedding.dimensions` | 整数 (>=1) | 組み込みテーブルにないモデルでは必須です（以下を参照）。               |

リクエストパスは 2 つあります。

- **プロバイダーアダプターパス**（デフォルト）: `embedding.provider` を設定し、`embedding.apiKey`/`embedding.baseUrl` は省略します。plugin は、`memory-core` が使用するものと同じメモリ埋め込みアダプターを介して、プロバイダーに設定された認証プロファイル、環境変数、または `models.providers.<provider>.apiKey` を解決します。これは、`github-copilot`、`ollama`、および埋め込みに対応するその他の同梱プロバイダー向けのパスです。
- **OpenAI 互換クライアントの直接パス**: `embedding.provider` を未設定（または `"openai"`）のままにし、`embedding.apiKey` と `embedding.baseUrl` を設定します。同梱のプロバイダーアダプターがない、生の OpenAI 互換埋め込みエンドポイントに使用します。

OpenAI Codex / ChatGPT OAuth は、OpenAI Platform の埋め込み認証情報ではありません。OpenAI の埋め込みには、OpenAI API キーの認証プロファイル、`OPENAI_API_KEY`、または `models.providers.openai.apiKey` を使用します。OAuth のみを使用するユーザーは、`github-copilot` や `ollama` など、埋め込みに対応する別のプロバイダーを選択してください。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "github-copilot",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

一部の OpenAI 互換埋め込みエンドポイントは `encoding_format` パラメーターを拒否します。ほかのエンドポイントはこれを無視し、常に `number[]` を返します。`memory-lancedb` はリクエストで `encoding_format` を省略し、float 配列または base64 エンコードされた float32 の応答を受け入れるため、どちらの応答形式も設定なしで動作します。

### 次元数

OpenClaw に組み込まれている次元数は、`text-embedding-3-small` (1536) と `text-embedding-3-large` (3072) のみです。それ以外のモデルでは、LanceDB がベクトル列を作成できるように、明示的な `embedding.dimensions` が必要です。たとえば、2048 次元の ZhiPu `embedding-3` は次のように設定します。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            apiKey: "${ZHIPU_API_KEY}",
            baseUrl: "https://open.bigmodel.cn/api/paas/v4",
            model: "embedding-3",
            dimensions: 2048,
          },
        },
      },
    },
  },
}
```

## Ollama の埋め込み

同梱の Ollama プロバイダーアダプターパス（`embedding.provider: "ollama"`）を使用します。これは Ollama ネイティブの `/api/embed` エンドポイントを呼び出し、[Ollama](/ja-JP/providers/ollama) プロバイダーと同じ認証およびベース URL のルールに従います。

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "ollama",
            baseUrl: "http://127.0.0.1:11434",
            model: "mxbai-embed-large",
            dimensions: 1024,
          },
          recallMaxChars: 400,
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

`mxbai-embed-large` は組み込みの次元数テーブルにないため、`dimensions` が必須です。小規模なローカル埋め込みモデルでは、ローカルサーバーがコンテキスト長エラーを返す場合、`recallMaxChars` を小さくしてください。

## 呼び出しと取り込みの制限

| 設定           | デフォルト | 範囲                        | 適用対象                                                 |
| ----------------- | ------- | ---------------------------- | ---------------------------------------------------------- |
| `recallMaxChars`  | `1000`  | 100-10000                    | 呼び出しのために埋め込み API へ送信されるテキスト。                 |
| `captureMaxChars` | `500`   | 100-10000                    | 自動取り込みの対象となり得るメッセージの長さ。                  |
| `customTriggers`  | `[]`    | 0-50 項目、各項目 <=100 文字 | 自動取り込みでメッセージを検討対象にするリテラルフレーズ。 |

`recallMaxChars` は、`before_prompt_build` の自動呼び出しクエリ、`memory_recall` ツール、`memory_forget` クエリパス、および `openclaw ltm
search` の上限を設定します。自動呼び出しでは、ターン内の最新のユーザーメッセージを埋め込み、ユーザーメッセージが存在しない場合にのみプロンプト全体へフォールバックします。これにより、チャンネルメタデータや大きなプロンプトブロックが埋め込みリクエストに含まれないようにします。

`captureMaxChars` は、ターンの `agent_end` イベントからのユーザーメッセージが、自動取り込みの検討対象となるのに十分短いかどうかを制御します。呼び出しクエリには影響しません。

`customTriggers` は、正規表現を使用せずにリテラルの自動取り込みフレーズを追加します。組み込みトリガーは、英語、チェコ語、中国語、日本語、韓国語の一般的な記憶フレーズ（`remember`、`prefer`、`记住`、`覚えて`、`기억해` など）に対応しています。

自動取り込みでは、エンベロープやトランスポートのメタデータ、プロンプトインジェクションのペイロード、またはすでに注入済みの `<relevant-memories>` コンテキストに見えるテキストも拒否し、エージェントのターンごとに取り込む記憶を最大 3 件に制限します。

各記憶は 1 つのエージェントによって所有されます。呼び出し、重複検出、取り込み、一覧表示、生クエリ、削除ではすべて、行を返すか変更する前にその所有者を適用します。`agents.entries.*` エントリに `memory.search.enabled: false` があるエージェント、または無効化されたトップレベル検索を継承するエージェントには、`memory_recall`、`memory_store`、`memory_forget` のいずれのツールも提供されません。また、plugin レベルの `autoRecall`/`autoCapture` フラグがオンでも、自動呼び出しや取り込みには参加しません。

## コマンド

`memory-lancedb` は、インストールされている場合、アクティブなメモリスロットを所有しているときだけでなく、常に `ltm` CLI 名前空間を登録します。

```bash
openclaw ltm list [--agent <id>] [--limit <n>] [--order-by-created-at]
openclaw ltm search <query> [--agent <id>] [--limit <n>]
openclaw ltm stats [--agent <id>]
```

`ltm query` は、LanceDB テーブルに対して非ベクトルクエリを直接実行します。

```bash
openclaw ltm query --agent research --cols id,text,createdAt --limit 20
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc
```

| フラグ                              | デフォルト                                 | 注記                                                                                                                                     |
| --------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--agent <id>`                    | 設定されたデフォルトエージェント                | 非公開のエージェント名前空間を選択します。`list`、`search`、`query`、`stats` で使用できます。                                                 |
| `--cols <columns>`                | `id,text,importance,category,createdAt` | コンマ区切りの列許可リスト。                                                                                                         |
| `--filter <condition>`            | なし                                    | 出力列に対する 1 つの比較（`category = 'preference'` や `importance >= 0.8` など）。文字列値は引用符で囲む必要があります。             |
| `--limit <n>`                     | `10`                                    | 正の整数。                                                                                                                         |
| `--order-by <column>:<asc\|desc>` | なし                                    | フィルター実行後にメモリ内でソートされます。ソート列はプロジェクションに自動追加され、要求されていなかった場合は出力から除去されます。 |

エージェントには、アクティブなメモリ plugin から 3 つのツールが提供されます。

- `memory_recall`: 保存された記憶を対象とするベクトル検索。
- `memory_store`: 事実、設定、決定、またはエンティティを保存します（プロンプトインジェクションのペイロードに見えるテキストは拒否し、ほぼ重複する保存はスキップします）。
- `memory_forget`: `memoryId` または `query` で削除します（スコアが 90% を超える一致が 1 件の場合は自動削除し、それ以外の場合は候補 ID を一覧表示して曖昧さを解消します）。

## ストレージ

LanceDB データのデフォルトは `~/.openclaw/memory/lancedb` です。`dbPath` で上書きします。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "~/.openclaw/memory/lancedb",
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

plugin は 1 つの LanceDB テーブルを保持し、各行に正規化されたエージェント所有者を保存します。これは検索後のフィルターではなく、ストレージ境界です。エージェントの所有権はベクトルランキングの前に適用され、一覧、クエリ、件数取得、削除の述語にも含まれます。`ltm query --filter` は、公開出力列に対する検証済みの比較を 1 つ受け入れます。ストアはその比較を必須の所有者述語とは別に構築するため、フィルターによってクエリ対象を別のエージェントへ広げることはできません。

エージェント単位の所有権が導入される前に作成されたデータベースには、信頼できる行の出自情報がありません。アップグレード時に、`openclaw doctor --fix` はこれらのレガシー行を、設定されたデフォルトエージェントへ一度だけ割り当てます。その移行が完了するまで、ランタイムアクセスはフェイルクローズします。ほかのエージェントが古い共有行を継承することはありません。

`storageOptions` は、LanceDB ストレージバックエンド（例: S3 互換オブジェクトストレージ）用の文字列キー/値ペアを受け入れ、`${ENV_VAR}` の展開をサポートします。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "s3://memory-bucket/openclaw",
          storageOptions: {
            access_key: "${AWS_ACCESS_KEY_ID}",
            secret_key: "${AWS_SECRET_ACCESS_KEY}",
            endpoint: "${AWS_ENDPOINT_URL}",
          },
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

## ランタイム依存関係とプラットフォームサポート

`memory-lancedb` は、Plugin パッケージ（OpenClaw コアの配布物ではありません）が所有するネイティブの `@lancedb/lancedb` パッケージに依存します。Gateway の起動時に Plugin の依存関係は修復されません。ネイティブ依存関係が見つからない場合や読み込みに失敗した場合は、Plugin パッケージを再インストールまたは更新し、Gateway を再起動してください。

`@lancedb/lancedb` は、`darwin-x64`（Intel Mac）向けのネイティブビルドを公開していません。このプラットフォームでは、Plugin の読み込み時に LanceDB が利用できないことがログに記録されます。デフォルトのメモリバックエンドを使用するか、サポートされているプラットフォーム/アーキテクチャで Gateway を実行するか、`memory-lancedb` を無効にしてください。

## トラブルシューティング

### 入力長がコンテキスト長を超える

埋め込みモデルが再呼び出しクエリを拒否しました。

```text
memory-lancedb: 再呼び出しに失敗しました: エラー: 400 入力長がコンテキスト長を超えています
```

`recallMaxChars` を小さくしてから、Gateway を再起動してください。

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        config: {
          recallMaxChars: 400,
        },
      },
    },
  },
}
```

Ollama の場合は、ネイティブの埋め込みエンドポイントを使用して、Gateway ホストから埋め込みサーバーに到達できることも確認してください。

```bash
curl http://127.0.0.1:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"hello"}'
```

### サポートされていない埋め込みモデル

`embedding.dimensions` がない場合、組み込みの OpenAI 埋め込み次元（`text-embedding-3-small`、`text-embedding-3-large`）のみが認識されます。それ以外のモデルでは、`embedding.dimensions` をそのモデルが報告するベクトルサイズに設定してください。

### Plugin は読み込まれるがメモリが表示されない

`plugins.slots.memory` が `memory-lancedb` を指していることを確認してから、次を実行してください。

```bash
openclaw ltm stats
openclaw ltm search "recent preference"
```

`autoCapture` が無効な場合でも、Plugin は既存のメモリを再呼び出しますが、新しいメモリを自動的に保存しません。`memory_store` ツールを使用するか、`autoCapture` を有効にしてください。

## 関連項目

- [メモリの概要](/ja-JP/concepts/memory)
- [Active Memory](/ja-JP/concepts/active-memory)
- [メモリ検索](/ja-JP/concepts/memory-search)
- [メモリ Wiki](/ja-JP/plugins/memory-wiki)
- [Ollama](/ja-JP/providers/ollama)
