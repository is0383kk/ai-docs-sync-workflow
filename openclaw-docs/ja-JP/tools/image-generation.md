---
read_when:
    - エージェントによる画像の生成または編集
    - 画像生成プロバイダーとモデルの設定
    - image_generate ツールのパラメーターを理解する
sidebarTitle: Image generation
summary: OpenAI、Google、fal、Microsoft Foundry、MiniMax、ComfyUI、DeepInfra、OpenRouter、LiteLLM、xAI、Vydra の各サービスで image_generate を使用して画像を生成・編集する
title: 画像生成
x-i18n:
    generated_at: "2026-07-26T09:21:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9688b1bc649713d8ed345a69a28d20b36ecd768b6a6d28a2d6c022d65b081862
    source_path: tools/image-generation.md
    workflow: 16
---

`image_generate` ツールは、設定済みのプロバイダーを通じて画像を生成・編集します。チャットセッションでは非同期で実行されます。OpenClaw はバックグラウンドタスクを記録し、タスク ID を直ちに返し、プロバイダーの処理が完了するとエージェントを起動します。完了処理を行うエージェントは、セッションの通常の表示応答モードに従います。設定されている場合は最終応答を自動的に配信し、セッションでメッセージツールが必要な場合は `message(action="send")` を使用します。リクエスト元のセッションが非アクティブであるか、アクティブな起動に失敗した場合、結果が失われないよう、OpenClaw は生成された画像を含む冪等な直接フォールバックを送信します。

<Note>
このツールは、画像生成プロバイダーが少なくとも 1 つ利用可能な場合にのみ表示されます。エージェントのツールに `image_generate` が表示されない場合は、`agents.defaults.mediaModels.image` を設定し、プロバイダーの API キーをセットアップするか、OpenAI ChatGPT/Codex OAuth でサインインしてください。
</Note>

## クイックスタート

