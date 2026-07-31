---
read_when:
    - doctor マイグレーションの追加または変更
    - 破壊的な設定変更の導入
sidebarTitle: Doctor
summary: Doctor コマンド：ヘルスチェック、設定の移行、修復手順
title: 診断ツール
x-i18n:
    generated_at: "2026-07-26T10:13:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` は、OpenClaw の修復および移行ツールです。古い設定や状態を修正し、健全性をチェックして、実行可能な修復手順を提示します。

## クイックスタート

```bash
openclaw doctor
```

### ヘッドレスモードと自動化モード

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    確認せずにデフォルトを受け入れます（該当する場合は、再起動、サービス、サンドボックスの修復手順を含みます）。

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    確認せずに推奨修復を適用します（`--repair` はエイリアスです）。

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    CI または事前確認の自動化向けに、構造化された健全性チェックを実行します。読み取り専用であり、
    確認、修復、移行、再起動、状態の書き込みは行いません。

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    強力な修復も適用します（カスタムのスーパーバイザー設定を上書きします）。

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    確認なしで実行し、安全な移行（設定の正規化と
    ディスク上の状態の移動）のみを適用します。人による確認が必要な再起動、サービス、
    サンドボックスの操作はスキップします。レガシー状態の移行は、検出されると引き続き自動的に実行されます。

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    追加の Gateway インストールがないかシステムサービスをスキャンします（launchd/systemd/schtasks）。

  </Tab>
</Tabs>

書き込み前に変更を確認するには、まず設定ファイルを開きます。

```bash
cat ~/.openclaw/openclaw.json
```

## 読み取り専用 lint モード

`openclaw doctor --lint` は、`openclaw doctor --fix` と対になる自動化向けの機能です。
両者は同じ Doctor ルールレジストリを共有しますが、ルールの選択方法と
ルールに基づく処理方法は同じではありません。

| モード                     | 確認   | 設定/状態の書き込み     | 出力                 | 用途                      |
| ------------------------ | --------- | ----------------------- | ---------------------- | ------------------------------- |
| `openclaw doctor`        | あり       | なし                      | わかりやすい健全性レポート | 人が状態を確認する場合         |
| `openclaw doctor --fix`  | 場合による | 修復ポリシーに従って実行 | わかりやすい修復ログ    | 承認済みの修復を適用する場合       |
| `openclaw doctor --lint` | なし        | なし                      | 構造化された検出結果    | CI、事前確認、レビューゲート |

デフォルトの `doctor --lint` は、広範かつ安全な自動化プロファイルを実行します。これは、
静的かつローカルで、CI や事前確認の出力に有用なチェックを行います。助言のみのチェック、
環境に依存するチェック、稼働中のサービスに依存するチェック、アカウント/ワークスペースの
インベントリ、履歴のクリーンアップなど、オプトインのチェックはスキップします。これらの
オプトインチェックを含む登録済み lint 監査をすべて実行する場合は `doctor --lint --all` を、
特定のチェックを実行する場合は `--only <id>` を使用します。

`doctor --fix` は lint のデフォルトプロファイルを使用せず、
`--all` を受け付けません。Doctor の順序付き修復パスを実行します。最新の健全性チェックでは、
任意の `repair()` 実装を提供でき、古い領域では引き続きレガシーな
Doctor 修復フローを使用します。一部の lint 検出結果は意図的に診断専用であるため、
`--lint --all` にチェックが表示されても、`--fix` がその領域を変更するとは限りません。
この契約では、`detect()`（検出結果を報告）と `repair()`（変更、差分、副作用を報告）を
分離しています。これにより、lint チェックを変更プランナーにすることなく、将来の
`doctor --fix --dry-run` に向けた拡張余地を維持しています。

一部の組み込みチェックは内部的にデフォルトで無効になっています。これにより、デフォルトの
`doctor --lint` 自動化プロファイルに含めずに、`--all`、
`--only`、Doctor の修復フローで利用できます。検出結果の重大度は、引き続き
各検出結果ごとに（`info`、`warning`、`error` のいずれかとして）出力されます。
デフォルトで選択されるかどうかは重大度レベルではありません。

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

JSON 出力フィールド：

- `ok`：選択した重大度のしきい値を満たす検出結果があったかどうか
- `checksRun` / `checksSkipped`：件数（プロファイル、`--only`、または `--skip` によってスキップ）
- `findings`：`checkId`、`severity`、`message`、および任意の `path`、`line`、`column`、`ocPath`、`source`、`target`、`requirement`、`fixHint` を含む構造化された診断

終了コード：

| コード | 意味                                                  |
| ---- | -------------------------------------------------------- |
| `0`  | 選択したしきい値以上の検出結果なし           |
| `1`  | 1 件以上の検出結果が選択したしきい値を満たした          |
| `2`  | 検出結果を出力する前にコマンドまたはランタイムで障害が発生 |

フラグ：

- `--severity-min info|warning|error`（デフォルト `warning`）：出力内容と、ゼロ以外の終了コードになる条件の両方を制御します。
- `--all`：デフォルトの自動化セットから除外されたオプトインチェックを含め、登録済みのすべての lint チェックを実行します。
- `--only <id>`（繰り返し指定可能）：指定したチェック ID のみを実行します。不明な ID はエラーの検出結果として報告されます。
- `--skip <id>`（繰り返し指定可能）：残りの実行を継続しながら、特定のチェックを除外します。
- `--json`、`--severity-min`、`--all`、`--only`、`--skip` には `--lint` が必要です。通常の `openclaw doctor` および `--fix` の実行では拒否されます。

## 実行内容（概要）

