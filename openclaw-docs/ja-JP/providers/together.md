---
read_when:
    - OpenClaw で Together AI を使用する場合
    - API キーの環境変数または CLI 認証の選択が必要です
summary: Together AI のセットアップ（認証 + モデル選択）
title: Together AI
x-i18n:
    generated_at: "2026-07-26T09:59:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9b08cae93c1ea7df46e1d2fbe78692f73bb3e56809122f70a56eec8b3dc5d8a4
    source_path: providers/together.md
    workflow: 16
---

[Together AI](https://together.ai) は、統合 API を通じて Llama、DeepSeek、Kimi などの主要なオープンソースモデルへのアクセスを提供します。
OpenClaw には `together` プロバイダーとして同梱されています。

| プロパティ | 値                         |
| -------- | ----------------------------- |
| プロバイダー | `together`                    |
| 認証     | `TOGETHER_API_KEY`            |
| API      | OpenAI 互換             |
| ベース URL | `https://api.together.xyz/v1` |

## はじめに

<Steps>
  <Step title="API キーを取得">
    [api.together.ai/settings/api-keys](https://api.together.ai/settings/api-keys) で API キーを作成します。
  </Step>
  <Step title="オンボーディングを実行">
    ```bash
    openclaw onboard --auth-choice together-api-key
    ```
  </Step>
  <Step title="デフォルトモデルを設定">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "together/meta-llama/Llama-3.3-70B-Instruct-Turbo",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

### 非対話型の例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

<Note>
オンボーディングでは `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` がデフォルトモデルとして設定されます。
</Note>

## 組み込みカタログ

コストは 100 万トークンあたりの米ドル額です。

| モデル参照                                          | 名前                         | 入力       | コンテキスト | 最大出力 | コスト（入力/出力） | 注記               |
| -------------------------------------------------- | ---------------------------- | ----------- | ------- | ---------- | ------------- | ------------------- |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` | Llama 3.3 70B Instruct Turbo | テキスト        | 131,072 | 8,192      | 0.88 / 0.88   | デフォルトモデル       |
| `together/moonshotai/Kimi-K2.6`                    | Kimi K2.6 FP4                | テキスト、画像 | 262,144 | 32,768     | 1.20 / 4.50   | 推論モデル     |
| `together/deepseek-ai/DeepSeek-V4-Pro`             | DeepSeek V4 Pro              | テキスト        | 512,000 | 8,192      | 2.10 / 4.40   | 推論モデル     |
| `together/Qwen/Qwen2.5-7B-Instruct-Turbo`          | Qwen2.5 7B Instruct Turbo    | テキスト        | 32,768  | 8,192      | 0.30 / 0.30   | 高速、非推論 |
| `together/zai-org/GLM-5.1`                         | GLM 5.1 FP4                  | テキスト        | 202,752 | 8,192      | 1.40 / 4.40   | 推論モデル     |

## 動画生成

同梱の `together` プラグインは、共有の `video_generate` ツールを通じた動画生成も登録します。

| プロパティ             | 値                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------- |
| デフォルトの動画モデル  | `Wan-AI/Wan2.2-T2V-A14B`                                                                  |
| その他のモデル         | `Wan-AI/Wan2.2-I2V-A14B`、`minimax/hailuo-02`、`kwaivgI/kling-2.1-master`                 |
| モード                | テキストから動画、`Wan-AI/Wan2.2-I2V-A14B` のみ画像から動画（参照画像 1 枚） |
| 長さ             | 1～10 秒                                                                              |
| サポートされるパラメーター | `size`（`<width>x<height>` として解析）、`aspectRatio`/`resolution` は読み取られません            |

Together をデフォルトの動画プロバイダーとして使用するには、次のように設定します。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

<Tip>
共有ツールのパラメーター、プロバイダーの選択、フェイルオーバーの動作については、[動画生成](/ja-JP/tools/video-generation)を参照してください。
</Tip>

<AccordionGroup>
  <Accordion title="環境に関する注意">
    Gateway がデーモン（launchd/systemd）として実行される場合は、そのプロセスから
    `TOGETHER_API_KEY` を利用できることを確認してください（たとえば、
    `~/.openclaw/.env` 内または `env.shellEnv` 経由）。

    <Warning>
    対話型シェルでのみ設定されたキーは、デーモン管理の Gateway プロセスから参照できません。
    永続的に利用できるようにするには、`~/.openclaw/.env` または `env.shellEnv` の設定を使用してください。
    </Warning>

  </Accordion>

  <Accordion title="トラブルシューティング">
    - キーが機能することを確認します：`openclaw models list --provider together`
    - モデルが表示されない場合は、Gateway プロセスに対応する正しい環境で API キーが設定されていることを確認してください。
    - モデル参照には `together/<model-id>` の形式を使用します。

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダーのルール、モデル参照、フェイルオーバーの動作。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共有動画生成ツールのパラメーターとプロバイダーの選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    プロバイダー設定を含む完全な設定スキーマ。
  </Card>
  <Card title="Together AI" href="https://together.ai" icon="arrow-up-right-from-square">
    Together AI のダッシュボード、API ドキュメント、料金。
  </Card>
</CardGroup>
