---
read_when:
    - 主要なオープンソース LLM 向けに単一の API キーを使用したい場合
    - OpenClaw で DeepInfra の API 経由でモデルを実行する場合
summary: DeepInfra の統合 API を使用して、OpenClaw から最も人気のあるオープンソースモデルとフロンティアモデルにアクセスする
title: DeepInfra
x-i18n:
    generated_at: "2026-07-26T09:47:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra は、一般的なオープンソースモデルとフロンティアモデルへのリクエストを、
単一の OpenAI 互換エンドポイントと API キーを介してルーティングします。ほとんどの OpenAI SDK は、
ベース URL を切り替えるだけで利用できます。

## Plugin をインストール

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## API キーを取得

1. [deepinfra.com](https://deepinfra.com/) にサインインします
2. Dashboard / Keys に移動してキーを生成するか、自動作成されたキーを使用します

## CLI のセットアップ

```bash
openclaw onboard --deepinfra-api-key <key>
```

または、環境変数を設定します。

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## 設定スニペット

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## 対応サーフェス

チャット、画像生成、動画生成では、`DEEPINFRA_API_KEY` が設定されると、
`https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta` からモデルカタログが
リアルタイムで更新されます。ライブ検出により選択可能なモデルの一覧は増えますが、
各サーフェスのデフォルトモデルは、以下の静的な値のままです。
その他のサーフェスは、同じライブカタログへ移行するまで静的カタログを使用します。

| サーフェス               | デフォルトモデル                                                               | OpenClaw の設定/ツール                                |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| チャット / モデルプロバイダー | `deepseek-ai/DeepSeek-V4-Flash`（ライブカタログにより、さらに多くのチャットモデルが追加されます） | `agents.defaults.model`                               |
| 画像生成/編集            | `black-forest-labs/FLUX-1-schnell`（ライブカタログにより、さらに多くの `image-gen` モデルが追加されます） | `image_generate`, `agents.defaults.mediaModels.image` |
| メディア理解             | 画像には `moonshotai/Kimi-K2.5`                                              | 受信画像の理解                                        |
| 音声テキスト変換         | `openai/whisper-large-v3-turbo`                                                | 受信音声の文字起こし                                  |
| テキスト音声変換         | `hexgrad/Kokoro-82M`                                                           | `tts.provider: "deepinfra"`                           |
| 動画生成                 | `Pixverse/Pixverse-T2V`（ライブカタログにより、さらに多くの `video-gen` モデルが追加されます） | `video_generate`, `agents.defaults.mediaModels.video` |
| メモリ埋め込み           | `BAAI/bge-m3`                                                                  | `memory.search.provider: "deepinfra"`                 |

DeepInfra は、再ランキング、分類、物体検出、その他の
ネイティブモデルタイプも公開しています。OpenClaw には、これらのカテゴリに対応する
プロバイダー契約がまだないため、この Plugin では登録されません。

## 利用可能なモデル

キーが設定されると、OpenClaw は DeepInfra のモデルを動的に検出します。
現在の一覧を確認するには、`/models deepinfra` または `openclaw models list --provider deepinfra` を使用します。

[deepinfra.com](https://deepinfra.com/) にあるすべてのモデルは、
`deepinfra/` プレフィックスを付けて使用できます。

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
...ほか多数
```

## 注意事項

- モデル参照は `deepinfra/<provider>/<model>` です（例: `deepinfra/Qwen/Qwen3-Max`）。
- デフォルトのチャットモデル: `deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- ベース URL: `https://api.deepinfra.com/v1/openai`
- 動画生成では、OpenAI 互換の非同期エンドポイント `https://api.deepinfra.com/v1/openai/videos` を使用します（送信後にポーリング）。設定済みの `baseUrl` が適用されます。`openclaw doctor --fix` は、`api.deepinfra.com` にある従来の `nativeBaseUrl` または `/v1/inference` の値を `baseUrl` へ自動的に移行します。カスタムのネイティブエンドポイントは doctor の通知とともに廃止され、OpenAI 互換の `baseUrl` を手動で設定する必要があります。`baseUrl` が廃止済みの `/v1/inference` サーフェスを参照している間は、動画生成はリクエストを送信する前に、対処方法を示すエラーで失敗します。

## 関連項目

- [モデルプロバイダー](/ja-JP/concepts/model-providers)
- [すべてのプロバイダー](/ja-JP/providers/index)