<AccordionGroup>
  <Accordion title="健全性、UI、更新">
    - git インストール向けの任意の事前更新（対話モードのみ）。
    - UI プロトコルの鮮度チェック（プロトコルスキーマの方が新しい場合は Control UI を再ビルド）。
    - 健全性チェックと再起動の確認。
    - 問題がある場合のみ Skills と Plugin に関する注記を表示。正常なインベントリは `openclaw skills check` と `openclaw plugins list` に引き続き表示されます。

  </Accordion>
  <Accordion title="設定と移行">
    - レガシーな値の形式に対する設定の正規化。
    - レガシーなフラット形式の `talk.*` フィールドから `talk.provider` と `talk.providers.<provider>` への Talk 設定の移行。
    - レガシー Chrome 拡張機能の設定と Chrome MCP の準備状況に対するブラウザー移行チェック。
    - OpenCode プロバイダーのオーバーライドに関する警告（`models.providers.opencode` / `opencode-zen` / `opencode-go`）。
    - レガシー OpenAI Codex プロバイダー/プロファイルの移行（`openai-codex` → `openai`）と、古い `models.providers.openai-codex` によるシャドーイングの警告。
    - OpenAI Codex OAuth プロファイルに対する OAuth TLS 前提条件のチェック。
    - `plugins.allow` が制限的である一方、ツールポリシーがワイルドカードまたは Plugin 所有のツールを引き続き要求している場合の、Plugin/ツール許可リストに関する警告。
    - レガシーなディスク上の状態の移行（セッション/エージェントディレクトリ/WhatsApp 認証）。
    - レガシー Plugin マニフェスト契約キーの移行（`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders` → `contracts`）。
    - レガシー Cron ストアの移行（`jobId`、`schedule.cron`、トップレベルの配信/ペイロードフィールド、ペイロードの `provider`、`notify: true` Webhook フォールバックジョブ）。
    - `agents.defaults`、`agents.entries.*`、`models.providers.*`（モデルごとのエントリを含む）にわたる Codex CLI ランタイム固定値の修復（`agentRuntime.id: "codex-cli"` → `"codex"`）。
    - Plugin が有効な場合に古い Plugin 設定をクリーンアップ。`plugins.enabled=false` の場合は、古い Plugin 参照を無効な隔離設定として保持します。

  </Accordion>
  <Accordion title="状態と整合性">
    - セッションロックファイルの検査と古いロックのクリーンアップ。
    - 影響を受ける 2026.4.24 ビルドで作成された、重複するプロンプト書き換え分岐に対するセッショントランスクリプトの修復。
    - 停止したメインセッションとサブエージェントの再起動復旧トゥームストーンを検出。Doctor はブロックされたセッションを報告し、既存のトゥームストーンと競合する古い中断フラグのみを修復します。自動復旧を再度有効にはしません。
    - 状態の整合性と権限のチェック（セッション、トランスクリプト、状態ディレクトリ）。
    - ローカルで実行している場合の設定ファイル権限チェック（chmod 600）。
    - モデル認証の健全性：OAuth の有効期限をチェックし、期限切れが近いトークンを更新でき、認証プロファイルのクールダウン/無効状態を報告します。

  </Accordion>
  <Accordion title="Gateway、サービス、スーパーバイザー">
    - サンドボックスが有効な場合のサンドボックスイメージの修復。
    - レガシーサービスの移行と追加 Gateway の検出。
    - Matrix チャネルのレガシー状態の移行（`--fix` / `--repair` モード）。
    - Gateway ランタイムチェック（サービスはインストール済みだが未実行、キャッシュされた launchd ラベル）。
    - チャネル状態の警告（稼働中の Gateway からプローブ）。
    - チャネル固有の権限チェックは `openclaw channels capabilities` 配下にあります。たとえば、Discord ボイスチャネルの権限は `openclaw channels capabilities --channel discord --target channel:<channel-id>` で監査されます。
    - ローカル TUI クライアントがまだ実行中で、Gateway のイベントループの健全性が低下している場合の WhatsApp 応答性チェック。`--fix` は、確認済みのローカル TUI クライアントのみを停止します。
    - プライマリモデル、フォールバック、画像/動画生成モデル、Heartbeat/サブエージェント/Compaction のオーバーライド、フック、チャネルモデルのオーバーライド、セッションルートの固定値にある、レガシーな `openai-codex/*` モデル参照に対する Codex ルート修復。`--fix` はそれらを `openai/*` に書き換え、`openai-codex:*` の認証プロファイル/順序を `openai:*` に移行し、古いセッション/エージェント全体のランタイム固定値を削除して、修復後の有効なルートに基づいて Codex の互換性を判断します。
    - スーパーバイザー設定の監査（launchd/systemd/schtasks）と任意の修復。
    - インストールまたは更新時にシェルの `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` の値を取り込んだ Gateway サービスに対する、埋め込みプロキシ環境のクリーンアップ。
    - Gateway ランタイムチェック（サポートされていないレガシー Bun サービス、バージョンマネージャーのパス）。
    - Gateway ポート競合の診断（デフォルト `18789`）。

  </Accordion>
  <Accordion title="認証、セキュリティ、ペアリング">
    - オープンな DM ポリシーに対するセキュリティ警告。
    - ローカルトークンモードの Gateway 認証チェック（トークンソースが存在しない場合はトークン生成を提案し、トークンの SecretRef 設定は上書きしません）。
    - デバイスのペアリング問題を検出（初回ペアリング要求の保留、ロール/スコープのアップグレードの保留、古いローカルデバイストークンキャッシュのずれ、ペアリング済みレコードの認証のずれ）。

  </Accordion>
  <Accordion title="ワークスペースとシェル">
    - Linux での systemd linger チェック。
    - ワークスペースのブートストラップファイルサイズチェック（コンテキストファイルの切り詰め/上限間近の警告）。
    - デフォルトエージェントの Skills 準備状況チェック。必要なバイナリ、環境、設定、または OS 要件が不足している許可済み Skills を報告し、`--fix` は `skills.entries` で利用できない Skills を無効にできます。
    - シェル補完の状態チェックと自動インストール/アップグレード。
    - メモリ検索の埋め込みプロバイダー準備状況チェック（ローカルモデル、リモート API キー、または QMD バイナリ）。
    - ソースインストールのチェック（pnpm ワークスペースの不一致、UI アセットの欠落、tsx バイナリの欠落）。
    - 更新された設定とウィザードのメタデータを書き込みます。

  </Accordion>
</AccordionGroup>