<Steps>
  <Step title="認証を設定">
    少なくとも 1 つのプロバイダーの API キー（例: `OPENAI_API_KEY`、`GEMINI_API_KEY`、`OPENROUTER_API_KEY`）を設定するか、OpenAI Codex OAuth でサインインします。
  </Step>
  <Step title="デフォルトモデルを選択（任意）">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openai/gpt-image-2",
            timeoutMs: 180_000,
          },
        },
      },
    }
    ```

    ChatGPT/Codex OAuth は同じ `openai/gpt-image-2` モデル参照を使用します。`openai` OAuth プロファイルが設定されている場合、OpenClaw は最初に `OPENAI_API_KEY` を試すのではなく、その OAuth プロファイルを通じて画像リクエストをルーティングします。`models.providers.openai` の明示的な設定（API キー、カスタム/Azure ベース URL）を行うと、OpenAI Images API の直接ルートに戻ります。

  </Step>
  <Step title="エージェントに依頼">
    _「親しみやすいロボットのマスコット画像を生成してください。」_

    エージェントは `image_generate` を自動的に呼び出します。ツールの許可リストへの追加は不要です。プロバイダーが利用可能な場合、デフォルトで有効になります。ツールはバックグラウンドタスク ID を返し、準備が完了すると、完了処理を行うエージェントが `message` ツールを通じて生成された添付ファイルを送信します。

  </Step>
</Steps>

<Warning>
LocalAI などの OpenAI 互換 LAN エンドポイントでは、カスタム `models.providers.openai.baseUrl` を維持し、`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` で明示的にオプトインしてください。プライベートおよび内部の画像エンドポイントは、デフォルトでは引き続きブロックされます。
</Warning>

## 一般的なルート

| 目的                                                 | モデル参照                                         | 認証                                   |
| ---------------------------------------------------- | -------------------------------------------------- | -------------------------------------- |
| API 課金による OpenAI 画像生成                       | `openai/gpt-image-2`                               | `OPENAI_API_KEY`                       |
| Codex サブスクリプション認証による OpenAI 画像生成   | `openai/gpt-image-2`                               | OpenAI ChatGPT/Codex OAuth             |
| OpenAI の背景透過 PNG/WebP                           | `openai/gpt-image-1.5`                             | `OPENAI_API_KEY` または OpenAI Codex OAuth |
| DeepInfra 画像生成                                   | `deepinfra/black-forest-labs/FLUX-1-schnell`       | `DEEPINFRA_API_KEY`                    |
| fal Krea 2 の表現豊かでスタイル指定可能な生成         | `fal/krea/v2/medium/text-to-image`                 | `FAL_KEY`                              |
| OpenRouter 画像生成                                  | `openrouter/google/gemini-3.1-flash-image-preview` | `OPENROUTER_API_KEY`                   |
| LiteLLM 画像生成                                     | `litellm/gpt-image-2`                              | `LITELLM_API_KEY`                      |
| Microsoft Foundry MAI 画像生成                       | `microsoft-foundry/<deployment-name>`              | `AZURE_OPENAI_API_KEY` または Entra ID     |
| Google Gemini 画像生成                               | `google/gemini-3.1-flash-image`                    | `GEMINI_API_KEY` または `GOOGLE_API_KEY`   |

同じツールで、テキストからの画像生成と参照画像の編集を処理します。参照画像が 1 つの場合は `image`、複数の場合は `images` を使用します。fal の Krea 2 モデルでは、これらの参照画像は編集入力ではなくスタイル参照として送信されます。`quality`、`outputFormat`、`background` など、プロバイダーがサポートする出力ヒントは、利用可能な場合に転送されます。プロバイダーがサポートを宣言していない場合は、無視されたものとして報告されます。組み込みの背景透過サポートは OpenAI 固有です。他のプロバイダーでも、バックエンドが PNG アルファを出力する場合は維持されることがあります。

## サポート対象プロバイダー

| プロバイダー      | デフォルトモデル                        | 編集サポート                         | 認証                                                  |
| ----------------- | --------------------------------------- | ------------------------------------ | ----------------------------------------------------- |
| ComfyUI           | `workflow`                              | あり（1 画像、ワークフローで設定）   | `COMFY_API_KEY` またはクラウド用の `COMFY_CLOUD_API_KEY` |
| DeepInfra         | `black-forest-labs/FLUX-1-schnell`      | あり（1 画像）                       | `DEEPINFRA_API_KEY`                                   |
| fal               | `fal-ai/flux/dev`                       | あり（モデル固有の制限）             | `FAL_KEY`                                             |
| Google            | `gemini-3.1-flash-image`                | あり（最大 5 画像）                  | `GEMINI_API_KEY` または `GOOGLE_API_KEY`                  |
| LiteLLM           | `gpt-image-2`                           | あり（最大 5 入力画像）              | `LITELLM_API_KEY`                                     |
| Microsoft Foundry | `<deployment-name>`                     | あり（MAI-Image-2.5 モデルのみ）      | `AZURE_OPENAI_API_KEY` または Entra ID（`az login`）       |
| MiniMax           | `image-01`                              | あり（被写体参照）                   | `MINIMAX_API_KEY` または MiniMax OAuth（`minimax-portal`） |
| OpenAI            | `gpt-image-2`                           | あり（最大 5 画像）                  | `OPENAI_API_KEY` または OpenAI ChatGPT/Codex OAuth        |
| OpenRouter        | `google/gemini-3.1-flash-image-preview` | あり（最大 5 入力画像）              | `OPENROUTER_API_KEY`                                  |
| Vydra             | `grok-imagine`                          | なし                                 | `VYDRA_API_KEY`                                       |
| xAI               | `grok-imagine-image`                    | あり（最大 3 画像）                  | `XAI_API_KEY`                                         |

実行時に利用可能なプロバイダーとモデルを確認するには、`action: "list"` を使用します。

```text
/tool image_generate action=list
```

現在のセッションでアクティブな画像生成タスクを確認するには、`action: "status"` を使用します。

```text
/tool image_generate action=status
```

## プロバイダーの機能

| 機能                  | ComfyUI                    | DeepInfra | fal                                                   | Google         | Microsoft Foundry | MiniMax                   | OpenAI         | Vydra | xAI            |
| --------------------- | -------------------------- | --------- | ----------------------------------------------------- | -------------- | ----------------- | ------------------------- | -------------- | ----- | -------------- |
| 生成（最大数）        | 1                          | 4         | 4                                                     | 4              | 1                 | 9                         | 4              | 1     | 4              |
| 編集 / 参照           | 1 画像（ワークフロー）     | 1 画像    | Flux: 1; GPT: 10; Krea スタイル参照: 10; NB2: 14     | 最大 5 画像    | 1 画像            | 1 画像（被写体参照）      | 最大 5 画像    | -     | 最大 3 画像    |
| サイズ制御            | -                          | ✓         | ✓                                                     | ✓              | ✓                 | -                         | 最大 4K        | -     | -              |
| アスペクト比          | -                          | -         | ✓                                                     | ✓              | -                 | ✓                         | -              | -     | ✓              |
| 解像度（1K/2K/4K）    | -                          | -         | ✓                                                     | ✓              | -                 | -                         | -              | -     | 1K, 2K         |

## ツールパラメーター

<ParamField path="prompt" type="string" required>
  画像生成プロンプト。`action: "generate"` では必須です。
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  アクティブなセッションタスクを確認するには `"status"`、実行時に利用可能なプロバイダーとモデルを確認するには `"list"` を使用します。
</ParamField>
<ParamField path="model" type="string">
  プロバイダー/モデルの上書き（例: `openai/gpt-image-2`）。OpenAI の背景を透過するには `openai/gpt-image-1.5` を使用します。
</ParamField>
<ParamField path="image" type="string">
  編集モード用の単一の参照画像パスまたは URL。
</ParamField>
<ParamField path="images" type="string[]">
  編集モードまたはスタイル参照モデル用の複数の参照画像（共有ツールでは最大 14、プロバイダー固有の制限も引き続き適用されます）。
</ParamField>
<ParamField path="size" type="string">
  サイズヒント: `1024x1024`、`1536x1024`、`1024x1536`、`2048x2048`、`3840x2160`。
</ParamField>
<ParamField path="aspectRatio" type="string">
  アスペクト比: `1:1`、`2:1`、`20:9`、`19.5:9`、`2:3`、`3:2`、`2.35:1`、`3:4`、
  `4:3`、`4:5`、`5:4`、`9:16`、`9:19.5`、`9:20`、`16:9`、`21:9`、`1:2`、`4:1`、
  `1:4`、`8:1`、`1:8`。プロバイダーは、各モデル固有のサブセットを検証します。
</ParamField>
<ParamField path="resolution" type='"1K" | "2K" | "4K"'>解像度ヒント。</ParamField>
<ParamField path="quality" type='"low" | "medium" | "high" | "auto"'>
  プロバイダーがサポートしている場合の品質ヒント。
</ParamField>
<ParamField path="outputFormat" type='"png" | "jpeg" | "webp"'>
  プロバイダーがサポートしている場合の出力形式ヒント。
</ParamField>
<ParamField path="background" type='"transparent" | "opaque" | "auto"'>
  プロバイダーがサポートしている場合の背景ヒント。透過対応プロバイダーでは、`transparent` を `outputFormat: "png"` または `"webp"` と組み合わせて使用します。
</ParamField>
<ParamField path="count" type="number">生成する画像数（1-4）。</ParamField>
<ParamField path="timeoutMs" type="number">
  プロバイダーへのリクエストのタイムアウト（ミリ秒、省略可能）。Codex が動的ツールを通じて `image_generate` を呼び出す場合も、この呼び出しごとの値が設定済みのデフォルトを上書きし、上限は 600000 ms です。
</ParamField>
<ParamField path="filename" type="string">出力ファイル名のヒント。</ParamField>
<ParamField path="openai" type="object">
  OpenAI 専用のヒント: `background`、`moderation`、`outputCompression`、`user`。
</ParamField>
<ParamField path="fal.creativity" type='"raw" | "low" | "medium" | "high"'>
  fal Krea 2 の創造性制御。デフォルトは `medium` です。
</ParamField>

<Note>
すべてのプロバイダーがすべてのパラメーターをサポートしているわけではありません。フォールバックプロバイダーが、要求されたものと完全に一致するオプションではなく近いジオメトリオプションをサポートしている場合、OpenClaw は送信前に、最も近いサポート対象のサイズ、アスペクト比、または解像度に再マッピングします。サポートを宣言していないプロバイダーでは、サポートされていない出力ヒントは削除され、ツール結果で報告されます。ツール結果には適用された設定が表示され、`details.normalization` には要求値から適用値への変換が記録されます。
</Note>

## 設定

### モデルの選択

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-2",
        timeoutMs: 180_000,
        fallbacks: [
          "openrouter/google/gemini-3.1-flash-image-preview",
          "google/gemini-3.1-flash-image",
          "fal/fal-ai/flux/dev",
        ],
      },
    },
  },
}
```

