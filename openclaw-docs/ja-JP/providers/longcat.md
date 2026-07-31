---
read_when:
    - OpenClaw で LongCat-2.0 を使用する場合
    - LongCat API キーまたはモデルの制限が必要です
summary: LongCat-2.0 の LongCat API セットアップ
title: LongCat
x-i18n:
    generated_at: "2026-07-26T09:15:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c447f9c42e6547a69d2124debcb685c32fe59de29bfc551e18e791d9f280584
    source_path: providers/longcat.md
    workflow: 16
---

[LongCat](https://longcat.ai) は、コーディングおよびエージェント型ワークロード向けに構築された推論モデル LongCat-2.0 のホステッド API を提供します。OpenClaw は、LongCat の OpenAI 互換エンドポイント用の公式 `longcat` Plugin を提供します。

| プロパティ | 値 |
| ---------- | ---------------------------------- |
| プロバイダー | `longcat` |
| 認証 | `LONGCAT_API_KEY` |
| API | OpenAI 互換 Chat Completions |
| ベース URL | `https://api.longcat.chat/openai` |
| モデル | `longcat/LongCat-2.0` |
| コンテキスト | 1,048,576 tokens |
| 最大出力 | 131,072 tokens |
| 入力 | テキスト |

## Plugin のインストール

公式パッケージをインストールしてから、Gateway を再起動します。

```bash
openclaw plugins install @openclaw/longcat-provider
openclaw gateway restart
```

## はじめに

<Steps>
  <Step title="API キーを作成">
    [LongCat API プラットフォーム](https://longcat.chat/platform/)にサインインし、
    [API Keys](https://longcat.chat/platform/api_keys) ページでキーを作成します。
  </Step>
  <Step title="オンボーディングを実行">
    ```bash
    openclaw onboard --auth-choice longcat-api-key
    ```
  </Step>
  <Step title="モデルを確認">
    ```bash
    openclaw models list --provider longcat
    ```
  </Step>
</Steps>

プライマリモデルがまだ設定されていない場合、オンボーディングはホステッドカタログを追加し、`longcat/LongCat-2.0` を選択します。

### 非対話型セットアップ

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice longcat-api-key \
  --longcat-api-key "$LONGCAT_API_KEY"
```

## 推論の動作

LongCat は二値の思考制御を公開しています。OpenClaw は有効な思考レベルを
`thinking: { type: "enabled" }` に、`/think off` を
`thinking: { type: "disabled" }` にマッピングします。LongCat は現在
`reasoning_effort` を文書化していないため、OpenClaw は送信しません。

LongCat は `reasoning_content` に推論を返します。OpenClaw はアシスタントのツール呼び出しターンを再生するときにそのフィールドを保持するため、複数ターンのエージェントセッションではプロバイダーが想定するメッセージ形式が維持されます。

## 料金

組み込みカタログでは、LongCat の従量課金制の定価を 100 万 tokens あたりの USD で使用します。キャッシュなし入力は $0.75、キャッシュ済み入力は $0.015、出力は $2.95 です。LongCat は一時的な割引を提供する場合があります。[料金ページ](https://longcat.chat/platform/docs/Pricing/LongCat-2.0.html)と請求記録が正式な情報源です。

## セルフホスト型 LongCat-2.0

`longcat` プロバイダーは LongCat のホステッド API を対象としています。
[Hugging Face](https://huggingface.co/meituan-longcat/LongCat-2.0) のオープンウェイトを使用する場合は、OpenAI 互換ランタイムでモデルを提供し、代わりに OpenClaw の既存の
[vLLM](/ja-JP/providers/vllm) または [SGLang](/ja-JP/providers/sglang) プロバイダーを使用してください。

セルフホスト型プロバイダーカタログには、ランタイムの正確なモデル識別子を保持してください。ローカルデプロイを `longcat/LongCat-2.0` 経由でルーティングしないでください。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="シェルではキーが機能するが Gateway では機能しない">
    デーモン管理の Gateway プロセスは、対話型シェルのすべての変数を継承するわけではありません。`LONGCAT_API_KEY` を `~/.openclaw/.env` に配置するか、オンボーディングで設定するか、承認されたシークレット参照を使用してください。
  </Accordion>

  <Accordion title="リクエストが 402 または 429 で失敗する">
    `402` は、アカウントのトークンクォータが不足していることを意味します。`429` は、API キーがレート制限に達したことを意味します。[LongCat の使用状況](https://longcat.chat/platform/usage)を確認し、プロバイダーのバックオフ期間が経過してから、レート制限されたリクエストを再試行してください。
  </Accordion>

  <Accordion title="モデルが表示されない">
    `openclaw plugins list` を実行して `longcat` Plugin が有効になっていることを確認し、その後 `openclaw models list --provider longcat` を実行してください。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー設定、モデル参照、フェイルオーバーの動作。
  </Card>
  <Card title="LongCat API ドキュメント" href="https://longcat.chat/platform/docs/" icon="arrow-up-right-from-square">
    ホステッド API のエンドポイント、認証、制限、例。
  </Card>
  <Card title="LongCat-2.0 モデルカード" href="https://huggingface.co/meituan-longcat/LongCat-2.0" icon="arrow-up-right-from-square">
    アーキテクチャ、デプロイガイダンス、モデルの詳細。
  </Card>
  <Card title="シークレット" href="/ja-JP/gateway/secrets" icon="key">
    設定に平文を埋め込まずにプロバイダーの認証情報を保存します。
  </Card>
</CardGroup>
