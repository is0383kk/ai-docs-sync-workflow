---
read_when:
    - code_execution を有効化または設定する必要があります
    - ローカルシェルへのアクセスなしでリモート分析を行いたい場合
    - x_search または web_search とリモート Python 分析を組み合わせる場合
summary: 'code_execution: xAI を使用してサンドボックス化されたリモート Python 分析を実行する'
title: コード実行
x-i18n:
    generated_at: "2026-07-26T09:20:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution` は、xAI の Responses API
（`https://api.x.ai/v1/responses`、`x_search` が使用するものと同じエンドポイント）上で、サンドボックス化されたリモート Python 分析を実行します。バンドルされている `xai` Plugin により、`tools` コントラクトの下で登録されます。

<Warning>
  `code_execution` は xAI のサーバー上で実行されます。xAI の料金はツール呼び出し 1,000 回あたり $5 で、
  これにモデルの入力トークンと出力トークンの料金が加算されます。
</Warning>

| プロパティ           | 値                                                                             |
| ------------------ | --------------------------------------------------------------------------------- |
| ツール名          | `code_execution`                                                                  |
| プロバイダー Plugin    | `xai`（バンドル済み、`enabledByDefault: true`）                                         |
| 認証               | xAI 認証プロファイル、`XAI_API_KEY`、または `plugins.entries.xai.config.webSearch.apiKey` |
| デフォルトモデル      | `grok-4.3`                                                                        |
| デフォルトタイムアウト    | 30 秒                                                                        |
| デフォルトの `maxTurns` | 未設定（xAI が独自の内部制限を適用）                                        |

計算、集計、簡単な統計、グラフ形式の分析に使用できます。これには、`x_search` または `web_search` が返したデータの分析も含まれます。ローカルファイル、シェル、リポジトリ、ペアリング済みデバイスにはアクセスできず、呼び出し間で状態も保持されません。そのため、各呼び出しはノートブックセッションではなく、一時的な分析として扱ってください。最新の X データを使用する場合は、まず [`x_search`](/ja-JP/tools/web#x_search) を実行し、その結果をパイプで渡します。

ローカルで実行する場合は、代わりに [`exec`](/ja-JP/tools/exec) を使用してください。

## セットアップ

<Steps>
  <Step title="xAI の認証情報を指定する">
    OAuth には、対象となる SuperGrok または X Premium サブスクリプションが必要です
    （デバイスコード検証を使用するため、localhost コールバックなしでリモートホストからも利用できます）。

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    新規インストール時には、オンボーディングでも同じ選択肢を利用できます。

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    または API キーを使用します。

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    または設定を使用します。

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    これら 3 つのいずれを使用しても、`x_search` と Grok `web_search` も利用可能になります。

  </Step>

  <Step title="code_execution を有効化して調整する">
    `enabled` を省略した場合、`code_execution` が公開されるのは、アクティブなモデルのプロバイダーが `xai` であり、かつ xAI の認証情報を解決できる場合のみです。既知の xAI 以外のプロバイダーを使用するアクティブモデルでプロバイダーをまたいだ利用をオプトインするには、`plugins.entries.xai.config.codeExecution.enabled` を `true` に設定します。アクティブなモデルのプロバイダーが欠落しているか解決できない場合、ツールは非表示のままです。すべてのプロバイダーで無効にするには、`enabled` を `false` に設定します。xAI の認証情報は常に必要です。

    同じブロックを使用して、モデル、ターン上限、またはタイムアウトを上書きできます。

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // 既知の xAI 以外のモデルプロバイダーでは必須
                model: "grok-4.3", // デフォルトの xAI コード実行モデルを上書き
                maxTurns: 2,            // 内部ツールターン数の任意の上限
                timeoutSeconds: 30,     // リクエストタイムアウト（デフォルト: 30）
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Gateway を再起動する">
    ```bash
    openclaw gateway restart
    ```

    xAI Plugin が再登録され、上記のプロバイダー、有効化、認証の各チェックに合格すると、エージェントのツール一覧に `code_execution` が表示されます。

  </Step>
</Steps>

## 使用方法

分析の意図を明示してください。このツールが受け取るパラメーターは `task` 1 つだけなので、完全なリクエストとインラインデータを 1 つのプロンプトで送信します。

```text
code_execution を使用して、次の数値の 7 日移動平均を計算してください: ...
```

```text
x_search を使用して今週 OpenClaw に言及している投稿を検索し、次に code_execution を使用して日別に集計してください。
```

```text
web_search を使用して最新の AI ベンチマーク数値を収集し、次に code_execution を使用して変化率を比較してください。
```

## エラー

認証がない場合、ツールは例外をスローせず、構造化された JSON エラーを返すため、エージェントは自己修正できます。

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution には xAI の認証情報が必要です。Grok でサインインするには `openclaw onboard --auth-choice xai-oauth` を実行するか、`openclaw onboard --auth-choice xai-api-key` を実行するか、Gateway 環境に `XAI_API_KEY` を設定するか、`plugins.entries.xai.config.webSearch.apiKey` を設定してください。",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## 関連項目

<CardGroup cols={2}>
  <Card title="Exec ツール" href="/ja-JP/tools/exec" icon="terminal">
    使用中のマシンまたはペアリング済み Node 上でのローカルシェル実行。
  </Card>
  <Card title="Exec の承認" href="/ja-JP/tools/exec-approvals" icon="shield">
    シェル実行の許可・拒否ポリシー。
  </Card>
  <Card title="Web ツール" href="/ja-JP/tools/web" icon="globe">
    `web_search`、`x_search`、および `web_fetch`。
  </Card>
  <Card title="xAI プロバイダー" href="/ja-JP/providers/xai" icon="microchip">
    Grok モデル、Web/X 検索、およびコード実行の設定。
  </Card>
</CardGroup>