### プロバイダーの選択順序

OpenClaw は次の順序でプロバイダーを試行します。

1. ツール呼び出しの **`model` パラメーター**（エージェントが指定した場合）。
2. 設定の **`imageGenerationModel.primary`**。
3. 順番に **`imageGenerationModel.fallbacks`**。
4. **自動検出** - 認証情報のあるプロバイダーのデフォルトのみ：
   - 現在のデフォルトプロバイダーを最初に試行；
   - 残りの登録済み画像生成プロバイダーをプロバイダー ID 順に試行。

プロバイダーが失敗した場合（認証エラー、レート制限など）、次に設定された
候補が自動的に試行されます。すべて失敗した場合、エラーには各試行の
詳細が含まれます。

<AccordionGroup>
  <Accordion title="呼び出しごとのモデルオーバーライドは厳密に適用されます">
    呼び出しごとの `model` オーバーライドでは、そのプロバイダー／モデルのみが試行され、
    設定されたプライマリ／フォールバックや自動検出されたプロバイダーには進みません。
  </Accordion>
  <Accordion title="自動検出では認証状態が考慮されます">
    OpenClaw が実際にプロバイダーを認証できる場合にのみ、そのプロバイダーのデフォルトが
    候補リストに追加されます。認証済みプロバイダー間の自動フォールバックは
    常に有効です。呼び出しごとの `model` が最優先されます。
  </Accordion>
  <Accordion title="タイムアウト">
    低速な画像バックエンドには `agents.defaults.mediaModels.image.timeoutMs` を設定します。
    呼び出しごとのツールパラメーター `timeoutMs` は設定済みのデフォルトを上書きし、
    設定済みのデフォルトは Plugin が定義したプロバイダーのデフォルトを
    上書きします。Google および OpenRouter がホストする画像プロバイダーのデフォルトは 180 秒です。
    Microsoft Foundry MAI、xAI、Azure OpenAI の画像生成では
    600 秒です。Codex の動的ツール呼び出しでは、ブリッジのデフォルト
    `image_generate` として 120 秒を使用し、設定されている場合は同じタイムアウト予算を尊重しますが、
    OpenClaw の動的ツールブリッジの上限である 600000 ms に制限されます。
  </Accordion>
  <Accordion title="実行時に確認">
    現在登録されているプロバイダー、そのデフォルトモデル、
    および認証用環境変数のヒントを確認するには `action: "list"` を使用します。
  </Accordion>
