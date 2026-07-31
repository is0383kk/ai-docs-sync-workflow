---
read_when:
    - API キーやモデルサーバーを使わずにローカルでテキスト推論を実行したい場合
    - ローカルの GGUF モデルによるメモリ検索用埋め込みを使用する場合
    - memory.search.provider = "local" を設定しています
    - node-llama-cpp ランタイムを所有する OpenClaw Plugin が必要です
sidebarTitle: llama.cpp Provider
summary: llama.cpp を使用して OpenClaw でローカル GGUF テキスト推論とメモリエンベディングを実行する
title: llama.cpp プロバイダー
x-i18n:
    generated_at: "2026-07-26T09:51:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 88e6d66943adcbc602421b8cc00359b3ed87357194c3ffaa845c1db7fbcd9c38
    source_path: plugins/llama-cpp.md
    workflow: 16
---

`llama-cpp` は、プロセス内のローカル GGUF テキスト推論および埋め込み用の公式外部プロバイダー Plugin です。テキストプロバイダー `llama-cpp` と埋め込みプロバイダー `local` を登録し、`node-llama-cpp` ネイティブランタイムを所有します。

ローカル推論またはローカルメモリ埋め込みを使用する前にインストールしてください。

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

メインの `openclaw` npm パッケージには `node-llama-cpp` は含まれていません。ネイティブ依存関係をこの Plugin に保持することで、通常の OpenClaw npm 更新によって、OpenClaw パッケージディレクトリ内に手動でインストールしたランタイムが削除されるのを防ぎます。

## ローカルテキスト推論

対話型オンボーディング中に **ローカルモデル（llama.cpp）** を選択します。OpenClaw はデフォルトモデルをダウンロードする前に確認します。

`hf:bartowski/Qwen_Qwen3-4B-Instruct-2507-GGUF/Qwen_Qwen3-4B-Instruct-2507-Q4_K_M.gguf`

Qwen3 4B Instruct 2507 Q4_K_M ファイルは約 2.5 GB です。モデルの重みに約 3 GB の RAM を確保し、さらにコンテキストと OpenClaw ランタイムのオーバーヘッドを考慮してください。デフォルトのコンテキストは、8 GB のマシンでも実用的に動作するよう、上限 8,192 トークンで自動的にサイズ設定されます。マシンに十分なメモリがある場合にのみ、より大きなコンテキストを設定してください。

オンボーディングの検出チェックは読み取り専用です。デフォルトまたは設定済みの GGUF ファイルがモデルキャッシュにすでに存在する場合にのみ、llama.cpp を自動的に提示します。検出中にダウンロードすることはありません。Ollama と LM Studio は引き続き個別のローカルサービスの選択肢であり、それぞれ独自の検出フローを維持します。llama.cpp を手動で選択すると、デフォルトモデルのダウンロードを確認するフローに進みます。

このプロバイダーは、GGUF モデルに埋め込まれたチャットテンプレートと、node-llama-cpp のネイティブ関数呼び出しを使用します。テキストはトークン単位でストリーミングされます。ツール呼び出しは node-llama-cpp 内で実行されるのではなく、実行のために OpenClaw に返されます。

### 別の GGUF モデルを使用する

`models.providers.llama-cpp` にモデルを追加します。ローカルパスまたは完全な `hf:` ファイル URI を `params.modelPath` に指定します。

```json5
{
  models: {
    mode: "merge",
    providers: {
      "llama-cpp": {
        baseUrl: "local://llama-cpp",
        api: "openai-completions",
        params: {
          modelCacheDir: "~/.node-llama-cpp/models",
        },
        models: [
          {
            id: "my-local-model",
            name: "My local GGUF",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 2048,
            params: {
              modelPath: "~/Models/my-model.Q4_K_M.gguf",
              contextSize: 8192,
            },
            compat: { supportsTools: true },
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "llama-cpp/my-local-model" },
    },
  },
}
```

推論時に、不足しているモデルが暗黙的にダウンロードされることはありません。カスタム `hf:` URI の場合は、最初に GGUF を `modelCacheDir` にダウンロードしてください。検出では、リポジトリ、ブランチ、分割ファイルの命名を含む、node-llama-cpp 独自の読み取り専用キャッシュリゾルバーが使用されます。

## メモリ埋め込みの設定

`memory.search.provider` を `local` に設定します。

```json5
{
  memory: {
    search: {
      provider: "local",
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

`local.modelPath` のデフォルトは、上記の `hf:` URI（`embeddinggemma-300m-qat-Q8_0.gguf`）です。別のモデルを使用するには、異なる `hf:` URI またはローカルの `.gguf` ファイルを指定します。`local.modelCacheDir` は、ダウンロードされたモデルのキャッシュ先（デフォルト: `~/.node-llama-cpp/models`）を上書きし、`local.contextSize` には整数または `"auto"` を指定できます。

`local.contextSize` が数値の場合、プロバイダーはその要件を node-llama-cpp の GPU レイヤー自動配置にも渡します。これにより、node-llama-cpp はメモリ安全性チェックを維持しながら、モデルと埋め込みコンテキストを同時に収容できます。`"auto"` の場合、node-llama-cpp は通常の自動配置を維持します。

## ネイティブランタイム

ネイティブインストールを最も円滑に行うには Node 24 を使用してください。pnpm を使用するソースチェックアウトでは、ネイティブ依存関係の承認と再ビルドが必要になる場合があります。

```bash
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## メモリランタイムの診断

プロバイダーが読み込まれた後に `openclaw memory status --deep` を実行すると、選択されたバックエンドとビルド、デバイス名、GPU にオフロードされたレイヤー、要求されたコンテキストサイズ、および最後に観測された VRAM またはユニファイドメモリのスナップショットを確認できます。パッシブなステータス読み取りではモデルの再読み込みやデバイスのポーリングを行わないため、VRAM の値には観測時刻のタイムスタンプが含まれます。

実行中の Gateway がローカルプロバイダーをすでに使用している場合、同じ最終確認済みの情報が `openclaw doctor` に表示されることがあります。通常のステータスまたは doctor コマンドでは、診断情報を収集するためだけにモデルを読み込むことはありません。

## トラブルシューティング

`node-llama-cpp` が見つからないか読み込みに失敗した場合、OpenClaw は次の内容とともに失敗を報告します。

1. Plugin をインストールします: `openclaw plugins install @openclaw/llama-cpp-provider`。
2. ネイティブのインストールや更新には Node 24 を使用します。
3. pnpm のソースチェックアウトでは、`pnpm approve-builds` を実行してから `pnpm rebuild node-llama-cpp` を実行します。

プロセス内のネイティブ依存関係を使用せずにローカル推論を行うには、代わりに Ollama または LM Studio プロバイダーを使用してください。より手軽にローカル埋め込みを利用するには、代わりに `memory.search.provider` を `lmstudio`、`ollama`、`openai`、`voyage` などのリモート埋め込みプロバイダーに設定します。
