---
read_when:
    - OpenClaw で Hugging Face Inference を使用する場合
    - HF トークンの環境変数または CLI 認証の選択が必要です
summary: Hugging Face Inference のセットアップ（認証 + モデル選択）
title: Hugging Face（推論）
x-i18n:
    generated_at: "2026-07-26T09:15:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) は、多数のホスト型モデル（DeepSeek、Llama など）の前段に、1 つのトークンで利用できる OpenAI 互換のチャット補完ルーターを提供します。OpenClaw が通信するのは**チャット補完エンドポイントのみ**です。テキストから画像への変換、埋め込み、音声には、[HF inference clients](https://huggingface.co/docs/api-inference/quicktour) を直接使用してください。

| プロパティ     | 値                                                                                                                       |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| プロバイダー ID  | `huggingface`                                                                                                               |
| Plugin       | バンドル済み（デフォルトで有効、インストール手順は不要）                                                                               |
| 認証環境変数 | `HUGGINGFACE_HUB_TOKEN` または `HF_TOKEN`（きめ細かなトークン）                                                                  |
| API          | OpenAI 互換（`https://router.huggingface.co/v1`）                                                                      |
| 課金      | 単一の HF トークン。[料金](https://huggingface.co/docs/inference-providers/pricing)は無料枠付きでプロバイダーの料金体系に従います |

## はじめに

<Steps>
  <Step title="きめ細かなトークンを作成">
    [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) に移動し、新しいきめ細かなトークンを作成します。

    <Warning>
    トークンでは **Make calls to Inference Providers** 権限を有効にする必要があります。有効でない場合、API リクエストは拒否されます。
    </Warning>

  </Step>
  <Step title="オンボーディングを実行">
    プロバイダーのドロップダウンで **Hugging Face** を選択し、プロンプトが表示されたら API キーを入力します。

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="デフォルトモデルを選択">
    **Default Hugging Face model** ドロップダウンでモデルを選択します。トークンが有効な場合、リストは Inference API から読み込まれます。それ以外の場合、OpenClaw は以下の組み込みカタログを表示します。選択内容は `agents.defaults.model.primary` として保存されます。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="モデルが利用可能か確認">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### 非対話型セットアップ

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

`huggingface/deepseek-ai/DeepSeek-R1` をデフォルトモデルとして設定します。

## モデル ID

モデル参照は `huggingface/<org>/<model>` の形式（Hub 形式の ID）を使用します。OpenClaw の組み込みカタログは次のとおりです。

| モデル         | 参照（先頭に `huggingface/` を付加） |
| ------------- | -------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`        |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`      |
| GPT-OSS 120B  | `openai/gpt-oss-120b`            |

<Tip>
トークンが有効な場合、OpenClaw はオンボーディング時と Gateway 起動時に **GET** `https://router.huggingface.co/v1/models` からその他のモデルも検出するため、カタログには上記 3 つよりはるかに多くのモデルを含められます。任意のモデル ID に `:fastest` または `:cheapest` を追加できます。HF のルーターが一致する推論プロバイダーへルーティングします。[Inference Provider settings](https://hf.co/settings/inference-providers) でデフォルトのプロバイダー順序を設定してください。
</Tip>

## 高度な設定

<AccordionGroup>
  <Accordion title="モデル検出とオンボーディングのドロップダウン">
    OpenClaw は次のリクエストでモデルを検出します。

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # または $HF_TOKEN
    ```

    レスポンスは OpenAI 形式の `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }` です。

    キーが設定されている場合（オンボーディング、`HUGGINGFACE_HUB_TOKEN`、または `HF_TOKEN`）、対話型セットアップ中の **Default Hugging Face model** ドロップダウンには、このエンドポイントから取得した内容が入力されます。Gateway の起動時にも同じ呼び出しを繰り返してカタログを更新します。検出されたモデルは上記の組み込みカタログとマージされます（ID が一致する場合、コンテキストウィンドウやコストなどのメタデータに使用されます）。リクエストが失敗する、データが返されない、またはキーが設定されていない場合、OpenClaw は組み込みカタログのみにフォールバックします。

    プロバイダーを削除せずに検出を無効化するには、次を実行します。

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="モデル名、エイリアス、ポリシーサフィックス">
    - **API からの名前:** 検出されたモデルでは、API の `name`、`title`、または `display_name` が存在する場合はそれを使用します。それ以外の場合、OpenClaw はモデル ID から名前を生成します（例: `deepseek-ai/DeepSeek-R1` は「DeepSeek R1」になります）。
    - **表示名の上書き:** 設定でモデルごとにカスタムラベルを設定します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **ポリシーサフィックス:** `:fastest` と `:cheapest` は HF ルーターの規則であり、OpenClaw が書き換えるものではありません。サフィックスはモデル ID の一部としてそのまま送信され、HF のルーターが一致する推論プロバイダーを選択します。サフィックスごとに異なるエイリアスを使用する場合は、各バリアントを `models.providers.huggingface.models`（または `model.primary`）の下に個別のエントリとして追加してください。
    - **設定のマージ:** `models.providers.huggingface.models` 内の既存エントリ（例: `models.json` 内）は設定のマージ時にも保持されるため、そこで設定したカスタムの `name`、`alias`、またはモデルオプションは再起動後も維持されます。

  </Accordion>

  <Accordion title="環境とデーモンのセットアップ">
    Gateway がデーモン（launchd/systemd）として実行される場合、そのプロセスで `HUGGINGFACE_HUB_TOKEN` または `HF_TOKEN` を利用できることを確認してください（たとえば、`~/.openclaw/.env` 内または `env.shellEnv` 経由）。

    <Note>
    OpenClaw は `HUGGINGFACE_HUB_TOKEN` と `HF_TOKEN` の両方を受け付けます。両方が設定されている場合、`HUGGINGFACE_HUB_TOKEN` が優先されます。
    </Note>

  </Accordion>

  <Accordion title="設定: フォールバック付き DeepSeek R1">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="設定: 最安および最速バリアントを使用する DeepSeek">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="設定: エイリアス付き DeepSeek + GPT-OSS">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    すべてのプロバイダー、モデル参照、フェイルオーバー動作の概要。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/models" icon="brain">
    モデルの選択および設定方法。
  </Card>
  <Card title="Inference Providers のドキュメント" href="https://huggingface.co/docs/inference-providers" icon="book">
    Hugging Face Inference Providers の公式ドキュメント。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    完全な設定リファレンス。
  </Card>
</CardGroup>