</AccordionGroup>

### 画像編集

OpenAI、OpenRouter、Google、DeepInfra、fal、Microsoft Foundry、MiniMax、
ComfyUI、xAI は、参照画像の編集をサポートしています。fal の Krea 2 モデルでは、
編集入力ではなくスタイル参照として、同じ `image` / `images` フィールドを
使用します。参照画像のパスまたは URL を渡します。

```text
「この写真の水彩画バージョンを生成」+ image: "/path/to/photo.jpg"
```

OpenAI、OpenRouter、Google は `images` パラメーターを介して最大 5 枚の参照画像を
サポートし、xAI は最大 3 枚をサポートします。fal は Flux の image-to-image では参照画像 1 枚、
GPT Image 2 の編集では最大 10 枚、Krea 2 のスタイル参照では最大 10 枚、
Nano Banana 2 の編集では最大 14 枚をサポートします。Microsoft Foundry、MiniMax、
ComfyUI は 1 枚をサポートします。

## プロバイダーの詳細

<AccordionGroup>
  <Accordion title="OpenAI gpt-image-2（および gpt-image-1.5）">
    OpenAI の画像生成では、デフォルトで `openai/gpt-image-2` を使用します。
    `openai` OAuth プロファイルが設定されている場合、OpenClaw は
    Codex サブスクリプションのチャットモデルで使用されるものと同じ OAuth プロファイルを再利用し、
    Codex Responses バックエンドを介して画像リクエストを送信します。
    `https://chatgpt.com/backend-api` などの従来の Codex ベース URL は、
    画像リクエスト用に `https://chatgpt.com/backend-api/codex` へ正規化されます。OpenClaw は、
    そのリクエストを暗黙に `OPENAI_API_KEY` へフォールバック**しません**。
    OpenAI Images API へ直接ルーティングするには、API キー、カスタムベース URL、
    または Azure エンドポイントを指定して `models.providers.openai` を明示的に設定してください。

    `openai/gpt-image-1.5`、`openai/gpt-image-1`、
    `openai/gpt-image-1-mini` の各モデルも、引き続き明示的に選択できます。
    背景が透明な PNG/WebP 出力には `gpt-image-1.5` を使用します。現在の
    `gpt-image-2` API は `background: "transparent"` を拒否します。

    `gpt-image-2` は、同じ `image_generate` ツールを介して、
    テキストからの画像生成と参照画像の編集の両方をサポートします。
    OpenClaw は、`prompt`、`count`、`size`、`quality`、`outputFormat`、
    および参照画像を OpenAI に転送します。OpenAI が
    `aspectRatio` または `resolution` を直接受け取ることは**ありません**。
    可能な場合、OpenClaw はそれらをサポートされている `size` にマッピングし、
    それ以外の場合、ツールは無視されたオーバーライドとして報告します。

    OpenAI 固有のオプションは `openai` オブジェクト内にあります。

    ```json
    {
      "quality": "low",
      "outputFormat": "jpeg",
      "openai": {
        "background": "opaque",
        "moderation": "low",
        "outputCompression": 60,
        "user": "end-user-42"
      }
    }
    ```

    `openai.background` は `transparent`、`opaque`、`auto` のいずれかを受け付けます。
    透明な出力には、`outputFormat` `png` または `webp` と、
    透過に対応する OpenAI 画像モデルが必要です。OpenClaw は、デフォルトの
    `gpt-image-2` 透過背景リクエストを `gpt-image-1.5` にルーティングします。
    `openai.outputCompression` は JPEG/WebP 出力に適用され、PNG 出力では無視されます。

    トップレベルの `background` ヒントはプロバイダーに依存せず、現在は
    OpenAI プロバイダーが選択された場合に、同じ OpenAI の `background`
    リクエストフィールドへマッピングされます。背景サポートを宣言していないプロバイダーには、
    サポートされていないパラメーターを渡す代わりに、`ignoredOverrides` 内で返します。

    OpenAI の画像生成を `api.openai.com` ではなく Azure OpenAI デプロイメント経由で
    ルーティングするには、[Azure OpenAI エンドポイント](/ja-JP/providers/openai#azure-openai-endpoints)を
    参照してください。

  </Accordion>
  <Accordion title="Microsoft Foundry MAI 画像モデル">
    Microsoft Foundry の画像生成では、`microsoft-foundry/` プロバイダープレフィックスの下で、
    デプロイ済みの MAI 画像デプロイメント名を使用します。MAI API は
    `model` フィールドにデプロイメント名が指定されることを想定するため、
    プロバイダーレベルのデフォルトモデルはありません。

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "microsoft-foundry/<deployment-name>",
            timeoutMs: 600_000,
          },
        },
      },
    }
    ```

    このプロバイダーは OpenAI Images API ではなく、Microsoft Foundry の MAI API を使用します。

    - 生成エンドポイント：`/mai/v1/images/generations`
    - 編集エンドポイント：`/mai/v1/images/edits`
    - 認証：`AZURE_OPENAI_API_KEY` / プロバイダー API キー、または `az login` を介した Entra ID
    - 出力：PNG 画像 1 枚
    - サイズ：デフォルトは `1024x1024`。幅と高さはそれぞれ 768 px 以上、
      総ピクセル数は 1,048,576 以下である必要があります
    - 編集：PNG または JPEG の参照画像 1 枚。`MAI-Image-2.5-Flash` および
      `MAI-Image-2.5` デプロイメントのみがサポートします

    プロンプトのみの生成では、Foundry エンドポイントだけを設定して
    カスタムデプロイメント名を使用できます。カスタムデプロイメント名による編集には、
    そのデプロイメントが `MAI-Image-2.5-Flash` または `MAI-Image-2.5` を基盤としていることを
    OpenClaw が検証できるよう、オンボーディング／モデルのメタデータが必要です。

    現在の MAI 画像モデルは `MAI-Image-2.5-Flash`、`MAI-Image-2.5`、
    `MAI-Image-2e`、`MAI-Image-2` です。セットアップと
    チャットモデルの動作については、[Microsoft Foundry Plugin](/ja-JP/plugins/reference/microsoft-foundry)を
    参照してください。

  </Accordion>
  <Accordion title="OpenRouter 画像モデル">
    OpenRouter の画像生成では同じ `OPENROUTER_API_KEY` を使用し、
    OpenRouter の Chat Completions 画像 API を介してルーティングします。
    `openrouter/` プレフィックスで OpenRouter 画像モデルを選択します。

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openrouter/google/gemini-3.1-flash-image-preview",
          },
        },
      },
    }
    ```

    OpenClaw は、`prompt`、`count`、参照画像、および
    Gemini 互換の `aspectRatio` / `resolution` ヒントを OpenRouter に転送します。
    現在組み込まれている OpenRouter 画像モデルのショートカットには、
    `google/gemini-3.1-flash-image`、
    `google/gemini-3-pro-image`、`openai/gpt-5.4-image-2` があります。設定済みの Plugin が公開する内容を
    確認するには `action: "list"` を使用します。

  </Accordion>
  <Accordion title="fal Krea 2">
    fal の Krea 2 モデルでは、Flux が使用する汎用の
    `image_size` スキーマではなく、fal ネイティブの Krea スキーマを使用します。
    OpenClaw は次の値を送信します。

    - アスペクト比のヒントには `aspect_ratio`
    - `creativity`。デフォルトは `medium`
    - `image` または `images` が指定された場合は `image_style_references`

    より高速で表現力豊かなイラストには Krea 2 Medium を、
    より低速で詳細な写実的表現や質感のある外観には Krea 2 Large を選択します。

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/krea/v2/medium/text-to-image",
          },
        },
      },
    }
    ```

    Krea 2 は現在、リクエストごとに画像を 1 枚返します。Krea には `aspectRatio` の使用を
    推奨します。OpenClaw は `size` を、サポートされる最も近い Krea アスペクト比にマッピングし、
    Krea で `resolution` を破棄するのではなく拒否します。Krea ネイティブの創造性レベルを
    使用する場合は `fal.creativity` を使用します。

    ```json
    {
      "model": "fal/krea/v2/medium/text-to-image",
      "prompt": "リソグラフの質感を持つサイバージン風ポートレート",
      "aspectRatio": "9:16",
      "fal": {
        "creativity": "high"
      }
    }
    ```

  </Accordion>
  <Accordion title="MiniMax のデュアル認証">
    MiniMax の画像生成は、同梱されている両方の MiniMax
    認証経路で利用できます。

    - API キーによるセットアップには `minimax/image-01`
    - OAuth によるセットアップには `minimax-portal/image-01`

  </Accordion>
  <Accordion title="xAI grok-imagine-image">
    同梱の xAI プロバイダーは、プロンプトのみのリクエストには `/v1/images/generations` を使用し、
    `image` または `images` が存在する場合は `/v1/images/edits` を使用します。

    - モデル：`xai/grok-imagine-image`、`xai/grok-imagine-image-quality`
    - 枚数：最大 4
    - 参照：`image` 1 枚、または `images` 最大 3 枚
    - アスペクト比：`1:1`、`16:9`、`9:16`、`4:3`、`3:4`、`3:2`、`2:3`、`2:1`、
      `1:2`、`19.5:9`、`9:19.5`、`20:9`、`9:20`
    - 解像度：`1K`、`2K`
    - 出力：OpenClaw が管理する画像添付ファイルとして返されます

    OpenClaw は、これらの制御がプロバイダー間で共有される
    `image_generate` コントラクトに存在するようになるまで、xAI ネイティブの
    `quality`、`mask`、`user`、
    および `auto` アスペクト比を意図的に公開しません。

  </Accordion>
</AccordionGroup>

## 例

<Tabs>
  <Tab title="生成（4K 横長）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="OpenClaw 画像生成用のすっきりしたエディトリアルポスター" size=3840x2160 count=1
```
  </Tab>
  <Tab title="生成（透過 PNG）">
