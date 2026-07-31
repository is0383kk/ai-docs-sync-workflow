---
read_when:
    - OpenClaw で OpenAI モデルを使用する場合
    - API キーではなく Codex サブスクリプション認証を使用する場合
    - GPT-5 エージェントに、より厳格な実行動作が必要な場合
summary: OpenClaw で API キーまたは Codex サブスクリプションを使用して OpenAI を利用する
title: OpenAI
x-i18n:
    generated_at: "2026-07-26T09:59:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw は、直接の API キー認証と
ChatGPT/Codex サブスクリプション認証の両方に、1 つのプロバイダー ID `openai` を使用します。`openai/*` は正規のモデルルートです。
ランタイムポリシーが未設定または `auto` の埋め込みエージェントターンでは、OpenAI のルート情報により、
OpenClaw がバンドルされた Codex app-server ランタイムを暗黙的に選択できるかどうかが決まります。
`openai/*` プレフィックスだけではランタイムは選択されません。

- **エージェントモデル** - 明示的な
  `agentRuntime` 設定または OpenAI の暗黙的なルートポリシーによって選択されたランタイムを介する `openai/*`。ChatGPT/Codex サブスクリプションを使用する場合は Codex
  認証でサインインし、キーに基づく課金を使用する場合は API キー認証
  プロファイルを設定します。
- **エージェント以外の OpenAI API** - `OPENAI_API_KEY` または `openai` API キー認証プロファイルを介した、
  使用量に応じて課金される OpenAI Platform への直接アクセス。
- **レガシー設定** - `codex/*` および `openai-codex/*` の参照は、
  `openclaw doctor --fix` によって `openai/*` とモデルスコープの `agentRuntime.id: "codex"` に
  修復されます。

OpenAI は、OpenClaw のような外部ツールやワークフローでのサブスクリプション OAuth の使用を明示的にサポートしています。

## 使用量とコストの追跡

OpenClaw は、サブスクリプションのクォータと Platform API の課金を区別して扱います。

- ChatGPT/Codex OAuth には、サブスクリプションプラン、クォータ期間、クレジット残高が表示されます。
- `OPENAI_ADMIN_KEY` は、Control UI の **使用量** に、プロバイダーから報告された組織のコストと completions 使用量の過去 30 日分を表示します。これには、日別支出、リクエスト数とトークン数の合計、上位モデル、コストカテゴリーが含まれます。
- `OPENAI_PROJECT_ID` を使用すると、Admin API の履歴を任意で 1 つのプロジェクトに限定できます。
- OpenClaw は、`OPENAI_API_KEY` または `openai` 推論プロファイルを組織 API に送信することはありません。これらの認証情報は、カスタム、Azure、またはエージェントローカルのエンドポイントに属している可能性があります。

明示的な Admin キーは OAuth より優先されます。プロバイダーから報告された履歴は、OpenClaw がセッションから算出した推定コストとは統合されません。この履歴には、他のクライアントによる API アクティビティやプロバイダー側の課金調整が含まれる場合があります。

