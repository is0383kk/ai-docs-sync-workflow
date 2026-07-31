---
read_when:
    - 一般的なセットアップ、インストール、オンボーディング、ランタイムに関するサポート質問への回答
    - 詳細なデバッグ前のユーザー報告問題のトリアージ
summary: OpenClaw のセットアップ、設定、使用方法に関するよくある質問
title: よくある質問
x-i18n:
    generated_at: "2026-07-26T10:16:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

実環境のセットアップ（ローカル開発、VPS、マルチエージェント、OAuth/API キー、モデルのフェイルオーバー）について、簡潔な回答と詳細なトラブルシューティングを提供します。ランタイム診断については、[トラブルシューティング](/ja-JP/gateway/troubleshooting)を参照してください。設定の完全なリファレンスについては、[設定](/ja-JP/gateway/configuration)を参照してください。

## 問題が発生した場合の最初の 60 秒

<Steps>
  <Step title="クイックステータス">
    ```bash
    openclaw status
    ```
    ローカル環境の簡潔な概要：OS と更新、Gateway/サービスへの到達性、エージェント/セッション、プロバイダー設定とランタイムの問題（Gateway に到達できる場合）。
  </Step>
  <Step title="貼り付け可能なレポート（安全に共有可能）">
    ```bash
    openclaw status --all
    ```
    ログ末尾を含む読み取り専用の診断（トークンは秘匿化されます）。
  </Step>
  <Step title="デーモンとポートの状態">
    ```bash
    openclaw gateway status
    ```
    スーパーバイザーのランタイムと RPC の到達性、プローブ対象 URL、サービスが使用した可能性が高い設定を表示します。
  </Step>
  <Step title="詳細プローブ">
    ```bash
    openclaw status --deep
    ```
    サポートされている場合はチャネルプローブを含む、稼働中の Gateway ヘルスプローブ（到達可能な Gateway が必要です）。[ヘルス](/ja-JP/gateway/health)を参照してください。
  </Step>
  <Step title="最新ログを追跡">
    ```bash
    openclaw logs --follow
    ```
    RPC が停止している場合は、次の方法に切り替えます。
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # 名前付きプロファイルの例：
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    ファイルログはサービスログとは別です。[ロギング](/ja-JP/logging)と[トラブルシューティング](/ja-JP/gateway/troubleshooting)を参照してください。
  </Step>
  <Step title="doctor を実行（修復）">
    ```bash
    openclaw doctor
    ```
    設定と状態を修復または移行してから、ヘルスチェックを実行します。[Doctor](/ja-JP/gateway/doctor)を参照してください。
  </Step>
  <Step title="Gateway スナップショット（WS のみ）">
    ```bash
    openclaw health --json
    openclaw health --verbose   # エラー時に対象 URL と設定パスを表示
    ```
    稼働中の Gateway に完全なスナップショットを要求します。[ヘルス](/ja-JP/gateway/health)を参照してください。
  </Step>
</Steps>

## クイックスタートと初回セットアップ

初回実行に関する Q&A（インストール、オンボーディング、認証ルート、サブスクリプション、初期障害）は、[初回実行 FAQ](/ja-JP/help/faq-first-run)にあります。

## OpenClaw とは？

