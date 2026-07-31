---
read_when:
    - NovitaAI モデルで OpenClaw を実行する場合
    - Novita のプロバイダー ID、キー、またはエンドポイントが必要です
summary: OpenClaw で NovitaAI の OpenAI 互換 API を使用する
title: NovitaAI
x-i18n:
    generated_at: "2026-07-26T09:17:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83e0e43e68d85d73e790023858a49f971b683129dbbdf6092fbd8bba4d8da331
    source_path: providers/novita.md
    workflow: 16
---

NovitaAI は、OpenAI 互換 API を備えたホステッド AI インフラストラクチャプロバイダーです。
OpenClaw のバンドルプロバイダーとして提供されるため（個別の Plugin インストールは不要）、
認証情報には通常のモデル認証フローが使用され、モデル参照は
`novita/deepseek/deepseek-v3-0324` のような形式になります。

## セットアップ

[novita.ai/settings/key-management](https://novita.ai/settings/key-management) で API キーを作成してから、次を実行します。

```bash
openclaw onboard --auth-choice novita-api-key
```

または、次を設定します。

```bash
export NOVITA_API_KEY="<your-novita-api-key>" # pragma: allowlist secret
```

## デフォルト

| 設定          | 値                                 |
| ------------- | ---------------------------------- |
| プロバイダー ID | `novita`                           |
| エイリアス      | `novita-ai`, `novitaai`            |
| ベース URL      | `https://api.novita.ai/openai/v1`  |
| 環境変数        | `NOVITA_API_KEY`                   |
| デフォルトモデル | `novita/deepseek/deepseek-v3-0324` |

## バンドルモデルカタログ

- `novita/moonshotai/kimi-k2.5`
- `novita/minimax/minimax-m2.7`
- `novita/zai-org/glm-5`
- `novita/deepseek/deepseek-v3-0324`
- `novita/deepseek/deepseek-r1-0528`
- `novita/qwen/qwen3-235b-a22b-fp8`

これは出発点であり、リアルタイムのカタログではありません。アカウント、リージョン、または
Novita の現在の提供内容によって、ルートが追加、削除、制限される場合があります。長期間使用する
デフォルトを設定する前に確認してください。

```bash
openclaw models list --provider novita
```

## Novita を選ぶ場面

- OpenAI 互換 API による、オープンウェイトモデルへのホステッドアクセス。
- 単一のプロバイダーアカウントを通じた DeepSeek、Kimi、MiniMax、GLM、または Qwen ファミリーのルート。
- DeepInfra、GMI、OpenRouter、またはベンダーの直接 API に加えて利用できる、別のホステッドフォールバック経路。
- LM Studio、Ollama、SGLang、または vLLM のインフラストラクチャを保守する代わりとなる、プロバイダー側でのモデルホスティング。

ベンダー固有のリクエストパラメーターやサポート契約が必要な場合は、ベンダーの直接プロバイダーを選択してください。モデルを
自身のハードウェアまたはネットワーク境界内で実行する必要がある場合は、ローカルプロバイダーを選択してください。

## トラブルシューティング

- `401`/`403`: Novita のキーマネジメントページでキーを確認し、保存済みプロファイルが
  古い場合は `openclaw onboard --auth-choice novita-api-key` を再実行してください。
- 不明なモデルのエラー: `openclaw models list --provider novita` が返した正確な `novita/<route-id>` を使用してください。
- 低速または失敗するルート: 別の Novita モデルルートを試すか、プロバイダー固有の
  ばらつきを許容できるワークロードでは Novita をフォールバックプロバイダーとして設定してください。

## 関連項目

- [モデルプロバイダー](/ja-JP/concepts/model-providers)
- [プロバイダーディレクトリ](/ja-JP/providers/index)