```text
/tool image_generate action=generate model=openai/gpt-image-1.5 prompt="透明な背景上のシンプルな赤い丸形ステッカー" outputFormat=png background=transparent
```

同等の CLI：

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "透明な背景上のシンプルな赤い丸形ステッカー" \
  --json
```

  </Tab>
  <Tab title="生成（OpenAI の低品質設定）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="静かな生産性向上アプリ用の低コストなポスター案" quality=low openai='{"moderation":"low"}'
```

同等の CLI：

```bash
openclaw infer image generate \
  --model openai/gpt-image-2 \
  --quality low \
  --openai-moderation low \
  --prompt "静かな生産性向上アプリ向けの低コストなポスター草案" \
  --json
```

  </Tab>
  <Tab title="生成（正方形 2 枚）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="落ち着いた生産性向上アプリのアイコンについて、2 つのビジュアル案" size=1024x1024 count=2
```
  </Tab>
  <Tab title="編集（参照画像 1 枚）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="被写体はそのままに、背景を明るいスタジオセットに置き換える" image=/path/to/reference.png size=1024x1536
```
  </Tab>
  <Tab title="編集（複数の参照画像）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="最初の画像のキャラクターの特徴と、2 番目の画像のカラーパレットを組み合わせる" images='["/path/to/character.png","/path/to/palette.jpg"]' size=1536x1024