## Dreams UI のバックフィルとリセット

  Control UI の Dreams シーンには、grounded dreaming ワークフロー用の **バックフィル**、**リセット**、**Grounded をクリア** アクションがあります。これらは Gateway の doctor 形式の RPC メソッドを使用しますが、`openclaw doctor` CLI の修復／移行には含まれません。

  | アクション         | 実行内容                                                                                                                                                      |
  | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | バックフィル       | アクティブなワークスペース内の過去の `memory/YYYY-MM-DD.md` ファイルをスキャンし、grounded REM 日記パスを実行して、元に戻せるバックフィルエントリを `DREAMS.md` に書き込みます。 |
  | リセット          | マークされたバックフィル日記エントリのみを `DREAMS.md` から削除します。                                                                                                  |
  | Grounded をクリア | 過去のリプレイからステージングされた grounded 専用の短期エントリのうち、ライブリコールまたは日次サポートがまだ蓄積されていないものだけを削除します。                           |

  これらはいずれも `MEMORY.md` を編集せず、doctor の完全な移行を実行せず、grounded 候補を単独でライブ短期昇格ストアにステージングすることもありません。grounded の過去のリプレイを通常のディープ昇格レーンに投入するには、代わりに次の CLI フローを使用します。

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  これにより、grounded の永続候補が短期 dreaming ストアにステージングされる一方、`DREAMS.md` はレビュー用サーフェスとして維持されます。

  ## 詳細な動作と根拠

  <AccordionGroup>
  <Accordion title="0. 任意の更新（git インストール）">
    これが git チェックアウトで、doctor が対話的に実行されている場合、doctor の実行前に更新（fetch／rebase／build）するかどうかを確認します。
  </Accordion>
  <Accordion title="1. 設定の正規化">
    Doctor は従来の値の形式を現在のスキーマに正規化します。現在の Talk 音声設定は `talk.provider` + `talk.providers.<provider>` で、リアルタイム音声設定は `talk.realtime.*` の下にあります。Doctor は古い `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` の形式をプロバイダーマップに書き換え、従来のトップレベルのリアルタイムセレクター（`talk.mode`、`talk.transport`、`talk.brain`、`talk.model`、`talk.voice`）を `talk.realtime` に書き換えます。

    また、`plugins.allow` が空ではなく、ツールポリシーでワイルドカードまたは Plugin 所有のツールエントリが使用されている場合、Doctor は警告を表示します。`tools.allow: ["*"]` は、実際に読み込まれた Plugin のツールだけに一致します。排他的な Plugin 許可リストを回避するものではありません。

  </Accordion>
  <Accordion title="2. 従来の設定キーの移行">
    設定に有効な移行処理がある非推奨キーが含まれている場合、他のコマンドは実行を拒否し、`openclaw doctor` の実行を求めます。Doctor は、検出された従来のキーを説明し、適用した移行を示して、更新されたスキーマで `~/.openclaw/openclaw.json` を書き換えます。Gateway の起動処理は従来の設定形式を拒否し、`openclaw doctor --fix` の実行を求めます。起動時に `openclaw.json` を書き換えることはありません。Cron ジョブストアの移行も `openclaw doctor --fix` によって処理されます。

    <Note>
      Doctor が自動移行を提供するのは、キーの廃止後およそ 2 か月間のみです。
      それより古い従来のキー（たとえば、マルチエージェント対応前の設定形式にあった元の
      `routing.queue`、`routing.bindings`、`routing.agents`/`defaultAgentId`、
      `routing.transcribeAudio`、トップレベルの `agent.*`、またはトップレベルの `identity`）
      には移行パスがなくなっています。これらを使用する設定は、書き換えられる代わりに
      検証に失敗するようになりました。Doctor が処理を続行できるようにするには、
      現在の設定リファレンスに従って、それらのキーを手動で修正してください。
    </Note>

    有効な移行：

    | レガシーキー                                                                                    | 現行キー                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | 削除済み（WebChat は廃止済み）                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours`（およびアカウントごと）      | `...threadBindings.idleHours`                                               |
    | レガシー `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | レガシーのトップレベルのリアルタイム Talk セレクター（`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`） | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | トップレベルの `tts`                                                              |
    | `messages.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | 明示的なチャンネルブロックを持つ `messages.responsePrefix`                                           | 設定済みのチャンネル／アカウントの `responsePrefix` にコピー。暗黙的／カスタムチャンネル用のグローバルフォールバックは保持 |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`、フックのインストール、Cron ストア、バンドル検出、グローバル TTS 設定パス            | 共有 SQLite 状態                                                       |
    | TTS スピーカーフィールド `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>`（Discord を除くすべてのチャンネル）                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>`（Discord を含むすべてのチャンネル）                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"`（Gateway の起動時には、`api` が将来の／不明な列挙値であるプロバイダーも、フェイルクローズする代わりにスキップされる） |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | 削除済み（レガシーの Chrome 拡張機能リレー設定）                             |
    | `mcp.servers.*.type`（CLI ネイティブのエイリアス）                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | 逆の `mcp.servers.*.enabled`                                              |
    | MCP タイムアウトエイリアス `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | MCP のスネークケース形式のサーバーフィールド                                                                     | MCP の camelCase 形式のサーバーフィールド                                                   |
    | `tools.media.image/audio/video.models`                                                           | ケイパビリティタグ付きの `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | 削除済み                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | メディアモデルの `deepgram` オプション                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`、Discord リアルタイム `voice`                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | ワイルドカード対応の `browser.ssrfPolicy.allowedHostnames`                          |
    | サンドボックスブラウザの `enableNoVnc`                                                                    | `noVncEnabled`                                                                |
    | ルートの `media`                                                                                     | `attachments`                                                                |
    | チャンネル／アカウントの `heartbeat` 可視性ブロック                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | ルートの `audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | 生成モデルのデフォルト                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | 廃止された最終レイアウト調整オプション                                                               | 組み込みのデフォルト動作                                                     |
    | `channels.whatsapp.messagePrefix` およびレガシーの `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | グローバルの `messages.ackReaction`、および翻訳可能な場合は `ackReactionScope`        |
    | `cron.failureDestination`                                                                        | `cron.failureAlert` の宛先フィールド                                     |
    | `gateway.controlUi.chatMessageMaxWidth`、表示専用の `ui.prefs` キー                       | 削除済み（テキストの拡大率、チャット幅、ライブサイドバーのアクティビティはブラウザローカル） |
    | `agents.list`                                                                                    | キー付きの `agents.entries`                                                        |
    | トップレベルの `defaultModel`                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | トップレベルの `tui`                                                                                  | 削除済み（TUI フッターはコンパクトなデフォルトを使用）                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | 削除済み（Codex app-server は常に Codex ネイティブのワークスペースツールをネイティブのまま維持） |
    | `commands.modelsWrite`                                                                           | 削除済み（`/models add` は非推奨）                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | 削除済み（完全一致する `NO_REPLY` は、表示可能なフォールバックテキストに書き換えられなくなった）  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | 削除済み（OpenClaw が生成されるシステムプロンプトを所有）                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | 削除済み（低速なモデル／プロバイダーのタイムアウトには `models.providers.<id>.timeoutSeconds` を使用し、エージェント／実行タイムアウトの上限未満に維持） |
    | トップレベルの `memorySearch`、`agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path`（任意の階層）                                                            | 削除済み（メモリインデックスは各エージェントデータベースに格納）                       |
    | トップレベルの `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | `plugins.openai-codex` ポリシー ID                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`、`session.parentForkMaxTokens`                                 | 削除済み（非推奨）                                                        |
    | 2026.7 で廃止されたランタイムとチャネルの調整オプション                                               | 削除済み（組み込みの本番環境向けデフォルトが適用されます）                               |

    <Note>
      上記の `plugins.entries.voice-call.config.*` 行は、設定が読み込まれるたびに
      `openclaw
      doctor` ではなく Voice Call Plugin 自体によって正規化されます。この Plugin は、`openclaw
      doctor --fix` を示す起動時の警告もログに記録しますが、doctor は現在、これらのキーについて
      `openclaw.json` を書き換えません。実行時に変更を適用するのは
      Plugin 自体の正規化です。
    </Note>

    複数アカウント対応チャネルのデフォルトアカウントに関するガイダンス：

    - 2 つ以上の `channels.<channel>.accounts` エントリが `channels.<channel>.defaultAccount` または `accounts.default` なしで設定されている場合、フォールバックルーティングによって意図しないアカウントが選択される可能性があることを doctor が警告します。
    - `channels.<channel>.defaultAccount` に不明なアカウント ID が設定されている場合、doctor が警告し、設定済みのアカウント ID を一覧表示します。

  </Accordion>
  <Accordion title="2b. OpenCode プロバイダーのオーバーライド">
    `models.providers.opencode`、`opencode-zen`、または `opencode-go` を手動で追加した場合、`openclaw/plugin-sdk/llm` の組み込み OpenCode カタログがオーバーライドされます。これにより、モデルが誤った API を使用するよう強制されたり、コストがゼロに設定されたりする可能性があります。オーバーライドを削除してモデルごとの API ルーティングとコストを復元できるよう、doctor が警告します。
  </Accordion>
  <Accordion title="2c. ブラウザーの移行と Chrome MCP の準備状況">
    ブラウザー設定が削除済みの Chrome 拡張機能パスを参照している場合、doctor は現在のホストローカル Chrome MCP 接続モデルに正規化します（`browser.profiles.*.driver: "extension"` → `"existing-session"`、`browser.relayBindHost` は削除）。

    `defaultProfile: "user"` または設定済みの `existing-session` プロファイルを使用している場合、doctor はホストローカル Chrome MCP パスも監査します：

    - デフォルトの自動接続プロファイルについて、Google Chrome が同じホストにインストールされているか確認します
    - 検出された Chrome のバージョンを確認し、Chrome 144 未満の場合に警告します
    - ブラウザーの検査ページ（例：`chrome://inspect/#remote-debugging`、`brave://inspect/#remote-debugging`、または `edge://inspect/#remote-debugging`）でリモートデバッグを有効にするよう通知します

    doctor は Chrome 側の設定を有効にできません。ホストローカル Chrome MCP を使用するには、Gateway/Node ホスト上で Chromium ベースのブラウザー 144 以降がローカル実行され、リモートデバッグが有効で、初回接続時の同意プロンプトがブラウザーで承認されている必要があります。

    ここでの準備状況は、ローカル接続の前提条件のみを対象とします。既存セッションでは現在の Chrome MCP ルート制限が維持されます。`responsebody`、PDF エクスポート、ダウンロードのインターセプト、バッチアクションなどの高度なルートには、引き続き管理対象ブラウザーまたは raw CDP プロファイルが必要です。このチェックは Docker、サンドボックス、リモートブラウザー、その他のヘッドレスフローには適用されず、これらでは引き続き raw CDP が使用されます。

  </Accordion>
  <Accordion title="2d. OAuth TLS の前提条件">
    OpenAI Codex OAuth プロファイルが設定されている場合、doctor は OpenAI の認可エンドポイントをプローブし、ローカルの Node/OpenSSL TLS スタックが証明書チェーンを検証できることを確認します。証明書エラー（例：`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`、期限切れ証明書、自己署名証明書）でプローブが失敗した場合、doctor はプラットフォーム固有の修正ガイダンスを表示します。Homebrew Node を使用する macOS では、通常 `brew postinstall ca-certificates` で修正できます。`--deep` を使用すると、Gateway が正常でもプローブが実行されます。
  </Accordion>
  <Accordion title="2e. Codex OAuth プロバイダーのオーバーライド">
    以前に `models.providers.openai-codex` の下へ従来の OpenAI トランスポート設定を追加していた場合、その設定によって組み込みの Codex OAuth プロバイダーパスが隠される可能性があります。Codex OAuth とともに古いトランスポート設定が検出されると、古いトランスポートオーバーライドを削除または書き換えて現在のルーティング動作を復元できるよう、doctor が警告します。カスタムプロキシとヘッダーのみのオーバーライドは引き続きサポートされ、この警告は発生しませんが、それらの明示的に定義されたリクエストルートは Codex の暗黙的な選択の対象にはなりません。
  </Accordion>
  <Accordion title="2f. Codex ルートの修復">
    doctor は従来の `openai-codex/*` モデル参照を確認します。ネイティブ Codex ハーネスのルーティングでは正規の `openai/*` モデル参照を使用しますが、プレフィックスだけで Codex が選択されることはありません。実行時ポリシーが未設定または `auto` の場合、明示的なリクエストオーバーライドがない、公式の HTTPS Platform Responses または ChatGPT Responses ルートとの完全一致のみが対象です。[OpenAI の暗黙的なエージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)を参照してください。

    `--fix` / `--repair` モードでは、doctor は影響を受けるデフォルトエージェントおよびエージェントごとの参照を書き換えます。対象には、プライマリモデル、フォールバック、画像・動画生成モデル、Heartbeat/サブエージェント/Compaction のオーバーライド、フック、チャネルモデルのオーバーライド、古い永続化済みセッションルート状態が含まれます：

    - `openai-codex/gpt-*` は `openai/gpt-*` になります。
    - Codex の使用意図は、修復されたエージェントモデル参照のプロバイダー/モデル単位の `agentRuntime.id: "codex"` エントリに移動されます。
    - 実行時の選択はプロバイダー/モデル単位で行われるため、古いエージェント全体のランタイム設定と永続化済みセッションのランタイム固定は削除されます。
    - 修復された従来のモデル参照で以前の認証パスを維持するために Codex ルーティングが必要な場合を除き、既存のプロバイダー/モデルのランタイムポリシーは保持されます。
    - 既存のモデルフォールバックリストは、従来のエントリを書き換えたうえで保持されます。コピーされたモデルごとの設定は、従来のキーから正規の `openai/*` キーに移動されます。
    - 永続化済みセッションの `modelProvider`/`providerOverride`、`model`/`modelOverride`、フォールバック通知、認証プロファイルの固定が、検出されたすべてのエージェントセッションストアで修復されます。
    - doctor は、古い `agentRuntime.id: "codex-cli"` の固定（別の従来ランタイム ID）を、`agents.defaults`、`agents.entries.*`、`models.providers.*` の各モデルエントリにわたって `"codex"` に修復します。
    - `/codex ...` は「チャットからネイティブ Codex 会話を制御またはバインドする」ことを意味します。
    - `/acp ...` または `runtime: "acp"` は「外部 ACP/acpx アダプターを使用する」ことを意味します。

  </Accordion>
  <Accordion title="2g. セッションルートのクリーンアップ">
    doctor は、設定済みのモデルまたはランタイムを Codex などの Plugin 所有ルートから移動した後に残る、古い自動作成ルート状態がないか、検出されたエージェントセッションストアもスキャンします。

    `openclaw doctor --fix` は、その所有ルートが設定されなくなった場合、`modelOverrideSource: "auto"` モデルの固定、ランタイムモデルのメタデータ、固定されたハーネス ID、CLI セッションのバインディング、自動認証プロファイルのオーバーライドなど、自動作成された古い状態を消去できます。ユーザーが明示的に選択したセッションモデルまたは従来のセッションモデルは手動確認用として報告され、変更されません。そのルートを使用する意図がなくなった場合は、`/model ...`、`/new` で切り替えるか、セッションをリセットしてください。

  </Accordion>
  <Accordion title="3. 従来の状態の移行（ディスクレイアウト）">
    doctor は、以前のディスク上のレイアウトを現在の構造に移行できます：

    - セッションストアとトランスクリプト：`~/.openclaw/sessions/` から `~/.openclaw/agents/<agentId>/sessions/` へ
    - エージェントディレクトリ：`~/.openclaw/agent/` から `~/.openclaw/agents/<agentId>/agent/` へ
    - WhatsApp 認証状態（Baileys）：従来の `~/.openclaw/credentials/*.json`（`oauth.json` を除く）から `~/.openclaw/credentials/whatsapp/<accountId>/...` へ（デフォルトのアカウント ID：`default`）
    - 署名済みデバイス ID：`~/.openclaw/identity/device.json` から `state/openclaw.sqlite` 内の `primary` `device_identities` 行へ。個別のデバイス認証ファイルは変更されません

    これらの移行はベストエフォートかつ冪等です。従来のフォルダーをバックアップとして残す場合、doctor は警告を出します。Gateway/CLI も起動時に従来のセッションとエージェントディレクトリを自動移行するため、doctor を手動で実行しなくても履歴、認証、モデルがエージェントごとのパスに配置されます。WhatsApp 認証は意図的に `openclaw doctor` でのみ移行されます。Talk のプロバイダー/プロバイダーマップの正規化では構造的等価性で比較するため、キー順序のみの差異によって、変更を伴わない `doctor --fix` が繰り返し発生することはなくなりました。

  </Accordion>
  <Accordion title="3a. 従来の Plugin マニフェストの移行">
    doctor は、インストールされているすべての Plugin マニフェストをスキャンし、非推奨のトップレベル機能キー（`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders`）を検出します。検出すると、それらを `contracts` オブジェクトへ移動し、マニフェストファイルをその場で書き換えることを提案します。この移行は冪等です。`contracts` に同じ値がすでに存在する場合、データを重複させずに従来のキーが削除されます。
  </Accordion>
  <Accordion title="3b. 従来の Cron ストアの移行">
    doctor は、正規化された行を SQLite にインポートする前に、従来の Cron ジョブストア（`~/.openclaw/cron/jobs.json`）に古いジョブ形式がないかも確認します。

    現在の Cron クリーンアップには次が含まれます：

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - トップレベルのペイロードフィールド（`message`、`model`、`thinking`、...）→ `payload`
    - トップレベルの配信フィールド（`deliver`、`channel`、`to`、`provider`、...）→ `delivery`
    - ペイロードの `provider` 配信エイリアス → 明示的な `delivery.channel`
    - 従来の `notify: true` Webhook フォールバックジョブ → 廃止された raw `cron.webhook` 値が有効な場合、その値を使用する明示的な Webhook 配信。通知ジョブではチャット配信が維持され、`delivery.completionDestination` が設定されます。その後、doctor は古い設定キーを削除します。使用可能な従来の Webhook がない場合、ランタイム配信では読み取られないため、配信先のないジョブから機能しないトップレベルの `notify` マーカーが削除されます（通知を含む既存の配信は保持されます）。

    Gateway は読み込み時に不正な Cron 行もサニタイズするため、有効なジョブは実行を継続します。raw の不正な行は、`jobs.json` から削除される前に、アクティブなストアの隣にある `jobs-quarantine.json` にコピーされます。doctor は隔離された行を報告するため、手動で確認または修復できます。

    Gateway の起動時にはランタイムプロジェクションが正規化され、トップレベルの `notify` マーカーは無視されますが、永続化された Cron 状態は doctor による修復のために残されます。doctor は、移行先のないジョブ（`delivery.mode` が none/未指定、使用できない従来の Webhook 配信先、または既存の通知/チャット配信）から機能しないマーカーを削除し、既存の配信は変更しません。そのため、`doctor --fix` を繰り返し実行しても、同じジョブについて再度警告されることはなくなります。

    Linux では、ユーザーの crontab がまだ従来の `~/.openclaw/bin/ensure-whatsapp.sh` を呼び出している場合にも doctor が警告します。このホストローカルスクリプトは現在の OpenClaw では保守されておらず、Cron が systemd ユーザーバスに接続できない場合、誤った `Gateway inactive` メッセージを `~/.openclaw/logs/whatsapp-health.log` に書き込む可能性があります。`crontab -e` で古い crontab エントリを削除し、現在のヘルスチェックには `openclaw channels status --probe`、`openclaw doctor`、`openclaw gateway status` を使用してください。

  </Accordion>
  <Accordion title="3c. セッションロックのクリーンアップ">
    Doctor は、異常終了したセッションが残した古い書き込みロックファイルを検出するため、すべてのエージェントセッションディレクトリをスキャンします。検出した各ロックファイルについて、パス、PID、その PID がまだ稼働しているか、ロックの経過時間、古いと判断されるかどうか（PID が停止済み、所有者メタデータが不正、30 分超経過、または稼働中の PID が OpenClaw 以外のプロセスに属することが確認された場合）を報告します。`--fix` / `--repair` モードでは、停止済み、孤立、再利用済み、不正かつ古い、または OpenClaw 以外の所有者を持つロックを自動的に削除します。稼働中の OpenClaw プロセスが引き続き所有する古いロックは報告されますが、Doctor がアクティブなトランスクリプトライターを遮断しないよう、そのまま残されます。
  </Accordion>
  <Accordion title="3d. セッショントランスクリプトのブランチ修復">
    Doctor は、2026.4.24 のプロンプトトランスクリプト書き換えバグによって作成された重複ブランチ構造を検出するため、エージェントセッションの JSONL ファイルをスキャンします。この構造には、OpenClaw の内部ランタイムコンテキストを持つ放棄されたユーザーターンと、同じ可視ユーザープロンプトを含むアクティブな兄弟ブランチがあります。`--fix` / `--repair` モードでは、Doctor は影響を受けた各ファイルを元のファイルの隣にバックアップし、トランスクリプトをアクティブなブランチへ書き換えることで、Gateway の履歴リーダーとメモリリーダーに重複ターンが表示されないようにします。
  </Accordion>
  <Accordion title="4. 状態の整合性チェック（セッションの永続化、ルーティング、安全性）">
    状態ディレクトリは運用上の脳幹です。これが消失すると、別の場所にバックアップがない限り、セッション、認証情報、ログ、設定が失われます。

    Doctor は次の項目を確認します。

    - **状態ディレクトリがない**：壊滅的な状態損失について警告し、ディレクトリの再作成を促し、失われたデータは復旧できないことを通知します。
    - **状態ディレクトリの権限**：書き込み可能かを検証し、権限の修復を提案します（所有者またはグループの不一致が検出された場合は `chown` のヒントも表示します）。
    - **macOS のクラウド同期された状態ディレクトリ**：状態の実体が iCloud Drive（`~/Library/Mobile Documents/com~apple~CloudDocs/...`）または `~/Library/CloudStorage/...` の配下にある場合、同期対象のパスでは I/O が遅くなり、ロックと同期の競合が発生する可能性があるため警告します。
    - **Linux の SD または eMMC 上の状態ディレクトリ**：状態の実体が `mmcblk*` マウントソース上にある場合、SD/eMMC 上のランダム I/O は遅くなる可能性があり、セッションや認証情報の書き込みによって劣化が早まるため警告します。
    - **Linux の揮発性状態ディレクトリ**：状態の実体が `tmpfs` または `ramfs` にある場合、セッション、認証情報、設定、SQLite の状態（WAL/ジャーナルのサイドカーファイルを含む）が再起動時に消失するため警告します。Docker の `overlay` マウントは、コンテナが存続する間はホストの再起動後も書き込み可能レイヤーが維持されるため、意図的に警告対象外としています。
    - **セッションディレクトリがない**：履歴を永続化し、`ENOENT` のクラッシュを回避するには、`sessions/` とセッションストアディレクトリが必要です。
    - **トランスクリプトの不一致**：最近のセッションエントリに対応するトランスクリプトファイルがない場合に警告します。
    - **メインセッションの「1 行 JSONL」**：メインのトランスクリプトが 1 行しかない場合にフラグを立てます（履歴が蓄積されていません）。
    - **複数の状態ディレクトリ**：複数のホームディレクトリに `~/.openclaw` フォルダーが存在する場合、または `OPENCLAW_STATE_DIR` が別の場所を指している場合に警告します（履歴がインストール間で分割される可能性があります）。
    - **リモートモードの注意事項**：`gateway.mode=remote` の場合、Doctor はリモートホスト上で実行するよう通知します（状態はそこにあります）。
    - **設定ファイルの権限**：`~/.openclaw/openclaw.json` がグループまたは全ユーザーから読み取り可能な場合に警告し、`600` への制限強化を提案します。

  </Accordion>
  <Accordion title="5. モデル認証の健全性（OAuth の有効期限）">
    Doctor は認証ストア内の OAuth プロファイルを調べ、トークンが期限切れ間近または期限切れの場合に警告し、安全に実行できる場合は更新できます。Anthropic の OAuth/トークンプロファイルが古い場合は、Anthropic API キーまたは Anthropic のセットアップトークン経由の方法を提案します。更新を促すプロンプトは対話的（TTY）に実行している場合にのみ表示され、`--non-interactive` では更新の試行をスキップします。

    OAuth の更新が恒久的に失敗した場合（たとえば `refresh_token_reused`、`invalid_grant`、またはプロバイダーから再度サインインするよう求められた場合）、Doctor は再認証が必要であることを報告し、実行すべき正確な `openclaw models auth login --provider ...` コマンドを表示します。

    Doctor は、短時間のクールダウン（レート制限、タイムアウト、認証失敗）または長期間の無効化（請求またはクレジットの失敗）により、一時的に使用できない認証プロファイルも報告します。

    トークンが macOS Keychain に保存されている従来の Codex OAuth プロファイル（ファイルベースのサイドカーレイアウトより前の古いオンボーディング）は、Doctor でのみ修復されます。対話型ターミナルから `openclaw doctor --fix` を一度実行すると、Keychain に保存された従来のトークンが `auth-profiles.json` にインラインで移行されます。その後は、埋め込みターン（Telegram、cron、サブエージェントのディスパッチ）で標準の OpenAI OAuth プロファイルとして解決されます。

  </Accordion>
  <Accordion title="6. フックのモデル検証">
    `hooks.gmail.model` が設定されている場合、Doctor はモデル参照をカタログおよび許可リストと照合し、解決できない場合や許可されていない場合に警告します。
  </Accordion>
  <Accordion title="7. サンドボックスイメージの修復">
    サンドボックスが有効な場合、Doctor は Docker イメージを確認し、現在のイメージがない場合はビルドまたは従来の名前への切り替えを提案します。
  </Accordion>
  <Accordion title="7b. Plugin インストールのクリーンアップ">
    Doctor は、`openclaw doctor --fix` / `openclaw doctor --repair` モードで、OpenClaw が生成した従来の Plugin 依存関係ステージング状態を削除します。対象には、古い生成済み依存関係ルート、旧インストールステージディレクトリ、以前のバンドル Plugin 依存関係修復コードによるパッケージローカルの残骸、および現在のバンドルマニフェストを覆い隠す可能性がある、孤立または復旧されたバンドル `@openclaw/*` Plugin の管理対象 npm コピーが含まれます。また Doctor は、`peerDependencies.openclaw` を宣言する管理対象 npm Plugin にホストの `openclaw` パッケージを再リンクし、`openclaw/plugin-sdk/*` などのパッケージローカルなランタイムインポートが更新または npm 修復後も引き続き解決されるようにします。

    設定から参照されているもののローカル Plugin レジストリで見つからない、ダウンロード可能な Plugin を再インストールすることもできます（実体のある `plugins.entries`、設定済みのチャンネル、プロバイダー、検索設定、設定済みのエージェントランタイム）。パッケージの更新中は、コアパッケージの入れ替え中に Plugin パッケージを再インストールしないようにします。設定済みの Plugin に引き続き復旧が必要な場合は、更新後に `openclaw doctor --fix` をもう一度実行してください。以下のコンテナイメージ起動時の例外を除き、Gateway の起動および設定の再読み込みではパッケージ修復を実行しません。Plugin のインストールは、明示的な Doctor、インストール、更新の作業として扱われます。

    コンテナ化された Gateway の起動には限定的なアップグレード例外があります。`openclaw gateway run` が新しい OpenClaw バージョンで起動すると、準備完了になる前に安全な状態移行と既存のコア更新後の Plugin 収束処理を実行し、その後バージョンごとのチェックポイントを記録します。この起動処理では、古いバンドル Plugin レコードのクリーンアップ、ローカル Plugin リンクの修復、収束処理で必要な場合の設定済み Plugin パッケージの再インストール、アクティブな Plugin ペイロードの確認を実行できます。起動時に安全に修復できない場合は、コンテナを通常どおり再起動する前に、同じマウント済みの状態および設定に対して、同じイメージを `openclaw doctor --fix` で一度実行してください。

  </Accordion>
  <Accordion title="8. Gateway サービスの移行とクリーンアップのヒント">
    Doctor は従来の Gateway サービス（launchd/systemd/schtasks）を検出し、それらを削除して現在の Gateway ポートを使用する OpenClaw サービスをインストールすることを提案します。また、追加の Gateway 形式のサービスをスキャンし、クリーンアップのヒントを表示できます。プロファイル名付きの OpenClaw Gateway サービスは正規のサービスと見なされ、「追加」としてフラグを立てません。

    Linux では、ユーザーレベルの Gateway サービスがなくても、システムレベルの OpenClaw Gateway サービスが存在する場合、Doctor は 2 つ目のユーザーレベルサービスを自動的にインストールしません。`openclaw gateway status --deep` または `openclaw doctor --deep` で確認した後、重複を削除するか、システムスーパーバイザーが Gateway のライフサイクルを管理している場合は `OPENCLAW_SERVICE_REPAIR_POLICY=external` を設定してください。

  </Accordion>
  <Accordion title="8b. 起動時の Matrix 移行">
    Matrix チャンネルアカウントに保留中または実行可能な従来の状態移行がある場合、Doctor は（`--fix` / `--repair` モードで）移行前のスナップショットを作成し、ベストエフォートの移行手順を実行します。対象は、従来の Matrix 状態の移行と、従来の暗号化状態の準備です。どちらの手順も致命的ではなく、エラーはログに記録され、起動は続行されます。読み取り専用モード（`--fix` なしの `openclaw doctor`）では、このチェック全体がスキップされます。
  </Accordion>
  <Accordion title="8c. デバイスのペアリングと認証のドリフト">
    Doctor は通常の健全性チェックの一環としてデバイスペアリングの状態を調べ、次の項目を報告します。

    - 保留中の初回ペアリングリクエスト
    - ペアリング済みデバイスに対する保留中のロールまたはスコープのアップグレード
    - デバイス ID は一致しているものの、デバイス ID 情報が承認済みレコードと一致しなくなった場合の公開鍵不一致の修復
    - 承認済みロールのアクティブなトークンがないペアリング済みレコード
    - 承認済みペアリング基準外にスコープがドリフトしたペアリング済みトークン
    - Gateway 側のトークンローテーションより前に作成された、または古いスコープメタデータを持つ、現在のマシン用のローカルキャッシュ済みデバイストークンエントリ

    Doctor はペアリングリクエストを自動承認せず、デバイストークンも自動ローテーションしません。実行すべき正確な次の手順を表示します。

    - `openclaw devices list` で保留中のリクエストを確認する
    - `openclaw devices approve <requestId>` で対象のリクエストを承認する
    - `openclaw devices rotate --device <deviceId> --role <role>` で新しいトークンをローテーションする
    - `openclaw devices remove <deviceId>` で古いレコードを削除して再承認する

    これにより、初回ペアリング、保留中のロールまたはスコープのアップグレード、古いトークンまたはデバイス ID 情報のドリフトを区別し、よくある「すでにペアリング済みなのに、ペアリングが必要と表示され続ける」問題を解消します。

  </Accordion>
  <Accordion title="9. セキュリティ警告">
    Doctor は、許可リストなしで DM に開放されているプロバイダーや、危険な設定のポリシーなど、警告を検出した場合にのみセキュリティに関する注記を表示します。セキュリティ項目の完全な一覧には `openclaw security audit` を使用してください。
  </Accordion>
  <Accordion title="10. systemd linger（Linux）">
    systemd ユーザーサービスとして実行している場合、Doctor はログアウト後も Gateway が稼働し続けるよう、linger が有効であることを確認します。
  </Accordion>
  <Accordion title="11. ワークスペースの状態（Skills、Plugin、TaskFlow）">
    Doctor は正常状態の一覧ではなく、デフォルトエージェントの問題と対処方法を表示します。

    - **Skills**：許可されているものの使用できないスキル名を一覧表示します。要件の詳細と全件数には `openclaw skills check` を使用してください。
    - **Plugin**：エラーが発生した Plugin ID のみを報告します。読み込み済み、インポート済み、無効化済み、およびバンドル Plugin の一覧には `openclaw plugins list` を使用してください。
    - **Plugin の互換性警告**：現在のランタイムとの互換性に問題がある Plugin にフラグを立てます。
    - **Plugin の診断**：Plugin レジストリが読み込み時に出力した警告またはエラーを表示します。
    - **TaskFlow の復旧**：手動での確認またはキャンセルが必要な、不審な管理対象 TaskFlow を表示します。
    - **Claude CLI**：バイナリ、認証、プロファイル、ワークスペース、またはプロジェクトディレクトリの問題のみを報告します。正常なプローブの詳細は省略されます。

  </Accordion>
  <Accordion title="11b. ブートストラップファイルのサイズ">
    Doctor は、ワークスペースのブートストラップファイル（たとえば `AGENTS.md`、`CLAUDE.md`、またはその他の注入されたコンテキストファイル）が、設定された文字数の上限に近いか、超えているかを確認します。ファイルごとの元の文字数と注入後の文字数、切り詰め率、切り詰めの原因（`max/file` または `max/total`）、および全体の上限に対する注入済み文字数の合計割合を報告します。ファイルが切り詰められている場合や上限に近い場合、Doctor は `agents.defaults.bootstrapMaxChars` と `agents.defaults.bootstrapTotalMaxChars` を調整するためのヒントを表示します。
  </Accordion>
  <Accordion title="11c. シェル補完">
    Doctor は、現在のシェル（zsh、bash、fish、PowerShell）にタブ補完がインストールされているかを確認します。

    - シェルプロファイルで低速な動的補完パターン（`source <(openclaw completion ...)`）が使用されている場合、doctor はそれをより高速なキャッシュファイル形式にアップグレードします。
    - プロファイルで補完が設定されているもののキャッシュファイルがない場合、doctor はキャッシュを自動的に再生成します。
    - 補完がまったく設定されていない場合、doctor はインストールするか確認します（対話モードのみ。`--non-interactive` の場合はスキップされます）。

    キャッシュを手動で再生成するには、`openclaw completion --write-state` を実行します。

  </Accordion>
  <Accordion title="11d. 古いチャンネル Plugin のクリーンアップ">
    `openclaw doctor --fix` が存在しないチャンネル Plugin を削除すると、その Plugin を参照していた未解決のチャンネルスコープ設定も削除されます。対象は、`channels.<id>` エントリ、該当チャンネルを指定した Heartbeat ターゲット、`agents.*.models["<channel>/*"]` オーバーライドです。これにより、チャンネルランタイムがなくなっているにもかかわらず、設定が引き続き Gateway にそのチャンネルへのバインドを要求することで発生する Gateway の起動ループを防ぎます。
  </Accordion>
  <Accordion title="12. Gateway 認証チェック（ローカルトークン）">
    Doctor はローカル Gateway のトークン認証が使用可能な状態かを確認します。

    - トークンモードでトークンが必要にもかかわらず、トークンソースが存在しない場合、doctor はトークンの生成を提案します。
    - `gateway.auth.token` が SecretRef で管理されているものの利用できない場合、doctor は警告を表示し、平文で上書きしません。
    - `openclaw doctor --generate-gateway-token` は、トークンの SecretRef が設定されていない場合に限り、生成を強制します。

  </Accordion>
  <Accordion title="12b. SecretRef 対応の読み取り専用修復">
    一部の修復フローでは、ランタイムのフェイルファスト動作を弱めずに、設定済みの認証情報を検査する必要があります。

    - `openclaw doctor --fix` は、対象を限定した設定修復に、ステータス系コマンドと同じ読み取り専用の SecretRef サマリーモデルを使用します。
    - 例：Telegram の `allowFrom` / `groupAllowFrom` `@username` 修復では、利用可能な場合に設定済みのボット認証情報を使用しようとします。
    - Telegram ボットトークンが SecretRef 経由で設定されているものの、現在のコマンドパスでは利用できない場合、doctor は認証情報が「設定済みだが利用不可」であることを報告し、クラッシュしたりトークンがないと誤って報告したりせず、自動解決をスキップします。

  </Accordion>
  <Accordion title="13. Gateway のヘルスチェックと再起動">
    Doctor はヘルスチェックを実行し、Gateway が正常でないと思われる場合は再起動を提案します。
  </Accordion>
  <Accordion title="13b. メモリ検索の準備状態">
    Doctor は、設定されたメモリ検索用埋め込みプロバイダーがデフォルトエージェントで使用可能な状態かを確認します。動作は、設定されたバックエンドとプロバイダーによって異なります。

    - **QMD バックエンド**：`qmd` バイナリが利用可能で起動できるかを検査します。利用できない場合は、`npm install -g @tobilu/qmd`（または Bun の同等コマンド）や手動でのバイナリパス指定など、修正方法を表示します。
    - **明示的なローカルプロバイダー**：ローカルモデルファイル、または認識可能なリモート／ダウンロード可能なモデル URL があるかを確認します。ない場合は、リモートプロバイダーへの切り替えを提案します。
    - **明示的なリモートプロバイダー**（`openai`、`voyage` など）：API キーが環境または認証ストアに存在することを確認します。ない場合は、実行可能な修正ヒントを表示します。
    - **従来の自動プロバイダー**：`memorySearch.provider: "auto"` を OpenAI として扱い、OpenAI の準備状態を確認し、`doctor --fix` によって `provider: "openai"` へ書き換えます。

    キャッシュされた Gateway プローブ結果が利用可能な場合（チェック時点で Gateway が正常だった場合）、doctor はその結果を CLI から確認できる設定と照合し、不一致があれば通知します。Doctor はデフォルトの処理では新たな埋め込み ping を開始しません。プロバイダーをライブで確認する場合は、詳細メモリステータスコマンドを使用します。

    ランタイムで埋め込みの準備状態を確認するには、`openclaw memory status --deep` を使用します。

  </Accordion>
  <Accordion title="14. チャンネルステータスの警告">
    Gateway が正常な場合、doctor はチャンネルステータスのプローブを実行し、推奨される修正方法とともに警告を報告します。
  </Accordion>
  <Accordion title="15. スーパーバイザー設定の監査と修復">
    Doctor は、インストール済みのスーパーバイザー設定（launchd/systemd/schtasks）に、デフォルト値の欠落や古い値がないかを確認します（たとえば、systemd の network-online 依存関係や再起動の遅延）。不一致が見つかると更新を推奨し、サービスファイル／タスクを現在のデフォルト設定に書き換えることができます。

    注意事項：

    - `openclaw doctor` は、スーパーバイザー設定を書き換える前に確認を求めます。
    - `openclaw doctor --yes` は、デフォルトの修復確認を受け入れます。
    - `openclaw doctor --fix` は、確認なしで推奨される修正を適用します（`--repair` はエイリアスです）。
    - `openclaw doctor --fix --force` は、カスタムのスーパーバイザー設定を上書きします。
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` は、Gateway サービスのライフサイクルに関して doctor を読み取り専用に保ちます。サービスの正常性の報告とサービス以外の修復は引き続き実行しますが、外部スーパーバイザーがそのライフサイクルを管理するため、サービスのインストール／起動／再起動／ブートストラップ、スーパーバイザー設定の書き換え、従来のサービスのクリーンアップはスキップします。
    - Linux では、対応する systemd Gateway ユニットがアクティブな間、doctor はコマンド／エントリポイントのメタデータを書き換えません。また、重複サービスのスキャン時には、非アクティブかつレガシーではない追加の Gateway 類似ユニットを無視するため、補助サービスファイルによって不要なクリーンアップ通知が発生することはありません。
    - トークン認証でトークンが必要であり、`gateway.auth.token` が SecretRef で管理されている場合、doctor によるサービスのインストール／修復では SecretRef を検証しますが、解決された平文のトークン値をスーパーバイザーサービスの環境メタデータには保存しません。
    - Doctor は、古い LaunchAgent、systemd、または Windows Scheduled Task のインストールでインラインに埋め込まれた、管理対象の `.env`／SecretRef ベースのサービス環境値を検出し、それらの値がスーパーバイザー定義ではなくランタイムソースから読み込まれるようにサービスメタデータを書き換えます。
    - Doctor は、`gateway.port` の変更後もサービスコマンドが古い `--port` に固定されている場合にそれを検出し、サービスメタデータを現在のポートに書き換えます。
    - トークン認証でトークンが必要であり、設定されたトークンの SecretRef を解決できない場合、doctor は実行可能な対処方法を提示して、インストール／修復処理をブロックします。
    - `gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、doctor はモードが明示的に設定されるまでインストール／修復をブロックします。
    - Linux のユーザー systemd ユニットでは、doctor がサービス認証メタデータを比較するとき、トークンの差異チェックに `Environment=` と `EnvironmentFile=` の両方のソースが含まれます。
    - 設定が最後に新しいバージョンによって書き込まれている場合、doctor のサービス修復は、古い OpenClaw バイナリから Gateway サービスを書き換え、停止、または再起動することを拒否します。[Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting#split-brain-installs-and-newer-config-guard)を参照してください。
    - `openclaw gateway install --force` を使用すれば、いつでも完全な書き換えを強制できます。

  </Accordion>
  <Accordion title="16. Gateway ランタイムとポートの診断">
    Doctor はサービスランタイム（PID、直近の終了ステータス）を調査し、サービスがインストールされているものの実際には実行されていない場合に警告します。また、Gateway ポート（デフォルトは `18789`）でポートの競合がないかを確認し、考えられる原因（Gateway がすでに実行中、SSH トンネル）を報告します。
  </Accordion>
  <Accordion title="17. Gateway ランタイムのベストプラクティス">
    Gateway サービスが Bun またはバージョン管理された Node パス（`nvm`、`fnm`、`volta`、`asdf` など）で実行されている場合、doctor は警告します。Bun は OpenClaw の `node:sqlite` 状態ストアを開けないため、修復時に従来の Bun サービスを Node に移行します。バージョンマネージャーのパスは、サービスがシェル初期化設定を読み込まないため、アップグレード後に動作しなくなる可能性があります。Doctor は、利用可能な場合はシステムの Node インストール（Homebrew/apt/choco）への移行を提案します。

    新しくインストールまたは修復された macOS LaunchAgent は、対話型シェルの PATH をコピーする代わりに、標準化されたシステム PATH（`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`）を使用します。これにより、Homebrew で管理されるシステムバイナリを引き続き利用できる一方で、Volta、asdf、fnm、pnpm、その他のバージョンマネージャーのディレクトリによって Node 子プロセスが解決する対象が変わることはありません。Linux サービスでは引き続き、明示的な環境ルート（`NVM_DIR`、`FNM_DIR`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`BUN_INSTALL`、`PNPM_HOME`）と安定したユーザーバイナリディレクトリを保持しますが、推測されたバージョンマネージャーのフォールバックディレクトリがサービスの PATH に書き込まれるのは、そのディレクトリがディスク上に存在する場合だけです。

  </Accordion>
  <Accordion title="18. 設定の書き込みとウィザードメタデータ">
    Doctor は設定への変更を保存し、doctor の実行を記録するためにウィザードメタデータを付与します。
  </Accordion>
  <Accordion title="19. ワークスペースのヒント（バックアップとメモリシステム）">
    Doctor は、ワークスペースのメモリシステムがない場合にその導入を提案し、ワークスペースがまだ git で管理されていない場合はバックアップのヒントを表示します。

    ワークスペースの構成と git バックアップ（非公開の GitHub または GitLab を推奨）についての詳細なガイドは、[/concepts/agent-workspace](/ja-JP/concepts/agent-workspace)を参照してください。

  </Accordion>
</AccordionGroup>

## 関連情報

- [Gateway 運用手順書](/ja-JP/gateway)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)