<AccordionGroup>
  <Accordion title="OpenClaw とは何ですか？一段落で説明してください">
    OpenClaw は、自分のデバイス上で実行するパーソナル AI アシスタントです。普段使用しているメッセージング環境（Discord、Google Chat、iMessage、Mattermost、Signal、Slack、Telegram、WebChat、WhatsApp、および QQ Bot などの同梱チャネル Plugin）で応答し、対応プラットフォームでは音声機能とライブ Canvas も利用できます。**Gateway** は常時稼働するコントロールプレーンであり、アシスタントそのものが製品です。
  </Accordion>

  <Accordion title="価値提案">
    OpenClaw は「単なる Claude ラッパー」ではありません。**ローカルファーストのコントロールプレーン**として、**自分のハードウェア**上で高性能なアシスタントを実行し、普段使っているチャットアプリからアクセスできます。ステートフルなセッション、メモリ、ツールを備え、ワークフローをホスト型 SaaS に委ねる必要はありません。

    - **自分のデバイス、自分のデータ**：Gateway を任意の場所（Mac、Linux、VPS）で実行し、ワークスペースとセッション履歴をローカルに保持できます。
    - **Web サンドボックスではなく、実際のチャネル**：Discord/iMessage/Signal/Slack/Telegram/WhatsApp などに加え、対応プラットフォームではモバイル音声と Canvas を利用できます。
    - **モデル非依存**：Anthropic、MiniMax、OpenAI、OpenRouter などを、エージェントごとのルーティングとフェイルオーバー付きで使用できます。
    - **ローカルのみの選択肢**：ローカルモデルを実行し、すべてのデータをデバイス上に保持できます。
    - **マルチエージェントルーティング**：チャネル、アカウント、タスクごとにエージェントを分離し、それぞれに独自のワークスペースとデフォルトを設定できます。
    - **オープンソースでカスタマイズ可能**：ベンダーロックインなしで、調査、拡張、セルフホストが可能です。

    ドキュメント：[Gateway](/ja-JP/gateway)、[チャネル](/ja-JP/channels)、[マルチエージェント](/ja-JP/concepts/multi-agent)、[メモリ](/ja-JP/concepts/memory)。

  </Accordion>

  <Accordion title="セットアップしたばかりです。最初に何をすればよいですか？">
    最初のプロジェクトとして適しているもの：Web サイト（WordPress、Shopify、または静的サイト）の構築、モバイルアプリ（概要、画面、API 計画）のプロトタイプ作成、ファイルやフォルダーの整理、Gmail への接続と要約やフォローアップの自動化。

    大規模なタスクにも対応できますが、フェーズに分割し、並列作業にサブエージェントを使用すると最も効果的です。

  </Accordion>

  <Accordion title="OpenClaw の日常的な用途トップ 5 は何ですか？">
    - **パーソナルブリーフィング**：受信トレイ、カレンダー、関心のあるニュースの要約。
    - **調査と下書き**：簡単な調査、要約、メールやドキュメントの初稿作成。
    - **リマインダーとフォローアップ**：Cron または Heartbeat による通知やチェックリスト。
    - **ブラウザー自動化**：フォームへの入力、データ収集、Web タスクの反復。
    - **デバイス間の連携**：スマートフォンからタスクを送信し、Gateway にサーバー上で実行させ、結果をチャットで受け取ります。

  </Accordion>

  <Accordion title="OpenClaw は SaaS の見込み顧客獲得、アウトリーチ、広告、ブログに役立ちますか？">
    はい。**調査、選定、下書き**に活用できます。サイトの調査、候補リストの作成、見込み顧客の要約、アウトリーチ文面や広告コピーの下書きなどに対応します。

    **アウトリーチや広告の実施**では、必ず人間が関与するようにしてください。スパムを避け、現地の法律とプラットフォームのポリシーに従い、送信前にすべての内容を確認してください。OpenClaw に下書きを作成させ、利用者が承認します。

    ドキュメント：[セキュリティ](/ja-JP/gateway/security)。

  </Accordion>

  <Accordion title="Web 開発において Claude Code と比べた利点は何ですか？">
    OpenClaw は**パーソナルアシスタント**兼調整レイヤーであり、IDE の代替ではありません。リポジトリ内で最速の直接的なコーディングループを実現するには Claude Code または Codex を使用してください。永続的なメモリ、デバイス間アクセス、ツールのオーケストレーションには OpenClaw を使用してください。

    - セッションをまたぐ永続的なメモリとワークスペース。
    - マルチプラットフォームアクセス（Telegram、WhatsApp、TUI、WebChat）。
    - ツールのオーケストレーション（ブラウザー、ファイル、スケジュール、フック）。
    - 常時稼働の Gateway（VPS 上で実行し、どこからでも操作可能）。
    - ローカルのブラウザー、画面、カメラ、コマンド実行用の Node。

    ショーケース：[https://openclaw.ai/showcase](https://openclaw.ai/showcase)。

  </Accordion>
</AccordionGroup>

## Skills と自動化

<AccordionGroup>
  <Accordion title="リポジトリを変更状態にせずに Skills をカスタマイズするにはどうすればよいですか？">
    リポジトリ内のコピーを編集する代わりに、管理対象のオーバーライドを使用します。変更は `~/.openclaw/skills/<name>/SKILL.md` に配置します（または `~/.openclaw/openclaw.json` の `skills.load.extraDirs` を使用してフォルダーを追加します）。優先順位は `<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> 同梱版 -> `skills.load.extraDirs` です。そのため、git に触れることなく、管理対象のオーバーライドが同梱 Skills より優先されます。グローバルにインストールしつつ一部のエージェントのみに表示するには、共有コピーを `~/.openclaw/skills` に保持し、`agents.defaults.skills` / `agents.entries.*.skills` で表示範囲を制御します。アップストリームに取り込む価値のある編集のみ、リポジトリ内のコピーに対する PR として提出してください。
  </Accordion>

  <Accordion title="カスタムフォルダーから Skills を読み込めますか？">
    はい。`~/.openclaw/openclaw.json` の `skills.load.extraDirs` を使用してディレクトリを追加します（上記の順序では最も低い優先順位です）。`clawhub` はデフォルトで `./skills` にインストールし、OpenClaw は次のセッションでこれを `<workspace>/skills` として扱います。特定のエージェントのみに表示するには、`agents.defaults.skills` または `agents.entries.*.skills` と組み合わせてください。
  </Accordion>

  <Accordion title="タスクごとに異なるモデルや設定を使用するにはどうすればよいですか？">
    対応しているパターン：

    - **Cron ジョブ**：分離されたジョブごとに `model` オーバーライドを設定できます。
    - **エージェント**：デフォルトモデル、思考レベル、ストリームパラメーターが異なる個別のエージェントにタスクをルーティングします。
    - **オンデマンド切り替え**：`/model` を使用すると、現在のセッションのモデルをいつでも切り替えられます。

    例 — 同じモデルに対するエージェントごとの異なる設定：

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    モデルごとに共有するデフォルトは `agents.defaults.models["provider/model"].params` に配置し、エージェント固有のオーバーライドはフラットな `agents.entries.*.params` に配置します。ネストされた `agents.entries.*.models["provider/model"].params` の下に同じモデルを重複して配置しないでください。このパスは、エージェントごとのモデルカタログとランタイムオーバーライド用です。

    [Cron ジョブ](/ja-JP/automation/cron-jobs)、[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)、[設定](/ja-JP/gateway/config-agents)、[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

  </Accordion>

  <Accordion title="負荷の高い処理中にボットが停止します。処理をオフロードするにはどうすればよいですか？">
    長時間または並列のタスクには**サブエージェント**を使用します。サブエージェントは独自のセッションで実行され、要約を返すため、メインチャットの応答性を維持できます。ボットに「このタスク用のサブエージェントを起動して」と依頼するか、`/subagents` を使用します。Gateway が現在処理中かどうかを確認するには、`/status` を使用します。

    長時間のタスクとサブエージェントはいずれもトークンを消費します。コストが重要な場合は、`agents.defaults.subagents.model` を使用してサブエージェントに安価なモデルを設定してください。

    ドキュメント：[サブエージェント](/ja-JP/tools/subagents)、[バックグラウンドタスク](/ja-JP/automation/tasks)。

  </Accordion>

  <Accordion title="Discord でスレッドに紐付けられたサブエージェントセッションはどのように機能しますか？">
    Discord スレッドをサブエージェントまたはセッション対象に紐付けると、そのスレッド内の後続メッセージが紐付けられたセッションに送られます。

    - `thread: true` を指定した `sessions_spawn` で起動します（永続的なフォローアップには、必要に応じて `mode: "session"` を指定します）。
    - または、`/focus <target>` を使用して手動で紐付けます。
    - `/agents` で紐付け状態を確認します。
    - `/session idle <duration|off>` と `/session max-age <duration|off>` で自動フォーカス解除を制御します。
    - `/unfocus` でスレッドの紐付けを解除します。

    設定：`session.threadBindings.enabled`（グローバルスイッチ）、`session.threadBindings.idleHours`（デフォルトは `24`、`0` で無効化）、`session.threadBindings.maxAgeHours`（デフォルトは `0` = 上限なし）、および起動時の自動紐付け用の `session.threadBindings.spawnSessions`（デフォルトは `true`）。

    ドキュメント：[サブエージェント](/ja-JP/tools/subagents)、[Discord](/ja-JP/channels/discord)、[設定リファレンス](/ja-JP/gateway/configuration-reference)、[スラッシュコマンド](/ja-JP/tools/slash-commands)。

  </Accordion>

  <Accordion title="サブエージェントが完了しましたが、完了通知が誤った場所に送られたか、投稿されませんでした。何を確認すればよいですか？">
    解決された要求元ルートを確認してください。

    - 完了モードのサブエージェント配信では、紐付けられたスレッドまたは会話ルートが存在する場合、それが優先されます。
    - 完了元にチャネルしか含まれていない場合、OpenClaw は要求元セッションに保存されているルート（`lastChannel` / `lastTo` / `lastAccountId`）にフォールバックし、直接配信を試みます。
    - 紐付けられたルートも使用可能な保存済みルートもない場合、直接配信が失敗する可能性があり、結果は即時投稿ではなく、キューに入れられたセッション配信にフォールバックします。
    - 無効または古い対象によって、キューへのフォールバックや最終的な配信失敗が発生することもあります。
    - 子セッションで最後に表示されたアシスタントの応答が正確に `NO_REPLY` / `no_reply` または `ANNOUNCE_SKIP` である場合、OpenClaw は古い以前の進捗を投稿する代わりに、意図的に通知を抑制します。

    デバッグ：`<lookup>` にタスク ID、実行 ID、またはセッションキーを指定して `openclaw tasks show <lookup>` を実行します。

    ドキュメント：[サブエージェント](/ja-JP/tools/subagents)、[バックグラウンドタスク](/ja-JP/automation/tasks)、[セッションツール](/ja-JP/concepts/session-tool)。

  </Accordion>

  <Accordion title="Cron またはリマインダーが実行されません。何を確認すればよいですか？">
    Cron は Gateway プロセス内で実行されます。Gateway が継続的に稼働していない場合は実行されません。

    - Cron が有効になっており（`cron.enabled`）、`OPENCLAW_SKIP_CRON` が設定されていないことを確認します。
    - Gateway が 24/7 稼働していることを確認します（スリープや再起動がないこと）。
    - ジョブのタイムゾーン（`--tz` とホストのタイムゾーン）を確認します。

    デバッグ:
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    ドキュメント: [Cron ジョブ](/ja-JP/automation/cron-jobs)、[自動化](/ja-JP/automation)。

  </Accordion>

  <Accordion title="Cron は実行されましたが、チャンネルには何も送信されませんでした。なぜですか？">
    配信モードを確認します。

    - `--no-deliver` / `delivery.mode: "none"`: ランナーによるフォールバック送信は行われません。
    - 通知先（`channel` / `to`）が未指定または無効です。ランナーは外部への配信をスキップしました。
    - チャンネル認証エラー（`unauthorized`、`Forbidden`）: ランナーは配信を試みましたが、認証情報によってブロックされました。
    - 無出力の分離実行結果（`NO_REPLY` / `no_reply` のみ）は意図的に配信不能と見なされるため、キューに登録されたフォールバック配信も抑止されます。

    分離 Cron ジョブでは、チャットルートが利用可能な場合、エージェントは `message` ツールを使用して直接送信できます。`--announce` が制御するのは、エージェント自身がまだ送信していない最終テキストに対するランナーのフォールバック配信のみです。

    デバッグ:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    ドキュメント: [Cron ジョブ](/ja-JP/automation/cron-jobs)、[バックグラウンドタスク](/ja-JP/automation/tasks)。

  </Accordion>

  <Accordion title="分離 Cron 実行でモデルが切り替わったり、1 回再試行されたりしたのはなぜですか？">
    これは重複スケジューリングではなく、実行中のモデル切り替えパスです。分離 Cron は実行時のモデル引き継ぎを永続化し、アクティブな実行が `LiveSessionModelSwitchError` をスローすると、切り替え後のプロバイダー／モデル（および切り替え後の認証プロファイルのオーバーライド）を維持して再試行します。

    モデル選択の優先順位: 最初に Gmail フックのモデルオーバーライド（`hooks.gmail.model`）、次にジョブごとの `model`、その次に保存済み Cron セッションのモデルオーバーライド、最後に通常のエージェント／デフォルトモデル選択です。

    再試行ループは最初の試行と 2 回の切り替え再試行までに制限されます。その後、Cron は無限ループせずに中止します。

    デバッグ:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    ドキュメント: [Cron ジョブ](/ja-JP/automation/cron-jobs)、[cron CLI](/ja-JP/cli/cron)。

  </Accordion>

  <Accordion title="Linux に Skills をインストールするにはどうすればよいですか？">
    ネイティブの `openclaw skills` コマンドを使用するか、ワークスペースに Skills を配置します。macOS の Skills UI は Linux では利用できません。[https://clawhub.ai](https://clawhub.ai) で Skills を参照できます。

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    ネイティブの `openclaw skills install` は、デフォルトでアクティブなワークスペースの `skills/` ディレクトリに書き込みます。すべてのローカルエージェント向けの共有管理 Skills ディレクトリにインストールするには、`--global` を追加します。別個の `clawhub` CLI は、自作の Skills を公開または同期する場合にのみインストールしてください。共有 Skills を表示できるエージェントを制限するには、`agents.defaults.skills` または `agents.entries.*.skills` を使用します。

  </Accordion>

  <Accordion title="OpenClaw はスケジュールに従って、またはバックグラウンドで継続的にタスクを実行できますか？">
    はい。Gateway スケジューラーを使用します。

    - スケジュール実行または定期実行するタスクには **Cron ジョブ**（再起動後も維持されます）。
    - メインセッションの定期チェックには **Heartbeat**。
    - 概要を投稿したりチャットに配信したりする自律エージェントには **分離ジョブ**。

    ドキュメント: [Cron ジョブ](/ja-JP/automation/cron-jobs)、[自動化](/ja-JP/automation)、[Heartbeat](/ja-JP/gateway/heartbeat)。

  </Accordion>

  <Accordion title="Apple macOS 専用の Skills を Linux から実行できますか？">
    直接は実行できません。macOS の Skills は `metadata.openclaw.os` と必要なバイナリによって制限され、**Gateway ホスト**で条件を満たす場合にのみ読み込まれます。Linux では、制限をオーバーライドしない限り、`darwin` 専用の Skills（`apple-notes`、`apple-reminders`、`things-mac`）は読み込まれません。

    サポートされる方法は 3 つあります。

    **オプション A - Mac で Gateway を実行する（最も簡単）**。macOS バイナリが存在する環境で Gateway を実行し、Linux から[リモートモード](#gateway-ports-already-running-and-remote-mode)または Tailscale 経由で接続します。Gateway ホストが macOS であるため、Skills は通常どおり読み込まれます。

    **オプション B - macOS Node を使用する（SSH 不要）**。Linux で Gateway を実行し、macOS Node（メニューバーアプリ）をペアリングして、Mac の **Node Run Commands** を "Always Ask" または "Always Allow" に設定します。Node に必要なバイナリが存在する場合、OpenClaw は macOS 専用 Skills を利用可能と見なし、エージェントは `nodes` ツールを介して実行します。"Always Ask" の場合、プロンプトで "Always Allow" を承認すると、そのコマンドが許可リストに追加されます。

    **オプション C - SSH 経由で macOS バイナリをプロキシする（上級者向け）**。Gateway は Linux 上で維持しつつ、必要な CLI バイナリが Mac 上で実行される SSH ラッパーとして解決されるようにします。その後、Linux を許可するよう Skills をオーバーライドし、利用可能な状態を維持します。

    1. バイナリ用の SSH ラッパーを作成します（例: Apple Notes 用の `memo`）。
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. Linux ホスト上の `PATH` にラッパーを配置します（例: `~/bin/memo`）。
    3. Linux を許可するよう Skills のメタデータ（ワークスペースまたは `~/.openclaw/skills`）をオーバーライドします。
       ```markdown
       ---
       name: apple-notes
       description: macOS 上の memo CLI を使用して Apple Notes を管理します。
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. Skills のスナップショットを更新するため、新しいセッションを開始します。

  </Accordion>

  <Accordion title="Notion または HeyGen の連携機能はありますか？">
    現時点では組み込まれていません。選択肢は次のとおりです。

    - **カスタム Skills / Plugin**: 信頼性の高い API アクセスに最適です（どちらにも API があります）。
    - **ブラウザー自動化**: コードなしで動作しますが、より低速で脆弱です。

    代理店形式でクライアントごとのコンテキストを管理する場合は、クライアントごとに 1 つの Notion ページ（コンテキスト、設定、進行中の作業）を用意し、セッションの開始時にそのページを取得するようエージェントに依頼します。

    ネイティブ連携が必要な場合は、機能リクエストを作成するか、それらの API に対応する Skills を構築してください。

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    ネイティブインストールでは、アクティブなワークスペースの `skills/` ディレクトリに配置されます。すべてのローカルエージェント向けには `--global` を使用し、表示範囲を制限するには `agents.defaults.skills` / `agents.entries.*.skills` を設定します。一部の Skills は Homebrew でインストールされたバイナリを前提とします。Linux では Linuxbrew が必要です。

    [Skills](/ja-JP/tools/skills)、[Skills の設定](/ja-JP/tools/skills-config)、[ClawHub](/tools/clawhub)を参照してください。

  </Accordion>

  <Accordion title="既存のログイン済み Chrome を OpenClaw で使用するにはどうすればよいですか？">
    Chrome DevTools MCP 経由で接続する組み込みの `user` ブラウザープロファイルを使用します。

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    カスタム名を使用する場合は、明示的な MCP プロファイルを作成します。

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    ローカルホストのブラウザーまたは接続済みのブラウザー Node を使用できます。Gateway が別の場所で実行されている場合は、ブラウザーのマシンで Node ホストを実行するか、代わりにリモート CDP を使用します。

    管理対象の `openclaw` プロファイルと比較した、`existing-session` / `user` プロファイルの現在の制限事項:

    - `click`、`type`、`hover`、`scrollIntoView`、`drag`、`select` には、CSS セレクターではなくスナップショット参照が必要です。
    - アップロードフックには `ref` または `inputRef` が必要で、ファイルは一度に 1 つのみです。CSS `element` は使用できません。
    - `responsebody`、PDF エクスポート、ダウンロードのインターセプト、バッチアクションには、引き続き管理対象ブラウザーのパスが必要です。

    詳細な比較については、[ブラウザー](/ja-JP/tools/browser#existing-session-via-chrome-devtools-mcp)を参照してください。

  </Accordion>
</AccordionGroup>

## サンドボックス化とメモリ

<AccordionGroup>
  <Accordion title="サンドボックス化専用のドキュメントはありますか？">
    はい。[サンドボックス化](/ja-JP/gateway/sandboxing)を参照してください。Docker 固有のセットアップ（Docker 内の完全な Gateway またはサンドボックスイメージ）については、[Docker](/ja-JP/install/docker)を参照してください。
  </Accordion>

  <Accordion title="Docker の機能が制限されているように感じます。すべての機能を有効にするにはどうすればよいですか？">
    デフォルトイメージはセキュリティを優先し、`node` ユーザーとして実行されるため、システムパッケージ、Homebrew、同梱ブラウザーは含まれません。より完全なセットアップを行うには、次の手順を実行します。

    - キャッシュを維持するため、`OPENCLAW_HOME_VOLUME` を使用して `/home/node` を永続化します。
    - `OPENCLAW_IMAGE_APT_PACKAGES` を使用して、システム依存関係をイメージに組み込みます。
    - 同梱 CLI を使用して Playwright ブラウザーをインストールします: `node /app/node_modules/playwright-core/cli.js install chromium`。
    - `PLAYWRIGHT_BROWSERS_PATH` を設定し、そのパスを永続化します。

    ドキュメント: [Docker](/ja-JP/install/docker)、[ブラウザー](/ja-JP/tools/browser)。

  </Accordion>

  <Accordion title="1 つのエージェントで、DM は個人用のままにし、グループを公開／サンドボックス化できますか？">
    はい。プライベートなトラフィックが **DM**、公開トラフィックが **グループ**の場合に可能です。`agents.defaults.sandbox.mode: "non-main"` を設定すると、グループ／チャンネルセッション（メイン以外のキー）は設定されたサンドボックスバックエンドで実行され、メインの DM セッションはホスト上に残ります。サンドボックス化を有効にすると、Docker がデフォルトのバックエンドになります。サンドボックス化されたセッションで使用可能なツールは、`tools.sandbox.tools` で制限します。

    セットアップ手順: [グループ: 個人用 DM + 公開グループ](/ja-JP/channels/groups#pattern-personal-dms-public-groups-single-agent)。主要リファレンス: [Gateway の設定](/ja-JP/gateway/config-agents#agentsdefaultssandbox)。

  </Accordion>

  <Accordion title="ホストのフォルダーをサンドボックスにバインドするにはどうすればよいですか？">
    `agents.defaults.sandbox.docker.binds` を `["host:container:mode"]` に設定します（例: `"/home/user/src:/src:ro"`）。グローバルバインドとエージェントごとのバインドはマージされます。`scope: "shared"` の場合、エージェントごとのバインドは無視されます。機密性の高いものには `:ro` を使用してください。バインドはサンドボックスのファイルシステム境界を迂回します。

    OpenClaw は、正規化されたパスと、存在する最深の祖先を介して解決された正準パスの両方に対してバインド元を検証します。そのため、最終パスセグメントがまだ存在しない場合でも、シンボリックリンクの親を利用した脱出は安全側に倒して失敗します。

    [サンドボックス化](/ja-JP/gateway/sandboxing#custom-bind-mounts)および[サンドボックス、ツールポリシー、昇格の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check)を参照してください。

  </Accordion>

  <Accordion title="メモリはどのように機能しますか？">
    OpenClaw のメモリは、エージェントのワークスペース内にある Markdown ファイルです。日次ノートは `memory/YYYY-MM-DD.md`、整理された長期ノートは `MEMORY.md` に保存されます（メイン／プライベートセッションのみ）。

    OpenClaw は、Compaction が会話を要約する前に、無出力の **Compaction 前メモリフラッシュ**も実行し、最初に永続的なノートを書き込むようモデルに促します。これはワークスペースが書き込み可能な場合にのみ実行されます（読み取り専用サンドボックスではスキップされます）。無効にするには `agents.defaults.compaction.memoryFlush.enabled: false` を使用します。[メモリ](/ja-JP/concepts/memory)を参照してください。

  </Accordion>

  <Accordion title="メモリがすぐに忘れてしまいます。記憶を定着させるにはどうすればよいですか？">
    ボットに**事実をメモリへ書き込む**よう依頼します。長期ノートは `MEMORY.md`、短期コンテキストは `memory/YYYY-MM-DD.md` に保存します。メモリを保存するようモデルに念押しすると、通常は解決します。それでも忘れ続ける場合は、Gateway が実行ごとに同じワークスペースを使用していることを確認してください。

    ドキュメント: [メモリ](/ja-JP/concepts/memory)、[エージェントのワークスペース](/ja-JP/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="メモリは永続的に保持されますか？制限は何ですか？">
    メモリファイルはディスク上に保存され、削除されるまで保持されます。制限となるのはモデルではなくストレージです。**セッションコンテキスト**には引き続きモデルのコンテキストウィンドウによる制限があるため、長い会話は圧縮または切り詰められることがあります。そのためにメモリ検索があり、関連する部分だけをコンテキストに戻します。

    ドキュメント：[メモリ](/ja-JP/concepts/memory)、[コンテキスト](/ja-JP/concepts/context)。

  </Accordion>

  <Accordion title="セマンティックメモリ検索には OpenAI API キーが必要ですか？">
    必要なのは、デフォルトプロバイダーである **OpenAI embeddings** を使用する場合のみです。Codex OAuth が対象とするのはチャット／補完であり、embeddings へのアクセス権は付与されません。そのため、Codex（OAuth または Codex CLI ログイン）でサインインしても、セマンティックメモリ検索は有効になりません。OpenAI embeddings には引き続き実際の API キー（`OPENAI_API_KEY` または `models.providers.openai.apiKey`）が必要です。

    ローカル環境内に留めるには、`memory.search.provider: "local"`（GGUF/llama.cpp）を設定します。その他の対応プロバイダー：Bedrock、DeepInfra、Gemini（`GEMINI_API_KEY` または `memory.search.remote.apiKey`）、GitHub Copilot、LM Studio、Mistral、Ollama、OpenAI 互換、Voyage。設定の詳細については、[メモリ](/ja-JP/concepts/memory)と[メモリ検索](/ja-JP/concepts/memory-search)を参照してください。

  </Accordion>
</AccordionGroup>

## ディスク上の保存場所

<AccordionGroup>
  <Accordion title="OpenClaw で使用されるすべてのデータはローカルに保存されますか？">
    いいえ。**OpenClaw 自体の状態はローカル**ですが、**外部サービスには送信した内容が引き続き渡ります**。

    - **デフォルトではローカル**：セッション、メモリファイル、設定、ワークスペースは Gateway ホスト（`~/.openclaw` とワークスペースディレクトリ）に保存されます。
    - **必然的にリモート**：モデルプロバイダー（Anthropic／OpenAI など）に送信されたメッセージは各社の API に送られ、チャットプラットフォーム（Slack／Telegram／WhatsApp など）はメッセージデータを各社のサーバーに保存します。
    - **データの範囲は制御可能**：ローカルモデルを使用するとプロンプトは使用中のマシン内に留まりますが、チャネルの通信は引き続きそのチャネルのサーバーを経由します。

    関連項目：[エージェントワークスペース](/ja-JP/concepts/agent-workspace)、[メモリ](/ja-JP/concepts/memory)。

  </Accordion>

  <Accordion title="OpenClaw はデータをどこに保存しますか？">
    すべてが `$OPENCLAW_STATE_DIR`（デフォルト：`~/.openclaw`）以下に保存されます。

    | パス                                                               | 用途                                                            |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | メイン設定（JSON5）                                                 |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | 旧 OAuth インポート（初回使用時に認証プロファイルへコピー）        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | 認証プロファイル（OAuth、API キー、任意の `keyRef`/`tokenRef`）        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | `file` SecretRef プロバイダー用の任意のファイルベースシークレットペイロード   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | 旧互換性ファイル（静的な `api_key` エントリは除去済み）        |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | プロバイダーの状態（例：`whatsapp/<accountId>/creds.json`）      |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | エージェントごとの状態（agentDir と旧形式／アーカイブ済みセッション成果物）        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | セッション行とトランスクリプトを含む、エージェントごとの SQLite 状態      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | 旧セッションの移行元とアーカイブ／サポート成果物      |

    旧単一エージェントパス `~/.openclaw/agent/*` は `openclaw doctor` によって移行されます。

    **ワークスペース**（AGENTS.md、メモリファイル、Skills など）は別にあり、`agents.defaults.workspace`（デフォルト：`~/.openclaw/workspace`）で設定します。

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md はどこに配置すべきですか？">
    これらは `~/.openclaw` ではなく、**エージェントワークスペース**に配置します。

    - **ワークスペース（エージェントごと）**：`AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、`MEMORY.md`、`memory/YYYY-MM-DD.md`、任意の `HEARTBEAT.md`。小文字のルート `memory.md` は旧形式の修復入力専用です。両方が存在する場合、`openclaw doctor --fix` でこれを `MEMORY.md` に統合できます。
    - **状態ディレクトリ（`~/.openclaw`）**：設定、チャネル／プロバイダーの状態、認証プロファイル、セッション、ログ、共有 Skills（`~/.openclaw/skills`）。

    デフォルトのワークスペースは `~/.openclaw/workspace` で、次のように設定できます。

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    再起動後にボットが「忘れる」場合は、Gateway が起動のたびに同じワークスペースを使用していることを確認してください（リモートモードでは、ローカルのノート PC ではなく、**Gateway ホスト側**のワークスペースが使用されます）。

    ヒント：動作や設定を永続化するには、チャット履歴に頼るのではなく、ボットに **AGENTS.md または MEMORY.md へ書き込む**よう依頼してください。

    [エージェントワークスペース](/ja-JP/concepts/agent-workspace)と[メモリ](/ja-JP/concepts/memory)を参照してください。

  </Accordion>

  <Accordion title="SOUL.md を大きくできますか？">
    はい。`SOUL.md` は、エージェントコンテキストに挿入されるワークスペースのブートストラップファイルの 1 つです。ファイルごとのデフォルト挿入上限は `20000` 文字で、ファイル全体のブートストラップ予算は `60000` 文字です。

    共有デフォルトは次のように変更します。

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    または、`agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars` 以下で特定のエージェントを上書きします。

    `/context` を使用すると、元のサイズと挿入後のサイズ、および切り詰めが発生したかどうかを確認できます。`SOUL.md` には口調、姿勢、人格のみを簡潔に記載し、運用ルールは `AGENTS.md`、永続的な事実はメモリに記載してください。

    [コンテキスト](/ja-JP/concepts/context)と[エージェント設定](/ja-JP/gateway/config-agents)を参照してください。

  </Accordion>

  <Accordion title="推奨バックアップ戦略">
    **エージェントワークスペース**を**非公開**の git リポジトリに置き、非公開の場所（GitHub のプライベートリポジトリなど）にバックアップしてください。これにより、メモリと AGENTS／SOUL／USER ファイルを保存でき、後でアシスタントの「思考」を復元できます。

    `~/.openclaw` 以下にあるもの（認証情報、セッション、トークン、暗号化されたシークレットペイロード）は**コミットしないでください**。完全に復元できるようにするには、ワークスペースと状態ディレクトリを別々にバックアップしてください。

    ドキュメント：[エージェントワークスペース](/ja-JP/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="OpenClaw を完全にアンインストールするにはどうすればよいですか？">
    [アンインストール](/ja-JP/install/uninstall)を参照してください。
  </Accordion>

  <Accordion title="エージェントはワークスペースの外部で作業できますか？">
    はい。ワークスペースは**デフォルトの cwd** およびメモリの基点であり、厳格なサンドボックスではありません。相対パスはワークスペース内で解決されますが、サンドボックスが有効でない限り、絶対パスからホスト上の他の場所にアクセスできます。分離するには、[`agents.defaults.sandbox`](/ja-JP/gateway/sandboxing)またはエージェントごとのサンドボックス設定を使用してください。リポジトリをデフォルトの作業ディレクトリにするには、そのエージェントの `workspace` をリポジトリのルートに指定します。OpenClaw リポジトリ自体は単なるソースコードなので、エージェントに意図的にその中で作業させる場合を除き、ワークスペースは分けてください。

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="リモートモード：セッションストアはどこにありますか？">
    セッションの状態は **Gateway ホスト**が管理します。リモートモードでは、対象となるセッションストアはローカルのノート PC ではなく、リモートマシン上にあります。[セッション管理](/ja-JP/concepts/session)を参照してください。
  </Accordion>
</AccordionGroup>

## 設定の基本

<AccordionGroup>
  <Accordion title="設定の形式と保存場所は？">
    OpenClaw は `$OPENCLAW_CONFIG_PATH`（デフォルト：`~/.openclaw/openclaw.json`）から任意の **JSON5** 設定を読み込みます。ファイルがない場合は、デフォルトのワークスペース `~/.openclaw/workspace` など、比較的安全なデフォルト値を使用します。
  </Accordion>

  <Accordion title='gateway.bind: "lan"（または "tailnet"）を設定したところ、何もリッスンしない／UI に未認証と表示される'>
    ループバック以外へのバインドには、**有効な Gateway 認証経路が必要です**。共有シークレット認証（トークンまたはパスワード）、または正しく設定された ID 対応リバースプロキシの背後にある `gateway.auth.mode: "trusted-proxy"` を使用します。

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password` は、それだけではローカル Gateway 認証を有効にしません。ローカル呼び出し経路で `gateway.remote.*` をフォールバックとして使用できるのは、`gateway.auth.*` が未設定の場合のみです。
    - パスワード認証では、`gateway.auth.mode: "password"` と `gateway.auth.password`（または `OPENCLAW_GATEWAY_PASSWORD`）を設定します。
    - `gateway.auth.token` / `.password` が SecretRef で明示的に設定されているにもかかわらず解決できない場合、解決処理はフェイルクローズします（リモートフォールバックによる隠蔽は行われません）。
    - 共有シークレットを使用する Control UI 設定では、`connect.params.auth.token` または `connect.params.auth.password`（アプリ／UI 設定に保存）を使用して認証します。Tailscale Serve や `trusted-proxy` など、ID 情報を伴うモードでは代わりにリクエストヘッダーを使用します。共有シークレットを URL に含めないでください。
    - `gateway.auth.mode: "trusted-proxy"` を使用する場合、同一ホスト上のループバックリバースプロキシでは、`gateway.auth.trustedProxy.allowLoopback = true` の明示的な設定と `gateway.trustedProxies` 内のループバックエントリが必要です。

  </Accordion>

  <Accordion title="localhost でもトークンが必要になったのはなぜですか？">
    OpenClaw はループバックを含め、デフォルトで Gateway 認証を強制します。明示的な認証経路が設定されていない場合、起動時にトークンモードが選択され、その起動中のみ有効なトークンが生成されるため、ローカル WS クライアントにも認証が必要です。これにより、他のローカルプロセスが Gateway を呼び出すことを防ぎます。

    クライアントが再起動後も同じシークレットを必要とする場合は、`gateway.auth.token`、`gateway.auth.password`、`OPENCLAW_GATEWAY_TOKEN`、または `OPENCLAW_GATEWAY_PASSWORD` を明示的に設定してください。パスワードモード、または ID 対応リバースプロキシ用の `trusted-proxy` も選択できます。認証なしのループバックを使用するには、`gateway.auth.mode: "none"` を明示的に設定します。`openclaw doctor --generate-gateway-token` を使用すると、いつでもトークンを生成できます。

  </Accordion>

  <Accordion title="設定を変更した後に再起動する必要がありますか？">
    Gateway は設定を監視し、ホットリロードに対応しています。`gateway.reload.mode: "hybrid"`（デフォルト）は、安全な変更をホット適用し、重要な変更では再起動します。`hot`、`restart`、`off` にも対応しています。ほとんどの `tools.*`、`agents.*` ポリシー、`session.*`、`messages.*` の変更は、リロード操作なしですぐに反映されます。`gateway.*` のバインド／ポート変更には再起動が必要です。
  </Accordion>

  <Accordion title="Web 検索（および Web 取得）を有効にするにはどうすればよいですか？">
    `web_fetch` は API キーなしで動作します。`web_search` は選択したプロバイダーによって異なります。

    | プロバイダー | キー不要 | 環境変数 |
    | --- | --- | --- |
    | Brave | いいえ | `BRAVE_API_KEY` |
    | DuckDuckGo | はい（非公式の HTML ベース） | - |
    | Exa | いいえ | `EXA_API_KEY` |
    | Firecrawl | いいえ | `FIRECRAWL_API_KEY` |
    | Gemini | いいえ | `GEMINI_API_KEY` |
    | Grok | いいえ（xAI OAuth またはキー） | `XAI_API_KEY` |
    | Kimi | いいえ | `KIMI_API_KEY` または `MOONSHOT_API_KEY` |
    | MiniMax Search | いいえ | `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、または `MINIMAX_API_KEY` |
    | Ollama Web Search | はい（`ollama signin` が必要） | - |
    | Perplexity | いいえ | `PERPLEXITY_API_KEY` または `OPENROUTER_API_KEY` |
    | SearXNG | はい（セルフホスト） | `SEARXNG_BASE_URL` |
    | Tavily | いいえ | `TAVILY_API_KEY` |

    Grok は、モデル認証（`openclaw onboard --auth-choice xai-oauth`）の xAI OAuth を再利用することもできます。

    **推奨**：`openclaw configure --section web` を実行し、プロバイダーを選択してください。

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // 任意。自動検出する場合は省略
          },
        },
      },
    }
    ```

    プロバイダー固有のウェブ検索設定は `plugins.entries.<plugin>.config.webSearch.*` にあります。従来の `tools.web.search.*` プロバイダーパスも互換性のため引き続き読み込まれますが、新しい設定では使用しないでください。Firecrawl のウェブ取得フォールバック設定は `plugins.entries.firecrawl.config.webFetch.*` にあります。

    - 許可リスト: `web_search`/`web_fetch`/`x_search` を追加するか、3 つすべてを対象にする場合は `group:web` を追加します。
    - `web_fetch` はデフォルトで有効です。
    - `tools.web.fetch.provider` を省略すると、OpenClaw は利用可能な認証情報から、準備が整っている最初の取得フォールバックプロバイダーを自動検出します。公式の Firecrawl Plugin がそのフォールバックを提供します。
    - デーモンは `~/.openclaw/.env`（またはサービス環境）から環境変数を読み取ります。

    ドキュメント: [ウェブツール](/ja-JP/tools/web)。

  </Accordion>

  <Accordion title="config.apply によって設定が消去されました。復旧し、再発を防ぐにはどうすればよいですか？">
    `config.apply` は**設定全体**を置き換えます。一部のみを含むオブジェクトを渡すと、それ以外はすべて削除されます。

    現在の OpenClaw は、誤操作による上書きの大半を防止します。

    - OpenClaw が行う設定の書き込みでは、書き込む前に変更後の設定全体を検証します。
    - OpenClaw による無効または破壊的な書き込みは拒否され、`openclaw.json.rejected.*` として保存されます。
    - 起動またはホットリロードを壊す直接編集が行われた場合、Gateway はフェイルクローズするかリロードをスキップし、`openclaw.json` を書き換えることはありません。
    - `openclaw doctor --fix` が修復を担当し、最後に正常だった状態を復元できます。また、拒否されたファイルを `openclaw.json.clobbered.*` として保存します。

    復旧手順:

    - `openclaw logs --follow` で `Invalid config at`、`Config write rejected:`、または `config reload skipped (invalid config)` を確認します。
    - 有効な設定の隣にある最新の `openclaw.json.clobbered.*` または `openclaw.json.rejected.*` を調べます。
    - `openclaw config validate` と `openclaw doctor --fix` を実行します。
    - `openclaw config set` または `config.patch` を使用して、意図したキーのみを戻します。
    - 最後に正常だった状態や拒否されたペイロードがない場合は、バックアップから復元するか、`openclaw doctor` を再実行してチャンネルとモデルを再設定します。
    - 予期しない消失が発生した場合は、最後に確認できた設定またはバックアップを添えてバグを報告してください。ローカルのコーディングエージェントは、多くの場合、ログや履歴から動作する設定を再構築できます。

    回避方法: 小さな変更には `openclaw config set`、対話形式の編集には `openclaw configure`、不明なパスの確認には `config.schema.lookup`（浅いスキーマノードと直下の子要素の概要を返します）、部分的な RPC 編集には `config.patch` を使用し、`config.apply` は設定全体の置き換えにのみ使用してください。エージェント向けの `gateway` ランタイムツールは、従来の `tools.bash.*` エイリアス経由であっても、`tools.exec.ask` / `tools.exec.security` の書き換えを拒否します。

    ドキュメント: [設定](/ja-JP/cli/config)、[設定ウィザード](/ja-JP/cli/configure)、[Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting#gateway-rejected-invalid-config)、[Doctor](/ja-JP/gateway/doctor)。

  </Accordion>

  <Accordion title="複数のデバイスに特化したワーカーを配置し、中央の Gateway で実行するにはどうすればよいですか？">
    一般的な構成は、**1 つの Gateway**（たとえば Raspberry Pi）と、**ノード**および**エージェント**の組み合わせです。

    - **Gateway（中央）**: チャンネル（Signal/WhatsApp）、ルーティング、セッションを管理します。
    - **ノード（デバイス）**: Mac/iOS/Android が周辺デバイスとして接続し、ローカルツール（`system.run`、`canvas`、`camera`）を公開します。
    - **エージェント（ワーカー）**: 特定の役割（たとえば運用データと個人データ）ごとに分離された頭脳とワークスペースです。
    - **サブエージェント**: 並列処理のため、メインエージェントからバックグラウンド作業を生成します。
    - **TUI**: Gateway に接続し、エージェントやセッションを切り替えます。

    ドキュメント: [ノード](/ja-JP/nodes)、[リモートアクセス](/ja-JP/gateway/remote)、[マルチエージェントルーティング](/ja-JP/concepts/multi-agent)、[サブエージェント](/ja-JP/tools/subagents)、[TUI](/ja-JP/web/tui)。

  </Accordion>

  <Accordion title="OpenClaw ブラウザーはヘッドレスで実行できますか？">
    はい。

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    デフォルトは `false`（ヘッドあり）です。ヘッドレスでは、一部のサイトでボット対策チェックが作動する可能性が高くなります（X/Twitter はヘッドレスセッションを頻繁にブロックします）。同じ Chromium エンジンを使用し、ほとんどの自動化で動作します。主な違いはブラウザーウィンドウが表示されないことです（画面の確認にはスクリーンショットを使用してください）。[ブラウザー](/ja-JP/tools/browser)を参照してください。

  </Accordion>

  <Accordion title="ブラウザー操作に Brave を使用するにはどうすればよいですか？">
    `browser.executablePath` を Brave のバイナリ（または任意の Chromium ベースのブラウザー）に設定し、Gateway を再起動します。[ブラウザー](/ja-JP/tools/browser#use-brave-or-another-chromium-based-browser)を参照してください。
  </Accordion>
</AccordionGroup>

## リモート Gateway とノード

<AccordionGroup>
  <Accordion title="Telegram、Gateway、ノードの間でコマンドはどのように伝達されますか？">
    Telegram メッセージは**Gateway**によって処理されます。Gateway がエージェントを実行し、ノードツールが必要な場合にのみ、**Gateway WebSocket** 経由でノードを呼び出します。

    Telegram -> Gateway -> エージェント -> `node.*` -> Node -> Gateway -> Telegram

    ノードはプロバイダーからの受信トラフィックを認識せず、ノードの RPC 呼び出しのみを受信します。

  </Accordion>

  <Accordion title="Gateway がリモートでホストされている場合、エージェントから自分のコンピューターにアクセスするにはどうすればよいですか？">
    コンピューターを**ノード**としてペアリングします。Gateway は別の場所で実行されますが、Gateway WebSocket 経由でローカルマシン上の `node.*` ツール（画面、カメラ、システム）を呼び出せます。

    1. 常時稼働するホスト（VPS/ホームサーバー）で Gateway を実行します。
    2. Gateway ホストとコンピューターを同じ tailnet に配置します。
    3. Gateway WS に到達できることを確認します（tailnet へのバインドまたは SSH トンネル）。
    4. macOS アプリをローカルで開き、ノードとして登録されるように **Remote over SSH** モード（または直接 tailnet）で接続します。
    5. ノードを承認します。
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    別個の TCP ブリッジは不要です。ノードは Gateway WebSocket 経由で接続します。

    セキュリティ上の注意: macOS ノードをペアリングすると、そのマシンで `system.run` が許可されます。信頼できるデバイスのみをペアリングし、[セキュリティ](/ja-JP/gateway/security)を確認してください。

    ドキュメント: [ノード](/ja-JP/nodes)、[Gateway プロトコル](/ja-JP/gateway/protocol)、[macOS リモートモード](/ja-JP/platforms/mac/remote)、[セキュリティ](/ja-JP/gateway/security)。

  </Accordion>

  <Accordion title="Tailscale は接続されていますが、応答がありません。どうすればよいですか？">
    基本事項を確認します。

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    次に、認証とルーティングを確認します。Tailscale Serve を使用している場合は、`gateway.auth.allowTailscale` が正しく設定されていることを確認します。SSH トンネル経由で接続している場合は、トンネルが稼働し、正しいポートを指していることを確認します。また、DM/グループの許可リストに自分のアカウントが含まれていることを確認します。

    ドキュメント: [Tailscale](/ja-JP/gateway/tailscale)、[リモートアクセス](/ja-JP/gateway/remote)、[チャンネル](/ja-JP/channels)。

  </Accordion>

  <Accordion title="2 つの OpenClaw インスタンス（ローカル + VPS）を相互に通信させることはできますか？">
    はい。ただし、ボット間のブリッジは組み込まれていません。

    **最も簡単な方法**: 両方のボットがアクセスできる通常のチャットチャンネル（Slack/Telegram/WhatsApp）を使用します。ボット A からボット B にメッセージを送り、通常どおりボット B に返信させます。

    **CLI ブリッジ（汎用）**: `openclaw agent --message ... --deliver` を使用して相手の Gateway を呼び出し、相手のボットが監視しているチャットを対象にするスクリプトを実行します。一方のボットがリモート VPS 上にある場合は、SSH/Tailscale 経由で CLI の接続先をそのリモート Gateway に設定します（[リモートアクセス](/ja-JP/gateway/remote)を参照）。

    ```bash
    openclaw agent --message "ローカルボットからこんにちは" --deliver --channel telegram --reply-to <chat-id>
    ```

    2 つのボットが無限に応答し合わないよう、ガードレールを追加します（メンション時のみ、チャンネル許可リスト、または「ボットのメッセージには返信しない」ルール）。

    ドキュメント: [リモートアクセス](/ja-JP/gateway/remote)、[エージェント CLI](/ja-JP/cli/agent)、[エージェント送信](/ja-JP/tools/agent-send)。

  </Accordion>

  <Accordion title="複数のエージェントに個別の VPS が必要ですか？">
    いいえ。1 つの Gateway で複数のエージェントをホストでき、それぞれに独自のワークスペース、モデルのデフォルト設定、ルーティングを設定できます。これが通常の構成であり、エージェントごとに VPS を用意するよりも大幅に安価で簡単です。個別の VPS は、強固な分離（セキュリティ境界）が必要な場合や、共有したくない大きく異なる設定がある場合にのみ使用してください。
  </Accordion>

  <Accordion title="VPS から SSH 接続する代わりに、個人用ノート PC をノードとして使用する利点はありますか？">
    はい。ノードは、リモート Gateway からノート PC にアクセスするための第一級の手段であり、シェルアクセス以上の機能を利用できます。Gateway は macOS/Linux（Windows では WSL2 経由）で動作し、軽量です（小規模な VPS や Raspberry Pi クラスのマシンで十分で、4 GB RAM あれば余裕があります）。そのため、常時稼働するホストと、ノードとして使用するノート PC を組み合わせるのが一般的です。

    - **受信 SSH は不要** - ノードはデバイスのペアリングを通じて Gateway WebSocket に外向きに接続します。
    - **より安全な実行制御** - `system.run` は、そのノート PC 上のノード許可リストと承認によって制限されます。
    - **より多くのデバイスツール** - ノードは `system.run` に加えて、`canvas`、`camera`、`screen` を公開します。
    - **ローカルブラウザーの自動化** - Gateway を VPS 上で稼働させたまま、ノードホストを通じて Chrome をローカルで実行するか、Chrome MCP 経由でローカルの Chrome に接続できます。

    一時的なシェルアクセスには SSH で十分ですが、継続的なエージェントワークフローやデバイス自動化にはノードの方が簡単です。

    ドキュメント: [ノード](/ja-JP/nodes)、[ノード CLI](/ja-JP/cli/nodes)、[ブラウザー](/ja-JP/tools/browser)。

  </Accordion>

  <Accordion title="ノードは Gateway サービスを実行しますか？">
    いいえ。意図的に分離されたプロファイルを実行する場合を除き、ホストごとに実行する **Gateway は 1 つだけ**にしてください（[複数の Gateway](/ja-JP/gateway/multiple-gateways)を参照）。ノードは Gateway に接続する周辺デバイスです（iOS/Android ノード、またはメニューバーアプリの macOS「ノードモード」）。ヘッドレスのノードホストと CLI 操作については、[ノードホスト CLI](/ja-JP/cli/node)を参照してください。

    `gateway`、`discovery`、およびホストされる Plugin サーフェスの変更には、完全な再起動が必要です。

  </Accordion>

  <Accordion title="API / RPC を使用して設定を適用できますか？">
    はい。

    - `config.schema.lookup`: 書き込む前に、浅いスキーマノード、一致する UI ヒント、直下の子要素の概要とともに、設定の 1 つのサブツリーを確認します。
    - `config.get`: 現在のスナップショットとハッシュを取得します。
    - `config.patch`: 安全な部分更新（ほとんどの RPC 編集で推奨）。可能な場合はホットリロードし、必要な場合は再起動します。
    - `config.apply`: 設定全体を検証して置き換えます。可能な場合はホットリロードし、必要な場合は再起動します。
    - エージェント向けの `gateway` ランタイムツールは、引き続き `tools.exec.ask` / `tools.exec.security` の書き換えを拒否します。従来の `tools.bash.*` エイリアスは、同じ保護対象のパスに正規化されます。

  </Accordion>

  <Accordion title="初回インストール向けの最小限かつ妥当な設定">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    ワークスペースを設定し、ボットを起動できるユーザーを制限します。

  </Accordion>

  <Accordion title="VPS に Tailscale をセットアップし、Mac から接続するにはどうすればよいですか？">
    1. **VPS にインストールしてログイン**:
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. Tailscale アプリを使用して **Mac にインストールしてログイン**し、同じ tailnet に接続します。
    3. Tailscale 管理コンソールで **MagicDNS を有効化**し、VPS に安定した名前を割り当てます。
    4. **tailnet ホスト名を使用**します: SSH `ssh user@your-vps.tailnet-xxxx.ts.net`; Gateway WS `ws://your-vps.tailnet-xxxx.ts.net:18789`。

    SSH を使わずに Control UI を利用する場合は、VPS で Tailscale Serve を使用します:

    ```bash
    openclaw gateway --tailscale serve
    ```

    これにより Gateway はループバックにバインドされたままになり、Tailscale 経由で HTTPS が公開されます。[Tailscale](/ja-JP/gateway/tailscale)を参照してください。

  </Accordion>

  <Accordion title="Mac の Node をリモート Gateway（Tailscale Serve）に接続するにはどうすればよいですか？">
    Serve は **Gateway Control UI + WS** を公開します。Node は同じ Gateway WS エンドポイント経由で接続します。

    1. VPS と Mac が同じ tailnet に接続されていることを確認します。
    2. macOS アプリを Remote モードで使用します（SSH ターゲットには tailnet ホスト名を指定できます）。これにより Gateway ポートがトンネリングされ、Node として接続されます。
    3. Node を承認します:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    ドキュメント: [Gateway プロトコル](/ja-JP/gateway/protocol)、[検出](/ja-JP/gateway/discovery)、[macOS リモートモード](/ja-JP/platforms/mac/remote)。

  </Accordion>

  <Accordion title="2 台目のノート PC にインストールするべきですか、それとも Node を追加するだけでよいですか？">
    2 台目のノート PC で **ローカルツールのみ**（画面、カメラ、実行）を使用する場合は、**Node** として追加します。Gateway は 1 つのままで、設定も重複しません。ローカル Node ツールは現在 macOS でのみ利用できます。**厳密な分離**が必要な場合、または完全に独立した 2 つのボットを運用する場合にのみ、2 つ目の Gateway をインストールしてください。

    ドキュメント: [Node](/ja-JP/nodes)、[Node CLI](/ja-JP/cli/nodes)、[複数の Gateway](/ja-JP/gateway/multiple-gateways)。

  </Accordion>
</AccordionGroup>

## 環境変数と .env の読み込み

<AccordionGroup>
  <Accordion title="OpenClaw は環境変数をどのように読み込みますか？">
    OpenClaw は親プロセス（シェル、launchd/systemd、CI など）から環境変数を読み取り、さらに以下を読み込みます:

    - 現在の作業ディレクトリにある `.env`。
    - `~/.openclaw/.env` にあるグローバルフォールバック `.env`（`$OPENCLAW_STATE_DIR/.env`）。

    どちらの `.env` ファイルも、既存の環境変数を上書きしません。ただし、ワークスペースの `.env` では、プロバイダーの認証情報およびエンドポイントルーティングキーは例外です。`GEMINI_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY` などのキー、または `_ENDPOINT` で終わる任意のキー（およびその他の同梱プロバイダーの認証またはエンドポイント環境変数）は、ワークスペースの `.env` からは無視されます。これらはプロセス環境、`~/.openclaw/.env`、または設定 `env` に配置してください。

    設定内のインライン環境変数は、プロセス環境に存在しない場合にのみ適用されます:

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    完全な優先順位とソースについては、[/environment](/ja-JP/help/environment)を参照してください。

  </Accordion>

  <Accordion title="サービス経由で Gateway を起動したところ、環境変数が消えました。どうすればよいですか？">
    解決方法は 2 つあります:

    1. 不足しているキーを `~/.openclaw/.env` に配置すると、サービスがシェル環境を継承しない場合でも読み込まれます。
    2. シェルインポートを有効にします（任意の利便機能）:
       ```json5
       {
         env: {
           shellEnv: {
             enabled: true,
             timeoutMs: 15000,
           },
         },
       }
       ```
       これによりログインシェルが実行され、不足している想定キーのみがインポートされます（上書きは一切行いません）。対応する環境変数: `OPENCLAW_LOAD_SHELL_ENV=1`、`OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`。

  </Accordion>

  <Accordion title='COPILOT_GITHUB_TOKEN を設定しましたが、モデルのステータスに「Shell env: off.」と表示されます。なぜですか？'>
    `openclaw models status` は、**シェル環境のインポート**が有効かどうかを報告します。「Shell env: off」は環境変数が不足しているという意味ではありません。OpenClaw がログインシェルを自動的に読み込まないという意味にすぎません。

    Gateway がサービス（launchd/systemd）として実行されている場合、シェル環境は継承されません。トークンを `~/.openclaw/.env` に配置するか、`env.shellEnv.enabled: true` を有効にするか、設定 `env` に追加して（存在しない場合にのみ適用）、Gateway を再起動してから再確認してください:

    ```bash
    openclaw models status
    ```

    Copilot トークンは、`OPENCLAW_GITHUB_TOKEN`、`COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、`GITHUB_TOKEN` の順に解決されます。

    [/concepts/model-providers](/ja-JP/concepts/model-providers)および[/environment](/ja-JP/help/environment)を参照してください。

  </Accordion>
</AccordionGroup>

## セッションと複数のチャット

<AccordionGroup>
  <Accordion title="新しい会話を開始するにはどうすればよいですか？">
    `/new` または `/reset` を単独のメッセージとして送信します。[セッション管理](/ja-JP/concepts/session)を参照してください。
  </Accordion>

  <Accordion title="/new を送信しなかった場合、セッションは自動的にリセットされますか？">
    いいえ、デフォルトではリセットされません。セッションは同じ `sessionId` を維持し、会話が長くなると Compaction によってアクティブなモデルコンテキストが制限されます。`/new` と `/reset` は引き続き使用できます。また、`mode: "daily"` または `mode: "idle"` を使用して自動リセットを有効にできます。日次モードは Gateway ホスト上の `session.reset.atHour`（デフォルトは `4`、0-23）で切り替わります。アイドルモードでは、Heartbeat/Cron/実行システムイベントではなく、最後の実際の操作からの `session.reset.idleMinutes` が使用されます。

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType` は `direct`、`group`、`thread` をサポートします。Doctor は従来の `dm` エントリを `direct` に移行します。スキーマは `dm` を拒否します。従来のトップレベル `session.idleMinutes` は、`session.reset`/`resetByType` ブロックが設定されていない場合、アイドルモードのデフォルトに対する互換エイリアスとして引き続き機能します。ライフサイクル全体については、[セッション管理](/ja-JP/concepts/session)を参照してください。

  </Accordion>

  <Accordion title="OpenClaw インスタンスのチーム（1 人の CEO と多数のエージェント）を構成できますか？">
    はい。**マルチエージェントルーティング**と**サブエージェント**を使用できます。1 つのコーディネーターエージェントと、それぞれ独自のワークスペースおよびモデルを持つ複数のワーカーエージェントで構成します。

    これは楽しい実験として捉えるのが最適です。トークン消費量が多く、個別のセッションを持つ 1 つのボットより非効率になることがよくあります。一般的な構成は、会話するボットを 1 つ使用し、並列作業には異なるセッションを用意し、必要に応じてサブエージェントを起動するものです。

    ドキュメント: [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)、[サブエージェント](/ja-JP/tools/subagents)、[エージェント CLI](/ja-JP/cli/agents)。

  </Accordion>

  <Accordion title="タスクの途中でコンテキストが切り詰められたのはなぜですか？防ぐにはどうすればよいですか？">
    セッションコンテキストはモデルのウィンドウによって制限されます。長いチャット、大量のツール出力、または多数のファイルによって、Compaction や切り詰めが発生することがあります。

    - ボットに現在の状態を要約してファイルに書き込むよう依頼します。
    - 長いタスクの前には `/compact` を使用し、トピックを切り替える際には `/new` を使用します。
    - 重要なコンテキストをワークスペースに保存し、ボットに再度読み込むよう依頼します。
    - 長時間または並列の作業にはサブエージェントを使用し、メインチャットを小さく保ちます。
    - 頻繁に発生する場合は、コンテキストウィンドウがより大きいモデルを選択します。

  </Accordion>

  <Accordion title="OpenClaw をインストールしたまま完全にリセットするにはどうすればよいですか？">
    ```bash
    openclaw reset
    ```

    非対話式の完全リセット:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    その後、セットアップを再実行します:

    ```bash
    openclaw onboard --install-daemon
    ```

    既存の設定が検出された場合、オンボーディングには **リセット** オプションも表示されます。[オンボーディング（CLI）](/ja-JP/start/wizard)を参照してください。プロファイル（`--profile` / `OPENCLAW_PROFILE`）を使用していた場合は、各状態ディレクトリ（デフォルトは `~/.openclaw-<profile>`）をリセットしてください。開発専用のリセット: `openclaw gateway --dev --reset` は開発設定、認証情報、セッション、ワークスペースを消去します。

  </Accordion>

  <Accordion title='「context too large」エラーが発生します。リセットまたは Compaction するにはどうすればよいですか？'>
    - **Compaction**（会話を維持し、以前のやり取りを要約）: `/compact`、または要約を指示するには `/compact <instructions>`。
    - **リセット**（同じチャットキーに対する新しいセッション ID）: `/new` または `/reset`。

    繰り返し発生する場合は、**セッションプルーニング**（`agents.defaults.contextPruning`）を調整して古いツール出力を削減するか、コンテキストウィンドウがより大きいモデルを使用してください。

    ドキュメント: [Compaction](/ja-JP/concepts/compaction)、[セッションプルーニング](/ja-JP/concepts/session-pruning)、[セッション管理](/ja-JP/concepts/session)。

  </Accordion>

  <Accordion title='「LLM request rejected: messages.content.tool_use.input field required」と表示されるのはなぜですか？'>
    プロバイダーの検証エラーです。モデルが必須の `input` を含まない `tool_use` ブロックを出力しました。通常、セッション履歴が古いか破損していることを示します（長いスレッドの後や、ツールまたはスキーマの変更後によく発生します）。

    解決方法: `/new` を単独のメッセージとして送信し、新しいセッションを開始します。

  </Accordion>

  <Accordion title="30 分ごとに Heartbeat メッセージが届くのはなぜですか？">
    Heartbeat はデフォルトで **30m** ごとに実行されます。解決された認証モードが Anthropic OAuth/トークン認証（Claude CLI の再利用を含む）で、`heartbeat.every` が未設定の場合は **1h** ごとです。調整または無効化するには:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    `HEARTBEAT.md` が存在していても実質的に空の場合（空白行、Markdown/HTML コメント、ATX 見出し、フェンスマーカー、または空のリスト項目スタブのみ）、OpenClaw は API 呼び出しを節約するため Heartbeat の実行をスキップします。ファイルが存在しない場合でも Heartbeat は実行され、モデルが何をするかを判断します。

    エージェントごとの上書きには `agents.entries.*.heartbeat` を使用します。ドキュメント: [Heartbeat](/ja-JP/gateway/heartbeat)。

  </Accordion>

  <Accordion title='WhatsApp グループに「ボットアカウント」を追加する必要がありますか？'>
    いいえ。OpenClaw は **自分のアカウント**で実行されます。自分がグループに参加していれば、OpenClaw もそのグループを認識できます。デフォルトでは、送信者を許可するまでグループへの返信はブロックされます（`groupPolicy: "allowlist"`）。

    グループへの返信を自分だけに制限するには:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="WhatsApp グループの JID を取得するにはどうすればよいですか？">
    最も速い方法は、ログを追跡しながらグループでテストメッセージを送信することです。

    ```bash
    openclaw logs --follow --json
    ```

    `@g.us` で終わる `chatId`（または `from`）を探します。例: `1234567890-1234567890@g.us`。

    すでに設定済みまたは許可リストに登録済みの場合は、設定からグループを一覧表示します:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    ドキュメント: [WhatsApp](/ja-JP/channels/whatsapp)、[ディレクトリ](/ja-JP/cli/directory)、[ログ](/ja-JP/cli/logs)。

  </Accordion>

  <Accordion title="OpenClaw がグループで返信しないのはなぜですか？">
    よくある原因は 2 つあります。メンションゲートがデフォルトで有効になっている（ボットを @メンションするか、`mentionPatterns` に一致させる必要があります）か、`"*"` を指定せずに `channels.whatsapp.groups` を設定しており、そのグループが許可リストに登録されていない場合です。

    [グループ](/ja-JP/channels/groups)および[グループメッセージ](/ja-JP/channels/group-messages)を参照してください。

  </Accordion>

  <Accordion title="グループやスレッドは DM とコンテキストを共有しますか？">
    ダイレクトチャットはデフォルトでメインセッションに統合されます。グループやチャンネルには独自のセッションキーがあり、Telegram のトピックと Discord のスレッドは個別のセッションになります。[グループ](/ja-JP/channels/groups)および[グループメッセージ](/ja-JP/channels/group-messages)を参照してください。
  </Accordion>

  <Accordion title="作成できるワークスペースとエージェントの数はいくつですか？">
    厳密な上限はありません。数十個、さらには数百個でも問題ありませんが、以下に注意してください:

    - **ディスク使用量の増加**: アクティブなセッションとトランスクリプトはエージェントごとの SQLite データベースに保存されます。レガシー／アーカイブの成果物は引き続き `~/.openclaw/agents/<agentId>/sessions/` 配下に蓄積する可能性があります。
    - **トークンコスト**: エージェントが増えるほど、モデルの同時使用量も増えます。
    - **運用オーバーヘッド**: エージェントごとの認証プロファイル、ワークスペース、チャネルルーティング。

    エージェントごとに **アクティブな** ワークスペースを 1 つ (`agents.defaults.workspace`) に保ち、ディスク使用量が増えた場合は `openclaw sessions cleanup` で古いセッションを削除し（アクティブな SQLite の状態を手動で編集しないでください）、`openclaw doctor` を使用して不要なワークスペースやプロファイルの不一致を見つけます。

  </Accordion>

  <Accordion title="複数のボットまたはチャットを同時に実行できますか（Slack）。また、どのように設定すればよいですか？">
    はい、**マルチエージェントルーティング**を使用できます。複数の分離されたエージェントを実行し、受信メッセージをチャネル／アカウント／ピア別にルーティングします。Slack はチャネルとしてサポートされており、特定のエージェントにバインドできます。

    ブラウザーアクセスは強力ですが、「人間ができることなら何でもできる」わけではありません。ボット対策、CAPTCHA、MFA によって自動化がブロックされる場合があります。最も信頼性の高い制御を行うには、ホスト上のローカル Chrome MCP、またはブラウザーを実際に実行しているマシン上の CDP を使用してください。

    推奨設定: 常時稼働の Gateway ホスト（VPS／Mac mini）、役割ごとに 1 つのエージェント（バインディング）、それらのエージェントにバインドされた Slack チャネル、必要に応じて Chrome MCP または Node 経由のローカルブラウザー。

    ドキュメント: [マルチエージェントルーティング](/ja-JP/concepts/multi-agent)、[Slack](/ja-JP/channels/slack)、[ブラウザー](/ja-JP/tools/browser)、[Node](/ja-JP/nodes)。

  </Accordion>
</AccordionGroup>

## モデル、フェイルオーバー、認証プロファイル

デフォルト、選択、エイリアス、切り替え、フェイルオーバー、認証プロファイルに関するモデルの Q&A は、[モデル FAQ](/ja-JP/help/faq-models) にあります。

## Gateway: ポート、「すでに実行中」、リモートモード

<AccordionGroup>
  <Accordion title="Gateway はどのポートを使用しますか？">
    `gateway.port` は、WebSocket + HTTP（Control UI、フックなど）用の単一の多重化ポートを制御します。優先順位:

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > デフォルト 18789
    ```

  </Accordion>

  <Accordion title='openclaw gateway status で「Runtime: running」と表示されるのに、「Connectivity probe: failed」と表示されるのはなぜですか？'>
    「Running」は **スーパーバイザー**（launchd／systemd／schtasks）から見た状態です。一方、接続プローブでは CLI が実際に Gateway WebSocket へ接続します。`openclaw gateway status` の次の行を信頼してください: `Probe target:`（プローブが使用した URL）、`Listening:`（ポートに実際にバインドされているもの）、`Last gateway error:`（プロセスは稼働中でもポートがリッスンしていない場合によくある根本原因）。
  </Accordion>

  <Accordion title='openclaw gateway status で「Config (cli)」と「Config (service)」が異なるのはなぜですか？'>
    サービスが使用しているものとは別の設定ファイルを編集しています（多くの場合、`--profile` と `OPENCLAW_STATE_DIR` の不一致です）。

    修正するには、サービスで使用する `--profile`／環境と同じものから次を実行します:

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='「another gateway instance is already listening」とはどういう意味ですか？'>
    OpenClaw は起動直後に WebSocket リスナー（デフォルト `ws://127.0.0.1:18789`）をバインドすることで、ランタイムロックを適用します。`EADDRINUSE` によりバインドに失敗すると、`GatewayLockError`（「another gateway instance is already listening」）をスローします。

    修正: もう一方のインスタンスを停止するか、ポートを解放するか、`openclaw gateway --port <port>` を指定して実行します。

  </Accordion>

  <Accordion title="OpenClaw をリモートモード（クライアントが別の場所にある Gateway に接続）で実行するにはどうすればよいですか？">
    `gateway.mode: "remote"` を設定してリモート WebSocket URL を指定し、必要に応じて共有シークレットのリモート認証情報を設定します:

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` は、`gateway.mode` が `local` の場合（またはオーバーライドフラグを渡した場合）にのみ起動します。
    - macOS アプリは設定ファイルを監視し、これらの値が変更されると即座にモードを切り替えます。
    - `gateway.remote.token`／`.password` はクライアント側のリモート認証情報にすぎず、それ自体ではローカル Gateway の認証を有効にしません。

  </Accordion>

  <Accordion title='Control UI に「unauthorized」と表示される（または再接続を繰り返す）場合はどうすればよいですか？'>
    Gateway の認証経路と UI の認証方式が一致していません。

    事実（コードに基づく）:

    - Control UI はトークンを `sessionStorage` に保持し、現在のブラウザータブと選択された Gateway URL にスコープを限定します。そのため、localStorage にトークンを長期間保持しなくても、同じタブでの更新は引き続き機能します。
    - `AUTH_TOKEN_MISMATCH` では、Gateway が再試行ヒント（`canRetryWithDeviceToken=true`、`recommendedNextStep=retry_with_device_token`）を返した場合、信頼済みクライアントはキャッシュされたデバイストークンを使用して、上限 1 回の再試行を行えます。
    - そのキャッシュトークンによる再試行では、デバイストークンとともに保存された承認済みスコープを再利用します。明示的な `deviceToken`／明示的な `scopes` の呼び出し元は、キャッシュされたスコープを継承せず、要求したスコープセットを維持します。
    - この再試行経路以外では、接続認証の優先順位は、明示的な共有トークン／パスワード、明示的な `deviceToken`、保存済みデバイストークン、ブートストラップトークンの順です。
    - 組み込みのセットアップコードによるブートストラップは、`scopes: []` を持つ Node デバイストークンに加えて、信頼済みモバイルオンボーディング用の有効範囲が限定されたオペレーターハンドオフトークンを返します。オペレーターハンドオフはセットアップ時のネイティブ設定を読み取れますが、ペアリング変更スコープや `operator.admin` は付与しません。

    修正:

    - 最速: `openclaw dashboard`（ダッシュボード URL を表示してコピーし、開くことを試みます。ヘッドレス環境では SSH のヒントを表示します）。
    - トークンがまだない場合: `openclaw doctor --generate-gateway-token`。
    - リモートの場合: まず `ssh -N -L 18789:127.0.0.1:18789 user@host` でトンネルを確立し、次に `http://127.0.0.1:18789/` を開きます。
    - 共有シークレットモード: `gateway.auth.token`／`OPENCLAW_GATEWAY_TOKEN` または `gateway.auth.password`／`OPENCLAW_GATEWAY_PASSWORD` を設定し、一致するシークレットを Control UI の設定に貼り付けます。
    - Tailscale Serve モード: `gateway.auth.allowTailscale` が有効であること、および Tailscale の ID ヘッダーを迂回する未加工のループバック／Tailnet URL ではなく Serve URL を開いていることを確認します。
    - 信頼済みプロキシモード: 設定済みの ID 対応プロキシを経由していることを確認します。同一ホストのループバックプロキシには `gateway.auth.trustedProxy.allowLoopback = true` も必要です。
    - 1 回の再試行後も不一致が続く場合: ペアリング済みデバイストークンをローテーション／再承認します:
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - ローテーションが拒否された場合: ペアリング済みデバイスのセッションは、`operator.admin` も持っていない限り、**自身の** デバイスのみローテーションできます。また、明示的な `--scope` の値は、呼び出し元が現在持つオペレータースコープを超えることはできません。
    - それでも解決しない場合: `openclaw status --all` と [トラブルシューティング](/ja-JP/gateway/troubleshooting) を確認してください。認証の詳細については、[ダッシュボード](/ja-JP/web/dashboard) を参照してください。

  </Accordion>

  <Accordion title="gateway.bind を tailnet に設定しましたが、loopback でしかリッスンしません">
    `tailnet` バインドは、ネットワークインターフェースから Tailscale IP（100.64.0.0/10）を選択します。マシンが Tailscale に接続されていない場合（またはインターフェースがダウンしている場合）、Gateway は別のネットワークインターフェースを公開せず、loopback にフォールバックします。

    修正: そのホストで Tailscale を起動して Gateway を再起動するか、明示的に `gateway.bind: "loopback"`／`"lan"` に切り替えます。

    `tailnet` は明示的です。`auto` は loopback を優先します。必要な同一ホストの `127.0.0.1` リスナーを維持しながら、loopback 以外への公開を Tailnet に限定するには、`gateway.bind: "tailnet"` を使用します。

  </Accordion>

  <Accordion title="同じホストで複数の Gateway を実行できますか？">
    通常はできません。1 つの Gateway で複数のメッセージングチャネルとエージェントを実行できます。複数の Gateway は、冗長性（たとえばレスキューボット）または厳密な分離が必要な場合にのみ使用し、それぞれ固有の `OPENCLAW_CONFIG_PATH`、`OPENCLAW_STATE_DIR`、`agents.defaults.workspace`、`gateway.port` で分離してください。

    推奨: インスタンスごとに `openclaw --profile <name> ...`（`~/.openclaw-<name>` を自動作成）、プロファイル設定ごとに一意の `gateway.port`（手動実行の場合は `--port`）、および `openclaw --profile <name> gateway install` を指定したプロファイルごとのサービス。

    プロファイルはサービス名にもサフィックスを付けます: launchd `ai.openclaw.<profile>`、systemd `openclaw-gateway-<profile>.service`、Windows `OpenClaw Gateway (<profile>)`。修飾されていない `openclaw-gateway` systemd ユニットはデフォルトプロファイルにのみ存在します。名前変更前のレガシー systemd ユニット名 `clawdbot-gateway` は自動的に移行されます。

    完全なガイド: [複数の Gateway](/ja-JP/gateway/multiple-gateways)。

  </Accordion>

  <Accordion title='「invalid handshake」／コード 1008 とはどういう意味ですか？'>
    Gateway は **WebSocket サーバー**であり、最初のメッセージとして `connect` フレームを期待します。それ以外の場合、接続は **コード 1008**（ポリシー違反）で閉じられます。

    一般的な原因: WS クライアントではなくブラウザーで **HTTP** URL を開いた、ポート／パスが間違っている、またはプロキシ／トンネルが認証ヘッダーを取り除いたか、Gateway 以外のリクエストを送信した。

    修正: WS URL（`ws://<host>:18789`、または HTTPS 経由の `wss://...`）を使用し、通常のブラウザータブで WS ポートを開かず、認証が有効な場合は `connect` フレームにトークン／パスワードを含めます。CLI／TUI の例:

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    プロトコルの詳細: [Gateway プロトコル](/ja-JP/gateway/protocol)。

  </Accordion>
</AccordionGroup>

## ロギングとデバッグ

<AccordionGroup>
  <Accordion title="ログはどこにありますか？">
    ファイルログ（構造化）: デフォルトプロファイルでは `/tmp/openclaw/openclaw-YYYY-MM-DD.log`、名前付きプロファイルでは `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`。固定パスは `logging.file`、ファイルのログレベルは `logging.level`、コンソールの詳細度は `--verbose` と `logging.consoleLevel` で設定します。

    最速で追跡する方法:

    ```bash
    openclaw logs --follow
    ```

    サービス／スーパーバイザーのログ（Gateway が launchd／systemd 経由で実行される場合）:

    - macOS launchd の標準出力: `~/Library/Logs/openclaw/gateway.log`（プロファイルでは `gateway-<profile>.log` を使用。標準エラー出力は抑制されます）。
    - Linux: `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`。
    - Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`。

    詳細については、[トラブルシューティング](/ja-JP/gateway/troubleshooting) を参照してください。

  </Accordion>

  <Accordion title="Gateway サービスを開始／停止／再起動するにはどうすればよいですか？">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    Gateway を手動で実行している場合、`openclaw gateway --force` でポートを再取得できます。[Gateway](/ja-JP/gateway) を参照してください。

  </Accordion>

  <Accordion title="Windows でターミナルを閉じました。OpenClaw を再起動するにはどうすればよいですか？">
    Windows には 3 つのインストールモードがあります:

    **1) Windows Hub のローカルセットアップ**: ネイティブアプリが、アプリ所有のローカル WSL Gateway を管理します。スタートメニューまたはトレイから **OpenClaw Companion** を開き、**Gateway Setup** または Connections タブを使用します。

    **2) 手動 WSL2 Gateway**: Gateway は Linux 内で実行されます。
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    サービスをインストールしていない場合は、フォアグラウンドで起動します: `openclaw gateway run`。

    **3) ネイティブ Windows CLI／Gateway**: Windows 上で直接実行されます。
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    手動で実行する場合（サービスなし）: `openclaw gateway run`。

    ドキュメント: [Windows](/ja-JP/platforms/windows)、[Gateway サービス運用手順書](/ja-JP/gateway)。

  </Accordion>

  <Accordion title="Gateway は起動していますが、返信がまったく届きません。何を確認すればよいですか？">
    簡単なヘルスチェック:

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    一般的な原因: **Gateway ホスト**でモデル認証が読み込まれていない（`models status` を確認）、チャネルのペアリング／許可リストによって返信がブロックされている（チャネル設定とログを確認）、または正しいトークンなしで WebChat／ダッシュボードを開いている。リモートの場合は、トンネル／Tailscale 接続が稼働しており、Gateway WebSocket に到達できることを確認してください。

    ドキュメント: [チャンネル](/ja-JP/channels)、[トラブルシューティング](/ja-JP/gateway/troubleshooting)、[リモートアクセス](/ja-JP/gateway/remote)。

  </Accordion>

  <Accordion title='"Gateway から切断されました: 理由なし" - どうすればよいですか？'>
    通常、UI が WebSocket 接続を失ったことを意味します。次を確認してください: Gateway は実行中ですか (`openclaw gateway status`)？正常ですか (`openclaw status`)？UI に正しいトークンが設定されていますか (`openclaw dashboard`)？リモートの場合、トンネル/Tailscale リンクは接続されていますか？

    次に、ログを追跡します:

    ```bash
    openclaw logs --follow
    ```

    ドキュメント: [ダッシュボード](/ja-JP/web/dashboard)、[リモートアクセス](/ja-JP/gateway/remote)、[トラブルシューティング](/ja-JP/gateway/troubleshooting)。

  </Accordion>

  <Accordion title="Telegram の setMyCommands が失敗します。何を確認すればよいですか？">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    次に、該当するエラーを確認します:

    - `BOT_COMMANDS_TOO_MUCH`: Telegram メニューの項目が多すぎます。OpenClaw はすでに Telegram の上限に合わせて項目を削減し、コマンド数を減らして再試行しますが、一部のメニュー項目は引き続き除外される場合があります。Plugin/Skills/カスタムコマンドを減らすか、メニューが不要な場合は `channels.telegram.commands.native` を無効にしてください。
    - `TypeError: fetch failed`、`Network request for 'setMyCommands' failed!`、または同様のネットワークエラー: VPS 上またはプロキシの背後で実行している場合は、外向きの HTTPS が許可され、`api.telegram.org` の DNS が機能していることを確認してください。

    Gateway がリモートにある場合は、Gateway ホスト上のログを確認してください。

    ドキュメント: [Telegram](/ja-JP/channels/telegram)、[チャンネルのトラブルシューティング](/ja-JP/channels/troubleshooting)。

  </Accordion>

  <Accordion title="TUI に出力が表示されません。何を確認すればよいですか？">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    TUI で `/status` を使用して現在の状態を確認してください。チャットチャンネルで返信を受け取る想定の場合は、配信が有効になっていることを確認してください (`/deliver on`)。

    ドキュメント: [TUI](/ja-JP/web/tui)、[スラッシュコマンド](/ja-JP/tools/slash-commands)。

  </Accordion>

  <Accordion title="Gateway を完全に停止してから起動するにはどうすればよいですか？">
    サービスをインストールした場合（macOS では launchd、Linux では systemd）:

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    フォアグラウンドでは Ctrl-C で停止してから、`openclaw gateway run` を実行します。

    ドキュメント: [Gateway サービスのランブック](/ja-JP/gateway)。

  </Accordion>

  <Accordion title="簡単に説明: openclaw gateway restart と openclaw gateway の違い">
    `openclaw gateway restart` は**バックグラウンドサービス**（launchd/systemd）を再起動します。`openclaw gateway` は、このターミナルセッションで Gateway を**フォアグラウンド**実行します。サービスをインストールした場合は Gateway のサブコマンドを使用し、一度だけ実行する場合はサブコマンドなしのフォアグラウンド実行を使用してください。
  </Accordion>

  <Accordion title="問題が発生したときに詳細を確認する最速の方法">
    コンソールに詳細を表示するには `--verbose` を指定して Gateway を起動し、チャンネル認証、モデルルーティング、RPC エラーについてログファイルを調査します。
  </Accordion>
</AccordionGroup>

## メディアと添付ファイル

<AccordionGroup>
  <Accordion title="Skills で画像/PDF が生成されましたが、何も送信されませんでした">
    エージェントから送信する添付ファイルには、`media`、`mediaUrl`、`path`、`filePath` などの構造化メディアフィールドを使用する必要があります。[OpenClaw アシスタントのセットアップ](/ja-JP/start/openclaw)と[エージェント送信](/ja-JP/tools/agent-send)を参照してください。

    ```bash
    openclaw message send --target +15555550123 --message "どうぞ" --media /path/to/file.png
    ```

    次の点も確認してください: 対象チャンネルが外向きメディアに対応し、許可リストでブロックされていないこと。ファイルがプロバイダーのサイズ制限内であること（画像は最大辺 2048px にリサイズされます）。`tools.fs.workspaceOnly=true` は、ローカルパスからの送信をワークスペース、一時領域/メディアストア、サンドボックスで検証済みのファイルに制限します。`tools.fs.workspaceOnly=false`（デフォルト）では、構造化されたローカルメディア送信で、エージェントがすでに読み取り可能なホストローカルファイルを使用できます。対象はメディアおよび安全なドキュメント形式（画像、音声、動画、PDF、Office ドキュメント、Markdown/MD、TXT、JSON、YAML/YML などの検証済みテキストドキュメント）です。これはシークレットスキャナーではありません。拡張子と内容の検証が一致すれば、エージェントが読み取り可能な `secret.txt` または `config.json` を添付できます。機密ファイルはエージェントが読み取り可能なパスの外に置くか、ローカルパスからの送信をより厳格にするため `tools.fs.workspaceOnly=true` を維持してください。

    [画像](/ja-JP/nodes/images)を参照してください。

  </Accordion>
</AccordionGroup>

## セキュリティとアクセス制御

<AccordionGroup>
  <Accordion title="OpenClaw を受信 DM に公開しても安全ですか？">
    受信 DM は信頼されていない入力として扱ってください。デフォルト設定でリスクが軽減されます:

    - DM 対応チャンネルのデフォルト動作は**ペアリング**です。不明な送信者にはペアリングコードが送信され、そのメッセージは処理されません。`openclaw pairing approve --channel <channel> [--account <id>] <code>` で承認してください。保留中のリクエストは**チャンネルごとに 3 件**に制限されます。コードが届かなかった場合は `openclaw pairing list --channel <channel> [--account <id>]` を確認してください。
    - DM を一般公開するには、明示的なオプトインが必要です（`dmPolicy: "open"` と許可リスト `"*"`）。

    リスクのある DM ポリシーを検出するには、`openclaw doctor` を実行してください。

  </Accordion>

  <Accordion title="プロンプトインジェクションは公開ボットだけの問題ですか？">
    いいえ。プロンプトインジェクションで問題となるのは、ボットに DM を送信できる人物だけではなく、**信頼されていないコンテンツ**です。アシスタントが外部コンテンツ（Web 検索/取得、ブラウザページ、メール、ドキュメント、添付ファイル、貼り付けられたログ）を読み取る場合、そのコンテンツにはモデルを乗っ取ろうとする指示が含まれている可能性があります。送信者が自分だけの場合でも同様です。

    最大のリスクは、ツールが有効になっている場合です。モデルがだまされてコンテキストを外部へ漏えいしたり、ユーザーに代わってツールを呼び出したりする可能性があります。影響範囲を縮小してください:

    - 信頼されていないコンテンツの要約には、読み取り専用またはツールを無効にした「リーダー」エージェントを使用する
    - ツールが有効なエージェントでは、`web_search` / `web_fetch` / `browser` を無効にしておく
    - デコードされたファイル/ドキュメントのテキストも信頼されていないものとして扱う: OpenResponses の `input_file` とメディア添付ファイルの抽出はどちらも、生のファイルテキストを渡すのではなく、抽出したテキストを明示的な外部コンテンツ境界マーカーで囲む
    - サンドボックスを使用し、厳格なツール許可リストを適用する

    詳細: [セキュリティ](/ja-JP/gateway/security)。

  </Accordion>

  <Accordion title="OpenClaw は Rust/WASM ではなく TypeScript/Node を使用しているため、安全性が低いですか？">
    言語とランタイムも重要ですが、個人用エージェントにおける主なリスクではありません。実際のリスクは、Gateway の公開範囲、ボットにメッセージを送信できる人物、プロンプトインジェクション、ツールの権限範囲、認証情報の取り扱い、ブラウザへのアクセス、コマンド実行へのアクセス、サードパーティ製 Skills/Plugin の信頼性です。

    Rust と WASM は一部のコード分類でより強力な分離を提供できますが、プロンプトインジェクション、不適切な許可リスト、公開された Gateway、過度に広いツール権限、機密性の高いアカウントにすでにログインしているブラウザプロファイルは解決できません。次を主要な制御策として扱ってください: Gateway を非公開または認証付きにする、DM/グループにペアリングと許可リストを使用する、信頼されていない入力に対して危険なツールを拒否またはサンドボックス化する、信頼できる Plugin と Skills のみをインストールする、設定変更後に `openclaw security audit --deep` を実行する。

    詳細: [セキュリティ](/ja-JP/gateway/security)、[サンドボックス化](/ja-JP/gateway/sandboxing)。

  </Accordion>

  <Accordion title="公開された OpenClaw インスタンスに関する報告を見ました。何を確認すればよいですか？">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    より安全な基準: Gateway を `loopback` にバインドするか、認証済みのプライベートアクセス（tailnet、SSH トンネル、トークン/パスワード認証、または正しく設定された信頼済みプロキシ）経由でのみ公開する。DM を `pairing` または `allowlist` モードにする。全メンバーが信頼できる場合を除き、グループを許可リストに登録してメンションを必須にする。信頼されていないコンテンツを読み取るエージェントでは、高リスクのツール（`exec`、`browser`、`gateway`、`cron`）を拒否するか、権限範囲を厳しく制限する。ツール実行の影響範囲を縮小する必要がある場合は、サンドボックス化を有効にする。

    認証なしの公開バインド、ツールが有効な状態で公開されている DM/グループ、公開されたブラウザ制御を最初に修正してください。詳細: [openclaw security audit](/ja-JP/gateway/security#openclaw-security-audit)。

  </Accordion>

  <Accordion title="ClawHub の Skills とサードパーティ製 Plugin は安全にインストールできますか？">
    サードパーティ製の Skills と Plugin は、信頼することを自ら選択するコードとして扱ってください。ClawHub の Skills ページではインストール前にスキャン状態を確認できますが、スキャンは完全なセキュリティ境界ではありません。OpenClaw は Plugin/Skills のインストールまたは更新時に、組み込みのローカル危険コードブロックを実行しません。ローカルでの許可/ブロックの判断には、運用者が管理する `security.installPolicy` を使用してください。

    より安全な方法: 信頼できる作成者と固定されたバージョンを優先し、有効にする前に Skills/Plugin を確認し、Plugin/Skills の許可リストを限定し、信頼されていない入力を扱うワークフローは最小限のツールを備えたサンドボックス内で実行し、サードパーティ製コードにファイルシステム、コマンド実行、ブラウザ、シークレットへの広範なアクセスを与えないでください。

    詳細: [Skills](/ja-JP/tools/skills)、[Plugin](/ja-JP/tools/plugin)、[セキュリティ](/ja-JP/gateway/security)。

  </Accordion>

  <Accordion title="ボット専用のメール、GitHub アカウント、電話番号を用意すべきですか？">
    ほとんどの構成では、用意することを推奨します。ボットに別のアカウントと電話番号を使用すると、問題が発生した場合の影響範囲を縮小でき、個人アカウントに影響を与えずに認証情報をローテーションしたり、アクセスを取り消したりしやすくなります。

    小さく始めてください。実際に必要なツールとアカウントだけにアクセスを許可し、必要に応じて後から拡張します。

    ドキュメント: [セキュリティ](/ja-JP/gateway/security)、[ペアリング](/ja-JP/channels/pairing)。

  </Accordion>

  <Accordion title="テキストメッセージを自律的に操作させてもよいですか？また、それは安全ですか？">
    個人メッセージに対する完全な自律操作は**推奨しません**。最も安全な方法は、DM を**ペアリングモード**または限定的な許可リストに設定し、ユーザーに代わってメッセージを送信させる場合は**別の番号またはアカウント**を使用し、送信前にユーザーが**承認**するまで下書きだけを作成させることです。

    試す場合は、専用の分離されたアカウントで実施してください。[セキュリティ](/ja-JP/gateway/security)を参照してください。

  </Accordion>

  <Accordion title="個人用アシスタントのタスクに安価なモデルを使用できますか？">
    はい。エージェントがチャット専用であり、入力が信頼できる場合に限ります。小規模なモデル階層は指示の乗っ取りを受けやすいため、ツールが有効なエージェントや、信頼されていないコンテンツを読み取る場合には使用しないでください。小規模なモデルを使用する必要がある場合は、ツールを厳しく制限し、サンドボックス内で実行してください。[セキュリティ](/ja-JP/gateway/security)を参照してください。
  </Accordion>

  <Accordion title="Telegram で /start を実行しましたが、ペアリングコードを取得できませんでした">
    ペアリングコードは、不明な送信者がボットにメッセージを送り、`dmPolicy: "pairing"` が有効になっている場合に**のみ**送信されます。`/start` だけではコードは生成されません。

    保留中のリクエストを確認します:

    ```bash
    openclaw pairing list telegram
    ```

    すぐにアクセスするには、送信者 ID を許可リストに追加するか、そのアカウントに `dmPolicy: "open"` を設定してください。

  </Accordion>

  <Accordion title="WhatsApp: 連絡先にメッセージを送信しますか？ペアリングはどのように機能しますか？">
    いいえ。WhatsApp のデフォルトの DM ポリシーは**ペアリング**です。不明な送信者にはペアリングコードだけが送信され、そのメッセージは**処理されません**。OpenClaw は、受信したチャット、またはユーザーが明示的に実行した送信に対してのみ返信します。

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    ウィザードの電話番号入力は、自分自身からの DM を許可するための**許可リスト/所有者**を設定します。自動送信には使用されません。個人用の WhatsApp 番号では、その番号を使用して `channels.whatsapp.selfChatMode` を有効にしてください。

  </Accordion>
</AccordionGroup>

## チャットコマンド、タスクの中止、「停止しない」場合

<AccordionGroup>
  <Accordion title="内部システムメッセージがチャットに表示されないようにするにはどうすればよいですか？">
    ほとんどの内部メッセージ/ツールメッセージは、そのセッションで **verbose**、**trace**、または **reasoning** が有効になっている場合にのみ表示されます。

    表示されているチャットで次を実行してください:

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    それでも表示が多い場合は、Control UI でセッション設定を確認し、verbose を **inherit** に設定してください。また、設定で `verboseDefault: "on"` が指定されたボットプロファイルを使用していないことを確認してください。

    ドキュメント: [思考と詳細出力](/ja-JP/tools/thinking)、[セキュリティ](/ja-JP/gateway/security/index#reasoning-and-verbose-output-in-groups)。

  </Accordion>

  <Accordion title="実行中のタスクを停止／キャンセルするにはどうすればよいですか？">
    中止するには、次のいずれかを（スラッシュなしの）**単独メッセージとして**送信します：`stop`、`stop action`、`stop current action`、`stop run`、`stop current run`、`stop agent`、`stop the agent`、`stop openclaw`、`openclaw stop`、`stop don't do anything`、`stop do not do anything`、`stop doing anything`、`do not do that`、`please stop`、`stop please`、`abort`、`esc`、`exit`、`interrupt`、`halt`。英語以外で一般的なトリガー（フランス語、ドイツ語、スペイン語、中国語、日本語、ヒンディー語、アラビア語、ロシア語）も使用できます。

    exec ツールで開始したバックグラウンドプロセスについては、エージェントに次を実行するよう依頼します：

    ```text
    process action:kill sessionId:XXX
    ```

    ほとんどのスラッシュコマンドは、`/` で始まる**単独の**メッセージとして送信する必要がありますが、一部のショートカット（`/status` など）は、許可リストに登録された送信者であれば文中でも使用できます。[スラッシュコマンド](/ja-JP/tools/slash-commands)を参照してください。

  </Accordion>

  <Accordion title='Telegram から Discord のメッセージを送信するにはどうすればよいですか？（「Cross-context messaging denied」）'>
    OpenClaw は、デフォルトで**プロバイダー間**のメッセージ送信をブロックします。ツール呼び出しが Telegram に紐付けられている場合、明示的に許可しない限り Discord には送信されません。この設定は即座に反映され、Gateway の再起動は不要です：

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='ボットが立て続けに送ったメッセージを「無視」しているように感じるのはなぜですか？'>
    デフォルトでは、実行中に送信されたプロンプトはアクティブな実行へ誘導されます。`/queue` を使用して、アクティブな実行中の動作を選択します：

    - `steer`（デフォルト）- 次のモデル境界でアクティブな実行を誘導します。
    - `followup` - メッセージをキューに追加し、現在の実行が終了した後に一つずつ実行します。
    - `collect` - 互換性のあるメッセージをキューに追加し、現在の実行が終了した後に一度だけ返信します。
    - `interrupt` - 現在の実行を中止し、新たに開始します。

    `debounce:0.5s cap:25 drop:summarize` のように、キューモードへオプションを追加できます。[コマンドキュー](/ja-JP/concepts/queue)と[ステアリングキュー](/ja-JP/concepts/queue-steering)を参照してください。

  </Accordion>
</AccordionGroup>

## その他

<AccordionGroup>
  <Accordion title='API キーを使用する場合、Anthropic のデフォルトモデルは何ですか？'>
    認証情報とモデル選択は別々です。`ANTHROPIC_API_KEY` を設定する（または認証プロファイルに Anthropic API キーを保存する）と認証が有効になりますが、実際のデフォルトモデルは `agents.defaults.model.primary` で設定したモデルです（例：`anthropic/claude-sonnet-4-6` または `anthropic/claude-opus-4-6`）。`No credentials found for profile "anthropic:default"` は、実行中のエージェントに対して想定される `auth-profiles.json` で Anthropic の認証情報を Gateway が見つけられなかったことを意味します。
  </Accordion>
</AccordionGroup>

---

まだ解決しない場合は、[Discord](https://discord.com/invite/clawd) で質問するか、[GitHub ディスカッション](https://github.com/openclaw/openclaw/discussions)を開始してください。

## 関連項目

- [初回実行に関する FAQ](/ja-JP/help/faq-first-run) - インストール、オンボーディング、認証、サブスクリプション、初期段階のエラー
- [モデルに関する FAQ](/ja-JP/help/faq-models) - モデル選択、フェイルオーバー、認証プロファイル
- [トラブルシューティング](/ja-JP/help/troubleshooting) - 症状を起点としたトリアージ
