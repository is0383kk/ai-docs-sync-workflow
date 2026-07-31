---
read_when:
    - OpenClaw で Runway の動画生成を使用する場合
    - Runway API キーと環境変数の設定が必要です
    - Runway をデフォルトの動画プロバイダーに設定したい場合
summary: OpenClaw での Runway 動画生成のセットアップ
title: Runway
x-i18n:
    generated_at: "2026-07-26T10:18:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw には、ホスト型動画生成用の `runway` プロバイダーが同梱されています。これはデフォルトで有効になっており、`videoGenerationProviders` コントラクトに登録されています。

| プロパティ        | 値                                                             |
| --------------- | ----------------------------------------------------------------- |
| プロバイダー ID     | `runway`                                                          |
| Plugin          | 同梱、`enabledByDefault: true`                                 |
| 認証環境変数   | `RUNWAYML_API_SECRET`（標準）または `RUNWAY_API_KEY`             |
| オンボーディングフラグ | `--auth-choice runway-api-key`                                    |
| CLI 直接指定フラグ | `--runway-api-key <key>`                                          |
| API             | Runway のタスクベース動画生成（`GET /v1/tasks/{id}` ポーリング） |
| デフォルトモデル   | `runway/gen4.5`                                                   |

## はじめに

<Steps>
  <Step title="API キーを設定する">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="Runway をデフォルトの動画プロバイダーに設定する">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="動画を生成する">
    エージェントに動画の生成を依頼します。Runway が自動的に使用されます。
  </Step>
</Steps>

## 対応するモードとモデル

このプロバイダーは、3 つのモードに分類された 7 つの Runway モデルを公開します。同じモデル ID が複数のモードに対応する場合があります（たとえば、`gen4.5` はテキストから動画への変換と画像から動画への変換の両方で使用できます）。

| モード           | モデル                                                                 | 参照入力         |
| -------------- | ---------------------------------------------------------------------- | ----------------------- |
| テキストから動画  | `gen4.5`（デフォルト）、`veo3.1`、`veo3.1_fast`、`veo3`                    | なし                    |
| 画像から動画 | `gen4.5`、`gen4_turbo`、`gen3a_turbo`、`veo3.1`、`veo3.1_fast`、`veo3` | ローカルまたはリモート画像 1 枚 |
| 動画から動画 | `gen4_aleph`                                                           | ローカルまたはリモート動画 1 本 |

ローカルの画像および動画の参照は、データ URI 経由でサポートされます。

| アスペクト比         | 使用可能な値                              |
| --------------------- | ------------------------------------------- |
| テキストから動画         | `16:9`、`9:16`                              |
| 画像および動画の編集 | `1:1`、`16:9`、`9:16`、`3:4`、`4:3`、`21:9` |

<Warning>
  現在、動画から動画への変換には `runway/gen4_aleph` が必要です。ほかの Runway モデル ID では、動画の参照入力が拒否されます。
</Warning>

<Note>
  誤った列から Runway モデル ID を選択すると、API リクエストが OpenClaw から送信される前に明示的なエラーが発生します。プロバイダーは `extensions/runway/video-generation-provider.ts` で、`model` をモードの許可リスト（`TEXT_ONLY_MODELS`、`IMAGE_MODELS`、`VIDEO_MODELS`）に照らして検証します。
</Note>

## 設定

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## 詳細設定

<AccordionGroup>
  <Accordion title="環境変数のエイリアス">
    OpenClaw は `RUNWAYML_API_SECRET`（標準）と `RUNWAY_API_KEY` の両方を認識します。
    どちらの変数でも Runway プロバイダーを認証できます。
  </Accordion>

  <Accordion title="タスクのポーリング">
    Runway はタスクベースの API を使用します。生成リクエストを送信した後、OpenClaw は動画の準備が完了するまで
    `GET /v1/tasks/{id}` をポーリングします。このポーリング動作に追加の
    設定は必要ありません。
  </Accordion>
</AccordionGroup>

## 関連情報

<CardGroup cols={2}>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共通のツールパラメーター、プロバイダーの選択、非同期動作。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/config-agents#agent-defaults" icon="gear">
    動画生成モデルを含む、エージェントのデフォルト設定。
  </Card>
</CardGroup>
