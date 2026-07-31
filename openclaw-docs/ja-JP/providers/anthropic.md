---
read_when:
    - OpenClaw で Anthropic モデルを使用する場合
    - ペアリングされたコンピューター間で Claude CLI または Claude Desktop のセッションを閲覧したい場合
summary: OpenClaw で API キーまたは Claude CLI を介して Anthropic Claude を使用する
title: Anthropic
x-i18n:
    generated_at: "2026-07-26T09:46:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic は **Claude** モデルファミリーを開発しています。OpenClaw は 2 つの認証経路をサポートしています。

- **API キー** - 使用量ベースの課金による Anthropic API への直接アクセス（`anthropic/*` モデル）
- **Claude CLI** - 同じホスト上の既存の Claude Code ログインを再利用

## 使用量とコストの追跡

OpenClaw は利用可能な Anthropic 認証情報を検出し、対応する使用量画面を選択します。

- Claude のサブスクリプション／セットアップ認証情報では、クォータ期間とオプションの追加使用量予算が表示されます。
- `ANTHROPIC_ADMIN_KEY` または `ANTHROPIC_ADMIN_API_KEY` では、プロバイダーから報告された過去 30 日間の組織コストと Messages API 使用量が Control UI の **使用量** に表示されます。これには、日次支出、トークン／キャッシュ合計、上位モデル、コストカテゴリが含まれます。
- Anthropic プロバイダープロファイルに保存された `sk-ant-admin...` 認証情報は、Admin API キーとして自動的に検出されます。

Admin API のコスト履歴は、Anthropic の [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api) から取得されます。これはプロバイダーによる実際の請求額であり、OpenClaw がセッションから算出する推定コストとは別です。

<Warning>
OpenClaw の Claude CLI バックエンドは、インストール済みの Claude Code CLI を
非対話型の出力モード（`claude -p`）で実行します。Anthropic の現在の Claude Code ドキュメントでは、
このモードを Agent SDK／プログラムによる使用として説明しています。Anthropic の 2026 年 6 月 15 日の
サポート更新により、発表済みだった Agent SDK の個別課金への変更は一時停止されました。Claude
Agent SDK、`claude -p`、およびサードパーティ製アプリの使用量は、引き続きログイン中の
サブスクリプションの使用量上限から消費されます。また、以前発表された月次 Agent SDK
クレジットは、Anthropic がそのプランを見直している間は利用できません。

対話型の Claude Code も、引き続きログイン中の Claude プランの上限から消費されます。
API キー認証は直接の従量課金であり、そのプランには依存しません。
長期間稼働する Gateway ホスト、共有オートメーション、および予測可能な本番環境の
支出には、Anthropic API キーを使用してください。

Anthropic の現在のサポート記事により、OpenClaw のリリースなしで
この動作が変更される可能性があります。