```
  </Tab>
  <Tab title="Krea スタイル参照">
```text
/tool image_generate action=generate model=fal/krea/v2/medium/text-to-image prompt="このカラーパレットと印刷テクスチャを使用した、表現力豊かなエディトリアルポートレート" images='["/path/to/palette.png","/path/to/texture.jpg"]' aspectRatio=9:16 fal='{"creativity":"high"}'
```
  </Tab>
</Tabs>

同じ `--output-format`、`--background`、`--quality`、および
`--openai-moderation` フラグを `openclaw infer image edit` でも使用できます。
`--openai-background` は OpenAI 固有のエイリアスとして引き続き使用できます。現在、
OpenAI 以外のバンドル済みプロバイダーは明示的な背景制御を宣言していないため、
それらでは `background: "transparent"` が無視されたものとして報告されます。

## 関連項目

- [ツールの概要](/ja-JP/tools) - 利用可能なすべてのエージェントツール
- [ComfyUI](/ja-JP/providers/comfy) - ローカル ComfyUI および Comfy Cloud のワークフロー設定
- [fal](/ja-JP/providers/fal) - fal の画像および動画プロバイダーの設定
- [Google（Gemini）](/ja-JP/providers/google) - Gemini 画像プロバイダーの設定
- [Microsoft Foundry Plugin](/ja-JP/plugins/reference/microsoft-foundry) - Microsoft Foundry のチャットおよび MAI 画像の設定
- [MiniMax](/ja-JP/providers/minimax) - MiniMax 画像プロバイダーの設定
- [OpenAI](/ja-JP/providers/openai) - OpenAI Images プロバイダーの設定
- [Vydra](/ja-JP/providers/vydra) - Vydra の画像、動画、音声の設定
- [xAI](/ja-JP/providers/xai) - Grok の画像、動画、検索、コード実行、TTS の設定
- [設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults) - `imageGenerationModel` の設定
- [モデル](/ja-JP/concepts/models) - モデルの設定とフェイルオーバー
