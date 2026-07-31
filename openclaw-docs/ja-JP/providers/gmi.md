---
read_when:
    - GMI Cloud モデルで OpenClaw を実行する場合
    - GMI のプロバイダー ID、キー、またはエンドポイントが必要です
summary: OpenClaw で GMI Cloud の OpenAI 互換 API を使用する
title: GMI Cloud
x-i18n:
    generated_at: "2026-07-26T09:58:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21fd2a997f44e1f78d97a0fba24ca2bbc00dd193323da712d650ed4ba105355
    source_path: providers/gmi.md
    workflow: 16
---

GMI Cloud は、OpenAI 互換 API を介してフロンティアモデルとオープンウェイトモデルを提供するホステッド推論プラットフォームです。OpenClaw では公式の外部プロバイダー Plugin として提供されています。一度インストールし、通常のモデル認証を通じて認証情報を保存すると、`gmi/google/gemini-3.1-flash-lite` のようなモデル参照を使用できます。

Anthropic、DeepSeek、Google、Moonshot、OpenAI、Z.AI など、GMI のカタログで公開されている複数のホステッドモデルファミリーを 1 つの API キーで利用したい場合は、GMI を使用します。モデルのフォールバック用のセカンダリプロバイダー、ベンダー間でホステッドルートを比較するためのプロバイダー、またはプライマリプロバイダーより先に GMI でモデルが利用可能になった場合のプロバイダーとして使用できます。プロバイダー ID、認証プロファイル、エイリアス、モデルカタログのシード、ベース URL は OpenClaw が管理し、実際のモデルの可用性、請求、レート制限、プロバイダー側のルーティングポリシーは GMI が管理します。

| プロパティ      | 値                                    |
| ------------- | ---------------------------------------- |
| プロバイダー ID   | `gmi`（エイリアス: `gmi-cloud`、`gmicloud`） |
| パッケージ       | `@openclaw/gmi-provider`                 |
| 認証環境変数  | `GMI_API_KEY`                            |
| API           | OpenAI 互換（`openai-completions`） |
| ベース URL      | `https://api.gmi-serving.com/v1`         |
| デフォルトモデル | `gmi/google/gemini-3.1-flash-lite`       |

## セットアップ

Plugin をインストールして Gateway を再起動し、GMI Cloud
（`https://www.gmicloud.ai/`）で API キーを作成します。

```bash
openclaw plugins install @openclaw/gmi-provider
openclaw gateway restart
```

次に、以下を実行します。

```bash
openclaw onboard --auth-choice gmi-api-key
```

非対話型セットアップでは `--gmi-api-key <key>` を渡すか、次のように設定できます。

```bash
export GMI_API_KEY="<your-gmi-api-key>" # pragma: allowlist secret
```

## GMI を選択する場合

- ローカルモデルサーバーではなく、ホステッド OpenAI 互換エンドポイントを使用したい場合。
- 1 つのプロバイダーアカウントを通じて、複数の商用モデルファミリーとオープンウェイトモデルファミリーを
  試したい場合。
- DeepInfra、OpenRouter、Together、またはベンダーの直接 API とは異なるアップストリームルーティングを持つ
  フォールバックプロバイダーが必要な場合。
- GMI 固有のモデル ID、料金体系、またはアカウント管理機能が必要な場合。

GMI の OpenAI 互換ルートでは公開されていないベンダーネイティブ機能が必要な場合は、ベンダーの直接プロバイダーを選択してください。ホステッド環境の利便性よりもデータの局所性やローカル GPU の制御が重要な場合は、LM Studio、Ollama、SGLang、vLLM などのローカルプロバイダーを選択してください。

## モデル

Plugin カタログには、一般的に利用可能な GMI Cloud のルート ID がシードされています。

| モデル参照                          | 入力        | コンテキスト   | 最大出力 |
| ---------------------------------- | ------------ | --------- | ---------- |
| `gmi/anthropic/claude-sonnet-4.6`  | テキスト + 画像 | 200,000   | 64,000     |
| `gmi/deepseek-ai/DeepSeek-V3.2`    | テキスト         | 163,840   | 65,536     |
| `gmi/google/gemini-3.1-flash-lite` | テキスト + 画像 | 1,048,576 | 65,536     |
| `gmi/moonshotai/Kimi-K2.5`         | テキスト + 画像 | 262,144   | 65,536     |
| `gmi/openai/gpt-5.4`               | テキスト + 画像 | 400,000   | 128,000    |
| `gmi/zai-org/GLM-5.1-FP8`          | テキスト         | 202,752   | 65,536     |

このカタログはシードであり、すべてのアカウントが常にすべてのモデルを呼び出せることを保証するものではありません。構成済みプロバイダーが環境内で報告するモデルを一覧表示します。

```bash
openclaw models list --provider gmi
```

## トラブルシューティング

- `401` または `403`: OpenClaw を実行しているプロセスに `GMI_API_KEY` が設定されていることを確認するか、オンボーディングを再実行してプロバイダー認証プロファイルにキーを保存してください。
- 不明なモデルのエラー: GMI アカウントにそのモデルが存在することを確認し、`openclaw models list --provider gmi` に表示される完全な `gmi/<route-id>` 参照を使用してください。
- 断続的なプロバイダーエラー: 別の GMI ルートを試すか、GMI を唯一のプライマリモデルプロバイダーではなくフォールバックとして構成してください。

## 関連項目

- [モデルプロバイダー](/ja-JP/concepts/model-providers)
- [すべてのプロバイダー](/ja-JP/providers/index)