- [Claude Code CLI リファレンス](https://code.claude.com/docs/en/cli-usage)
- [Claude プランで Claude Agent SDK を使用する](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [Pro または Max プランで Claude Code を使用する](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [Team または Enterprise プランで Claude Code を使用する](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [Claude Code のコストを管理する](https://code.claude.com/docs/en/costs)

</Warning>

## はじめに

<Tabs>
  <Tab title="API キー">
    **最適な用途：** 標準的な API アクセスと使用量ベースの課金。

    <Steps>
      <Step title="API キーを取得する">
        [Anthropic Console](https://console.anthropic.com/) で API キーを作成します。
      </Step>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard
        # 選択: Anthropic API key
        ```

        または、キーを直接渡します。

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### 設定例

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **最適な用途：** 個別の API キーを使用せずに、既存の Claude CLI ログインを再利用する場合。

    <Steps>
      <Step title="Claude CLI がインストールされ、ログイン済みであることを確認する">
        次のコマンドで確認します。

        ```bash
        claude --version
        ```
      </Step>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard
        # 選択: Claude CLI
        ```

        OpenClaw は既存の Claude CLI 認証情報を検出して再利用します。
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Claude CLI バックエンドのセットアップとランタイムの詳細については、[CLI バックエンド](/ja-JP/gateway/cli-backends)を参照してください。
    </Note>

    <Warning>
    Claude CLI の再利用では、OpenClaw プロセスが Claude CLI のログインと
    同じホスト上で実行される必要があります。Docker インストールでは、コンテナのホームを永続化し、
    そこで Claude Code にログインできます。詳細については、
    [Docker での Claude CLI バックエンド](/ja-JP/install/docker#claude-cli-backend-in-docker)を参照してください。
    [Podman](/ja-JP/install/podman) などのその他のコンテナインストールでは、ホストの
    `~/.claude` がセットアップ環境やランタイムにマウントされません。その場合は Anthropic API キーを使用するか、
    [OpenAI Codex](/ja-JP/providers/openai) など、OpenClaw が管理する OAuth を備えた
    プロバイダーを選択してください。
    </Warning>

    ### セットアップトークンを取得する

    Claude Code がインストールされている任意のマシンで `claude setup-token` を実行します。
    `sk-ant-oat01-` で始まる長期有効トークンが出力されます。

    オンボーディング中に、macOS アプリの **Connect with an API key or token** で
    **Anthropic setup-token** を選択してトークンを貼り付けるか、次のコマンドを使用します。

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### 設定例

    正規の Anthropic モデル参照と CLI ランタイムの上書きを組み合わせることを推奨します。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    互換性のため、従来の `claude-cli/claude-opus-4-7` モデル参照も引き続き機能しますが、
    新しい設定ではプロバイダー／モデルの選択を `anthropic/*` として維持し、
    実行バックエンドをプロバイダー／モデルのランタイムポリシーに配置する必要があります。

    ### 課金と `claude -p`

    OpenClaw は Claude CLI の実行に、Claude Code の非対話型 `claude -p` パスを
    使用します。Anthropic は現在、このパスを Agent SDK／プログラムによる使用として扱っています。

    - Anthropic の 2026 年 6 月 15 日のサポート更新により、以前発表された
      Agent SDK の個別クレジットプランは一時停止されました。
    - サブスクリプションプランの Claude Agent SDK、`claude -p`、およびサードパーティ製アプリの使用量は、
      引き続きログイン中のサブスクリプションの使用量上限から消費されます。
    - Anthropic がそのプランを見直している間、以前発表された月次 Agent SDK クレジットは
      利用できません。
    - Console／API キーによるログインでは従量課金制の API 課金が適用され、
      サブスクリプションの Agent SDK クレジットは付与されません。

    一時停止の通知については Anthropic の [Agent SDK プランに関する
    記事](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)を参照してください。また、サブスクリプションの動作については
    [Pro／Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    および
    [Team／Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
    向けの Claude Code プラン記事を参照してください。

    Anthropic は OpenClaw のリリースなしで、Claude Code の課金やレート制限の動作を
    変更する可能性があります。課金の予測可能性が重要な場合は、`claude auth status`、`/status`、および
    リンク先の Anthropic ドキュメントを確認してください。

    <Tip>
    共有の本番オートメーションでは、Claude CLI ではなく Anthropic API キーを
    使用してください。OpenClaw は、[OpenAI Codex](/ja-JP/providers/openai)、[Qwen Cloud](/ja-JP/providers/qwen)、
    [MiniMax](/ja-JP/providers/minimax)、および [Z.AI / GLM](/ja-JP/providers/zai) の
    サブスクリプション形式のオプションもサポートしています。
    </Tip>

  </Tab>
</Tabs>

## コンピューター間の Claude セッション

同梱の Anthropic Plugin は、通常のセッションサイドバーに **Claude Code** グループを
追加します。行を開くと通常のチャットペインに表示されます。Gateway および接続された Node ホスト上にある、
アーカイブされていない Claude Code セッションを検出します。

- Claude CLI セッションは、有効なプロジェクトインデックスレコードから取得されます。インデックスにない
  トランスクリプトについては、範囲を制限したメタデータフォールバックにより、`~/.claude/projects/` 配下にある、同時実行中でサイドチェーンではない
  対話型（`cli`）およびヘッドレス Agent SDK CLI（`sdk-cli`）セッションが認識されます。
- Claude Desktop セッションでは、そのメタデータが同じ Claude Code セッション ID を指している場合、
  Desktop のタイトル、アクティビティ時刻、アーカイブ状態が使用されます。
- CLI のみのセッションにはアーカイブフラグがないため、トランスクリプトが存在する間は
  表示されたままになります。

検出に追加の OpenClaw 設定は不要です。Anthropic Plugin は
同梱されており、デフォルトで有効です。ネイティブ macOS Node は、ローカルに `~/.claude/projects/`
ディレクトリが存在する場合、読み取り専用の Claude セッションコマンドを通知します。
これらのコマンドが初めて表示されたときに、Node ペアリングのアップグレードを承認してください。

サイドバーは Gateway またはペアリング済み Node ホストごとに行をグループ化し、
各コンピューターから応答があり次第、そのホストの最新の範囲限定ページを表示します。ホストの接続状態が変化した後、
ページが再びフォーカスを得たとき、および表示中は最大 30 秒ごとに再調整されるため、
OpenClaw の外部で作成された Claude セッションも再読み込みなしで表示されます。
カタログに変更があると、より早く追加の処理が実行されます。カタロググループの下にある **さらにセッションを読み込む**
を使用すると、さらに履歴があるすべてのホストについて次のページが追加されます。追加された行は表示されたままになり、
更新時にも同じ深さまで再取得されます。カタログクライアントは `sessions.catalog.list` を使用し、行を開くときは
`sessions.catalog.read` を使用します。

ターミナルの引き継ぎでは、サービス／デーモンの PATH より先に、所有ホストユーザーのログインシェルの
PATH から `claude` を解決します。これにより、アプリから起動されたセッションと、
オペレーターが通常のターミナルで使用する Claude CLI の整合性が保たれます。

行を選択すると、最初に最新のトランスクリプトページが読み込まれます。**古いトランスクリプト項目を
読み込む**では、不透明なバイトカーソルをたどり、履歴全体を読み込む代わりに
JSONL ファイルから範囲を制限した別のセクションを読み込みます。通常のユーザー、アシスタント、
推論、ツール呼び出し、ツール結果の内容は保持されます。個々の項目が
Node／Gateway の安全上限を超える場合は、切り詰められたことが明確に示されます。

Gateway ローカルの `claude-cli` 行では、通常の入力欄に入力すると
`sessions.catalog.continue` が呼び出されます。OpenClaw はローカルカタログレコードを再解決し、
モデルが固定されたネイティブセッションを作成または再利用して、表示可能な項目を最大 200 件
または 512 KiB までインポートし、Claude CLI バインディングを初期化します。最初のターンは
`--fork-session` で再開されます。Claude はフォークに新しいセッション ID を割り当てるため、
それ以降のターンではフォークが使用され、元のセッションは変更されません。

ヘッドレス Node ホストでも、以下の Node ローカル設定を有効にして
Node ホストを再起動することで、Claude CLI の行を継続可能にできます。

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Node は、設定が有効で、ローカルの `claude` 実行可能ファイルが解決できる場合にのみ、
`agent.cli.claude.run.v1` を通知します。OpenClaw はその Node 上でカタログレコードを再解決し、
同じ範囲限定の履歴をインポートして、引き継いだセッションを Node およびカタログから報告された
作業ディレクトリにバインドします。各ターンでは、その Node の Claude ファイルとログインを使用して、
Node 上の実際の `claude -p` プロセスが実行されます。Node の実行承認ポリシーは引き続き適用され、
Gateway がオプトインを強制することはできません。

Node 継続機能 v1 は 1 回限りです。Gateway の local loopback MCP 設定と
Gateway の Skills Plugin 引数は省略され、Gateway トランスクリプトからの再初期化は行われず、
添付ファイルと画像は拒否されます。Claude Desktop の行は引き続き閲覧専用です。ネイティブ
macOS アプリの Node も、アプリが実行コマンドを通知するまでは閲覧専用のままです。

<Note>
ペアリング済み Node の Claude セッションは、ヘッドレス Node が明示的に
`agent.cli.claude.run.v1` を通知しない限り、読み取り専用のままです。OpenClaw が Claude Desktop の
メタデータを変更したり、Claude セッションをアーカイブしたりすることはありません。このページでは、認証済みの
`node.invoke` を使用するため、書き込みスコープを持つオペレーター接続が必要です。継続機能が有効な Node でも、
一覧表示と読み取りは読み取り専用のままです。
</Note>

[Nodes：Claude セッションとトランスクリプト](/ja-JP/nodes#claude-sessions-and-transcripts)で、Node コマンドとセキュリティ境界について参照してください。

## Thinking のデフォルト（Claude Opus 5、Sonnet 5、Mythos 5、Fable 5、4.8、4.6）

`anthropic/claude-opus-5` は、デフォルトで `high` の effort による適応型 Thinking を使用します。
Thinking を無効にするには `/think off` を、モデルネイティブのより高い effort レベルを使用するには
`/think xhigh|max` を使用します。Anthropic は Opus 5 でこれらのリクエスト機能をサポートしていないため、
OpenClaw は手動の Thinking バジェット、カスタムサンプリングパラメータ、アシスタントのプリフィル、
Priority Tier を省略します。カタログには、1,000,000 トークンのコンテキストウィンドウ、
128,000 トークンの出力上限、画像入力、`$5/$25` の入出力料金が掲載されています。

`anthropic/claude-sonnet-5` は、同じ適応型 Thinking のデフォルトとリクエスト制限を使用します。
カタログでは、2026 年 8 月 31 日まで Anthropic の導入時 `$2/$10` 入出力料金を使用し、
2026 年 9 月 1 日から標準の `$3/$15` 料金が適用されます。

`anthropic/claude-fable-5` は常に適応型 Thinking を使用し、デフォルトの effort は `high`
です。Anthropic はこのモデルで Thinking を無効にすることを許可していないため、
`/think off` と `/think minimal` は代わりに `low` の effort にマッピングされます。
また、Anthropic は Thinking が有効なリクエストでの temperature のオーバーライドを拒否するため、
OpenClaw は Fable 5 リクエストのカスタム temperature 値も省略します。

`anthropic/claude-mythos-5` は、同じ常時有効の適応型 Thinking 契約を持つ限定アクセスモデルです。
OpenClaw のデフォルトは `high` で、`/think off` と
`/think minimal` を `low` にマッピングし、呼び出し元が選択したサンプリングパラメータを省略します。
カタログには、1,000,000 トークンのコンテキストウィンドウ、128,000 トークンの出力上限、
画像入力、`$10/$50` の入出力料金が掲載されています。

OpenClaw では、Claude Opus 4.8 の Thinking はデフォルトで無効です。`/think high|xhigh|max` で
適応型 Thinking を明示的に有効にすると、OpenClaw は Anthropic の Opus 4.8 用 effort 値を送信します。
Claude 4.6 モデル（Opus 4.6 と Sonnet 4.6）のデフォルトは `adaptive` です。

メッセージごとに `/think:<level>` で、またはモデルパラメータ内でオーバーライドします。

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
関連する Anthropic ドキュメント：
- [適応型 Thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [拡張 Thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## 安全性による拒否時のフォールバック（Claude Fable 5）

<Warning>
Claude Fable 5 を使用すると、Claude Opus 4.8 も使用されます。Fable 5 にはリクエストを拒否できる
安全性分類器が搭載されており、Anthropic が認可している復旧方法は、そのターンを
`claude-opus-4-8` に処理させることです。OpenClaw は API キーによる直接リクエストで
これを自動的に有効化するため、一部の Fable ターンは Claude Opus 4.8 によって応答され、
その料金で請求されます。ポリシーまたは予算上、Opus によって処理されるターンを許容できない場合は、
`anthropic/claude-fable-5` を選択しないでください。
</Warning>

### これが存在する理由

Fable 5 の分類器は、制限対象分野のリクエストに対して `stop_reason: "refusal"` を返します。
また、無害な作業に隣接する内容（セキュリティツール、ライフサイエンス、さらにはモデルに
生の推論を再現するよう求める場合）でも誤検知します。フォールバックがなければ、別の Claude モデルなら
問題なく処理できる場合でも、そのターンはエラーで終了します。Anthropic 自身の拒否メッセージでも、
API インテグレーターにフォールバックモデルを設定するよう案内しています。

### 動作の仕組み

1. `anthropic/claude-fable-5` への API キーによる直接リクエストごとに、OpenClaw は
   Anthropic のサーバー側フォールバックをオプトインするため、
   `server-side-fallback-2026-06-01` ベータヘッダーと
   `fallbacks: [{"model": "claude-opus-4-8"}]` を送信します。Anthropic が Fable 5 に対して許可している
   フォールバック先は Claude Opus 4.8 のみです。
2. フォールバックをトリガーするのは、安全性分類器による拒否だけです。レート制限、
   過負荷、サーバーエラーは従来どおりに動作し、OpenClaw の通常の
   [モデルフェイルオーバー](/ja-JP/concepts/model-failover)を経由します。
3. 救済処理は同じ呼び出し内で行われます。出力前に拒否された場合、レイテンシー以外からは
   判別できず、回答全体が Opus 4.8 から返されます。ストリーム途中で拒否された場合は、
   部分テキストがフォールバックモデルの継続元となるプレフィックスとして保持されますが、
   拒否したモデルの推論とツール呼び出しは Anthropic のリプレイルールに従って破棄されます
   （それらを応答に含めたり、実行したりしてはなりません）。
4. Claude Opus 4.8 も拒否した場合、そのターンでは、この機能が導入される前とまったく同様に、
   拒否がエラーとして表面化します。

フォールバックは Anthropic API レベルで行われるため、`claude-opus-4-8` を
設定済みモデルリストやフォールバックチェーンに含める必要はありません。Fable 対応の
API キーは常に Opus を処理できます。

### 可観測性と請求

- フォールバックで処理されたターンでは、アシスタントメッセージに
  `fromModel` と `toModel` を示す `provider_fallback` 診断が記録され、
  メッセージの `responseModel` には `claude-opus-4-8` が報告されます。
- Anthropic は試行ごとに請求します。出力前の拒否は無料で、救済処理には Claude Opus 4.8 の料金
  （現在は Fable 5 の料金の半額）が適用されます。OpenClaw のターンごとのコスト見積もりでも、
  これに合わせてフォールバックで処理されたターンを Opus の料金で計算します。
- ストリーム途中で拒否された場合、Anthropic 側では、すでにストリーミングされた Fable の
  部分出力にも追加で課金されます。この部分は API の試行ごとの使用量に報告されますが、
  OpenClaw のターンごとの見積もりには含まれません。

### 適用範囲

`api.anthropic.com` に対して API キー認証を使用する `anthropic/claude-fable-5` に適用されます。
OAuth（Claude CLI サブスクリプションの再利用）、プロキシベース URL、Bedrock、Vertex、
Foundry のリクエストは変更されず、引き続き拒否がエラーとして表面化します。

ライブ検証済み：Fable 5 に生の思考連鎖を再現するよう求める無害なプロンプトは、
フォールバックなしで送信すると `category: "reasoning_extraction"` により拒否されますが、
同じプロンプトを OpenClaw 経由で送信すると、`provider_fallback` 診断が付いた
Opus 処理による通常の回答が返されます。

基盤となる動作については、Anthropic の[拒否とフォールバックの
ガイド](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)を参照してください。

## プロンプトキャッシュ

OpenClaw は、API キー認証で Anthropic のプロンプトキャッシュ機能をサポートします。

| 値               | キャッシュ期間 | 説明                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"`（デフォルト） | 5 分      | API キー認証で自動的に適用 |
| `"long"`            | 1 時間         | 拡張キャッシュ                         |
| `"none"`            | キャッシュなし     | プロンプトキャッシュを無効化                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="エージェントごとのキャッシュオーバーライド">
    モデルレベルのパラメータをベースラインとして使用し、特定のエージェントを `agents.entries.*.params` でオーバーライドします。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    設定のマージ順序：

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params`（`id` が一致するものをキーごとにオーバーライド）

    これにより、同じモデルを使用する一方のエージェントでは長期間のキャッシュを維持し、別のエージェントではバースト性が高く再利用率の低いトラフィックに対してキャッシュを無効にできます。

  </Accordion>

  <Accordion title="Bedrock Claude に関する注意事項">
    - Bedrock 上の Anthropic Claude モデル（`amazon-bedrock/*anthropic.claude*`）は、設定されている場合、`cacheRetention` のパススルーを受け入れます。
    - Anthropic 以外の Bedrock モデルは、実行時に強制的に `cacheRetention: "none"` に設定されます。
    - API キーのスマートデフォルトでは、明示的な値が設定されていない場合、Bedrock 上の Claude の参照にも `cacheRetention: "short"` が設定されます。

  </Accordion>
</AccordionGroup>

## 高度な設定

<AccordionGroup>
  <Accordion title="高速モード">
    OpenClaw の共有 `/fast` トグルは、`api.anthropic.com` への API キーによる直接トラフィックに対して Anthropic の `service_tier` フィールドを設定します。

    | コマンド | マッピング先 |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - API キーを使用して行われる直接の `api.anthropic.com` リクエストにのみ適用されます。OAuth／サブスクリプショントークンのリクエストとプロキシルートには、`service_tier` フィールドが設定されることはありません。
    - `serviceTier` または `service_tier` パラメータが明示されている場合、両方が設定されていると `/fast` より優先されます。
    - Claude Opus 5 と Sonnet 5 は Priority Tier をサポートしていないため、OpenClaw はこれらのモデルで `service_tier` を省略します。
    - Priority Tier のキャパシティがないアカウントでは、`service_tier: "auto"` が `standard` に解決される場合があります。

    </Note>

  </Accordion>

  <Accordion title="メディア理解（画像と PDF）">
    バンドルされている Anthropic Plugin は、画像と PDF の理解機能を登録します。OpenClaw は
    設定済みの Anthropic 認証からメディア機能を自動解決するため、追加設定は不要です。

    | プロパティ        | 値                 |
    | --------------- | --------------------- |
    | デフォルトモデル   | `claude-opus-5`       |
    | サポートされる入力 | 画像、PDF ドキュメント |

    画像または PDF が会話に添付されると、OpenClaw は自動的に
    Anthropic のメディア理解プロバイダーを経由させます。

  </Accordion>

  <Accordion title="1M コンテキストウィンドウ">
    Claude Opus 5、Sonnet 5、Mythos 5、Fable 5 は正確に
    1,000,000 トークンの入力ウィンドウを持ち、最大 128,000 出力トークンをサポートします。
    Anthropic の 1M コンテキストウィンドウは、適応型 Thinking を使用する Claude 4.x モデルでも
    GA になっています：Opus 4.8、
    Opus 4.7、Opus 4.6、Sonnet 4.6。OpenClaw はこれらのモデルのサイズを
    自動的に設定するため、`params.context1m` は不要です。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    古い設定では `params.context1m: true` を維持できます。これはこれらのモデルでは無害な
    no-op であり、OpenClaw はいずれの場合も廃止された `context-1m-2025-08-07`
    ベータヘッダーを送信しなくなりました。その値を持つ古い `anthropicBeta` 設定エントリは、
    リクエストヘッダーの解決時に削除され、サポート対象外の古い Claude モデルは通常の
    コンテキストウィンドウを維持します。

    `params.context1m: true` は、Claude CLI バックエンド（`claude-cli/*`）でも同様に動作します。
    対象となる GA 対応の Opus と Sonnet モデルにはすでに 1M ウィンドウが自動的に適用されるため、
    そちらでもこのパラメータは任意です。

    <Warning>
    Anthropic の認証情報でロングコンテキストへのアクセスが必要です。OAuth／サブスクリプショントークン認証では必要な Anthropic ベータヘッダーを維持しますが、古い設定に廃止された 1M ベータヘッダーが残っている場合、OpenClaw はそれを削除します。
    </Warning>

  </Accordion>

  <Accordion title="Claude Opus 5 の 1M コンテキスト">
    `anthropic/claude-opus-5` とその `claude-cli` バリアントには、デフォルトで 1M の
    コンテキストウィンドウがあり、`params.context1m: true` は不要です。
  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="401 エラー／トークンが突然無効になった">
    Anthropic のトークン認証には有効期限があり、取り消される場合もあります。新規セットアップでは、代わりに Anthropic API キーを使用してください。
  </Accordion>

  <Accordion title='プロバイダー "anthropic" の API キーが見つかりません'>
    Anthropic の認証は**エージェントごと**です。新しいエージェントはメインエージェントのキーを継承しません。そのエージェントでオンボーディングを再実行するか、Gateway ホストで API キーを設定してから、`openclaw models status` で確認してください。
  </Accordion>

  <Accordion title='プロファイル "anthropic:default" の認証情報が見つかりません'>
    `openclaw models status` を実行して、アクティブな認証プロファイルを確認してください。オンボーディングを再実行するか、そのプロファイルパスに API キーを設定してください。
  </Accordion>

  <Accordion title="利用可能な認証プロファイルがありません（すべてクールダウン中）">
    `auth.unusableProfiles` については、`openclaw models status --json` を確認してください。Anthropic のレート制限によるクールダウンはモデル単位の場合があるため、同系統の別の Anthropic モデルは引き続き利用できる可能性があります。別の Anthropic プロファイルを追加するか、クールダウンが終了するまで待ってください。
  </Accordion>
</AccordionGroup>

<Note>
詳細なヘルプ：[トラブルシューティング](/ja-JP/help/troubleshooting)および[よくある質問](/ja-JP/help/faq)。
</Note>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択方法。
  </Card>
  <Card title="CLI バックエンド" href="/ja-JP/gateway/cli-backends" icon="terminal">
    Claude CLI バックエンドのセットアップとランタイムの詳細。
  </Card>
  <Card title="プロンプトキャッシュ" href="/ja-JP/reference/prompt-caching" icon="database">
    プロバイダー間でプロンプトキャッシュがどのように機能するか。
  </Card>
  <Card title="OAuth と認証" href="/ja-JP/gateway/authentication" icon="key">
    認証の詳細と認証情報の再利用ルール。
  </Card>
</CardGroup>