OpenAI の [API 使用状況ダッシュボード](https://help.openai.com/en/articles/10478918)のドキュメントでは、使用量データに必要な組織所有者権限と明示的な Usage Dashboard 権限について説明しています。

プロバイダー、モデル、ランタイム、チャンネルは、それぞれ別のレイヤーです。これらのラベルが
混同されている場合は、設定を変更する前に[エージェントランタイム](/ja-JP/concepts/agent-runtimes)を
参照してください。

## クイック選択

| 目的                                              | 使用するもの                                                                | 注記                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ChatGPT/Codex サブスクリプション、ネイティブ Codex ランタイム  | `openai/gpt-5.6-sol`                                               | 新規のサブスクリプション設定。Codex 認証でサインインします。                  |
| エージェントターンに対する直接の API キー課金            | `openai/gpt-5.6` と順序指定された API キー認証プロファイル              | 新規の API キー設定。修飾なしの直接 API ID は Sol に解決されます。        |
| GPT-5.6 の正確なティアを選択                      | `openai/gpt-5.6-sol`、`-terra`、または `-luna`                         | このアカウントで利用可能なティアは `models list` で確認します。        |
| GPT-5.6 にアクセスできないアカウント                    | `openai/gpt-5.5`                                                   | 明示的な復旧選択。OpenClaw は暗黙的にダウングレードしません。     |
| 直接の API キー課金、明示的な OpenClaw ランタイム | `openai/gpt-5.6` とプロバイダー/モデル `agentRuntime.id: "openclaw"` | 通常の `openai` API キープロファイルを選択します。                           |
| 最新の ChatGPT Instant モデルエイリアス                | `openai/chat-latest`                                               | 直接の API キーのみ。安定版のデフォルトではなく、移動するエイリアスです。          |
| 画像の生成または編集                       | `openai/gpt-image-2`                                               | `OPENAI_API_KEY` または Codex OAuth で動作します。                         |
| 背景が透明な画像                     | `openai/gpt-image-1.5`                                             | `outputFormat` を `png` または `webp` と `background=transparent` に設定します。 |

## 名前の対応表

| 表示される名前                            | レイヤー             | 意味                                                                                  |
| --------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------- |
| `openai`                                | プロバイダープレフィックス   | 正規の OpenAI モデルルート。ルート情報によって暗黙的なランタイムが決まります。                |
| `codex` Plugin                          | Plugin            | ネイティブ Codex app-server ランタイムと `/codex` チャットコントロールを提供するバンドル Plugin。 |
| プロバイダー/モデル `agentRuntime.id: codex` | エージェントランタイム     | 一致する埋め込みターンにネイティブ Codex app-server ハーネスを強制します。                   |
| `/codex ...`                            | チャットコマンドセット  | 会話から Codex app-server スレッドをバインドまたは制御します。                               |
| `runtime: "acp", agentId: "codex"`      | ACP セッションルート | ACP/acpx を介して Codex を実行する明示的なフォールバックパス。                                 |

## 暗黙的なエージェントランタイム

プロバイダー/モデルの `agentRuntime` ポリシーが未設定または `auto` の場合、OpenAI が所有する
プロバイダールートポリシーは、有効なエンドポイントとアダプターから暗黙的なランタイムを
選択します。

| 有効なルート情報                                                                                                                                                  | 暗黙的なランタイム      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `openai-responses` を使用する正確な公式 Platform HTTPS エンドポイント、または `openai-chatgpt-responses` を使用する正確な公式 ChatGPT HTTPS エンドポイント。作成されたリクエストオーバーライドなし | Codex が選択される場合があります |
| 作成された `openai-completions` アダプター                                                                                                                                  | OpenClaw              |
| カスタムエンドポイント                                                                                                                                                        | OpenClaw              |
| HTTP を使用する明示的かつ正確な公式エンドポイント                                                                                                                            | 拒否              |
| 作成されたプロバイダー/モデルのリクエストオーバーライドがあるルート                                                                                                                 | OpenClaw              |

明示的なデフォルト以外のプロバイダー/モデル `agentRuntime.id` は、引き続き優先されます。
たとえば、`agentRuntime.id: "openclaw"` は、本来 Codex を選択できるルートでも
OpenClaw を維持します。一方、`agentRuntime.id: "codex"` は Codex を必須とし、
有効なルートが Codex 互換と宣言されていない場合は安全側に倒して失敗します。
ランタイムの選択によって認証情報の種類や課金が変わることはありません。Platform API キー
認証と ChatGPT/Codex サブスクリプション認証は引き続き区別されます。

`openclaw doctor --fix` は、レガシーの `codex/*` および `openai-codex/*` モデル
参照、レガシー Codex 認証プロファイル ID、レガシー Codex 認証順序エントリを、
正規の `openai` ルートに移行します。移行されたモデル参照には、モデルスコープの
`agentRuntime.id: "codex"` が付与されます。新しい認証順序設定には `auth.order.openai` を使用してください。

<Note>
新規の OpenAI 設定では、プライマリモデルが設定されていない場合にのみ GPT-5.6 をプライマリに設定します。
OpenAI 認証を追加または更新しても、`openai/gpt-5.5` を含む既存の明示的な
選択は維持されます。ただし、`models auth login --set-default` または
`models set` を明示的に使用した場合を除きます。エージェントモデルに API キー認証を
使用する場合にのみ、API キー認証プロファイルを使用してください。
</Note>

## GPT-5.6 限定プレビュー

OpenClaw は、正確な `openai/gpt-5.6-sol`、
`openai/gpt-5.6-terra`、および `openai/gpt-5.6-luna` モデル ID を認識します。現在のカタログでは、3 つすべてが
`xhigh` および `max` 推論を公開しています。OpenAI は、Sol をフラッグシップティア、Terra をバランス型ティア、Luna を高速かつ
低コストのティアと説明しています。
[GPT-5.6 リリース発表](https://openai.com/index/previewing-gpt-5-6-sol/)
および[アクセスガイド](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna)を参照してください。

OpenAI API キーによる直接認証では、修飾なしの `openai/gpt-5.6` ID は
Sol のエイリアスであり、新規設定のデフォルトです。ネイティブ Codex カタログは、
この直接 API のエイリアスをクライアント側で適用しません。ワークスペースのアクセス権に応じて、
正確な Sol、Terra、Luna の ID が表示される場合があります。そのため、新規の ChatGPT/Codex OAuth 設定では
`openai/gpt-5.6-sol` を使用します。次のコマンドで現在のアカウントを確認してください。

```bash
openclaw models list --provider openai
```

API 組織と Codex ワークスペースのアクセス権は異なる場合があります。GPT-5.6 が
利用できない場合は、GPT-5.5 を明示的に選択してください。

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw はアップストリームのアクセスエラーを表示し、GPT-5.6 の選択を
暗黙的に GPT-5.5 に置き換えることはありません。

<Note>
対象となる正確な公式 HTTPS ルートでは、ランタイムポリシーが未設定または `auto` の場合、
バンドルされた Codex app-server Plugin が選択されることがあります。作成された Completions ルート、
カスタムエンドポイント、リクエストトランスポートのオーバーライドは OpenClaw のままです。平文の
公式 HTTP エンドポイントは拒否されます。明示的なプロバイダー/モデルのランタイム設定は引き続き
優先されます。古いレガシー Codex モデル参照、
`codex-cli/*` 参照、または明示的なランタイム設定によって設定されていない古いランタイムセッションの固定を修復するには、`openclaw doctor --fix` を実行します。
</Note>

## OpenClaw の機能対応範囲

| OpenAI の機能         | OpenClaw の提供機能                                                                              | 状態                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| チャット / Responses          | `openai/<model>` モデルプロバイダー                                                               | 対応                                                             |
| Codex サブスクリプションモデル | OpenAI OAuth を使用する `openai/<model>`                                                            | 対応                                                             |
| レガシー Codex モデル参照   | 古い Codex モデル参照、`codex-cli/<model>`                                                     | doctor により `openai/<model>` に修復                          |
| Codex app-server ハーネス  | ランタイムが未設定/`auto` の Codex 互換 HTTPS ルート、または明示的な `agentRuntime.id: codex`  | 対応                                                             |
| サーバー側ウェブ検索    | OpenAI Responses ネイティブツール                                                                  | ウェブ検索が有効で、ほかのプロバイダーが固定されていない場合に対応 |
| 画像                    | `image_generate`                                                                              | 対応                                                             |
| 動画                    | `video_generate`                                                                              | 対応                                                             |
| テキスト読み上げ            | `tts.provider: "openai"` / `tts`                                                              | 対応                                                             |
| バッチ音声テキスト変換      | `tools.media.audio` / メディア理解                                                     | 対応                                                             |
| ストリーミング音声テキスト変換  | Voice Call `streaming.provider: "openai"`                                                     | 対応                                                             |
| リアルタイム音声            | Voice Call `realtime.provider: "openai"` / Control UI Talk `talk.realtime.provider: "openai"` | 対応（OpenAI Platform API キー）                                   |
| 埋め込み                | メモリ埋め込みプロバイダー                                                                     | 対応                                                             |

<Note>
OpenAI のリアルタイム音声は、公開されている **OpenAI Platform Realtime
API** を経由し、Platform API キーが必要です。Codex OAuth トークンは
代わりに ChatGPT Codex バックエンドを認証するものであり、公開 Realtime
エンドポイント用の Platform API キーと互換性はありません。

API キー認証で請求情報が不足していると報告された場合は、API キー
認証の使用時に、リアルタイム認証情報を提供する組織の Platform クレジットを
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
で補充してください。リアルタイム音声では、
`openclaw onboard --auth-choice openai-api-key` によって作成された `openai` API キー認証プロファイル、
Control UI Talk 用に `talk.realtime.providers.openai.apiKey` で設定された Platform API キー、
Voice Call 用の `plugins.entries.voice-call.config.realtime.providers.openai.apiKey`、または
`OPENAI_API_KEY` 環境変数を使用できます。

Control UI Video Talk では、OpenAI WebRTC は必要に応じてカメラのコンテキストを受信します。
モデルが `describe_view` を呼び出すと、ブラウザーはサイズ制限された JPEG を 1 枚、
リアルタイムデータチャネル経由で送信します。OpenClaw は OpenAI セッションに
継続的なカメラトラックを添付しません。
</Note>

## メモリ埋め込み

OpenClaw は、`memory_search` のインデックス作成とクエリ埋め込みに、
OpenAI または OpenAI 互換の埋め込みエンドポイントを使用できます。

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

非対称の埋め込みラベルを必要とする OpenAI 互換エンドポイントでは、
`memory.search` の下に `queryInputType` と `documentInputType` を設定します。OpenClaw は
これらをプロバイダー固有の `input_type` リクエストフィールドとして転送します。クエリ
埋め込みでは `queryInputType` を使用し、インデックス化されたメモリチャンクとバッチインデックス作成では
`documentInputType` を使用します。完全な例については、
[メモリ設定リファレンス](/ja-JP/reference/memory-config#provider-specific-config)
を参照してください。

## はじめに

<Tabs>
  <Tab title="API キー（OpenAI Platform）">
    **最適な用途:** API への直接アクセスと従量課金。

    <Steps>
      <Step title="API キーを取得する">
        [OpenAI Platform ダッシュボード](https://platform.openai.com/api-keys)から API キーを作成またはコピーします。
      </Step>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        または、キーを直接渡します。

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### ルートの概要

    | モデル参照        | ランタイムポリシーまたはルートの詳細                                 | ルート                     | 認証                              |
    | ---------------- | ------------------------------------------------------------- | ------------------------- | --------------------------------- |
    | `openai/gpt-5.6` | 未設定/`auto`、公式の完全一致 HTTPS ネイティブルート、リクエストによる上書きなし | Codex が選択される場合あり     | 順序付けされた API キー認証プロファイル      |
    | `openai/gpt-5.6` | プロバイダー/モデル `agentRuntime.id: "openclaw"`                  | OpenClaw 組み込みランタイム | 選択された `openai` API キープロファイル |
    | `openai/gpt-5.5` | 明示的なプロバイダー/モデル `agentRuntime.id`                     | 選択されたエージェントランタイム    | 選択された OpenAI API キープロファイル   |
    | `openai/*`       | 明示的に指定された Completions、カスタム、またはリクエストによる上書き | OpenClaw 組み込みランタイム | 認証情報の種類は変更されない |
    | `openai/*`       | 平文の公式 HTTP エンドポイント                  | 拒否                 | 認証情報は送信されない             |

    <Note>
    ランタイムが未設定または `auto` の場合、条件を満たす公式の完全一致 HTTPS
    ネイティブルートのみが、Codex app-server ハーネスを暗黙的に選択できます。エージェントモデルで
    API キー認証を使用するには、`openai` API キー認証プロファイルを作成し、
    `auth.order.openai` で順序を設定します。`OPENAI_API_KEY` は、
    エージェント以外の OpenAI API サーフェス向けの直接フォールバックとして引き続き使用されます。古い
    レガシー Codex 認証順序エントリを移行するには、`openclaw doctor --fix` を実行します。
    </Note>

    ### 設定例

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    直接 API の単独の `gpt-5.6` ID は Sol ティアに解決されます。この API
    組織で GPT-5.6 が公開されていない場合は、プライマリを
    `openai/gpt-5.5` に明示的に設定します。

    OpenAI API から ChatGPT の現在の Instant モデルを試すには、モデルを
    `openai/chat-latest` に設定します。

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` は変動するエイリアスです。新しい OpenAI API キーのセットアップでは代わりに
    `openai/gpt-5.6` を使用し、その直接 API の単独 ID は Sol に解決されます。
    `openai/gpt-5.5` を含む既存の明示的なプライマリは変更されません。
    `chat-latest` エイリアスは `medium` のテキスト詳細度のみを受け付けます。このモデルに
    ほかの詳細度が要求された場合、OpenClaw は `medium` に強制します。

    <Warning>
    OpenClaw は、直接 OpenAI API キールートでは `gpt-5.3-codex-spark` を
    提供しません。サインイン済みアカウントで公開されている場合に限り、
    Codex サブスクリプションカタログのエントリを通じて利用できます。
    </Warning>

  </Tab>

  <Tab title="Codex サブスクリプション">
    **最適な用途:** 別の API キーの代わりに、ChatGPT/Codex サブスクリプションを
    ネイティブ Codex app-server 実行で使用する場合。Codex cloud には
    ChatGPT へのサインインが必要です。

    <Steps>
      <Step title="Codex OAuth を実行する">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        または、OAuth を直接実行します。

        ```bash
        openclaw models auth login --provider openai
        ```

        ヘッドレス環境やコールバックを利用できないセットアップでは、`--device-code` を追加すると、
        localhost のブラウザーコールバックの代わりに ChatGPT のデバイスコードフローで
        サインインできます。

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="正規の OpenAI モデルルートを使用する">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        この公式の完全一致 HTTPS ネイティブルートには、ランタイム設定は不要です。
        Codex app-server ランタイムが自動的に選択される場合があり、そのランタイムが
        選択されると、OpenClaw はバンドルされた Codex Plugin をインストールまたは修復します。
      </Step>
      <Step title="Codex 認証が利用可能であることを確認する">
        ```bash
        openclaw models list --provider openai
        ```

        Gateway の起動後、チャットで `/codex status` または `/codex models` を
        送信して、ネイティブ app-server ランタイムを確認します。
      </Step>
    </Steps>

    ### ルートの概要

    | モデル参照                | ランタイムポリシーまたはルートの詳細                                 | ルート                                                    | 認証                                               |
    | ------------------------ | ------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------- |
    | `openai/gpt-5.6-sol`     | 未設定/`auto`、公式の完全一致 HTTPS ネイティブルート、リクエストによる上書きなし | Codex が選択される場合あり                                    | Codex サインイン、または順序付けされた `openai` 認証プロファイル |
    | `openai/gpt-5.6-terra`   | 未設定/`auto`、公式の完全一致 HTTPS ネイティブルート、リクエストによる上書きなし | Codex が選択される場合あり                                    | カタログで Terra が公開されている場合の Codex サインイン       |
    | `openai/gpt-5.6-luna`    | 未設定/`auto`、公式の完全一致 HTTPS ネイティブルート、リクエストによる上書きなし | Codex が選択される場合あり                                    | カタログで Luna が公開されている場合の Codex サインイン        |
    | `openai/gpt-5.6-sol`     | プロバイダー/モデル `agentRuntime.id: "openclaw"`                  | OpenClaw 組み込みランタイム、内部 Codex 認証トランスポート | 選択された `openai` OAuth プロファイル                    |
    | `openai/gpt-5.5`         | 明示的なプロバイダー/モデル `agentRuntime.id`                     | 選択されたエージェントランタイム                                   | 選択された OpenAI 認証プロファイル                       |
    | `openai/*`               | 明示的に指定された Completions、カスタム、またはリクエストによる上書き | OpenClaw 組み込みランタイム                                | 認証情報の要件は引き続きルート固有      |
    | `openai/*`               | 平文の公式 HTTP エンドポイント                  | 拒否                                                 | 認証情報は送信されない                              |
    | レガシー Codex GPT-5.5 参照 | doctor により修復                                            | `openai/gpt-5.5` に書き換え                            | 移行された OpenAI OAuth プロファイル                      |
    | `codex-cli/gpt-5.5`      | doctor により修復                                            | `openai/gpt-5.5` に書き換え                            | Codex app-server 認証                              |

    <Warning>
    サブスクリプションを利用する新規セットアップでは、正確な `openai/gpt-5.6-sol` を使用します。
    ネイティブ Codex カタログには、正確な Terra または Luna の参照も表示される場合があります。
    アカウントで GPT-5.6 が利用できない場合は、`openai/gpt-5.5` を明示的に選択してください。古い
    Codex GPT 参照は従来の OpenClaw ルートであり、ネイティブ Codex ランタイムの
    パスではありません。既存の明示的な GPT-5.5 の選択をアップグレードせずに移行するには、
    `openclaw doctor --fix` を実行してください。`gpt-5.3-codex-spark` は、Codex サブスクリプション
    カタログにそれが掲載されているアカウントに引き続き限定されます。これに対する OpenAI
    API キーおよび Azure の直接参照は、引き続き非表示になります。
    </Warning>

    <Note>
    新しい設定では、OpenAI エージェントの認証順序を `auth.order.openai` に配置してください。
    doctor は、古い従来の Codex 認証順序エントリを移行します。
    </Note>

    ### 設定例

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    API キーによるバックアップを使用する場合は、選択したモデルを `openai/*` に保持し、
    認証順序を `openai` に配置してください。OpenClaw は Codex ハーネスを維持したまま、
    まずサブスクリプションを試し、次に API キーを試します。

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    オンボーディングでは、`~/.codex` から OAuth 情報をインポートしなくなりました。
    ブラウザー OAuth（デフォルト）または前述のデバイスコードフローでサインインしてください。
    OpenClaw は、生成された認証情報を独自のエージェント認証ストアで管理します。
    </Note>

    ### Codex OAuth ルーティングの確認と復旧

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    特定のエージェントの場合は、`--agent <id>` を追加します。

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    古い設定に従来の Codex GPT 参照が残っている場合、または明示的なランタイム設定なしに
    古い OpenAI ランタイムセッションの固定指定が残っている場合は、修復します。

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    `models auth list --provider openai` に使用可能なプロファイルが表示されない場合は、再度
    サインインします。

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    同じエージェントで複数の Codex OAuth ログインを使用するには `--profile-id` を使用し、
    認証順序または `/model ...@<profileId>` で制御します。

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    プロファイル順序に依存する前に、`openclaw doctor --fix` を実行して、古い従来の OpenAI Codex
    プレフィックスのプロファイル ID と順序エントリを移行してください。

    ### ステータスインジケーター

    チャットの `/status` は、現在のセッションでどのモデルランタイムが有効かを表示します。
    バンドルされた Codex app-server ハーネスは、対象となる暗黙的なルートまたは明示的な
    プロバイダー／モデルランタイムポリシーによって選択されると、
    `Runtime: OpenAI Codex` として表示されます。

    ### Doctor の警告

    従来の Codex モデル参照または古い OpenAI ランタイムの固定指定が設定やセッション状態に
    残っている場合、OpenClaw が明示的に設定されていない限り、`openclaw doctor --fix` はそれらを
    Codex ランタイムを使用する `openai/*` に書き換えます。

    ### コンテキストウィンドウのデフォルトと長いコンテキストのオプトイン

    OpenClaw は、ネイティブモデルの容量と有効なランタイム予算を
    別々の値として扱います。

    - `contextWindow` は、プロバイダーのモデルウィンドウ全体を宣言します。
    - `contextTokens` は、そのウィンドウのうち OpenClaw が有効な入力に使用する量を制限します。

    ChatGPT/Codex OAuth は、現在の Codex アカウントカタログに従います。現在の
    カタログでは通常、GPT-5.6 に対して `272000` トークンの有効ウィンドウが掲載されています。
    API キーを直接使用する GPT-5.5 および GPT-5.6 モデルも、Platform API がより大きなネイティブ
    ウィンドウを公開しているにもかかわらず、デフォルトで `272000`
    `contextTokens` になります。これにより、通常のレイテンシー、品質、コストの特性が
    認証モード間で一貫します。設定された `agents.defaults.contextTokens` 値によって
    この予算をさらに下げることはできますが、モデルを設定済みの
    `contextTokens` 上限より高くすることはできません。

    API キーを直接使用する GPT-5.5 および GPT-5.6 について、OpenAI は `1050000`
    トークンのプロバイダーウィンドウと、`128000` の最大出力トークンを文書化しています。
    出力許容量をすべて予約すると、入力用に `922000` トークンが残ります。これは算出された
    運用予算であり、プロバイダーが別途公開している入力上限ではありません。公式の
    [モデル比較](https://developers.openai.com/api/docs/models/compare)および
    [GPT-5.5 モデルページ](https://developers.openai.com/api/docs/models/gpt-5.5)を参照してください。
    次の例では、1 つの Terra モデルでその許容量をオプトインし、
    有効トークンが `700000` に達した時点で Compaction を行うよう OpenAI に要求します。

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    この例で `agentRuntime.id: "openclaw"` を指定しているのは意図的です。これにより、
    組み込みの OpenClaw Responses パスが、前述のモデルメタデータとサーバー側の
    Compaction 設定を使用していることを確認できます。一方、ネイティブ Codex ハーネスのスレッドは、
    Codex 設定内で独自のコンテキスト予算を管理します。詳細は
    [Codex ハーネスの長いコンテキスト](/ja-JP/plugins/codex-harness#direct-api-long-context)を参照してください。

    <Warning>
    GPT-5.5 または GPT-5.6 のリクエストが `272000` 入力トークンを超えると、
    OpenAI はより高い長コンテキスト料金を適用します。条件を満たすリクエスト全体に対し、
    入力は 2 倍、出力は 1.5 倍の料金が請求されます。大きなプロンプトはターンをまたいで
    再送または Compaction されるため、表示される応答が短い場合でも、オプトインしたセッションの
    コストはデフォルトより大幅に高くなる可能性があります。
    [OpenAI API の料金](https://developers.openai.com/api/docs/pricing)を参照してください。
    アカウントのアクセス権、実際の上限、請求については、引き続き API が正式な情報源です。
    </Warning>

    ### カタログの復旧

    OpenClaw は、`gpt-5.5` が存在する場合、その上流 Codex カタログメタデータを使用します。
    アカウントが認証済みであるにもかかわらず、ライブ Codex 検出で `gpt-5.5` の行が
    欠落している場合、OpenClaw はその OAuth モデル行を生成し、Cron、サブエージェント、
    および設定済みのデフォルトモデルによる実行が `Unknown model` で失敗しないようにします。

  </Tab>
</Tabs>

## ネイティブ Codex app-server の認証

対象となる正確な公式 HTTPS ルートによって暗黙的に選択された場合、またはプロバイダー／モデルの
`agentRuntime.id: "codex"` によって明示的に選択された場合、ネイティブ Codex app-server ハーネスは
`openai/*` モデル参照を使用します。その認証は引き続きアカウントベースです。
OpenClaw は次の順序で認証を選択します。

1. エージェント用に順序付けられた OpenAI 認証プロファイル。できるだけ
   `auth.order.openai` の下に配置してください。古い従来の Codex 認証プロファイル ID と認証順序を
   移行するには、`openclaw doctor --fix` を実行します。
2. ローカル Codex CLI の ChatGPT サインインなど、app-server の既存アカウント。
   デフォルトの分離されたエージェントホームでは、OpenClaw はそのネイティブ CLI アカウントを
   ログイン RPC を介して app-server に橋渡しします。CLI の設定、plugins、スレッドストアは共有しません。
3. ローカル stdio app-server の起動時のみ、かつ app-server がアカウントなしと
   報告した場合のみ、`CODEX_API_KEY`、続いて `OPENAI_API_KEY`。

Gateway プロセスに、OpenAI モデルまたは埋め込みを直接利用するための `OPENAI_API_KEY` も
設定されているという理由だけで、ローカルの ChatGPT/Codex サブスクリプションによる
サインインが置き換えられることはありません。環境 API キーのフォールバックは、ローカル stdio の
アカウントなしパスにのみ適用され、WebSocket app-server 接続経由で送信されることはありません。
サブスクリプション形式の Codex プロファイルが選択されている場合、OpenClaw は
`CODEX_API_KEY` と `OPENAI_API_KEY` を、生成された stdio app-server 子プロセスに
渡さず、代わりに選択された認証情報を app-server のログイン RPC 経由で送信します。

そのサブスクリプションプロファイルが Codex の使用上限によってブロックされると、OpenClaw は
Codex が通知したリセット時刻までプロファイルをブロック済みとしてマークし、選択したモデルを
変更したり Codex ハーネスから外れたりすることなく、認証順序に従って次の
`openai:*` プロファイルに切り替えます。リセット時刻を過ぎると、
サブスクリプションプロファイルは再び使用可能になります。

## 画像生成

バンドルされた `openai` plugin は、`image_generate` ツールを通じて
画像生成を登録します。同じ `openai/gpt-image-2` モデル参照を通じて、OpenAI API キーと
Codex OAuth の両方による画像生成をサポートします。

| 機能                      | OpenAI API キー                    | Codex OAuth                          |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| モデル参照                | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| 認証                      | `OPENAI_API_KEY`                   | OpenAI Codex OAuth サインイン         |
| 転送方式                  | OpenAI Images API                  | Codex Responses バックエンド          |
| リクエストあたりの最大画像数 | 4                                  | 4                                    |
| 編集モード                | 有効（参照画像は最大 5 枚）         | 有効（参照画像は最大 5 枚）           |
| サイズの上書き            | 2K/4K サイズを含めてサポート        | 2K/4K サイズを含めてサポート          |
| アスペクト比／解像度       | OpenAI Images API には転送されない  | 安全な場合はサポート対象サイズに対応付け |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
共通ツールパラメーター、プロバイダーの選択、フェイルオーバー動作については、
[画像生成](/ja-JP/tools/image-generation)を参照してください。
</Note>

`gpt-image-2` は、OpenAI のテキストからの画像生成および画像編集におけるデフォルトです。
`gpt-image-1.5`、`gpt-image-1`、`gpt-image-1-mini` は、明示的なモデルの上書きとして
引き続き使用できます。背景が透明な PNG/WebP 出力には `openai/gpt-image-1.5` を使用してください。
現在の `gpt-image-2` API は `background: "transparent"` を拒否します。

背景を透明にするリクエストでは、`model: "openai/gpt-image-1.5"`、`outputFormat: "png"`、または
`"webp"` と、`background: "transparent"` を指定して `image_generate` を呼び出します。
古い `openai.background` プロバイダーオプションも引き続き使用できます。OpenClaw はさらに、
デフォルトの `openai/gpt-image-2` 透明リクエストを `gpt-image-1.5` に書き換えることで、
公開 OpenAI および OpenAI Codex OAuth ルートを保護します。Azure およびカスタムの
OpenAI 互換エンドポイントでは、設定済みのデプロイメント名／モデル名を維持します。

同じ設定は、ヘッドレス CLI 実行でも公開されています。

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "透明な背景上のシンプルな赤い円形ステッカー" \
  --json
```

入力ファイルから開始する場合は、`openclaw infer image edit` で同じ `--output-format` および
`--background` フラグを使用します。
`--openai-background` は、OpenAI 固有のエイリアスとして引き続き利用できます。
OpenAI Images の品質とコストを制御するには、`--quality low|medium|high|auto` を使用します。
`image generate` または `image edit` から OpenAI のモデレーションヒントを
渡すには、`--openai-moderation low|auto` を使用します。

ChatGPT/Codex OAuth インストールでは、同じ `openai/gpt-image-2` ref を維持します。
`openai` OAuth プロファイルが設定されている場合、OpenClaw は保存されている OAuth
アクセストークンを解決し、Codex Responses バックエンドを通じて画像リクエストを送信します。
最初に `OPENAI_API_KEY` を試したり、暗黙的に API キーへフォールバックしたりすることはありません。
代わりに OpenAI Images API の直接ルートを使用する場合は、API キー、カスタムベース
URL、または Azure エンドポイントを使用して `models.providers.openai` を明示的に設定します。
そのカスタム画像エンドポイントが信頼済みの LAN/プライベートアドレス上にある場合は、
`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` も設定します。このオプトインがない限り、OpenClaw は
プライベート/内部の OpenAI 互換画像エンドポイントをブロックしたままにします。

生成:

```
/tool image_generate model=openai/gpt-image-2 prompt="macOS 上の OpenClaw 用の洗練されたローンチポスター" size=3840x2160 count=1
```

透明 PNG を生成:

```
/tool image_generate model=openai/gpt-image-1.5 prompt="透明な背景上のシンプルな赤い円形ステッカー" outputFormat=png background=transparent
```

編集:

```
/tool image_generate model=openai/gpt-image-2 prompt="オブジェクトの形状を維持し、素材を半透明のガラスに変更" image=/path/to/reference.png size=1024x1536
```

## 動画生成

同梱の `openai` Plugin は、`video_generate` ツールを通じて
動画生成を登録します。

| 機能             | 値                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------- |
| デフォルトモデル | `openai/sora-2`                                                                    |
| モード           | テキストから動画、画像から動画、単一動画の編集                                     |
| 参照入力         | 画像 1 枚または動画 1 本                                                           |
| サイズの上書き   | テキストから動画と画像から動画でサポート                                           |
| アスペクト比     | 未加工のまま転送せず、最も近いサポート対象サイズに変換                             |
| その他の上書き   | `resolution`、`audio`、`watermark` はサポートされず、ツール警告とともに破棄されます |

OpenAI の画像から動画へのリクエストでは、画像
`input_reference` とともに `POST /v1/videos` を使用します。単一動画の編集では、
アップロードした動画を `video` フィールドに指定して `POST /v1/videos/edits` を使用します。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
共有ツールのパラメーター、プロバイダーの選択、フェイルオーバーの動作については、
[動画生成](/ja-JP/tools/video-generation)を参照してください。

OpenAI プロバイダーは `supportsSize` を宣言しますが、`supportsAspectRatio` や
`supportsResolution` は宣言しません。OpenClaw の共有正規化レイヤーは、リクエストされた
`aspectRatio` を、リクエストがプロバイダーに到達する前に最も近い OpenAI
`size` に変換するため、通常はアスペクト比のリクエストも機能します。
`resolution` にはサイズのフォールバックがないため破棄され、
`Ignored unsupported overrides for openai/<model>: resolution=<value>` として呼び出し元に通知されます。
</Note>

## GPT-5 プロンプトへの追加

OpenClaw は、`openai` プロバイダー上の GPT-5 ファミリーモデルに対して、
共有 GPT-5 プロンプト追加要素を加えます（`openai/*` に正規化される、
修復前の従来の Codex ref を含む）。OpenRouter や opencode ルートなど、
GPT-5 ファミリーのモデル ID も提供する他のプロバイダーには、このオーバーレイは適用されません。
これはモデル ID だけではなく、プロバイダー ID `openai` によって制限されます。
古い GPT-4.x モデルには適用されません。

ネイティブ Codex app-server ハーネスは、developer instructions を通じて
ペルソナ/ツール規律の動作コントラクトや、親しみやすい対話スタイルのオーバーレイを受け取りません。
ネイティブ Codex は Codex が所有するベース、モデル、プロジェクトドキュメントの動作を維持し、
OpenClaw はネイティブスレッドで Codex の組み込みパーソナリティを無効にするため、
エージェントワークスペースのパーソナリティファイルが引き続き基準となります。
OpenClaw がネイティブ Codex スレッドに追加するのは、実行時コンテキストのみです。
これには、チャネル配信、OpenClaw 動的ツール、ACP 委任、ワークスペースコンテキスト、
OpenClaw Skills が含まれます。同じ追加要素に含まれる Heartbeat ガイダンステキストだけは例外です。
ネイティブ Codex の Heartbeat ターンにはこのテキストが適用されますが、
共有プロンプト追加フックではなく、専用のコラボレーション指示として注入されます。

GPT-5 追加要素は、一致する OpenClaw 組み立て済みプロンプトに対して、
ペルソナの持続性、実行の安全性、ツール規律、出力形式、完了チェック、検証に関する
タグ付き動作コントラクトを追加します。チャネル固有の返信とサイレントメッセージの動作は、
共有 OpenClaw システムプロンプトと送信配信ポリシーに引き続き含まれます。
親しみやすい対話スタイルのレイヤーは独立しており、設定可能です。

| 値                     | 効果                                           |
| ---------------------- | ---------------------------------------------- |
| `"friendly"`（デフォルト） | 親しみやすい対話スタイルのレイヤーを有効化     |
| `"on"`                 | `"friendly"` のエイリアス                      |
| `"off"`                | 親しみやすいスタイルのレイヤーのみを無効化     |

<Tabs>
  <Tab title="設定">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
実行時には値の大文字と小文字は区別されないため、`"Off"` と
`"off"` はどちらも親しみやすいスタイルのレイヤーを無効にします。
</Tip>

<Note>
共有の `agents.defaults.promptOverlays.gpt5.personality` 設定が未設定の場合、従来の
`plugins.entries.openai.config.personality` は互換性フォールバックとして引き続き読み取られます。
</Note>

## 音声とスピーチ

<AccordionGroup>
  <Accordion title="音声合成（TTS）">
    同梱の `openai` Plugin は、`tts` サーフェス向けに
    音声合成を登録します。

    | 設定         | 設定パス                                              | デフォルト                            |
    | ------------- | --------------------------------------------------------- | ----------------------------------- |
    | モデル       | `tts.providers.openai.model`                  | `gpt-4o-mini-tts`                |
    | 音声         | `tts.providers.openai.speakerVoice`           | `coral`                          |
    | 速度         | `tts.providers.openai.speed`                  | （未設定）                          |
    | 指示         | `tts.providers.openai.instructions`           | （未設定、`gpt-4o-mini-tts` のみ）  |
    | 形式         | `tts.providers.openai.responseFormat`         | ボイスメモは `opus`、ファイルは `mp3` |
    | API キー     | `tts.providers.openai.apiKey`                 | `OPENAI_API_KEY` にフォールバック   |
    | ベース URL   | `tts.providers.openai.baseUrl`                | `https://api.openai.com/v1`      |
    | 追加本文     | `tts.providers.openai.extraBody` / `extra_body` | （未設定）                        |

    利用可能なモデル: `gpt-4o-mini-tts`、`tts-1`、`tts-1-hd`。利用可能な音声:
    `alloy`、`ash`、`ballad`、`cedar`、`coral`、`echo`、`fable`、`juniper`、
    `marin`、`onyx`、`nova`、`sage`、`shimmer`、`verse`。

    `extraBody` は、OpenClaw が生成したフィールドの後に
    `/audio/speech` リクエスト JSON へマージされるため、`lang` などの
    追加キーを必要とする OpenAI 互換エンドポイントに使用します。プロトタイプキーは無視されます。

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    チャット API エンドポイントに影響を与えずに TTS のベース URL を上書きするには、
    `OPENAI_TTS_BASE_URL` を設定します。OpenAI TTS と Realtime 音声はどちらも
    OpenAI Platform API キーを通じて設定されます。OAuth のみのインストールでも
    Codex ベースのチャットモデルは使用できますが、OpenAI のライブ音声応答は使用できません。
    </Note>

  </Accordion>

  <Accordion title="音声テキスト変換">
    同梱の `openai` Plugin は、OpenClaw のメディア理解文字起こしサーフェスを通じて、
    バッチ音声テキスト変換を登録します。

    - デフォルトモデル: `gpt-4o-transcribe`
    - エンドポイント: OpenAI REST `/v1/audio/transcriptions`
    - 入力パス: マルチパート音声ファイルのアップロード
    - Discord ボイスチャネルのセグメントやチャネル音声添付ファイルを含め、
      受信音声の文字起こしが `tools.media.audio` を読み取るすべての場所で使用

    受信音声の文字起こしに OpenAI を強制するには:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    言語とプロンプトのヒントは、共有音声メディア設定または呼び出しごとの文字起こしリクエストで
    指定された場合、OpenAI に転送されます。

  </Accordion>

  <Accordion title="Realtime 文字起こし">
    同梱の `openai` Plugin は、Voice Call Plugin 向けに
    Realtime 文字起こしを登録します。

    | 設定             | 設定パス                                                              | デフォルト |
    | ----------------- | ----------------------------------------------------------------------- | --------- |
    | モデル           | `plugins.entries.voice-call.config.streaming.providers.openai.model` | `gpt-4o-transcribe` |
    | 言語             | `...openai.language`                                                 | （未設定） |
    | プロンプト       | `...openai.prompt`                                                   | （未設定） |
    | 無音時間         | `...openai.silenceDurationMs`                                        | `800`   |
    | VAD しきい値     | `...openai.vadThreshold`                                             | `0.5`   |
    | 認証             | `...openai.apiKey`、`OPENAI_API_KEY`、または `openai` API キープロファイル    | Platform API キーが必要 |

    <Note>
    G.711 u-law（`g711_ulaw` / `audio/pcmu`）音声で
    `wss://api.openai.com/v1/realtime` への WebSocket 接続を使用します。`openai` API キー
    プロファイルの場合、Gateway は WebSocket を開く前に一時的な Realtime 文字起こし
    クライアントシークレットを発行します。このストリーミングプロバイダーは Voice Call の
    Realtime 文字起こしパス向けです。現在 Discord ボイスは短いセグメントを録音し、
    代わりにバッチ `tools.media.audio` 文字起こしパスを使用します。
    </Note>

  </Accordion>

  <Accordion title="Realtime 音声">
    同梱の `openai` Plugin は、Voice Call
    Plugin 向けに Realtime 音声を登録します。

    | 設定                                    | 設定パス                                                                     | デフォルト                         |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------- |
    | モデル                                  | `plugins.entries.voice-call.config.realtime.providers.openai.model`                                                           | `gpt-realtime-2.1`                 |
    | 音声                                    | `...openai.voice`                                                           | `alloy`                 |
    | Temperature（Azure デプロイメントブリッジ） | `...openai.temperature`                                                           | `0.8`                 |
    | VAD しきい値                            | `...openai.vadThreshold`                                                           | `0.5`                 |
    | 無音時間                                | `...openai.silenceDurationMs`                                                           | `500`                 |
    | プレフィックスパディング                | `...openai.prefixPaddingMs`                                                           | `300`                 |
    | 推論エフォート                          | `...openai.reasoningEffort`                                                           | （未設定）                         |
    | 認証                                    | `openai` API キープロファイル、`...openai.apiKey`、または `OPENAI_API_KEY` | OpenAI Platform API キーが必要 |

    `gpt-realtime-2.1` で使用可能な組み込み Realtime 音声：`alloy`、`ash`、
    `ballad`、`coral`、`echo`、`sage`、`shimmer`、`verse`、`marin`、`cedar`。
    OpenAI は、最高の Realtime 品質を得るために `marin` と `cedar` を推奨しています。これは
    上記のテキスト読み上げ音声とは別のセットです。`fable`、`nova`、`onyx` などの TTS 専用音声は、
    Realtime セッションでは使用できません。より小規模で低コストな Realtime 2.1 バリアントを使用する場合は、
    モデルを明示的に `gpt-realtime-2.1-mini` に設定します。

    <Note>
    **GPT-Live（近日提供予定）。** OpenAI の全二重 `gpt-live-1` および
    `gpt-live-1-mini` モデルは、2026 年 7 月に ChatGPT 音声モードを置き換えました。
    開発者 API は早期アクセス対象の組織へ段階的に提供されています。OpenClaw は
    このモデルファミリーを認識しますが、まだ実行しません。GPT-Live セッションは
    WebRTC 専用で、ターンテイキングを独自に管理し（VAD なし）、OpenClaw の Realtime トランスポートが
    まだ実装していないハンドオフイベントプロトコルを介してエージェントの作業を委譲します。
    `gpt-live-*` モデルを設定すると、エージェントにアクセスできないまま音声へ暗黙的に接続するのではなく、
    WebSocket ブリッジと Talk ブラウザセッションの両方に関するガイダンスを示して安全側で失敗します。
    早期アクセス期間中は、API アクセスも OpenAI の組織ごとに制限されます。
    GPT-Live のサポートが実装されるまでは、`gpt-realtime-2.1`（デフォルト）を使用してください。
    </Note>

    <Note>
    バックエンドの OpenAI Realtime ブリッジは GA Realtime WebSocket セッション形式を使用します。
    この形式では `session.temperature` は使用できません。Azure OpenAI
    デプロイメントは引き続き `azureEndpoint` および `azureDeployment` を介して利用でき、
    デプロイメント互換のセッション形式（`temperature` を含む）を維持します。
    双方向のツール呼び出しと G.711 u-law 音声をサポートします。
    </Note>

    <Note>
    Realtime 音声はセッションの作成時に選択されます。OpenAI ではほとんどの
    セッションフィールドを後から変更できますが、そのセッションでモデルが音声を出力した後は、
    音声を変更できません。OpenClaw は現在、組み込み Realtime 音声 ID を文字列として公開しています。
    </Note>

    <Note>
    Control UI の Talk は、Gateway が発行する一時的なクライアントシークレットと、
    OpenAI Realtime API に対するブラウザからの直接的な WebRTC SDP 交換を使用して、
    OpenAI のブラウザ Realtime セッションを利用します。Gateway は、選択された
    `openai` の認証情報を使用して、そのクライアントシークレットを発行します。
    設定済みのキー、API キープロファイル、および `OPENAI_API_KEY` が優先され、
    `openai` OAuth プロファイルまたは外部 Codex ログインがフォールバックになります。
    Gateway リレーと Voice Call バックエンドの Realtime WebSocket ブリッジは、
    ネイティブ OpenAI エンドポイントに対して同じ認証情報の優先順位を使用します。
    メンテナー向けのライブ検証は
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`
    で利用できます。OpenAI 側の処理では、シークレットをログに記録せずに、
    バックエンド WebSocket ブリッジとブラウザ WebRTC SDP 交換の両方を検証します。
    Google の認証情報なしでこの 2 つの処理を実行するには、`--openai-only` を渡します。
    </Note>

  </Accordion>
</AccordionGroup>

## Azure OpenAI エンドポイント

バンドルされている `openai` プロバイダーは、ベース URL を上書きすることで、
画像生成用の Azure OpenAI リソースを対象にできます。画像生成パスでは、OpenClaw は
`models.providers.openai.baseUrl` 上の Azure ホスト名を検出し、
Azure のリクエスト形式へ自動的に切り替えます。

<Note>
Realtime 音声は別の設定パス
（`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`）
を使用し、`models.providers.openai.baseUrl` の影響を受けません。Azure の設定については、
[音声とスピーチ](#voice-and-speech)の **Realtime 音声** アコーディオンを参照してください。
</Note>

次の場合は Azure OpenAI を使用します。

- Azure OpenAI のサブスクリプション、クォータ、またはエンタープライズ契約をすでに保有している
- Azure が提供する地域別データレジデンシーまたはコンプライアンス制御が必要である
- 既存の Azure テナント内にトラフィックを維持したい

### 設定

バンドルされている `openai` プロバイダーを介して Azure 画像生成を使用するには、
`models.providers.openai.baseUrl` を Azure リソースに向け、`apiKey` を
Azure OpenAI キー（OpenAI Platform キーではありません）に設定します。

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw は、Azure 画像生成ルートで次の Azure ホストサフィックスを認識します。

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

認識された Azure ホストに対する画像生成リクエストでは、OpenClaw は次の処理を行います。

- `Authorization: Bearer` の代わりに `api-key` ヘッダーを送信する
- デプロイメントスコープのパス（`/openai/deployments/{deployment}/...`）を使用する
- 各リクエストに `?api-version=...` を追加する
- Azure 画像生成呼び出しでは、デフォルトのリクエストタイムアウトとして 600s を使用する。
  呼び出しごとの `timeoutMs` 値は、引き続きこのデフォルトを上書きします。

その他のベース URL（公開 OpenAI、OpenAI 互換プロキシ）では、標準の
OpenAI 画像リクエスト形式が維持されます。

<Note>
`openai` プロバイダーの画像生成パスで Azure ルーティングを使用するには、
OpenClaw 2026.4.22 以降が必要です。それ以前のバージョンでは、カスタム
`openai.baseUrl` が公開 OpenAI エンドポイントと同様に扱われるため、Azure の画像
デプロイメントに対するリクエストは失敗します。
</Note>

### API バージョン

Azure 画像生成パスに特定の Azure プレビュー版または GA 版を固定するには、
`AZURE_OPENAI_API_VERSION` を設定します。

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

変数が未設定の場合、デフォルトは `2024-12-01-preview` です。

### モデル名はデプロイメント名

Azure OpenAI では、モデルがデプロイメントに関連付けられます。バンドルされている
`openai` プロバイダーを介してルーティングされる Azure 画像生成リクエストでは、
OpenClaw の `model` フィールドに、公開 OpenAI モデル ID ではなく、
Azure ポータルで設定した **Azure デプロイメント名** を指定する必要があります。

`gpt-image-2` を提供する `gpt-image-2-prod` というデプロイメントを作成した場合：

```
/tool image_generate model=openai/gpt-image-2-prod prompt="すっきりしたポスター" size=1024x1024 count=1
```

同じデプロイメント名の規則が、バンドルされている `openai` プロバイダーを介して
ルーティングされるすべての画像生成呼び出しに適用されます。

### 利用可能なリージョン

Azure の画像生成は現在、一部のリージョンでのみ利用できます
（例：`eastus2`、`swedencentral`、`polandcentral`、`westus3`、
`uaenorth`）。デプロイメントを作成する前に Microsoft の最新リージョン一覧を確認し、
対象のモデルが使用するリージョンで提供されていることを確認してください。

### パラメーターの相違点

Azure OpenAI と公開 OpenAI では、常に同じ画像パラメーターを使用できるとは限りません。
Azure は、公開 OpenAI で使用できるオプション（たとえば `gpt-image-2` における特定の
`background` 値）を拒否したり、特定のモデルバージョンでのみ公開したりする場合があります。
こうした相違は OpenClaw ではなく、Azure と基盤モデルに起因します。Azure リクエストが
検証エラーで失敗した場合は、Azure ポータルで、使用しているデプロイメントと API バージョンが
サポートするパラメーターセットを確認してください。

<Note>
Azure OpenAI はネイティブトランスポートと互換動作を使用しますが、
OpenClaw の非表示の帰属ヘッダーは受け取りません。詳細については、
[高度な設定](#advanced-configuration)の **ネイティブと OpenAI 互換ルートの比較**
アコーディオンを参照してください。

画像生成以外の Azure 上のチャットまたは Responses トラフィックには、
オンボーディングフローまたは専用の Azure プロバイダー設定を使用してください。
`openai.baseUrl` だけでは Azure の API／認証形式は適用されません。
別の `azure-openai-responses/*` プロバイダーも存在します。下記のサーバー側 Compaction
アコーディオンを参照してください。
</Note>

## 高度な設定

以下のモデルごとの `params` の例は、OpenClaw の組み込みプロバイダーリクエストを
形成します。これらの設定は作成者が指定するリクエスト動作であるため、通常は対象となる
`auto` ルートでも、Codex が暗黙的に選択されるのではなく OpenClaw 上に維持されます。
ネイティブ Codex app-server ハーネスは、独自のトランスポートとリクエスト設定を所有します。
有効なルートが Codex 互換として宣言されていない場合、明示的な `agentRuntime.id: "codex"` は
安全側で失敗します。

<AccordionGroup>
  <Accordion title="トランスポート（WebSocket と SSE）">
    OpenClaw は `openai/*` に対して、SSE フォールバックを伴う WebSocket 優先
    （`"auto"`）を使用します。

    `"auto"` モードでは、OpenClaw は次の処理を行います。
    - 初期の WebSocket 障害を 1 回再試行してから SSE にフォールバックする
    - 障害後に WebSocket を 60 秒間縮退状態としてマークし、
      クールダウン中は SSE を使用する
    - 再試行と再接続のために、安定したセッションおよびターン識別ヘッダーを付加する
    - トランスポートのバリアント間で使用量カウンター（`input_tokens`／`prompt_tokens`）を
      正規化する

    | 値                         | 動作                                      |
    | -------------------------- | ----------------------------------------- |
    | `"auto"`（デフォルト） | WebSocket 優先、SSE フォールバック        |
    | `"sse"`         | SSE のみを強制                            |
    | `"websocket"`         | WebSocket のみを強制                      |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    関連する OpenAI ドキュメント：
    - [WebSocket を使用した Realtime API](https://platform.openai.com/docs/guides/realtime-websocket)
    - [API レスポンスのストリーミング（SSE）](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="高速モード">
    OpenClaw は `openai/*` に対して共有の高速モード切り替えを公開します。

    - **チャット／UI：** `/fast status|auto|on|off`
    - **設定：** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    有効にすると、OpenClaw は高速モードを OpenAI の優先処理
    （`service_tier = "priority"`）にマッピングします。既存の `service_tier` 値は
    保持され、高速モードは `reasoning` または
    `text.verbosity` を書き換えません。`fastMode: "auto"` は、自動カットオフまでは
    新しいモデル呼び出しを高速モードで開始し、その後の再試行、フォールバック、ツール結果、
    または継続呼び出しは高速モードなしで開始します。カットオフのデフォルトは 60 秒です。
    変更するには、アクティブなモデルに `params.fastAutoOnSeconds` を設定します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    セッションの上書きは設定より優先されます。Sessions UI でセッションの上書きを解除すると、
    セッションは設定済みのデフォルトに戻ります。
    </Note>

  </Accordion>

  <Accordion title="優先処理 (service_tier)">
    OpenAI の API は、`service_tier` を介して優先処理を提供します。OpenClaw では
    モデルごとに設定します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    サポートされる値: `auto`、`default`、`flex`、`priority`。

    <Warning>
    `serviceTier` は、ネイティブ OpenAI エンドポイント
    (`api.openai.com`) およびネイティブ Codex エンドポイント (`chatgpt.com/backend-api`) にのみ転送されます。
    いずれかのプロバイダーをプロキシ経由でルーティングする場合、OpenClaw は
    `service_tier` を変更しません。
    </Warning>

  </Accordion>

  <Accordion title="サーバー側 Compaction (Responses API)">
    OpenAI Responses モデルに直接接続する場合 (`api.openai.com` 上の `openai/*`)、
    OpenAI Plugin の OpenClaw ストリームラッパーはサーバー側
    Compaction を自動的に有効化します。

    - `store: true` を強制します (モデル互換設定で `supportsStore: false` が設定されている場合を除く)
    - `context_management: [{ type: "compaction", compact_threshold: ... }]` を挿入します
    - デフォルトの `compact_threshold`: `contextWindow` の 70% (利用できない場合は
      `80000`)

    これは組み込みの OpenClaw ランタイムパスと、埋め込み実行で使用される OpenAI プロバイダー
    フックに適用されます。ネイティブ Codex app-server ハーネスは Codex を介して
    独自にコンテキストを管理するため、この設定の影響を受けません。

    <Tabs>
      <Tab title="明示的に有効化">
        Azure OpenAI Responses などの互換エンドポイントに役立ちます。

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="カスタムしきい値">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="無効化">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` は `context_management` の挿入のみを制御します。
    OpenAI Responses モデルへの直接接続では、互換設定で `supportsStore: false` が設定されていない限り、
    引き続き `store: true` が強制されます。
    </Note>

  </Accordion>

  <Accordion title="Strict-agentic GPT モード">
    OpenClaw の埋め込みランタイムを介して実行される `openai` プロバイダーの
    GPT-5 ファミリーモデルでは、OpenClaw はすでに `strict-agentic` と呼ばれる
    より厳格な実行コントラクトをデフォルトで使用します。解決されたプロバイダーが
    `openai` で、モデル ID が GPT-5 ファミリーに一致する場合、設定で
    明示的にオプトアウトしない限り自動的に有効化されます。

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    `"strict-agentic"` を明示的に設定しても、サポートされるレーンでは何も変わらず
    (すでにデフォルトです)、サポートされないプロバイダーとモデルの組み合わせでは機能しません。

    `strict-agentic` が有効な場合、OpenClaw は以下を行います。
    - 大規模な作業では `update_plan` を自動的に有効化します
    - 構造的に空、または推論のみのターンを、表示可能な回答を生成する
      継続ターンとして再試行します
    - 選択したハーネスが提供する場合、明示的なハーネス計画イベントを
      使用します

    OpenClaw は、ターンが計画、進捗更新、最終回答のいずれであるかを判断するために、
    アシスタントの文章を分類することはありません。

    <Note>
    このコントラクトは、すべて OpenClaw の埋め込みエージェントランナー内にあります。
    独自のターンと計画の動作を管理するネイティブ Codex app-server ハーネスには
    適用されません。ネイティブ Codex の実行では、実行コントラクトの設定よりも
    ハーネスの選択の方が重要です。
    </Note>

  </Accordion>

  <Accordion title="ネイティブ経路と OpenAI 互換経路">
    OpenClaw は、OpenAI、Codex、Azure OpenAI の直接エンドポイントと、
    汎用の OpenAI 互換 `/v1` プロキシを異なる方法で処理します。

    **ネイティブ経路** (`openai/*`、Azure OpenAI):
    - OpenAI の `none` effort をサポートするモデルでのみ
      `reasoning: { effort: "none" }` を維持します
    - `reasoning.effort: "none"` を拒否するモデルまたはプロキシでは、
      無効化された推論を省略します
    - ツールスキーマのデフォルトを strict モードにします
    - 検証済みのネイティブホストにのみ非表示の帰属ヘッダーを付加します (Azure
      OpenAI はネイティブ経路ですが、これらのヘッダーは付加されません)
    - OpenAI 専用のリクエスト整形 (`service_tier`、`store`、
      推論互換、プロンプトキャッシュのヒント) を維持します

    **プロキシ/互換経路:**
    - より緩やかな互換動作を使用します
    - 非ネイティブの `openai-completions` ペイロードから Completions の
      `store` を削除します
    - OpenAI 互換 Completions プロキシ向けに、高度な
      `params.extra_body`/`params.extraBody` パススルー JSON を受け入れます
    - vLLM などの OpenAI 互換 Completions プロキシ向けに
      `params.chat_template_kwargs` を受け入れます
    - strict ツールスキーマやネイティブ専用ヘッダーを強制しません

  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="画像生成" href="/ja-JP/tools/image-generation" icon="image">
    共通の画像ツールパラメーターとプロバイダーの選択。
  </Card>
  <Card title="動画生成" href="/ja-JP/tools/video-generation" icon="video">
    共通の動画ツールパラメーターとプロバイダーの選択。
  </Card>
  <Card title="OAuth と認証" href="/ja-JP/gateway/authentication" icon="key">
    認証の詳細と認証情報の再利用ルール。
  </Card>
</CardGroup>
