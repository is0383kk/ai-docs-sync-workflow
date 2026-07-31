---
read_when:
    - CLI から Gateway を実行する（開発環境またはサーバー）
    - Gateway の認証、バインドモード、接続性のデバッグ
    - Bonjour による Gateway の検出（ローカル + 広域 DNS-SD）
    - 外部 Gateway プロセススーパーバイザーの統合
sidebarTitle: Gateway
summary: OpenClaw Gateway CLI（`openclaw gateway`）— Gateway の実行、照会、検出
title: Gateway
x-i18n:
    generated_at: "2026-07-26T09:29:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

Gateway は OpenClaw の WebSocket サーバー（チャンネル、Node、セッション、フック）です。以下のすべてのサブコマンドは `openclaw gateway ...` の配下にあります。

<CardGroup cols={3}>
  <Card title="Bonjour 検出" href="/ja-JP/gateway/bonjour">
    ローカル mDNS + 広域 DNS-SD のセットアップ。
  </Card>
  <Card title="検出の概要" href="/ja-JP/gateway/discovery">
    OpenClaw が Gateway をアドバタイズし、検出する仕組み。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration">
    トップレベルの Gateway 設定キー。
  </Card>
</CardGroup>

## Gateway を実行する

```bash
openclaw gateway
openclaw gateway run   # 同等の明示的な形式
```

<AccordionGroup>
  <Accordion title="起動時の動作">
    - `~/.openclaw/openclaw.json` で `gateway.mode=local` が設定されていない限り、起動を拒否します。アドホック実行や開発用の実行には `--allow-unconfigured` を使用してください。設定の書き込みや修復を行わずに、このガードを回避します。
    - 起動時に修復可能な無効な設定が検出されると、対話型ターミナルでは `openclaw doctor --fix` の実行を提案し、同意後に起動を 1 回再試行します。非対話型実行では自動修復を一切行わず、代わりにコマンドを表示します。修復後の設定が依然として無効な場合、起動は停止したままです。
    - `openclaw onboard --mode local` と `openclaw setup` は `gateway.mode=local` を書き込みます。設定ファイルは存在するものの `gateway.mode` がない場合、設定が破損または上書きされたものとして扱われ、Gateway は `local` を推測しません。オンボーディングを再実行するか、キーを手動で設定するか、`--allow-unconfigured` を渡してください。
    - 認証なしで loopback を超えてバインドすることはブロックされます。
    - `--bind` の値 `lan`、`tailnet`、`custom` は、現在 IPv4 専用パスで解決されます。IPv6 専用の独自ホスト構成では、Gateway の前段に IPv4 サイドカーまたはプロキシが必要です。
    - `SIGUSR1` は、許可されるとプロセス内再起動をトリガーします。`commands.restart`（デフォルト: 有効）は、外部から送信される `SIGUSR1` を制御します。手動の OS シグナルによる再起動をブロックするには、`false` に設定してください。エージェント向けの `gateway` ツールは読み取り専用です。エージェントは、人間が承認した `openclaw` 委譲ツールを通じて再起動を要求します。
    - `SIGINT`/`SIGTERM` はプロセスを停止しますが、カスタムのターミナル状態は復元しません。CLI を TUI や raw モード入力でラップしている場合は、終了前にターミナルを自身で復元してください。

  </Accordion>
</AccordionGroup>

### オプション

<ParamField path="--port <port>" type="number">
  WebSocket ポート（デフォルトは設定/環境変数から取得。通常は `18789`）。
</ParamField>
<ParamField path="--bind <mode>" type="string">
  バインドモード: `loopback`（デフォルト）、`lan`、`tailnet`、`auto`、`custom`。
</ParamField>
<ParamField path="--token <token>" type="string">
  `connect.params.auth.token` 用の共有トークン。設定されている場合、デフォルトは `OPENCLAW_GATEWAY_TOKEN` です。
</ParamField>
<ParamField path="--auth <mode>" type="string">
  認証モード: `none`、`token`、`password`、`trusted-proxy`。
</ParamField>
<ParamField path="--password <password>" type="string">
  `--auth password` 用のパスワード。
</ParamField>
<ParamField path="--password-file <path>" type="string">
  ファイルから Gateway のパスワードを読み取ります。
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale の公開方法: `off`、`serve`、`funnel`。
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  シャットダウン時に Tailscale の serve/funnel 設定をリセットします。
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  `gateway.mode=local` を強制せずに起動します。アドホック/開発用のブートストラップ専用です。設定の永続化や修復は行いません。
</ParamField>
<ParamField path="--dev" type="boolean">
  存在しない場合に開発用の設定とワークスペースを作成します（`BOOTSTRAP.md` はスキップします）。
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  開発用 Gateway が周囲の環境変数からチャンネルを自動設定できるようにします。`--dev` が必要です。
</ParamField>
<ParamField path="--reset" type="boolean">
  開発用の設定、資格情報、セッション、ワークスペースをリセットします。`--dev` が必要です。
</ParamField>
<ParamField path="--force" type="boolean">
  起動前に、対象ポート上の既存のリスナーをすべて終了します。非対話型シェルでは、検証済みの Gateway リスナーの終了を拒否します。代わりに `--dev`、または空きポートを指定した分離済みの `--profile` を使用してください。
</ParamField>
<ParamField path="--verbose" type="boolean">
  stdout/stderr への詳細ログ出力。
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  コンソールには CLI バックエンドログのみを表示します（stdout/stderr も有効になります）。
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket ログ形式: `auto`、`full`、`compact`。
</ParamField>
<ParamField path="--compact" type="boolean">
  `--ws-log compact` のエイリアス。
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  モデルの raw ストリームイベントを JSONL に記録します。
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  raw ストリームの JSONL パス。
</ParamField>

`--claude-cli-logs` は `--cli-backend-logs` の非推奨エイリアスです。

`--bind custom` の場合、`gateway.customBindHost` を IPv4 アドレスに設定してください。`127.0.0.1` または `0.0.0.0` 以外のアドレスでは、同一ホストのクライアント用に、同じポート上の `127.0.0.1` も必要です。いずれかのリスナーをバインドできない場合、起動は失敗します。ワイルドカード `0.0.0.0` では、必須の別エイリアスは追加されません。IPv6 専用の独自ホスト構成では、Gateway の前段に IPv4 サイドカーまたはプロキシが必要です。

## Gateway を再起動する

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` は、実行中の Gateway にアクティブな処理の事前確認を要求し、その処理が完了した後に、集約された 1 回の再起動をスケジュールします。待機時間の上限は 5 分です。この時間を超えると、再起動が強制されます。`--safe` は `--force` または `--wait` と併用できません。

`--skip-deferral` は安全な再起動時のアクティブ処理の延期ゲートを迂回するため、報告されたブロッカーが存在する場合でも Gateway を直ちに再起動します。`--safe` が必要です。暴走タスクによって延期が停止している場合に使用してください。

`--wait <duration>` は、通常の（安全モードではない）再起動時のドレイン時間の上限を上書きします。単位なしのミリ秒、または単位接尾辞 `ms`、`s`、`m`、`h`、`d`（例: `30s`、`5m`、`1h30m`）を指定できます。`--wait 0` は無期限に待機します。`--force` または `--safe` とは互換性がありません。

`--force` はアクティブ処理のドレインをスキップし、直ちに再起動します。通常の `restart`（フラグなし）では、既存のサービスマネージャーによる再起動動作が維持されます。

<Warning>
インラインの `--password` は、ローカルのプロセス一覧に表示される可能性があります。`--password-file`、環境変数、または SecretRef を使用する `gateway.auth.password` を推奨します。
</Warning>

### 外部スーパーバイザー

別のプロセスマネージャーが Gateway のライフサイクルを所有する場合にのみ、`OPENCLAW_SUPERVISOR_MODE=external` を設定してください。このモードでは次のように動作します。

- `openclaw gateway restart` は、launchd、systemd、または Task Scheduler ではなく、検証済みの実行中 Gateway を対象にしながら、既存の安全な再起動、強制再起動、待機時間制限付き再起動の動作を維持します。
- ネイティブサービスのインストール、起動、停止、アンインストール操作は拒否され、外部スーパーバイザーを使用するよう案内されます。
- スーパーバイザーが Gateway を停止し、ランタイムを置き換えて最終処理を行い、安全に再起動できるように、OpenClaw の自己更新は拒否されます。
- 新しいプロセスでの再起動では、正常終了する前に、上限付きの SQLite ハンドオフを書き込みます。永続化に失敗した場合、利用可能なハンドオフを残さずに終了する代わりに、Gateway はプロセス内再起動へフォールバックします。

`OPENCLAW_SERVICE_REPAIR_POLICY=external` は、独立した Doctor の修復ポリシーとして維持されます。これはランタイムの所有権を宣言するものではありません。両方の動作が必要なスーパーバイザーでは、両方の変数を設定してください。

外部スーパーバイザーは、非公開のマシン向けコントラクトを通じて、再起動ハンドオフをネゴシエートして取得できます。

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

プロトコルバージョン `1` は `consume` 操作をサポートします。取得時には、1 回の即時 SQLite トランザクション内で、想定 PID と上限付きハンドオフフィールドを検証します。受理されたハンドオフは成功を返す前に削除されるため、並行するコンシューマーや再実行されたコンシューマーが両方とも受理することはできません。PID が一致しない場合は、対応する所有者のために保持されます。欠落、期限切れ、無効な行によって再起動が許可されることはありません。

有効なマシン要求は、再起動しない結果も含め、終了コード `0` で JSON を返します。無効な引数は、終了コード `2` で `reason: "invalid-expected-pid"` を返します。状態ストアの障害は、終了コード `1` で `reason: "store-unavailable"` を返します。スーパーバイザーは、OpenClaw のバージョン文字列からサポート状況を推測したり、非公開の SQLite スキーマを直接読み取ったりせず、実際に使用するランタイムまたはランチャー上で `capabilities` をプローブしてください。

### Gateway のプロファイリング

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` は起動中の各フェーズの所要時間を記録します。これには、フェーズごとの `eventLoopMax` 遅延と、Plugin ルックアップテーブルの所要時間（インストール済みインデックス、マニフェストレジストリ、起動計画、所有者マップ処理）が含まれます。
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` は、再起動単位の `restart trace:` 行を記録します。シグナル処理、アクティブ処理のドレイン、シャットダウンフェーズ、次回起動、準備完了までの時間、メモリメトリクスが含まれます。
- `OPENCLAW_DIAGNOSTICS=timeline` と `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` を併用すると、外部 QA ハーネス向けに、ベストエフォートの JSONL 起動診断タイムラインを書き込みます（設定の `diagnostics.flags: ["timeline"]` と同等ですが、パスは引き続き環境変数でのみ指定できます）。イベントループのサンプルを含めるには、`OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` を追加してください。
- `pnpm build` の後に `pnpm test:startup:gateway -- --runs 5 --warmup 1` を実行すると、ビルド済み CLI エントリを基準に Gateway の起動をベンチマークします。最初のプロセス出力、`/healthz`、`/readyz`、起動トレースの所要時間、イベントループ遅延、Plugin ルックアップテーブルの所要時間が対象です。
- `pnpm build` の後に `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5` を実行すると、macOS または Linux 上でプロセス内再起動をベンチマークします（Windows ではサポートされていません。再起動には `SIGUSR1` が必要です）。`SIGUSR1` を使用し、子プロセスで両方のトレースを有効にして、次回の `/healthz`、次回の `/readyz`、ダウンタイム、準備完了までの時間、CPU、RSS、再起動トレースのメトリクスを記録します。
- `/healthz` は生存性、`/readyz` は利用可能な準備完了状態を表します。トレース行とベンチマーク出力は、単一の期間やサンプルから導く完全なパフォーマンス結論ではなく、所有者への帰属を判断するシグナルとして扱ってください。

## 実行中の Gateway に問い合わせる

すべての問い合わせコマンドは WebSocket RPC を使用します。

<Tabs>
  <Tab title="出力モード">
    - デフォルト: 人間が読みやすい形式（TTY では色付き）。
    - `--json`: 機械可読の JSON（装飾やスピナーなし）。
    - `--no-color`（または `NO_COLOR=1`）: 人間向けのレイアウトを維持しながら ANSI を無効にします。

  </Tab>
  <Tab title="共通オプション">
    - `--url <url>`: Gateway の WebSocket URL。
    - `--token <token>`: Gateway のトークン。
    - `--password <password>`: Gateway のパスワード。
    - `--timeout <ms>`: タイムアウト/時間上限（デフォルトはコマンドごとに異なります。以下の各コマンドを参照してください）。
    - `--expect-final`: 「final」レスポンスを待機します（エージェント呼び出し）。

  </Tab>
</Tabs>

<Note>
`--url` を設定すると、CLI は設定や環境変数の資格情報へフォールバックしません。`--token` または `--password` を明示的に渡してください。明示的な資格情報がない場合はエラーになります。
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` は稼働性プローブです。サーバーが HTTP に応答できるようになると、すぐに結果を返します。`/readyz` はより厳格で、起動中の Plugin サイドカー、チャンネル、または設定済みフックが安定するまでは異常状態のままです。ローカルまたは認証済みの詳細な `/readyz` レスポンスには、`eventLoop` 診断ブロック（遅延、使用率、CPU コア比率、`degraded` フラグ）が含まれます。

<ParamField path="--port <port>" type="number">
  このポート上の local loopback Gateway を対象にします。この呼び出しでは `OPENCLAW_GATEWAY_URL` と `OPENCLAW_GATEWAY_PORT` を上書きします。
</ParamField>

### `gateway usage-cost`

セッションログから使用コストの概要を取得します。

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  対象に含める日数。
</ParamField>
<ParamField path="--agent <id>" type="string">
  概要の対象を、設定済みの 1 つのエージェント ID に限定します。
</ParamField>
<ParamField path="--all-agents" type="boolean">
  設定済みのすべてのエージェントを集計します。`--agent` とは併用できません。
</ParamField>

### `gateway stability`

実行中の Gateway から最近の診断安定性レコーダーを取得します。

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  対象に含める最近のイベントの最大数（最大 `1000`）。
</ParamField>
<ParamField path="--type <type>" type="string">
  診断イベントタイプ（例: `payload.large` または `diagnostic.memory.pressure`）で絞り込みます。
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  診断シーケンス番号より後のイベントのみを含めます。
</ParamField>
<ParamField path="--bundle [path]" type="string">
  実行中の Gateway を呼び出す代わりに、永続化された安定性バンドルを読み取ります。`--bundle latest`（または引数なしの `--bundle`）を指定すると、状態ディレクトリ内の最新バンドルが選択されます。バンドルの JSON パスを直接渡すこともできます。
</ParamField>
<ParamField path="--export" type="boolean">
  安定性の詳細を出力する代わりに、共有可能なサポート診断用 zip を書き出します。
</ParamField>
<ParamField path="--output <path>" type="string">
  `--export` の出力パス。
</ParamField>

<AccordionGroup>
  <Accordion title="プライバシーとバンドルの動作">
    - レコードには、イベント名、件数、バイトサイズ、メモリ測定値、キュー／セッション状態、承認 ID、チャンネル／Plugin 名、秘匿化されたセッション概要などの運用メタデータが保持されます。チャットテキスト、Webhook 本文、ツール出力、生のリクエスト／レスポンス本文、トークン、Cookie、シークレット値、ホスト名、生のセッション ID は除外されます。レコーダーを完全に無効化するには、`diagnostics.enabled: false` を設定します。
    - レコーダーにイベントがある場合、Gateway の致命的な終了、シャットダウンのタイムアウト、再起動時の起動失敗では、同じ診断スナップショットが `~/.openclaw/logs/stability/openclaw-stability-*.json` に書き込まれます。最新のバンドルは `openclaw gateway stability --bundle latest` で確認できます。`--limit`、`--type`、`--since-seq` はバンドル出力にも適用されます。

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

バグ報告用に設計されたローカル診断 zip を書き出します。プライバシーモデルとバンドル内容については、[診断エクスポート](/ja-JP/gateway/diagnostics)を参照してください。

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  出力 zip のパス。デフォルトでは、状態ディレクトリ配下のサポート用エクスポートになります。
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  対象に含めるサニタイズ済みログ行の最大数。
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  検査するログの最大バイト数。
</ParamField>
<ParamField path="--url <url>" type="string">
  ヘルススナップショット用の Gateway WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  ヘルススナップショット用の Gateway トークン。
</ParamField>
<ParamField path="--password <password>" type="string">
  ヘルススナップショット用の Gateway パスワード。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  ステータス／ヘルススナップショットのタイムアウト。
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  永続化された安定性バンドルの検索をスキップします。
</ParamField>
<ParamField path="--json" type="boolean">
  書き込まれたパス、サイズ、マニフェストを JSON として出力します。
</ParamField>

エクスポートには、`manifest.json`（ファイル一覧）、`summary.md`（Markdown 概要）、`diagnostics.json`（最上位の設定／ログ／検出／安定性／ステータス／ヘルス概要）、`config/sanitized.json`、`status/gateway-status.json`、`health/gateway-health.json`、`logs/openclaw-sanitized.jsonl`、およびバンドルが存在する場合は `stability/latest.json` が含まれます。

このエクスポートは共有を前提に設計されています。安全なログフィールド、サブシステム名、ステータスコード、所要時間、設定済みモード、ポート、Plugin／プロバイダー ID、シークレットではない機能設定、秘匿化された運用ログメッセージなど、デバッグに役立つ運用詳細は保持されます。一方で、チャットテキスト、Webhook 本文、ツール出力、認証情報、Cookie、アカウント／メッセージ識別子、プロンプト／指示テキスト、ホスト名、シークレット値は省略または秘匿化されます。ログメッセージがユーザー／チャット／ツールのペイロードテキスト（例: 「user said」、「chat text」、「tool output」、「webhook body」）に見える場合、エクスポートにはメッセージが省略されたという事実とそのバイト数のみが保持されます。

### `gateway status`

Gateway サービス（launchd/systemd/schtasks）と、任意の接続性／認証プローブを表示します。

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  明示的なプローブ対象を追加します。設定済みのリモートと localhost も引き続きプローブされます。
</ParamField>
<ParamField path="--token <token>" type="string">
  プローブ用のトークン認証。
</ParamField>
<ParamField path="--password <password>" type="string">
  プローブ用のパスワード認証。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  プローブのタイムアウト。
</ParamField>
<ParamField path="--no-probe" type="boolean">
  接続性プローブをスキップします（サービスのみの表示）。
</ParamField>
<ParamField path="--deep" type="boolean">
  システムレベルのサービスもスキャンします。
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  接続性プローブを読み取りプローブに強化し、失敗した場合はゼロ以外の終了コードで終了します。`--no-probe` とは併用できません。
</ParamField>

<AccordionGroup>
  <Accordion title="ステータスの意味">
    - ローカル CLI 設定が存在しないか無効な場合でも、診断に使用できます。
    - デフォルト出力で確認できるのは、サービス状態、WebSocket 接続、ハンドシェイク時に確認できる認証機能です。読み取り／書き込み／管理操作ではありません。
    - 初回デバイス認証に対するプローブは非変更的です。既存のキャッシュ済みデバイストークンがあれば再利用しますが、ステータス確認のためだけに新しい CLI デバイス ID や読み取り専用ペアリングレコードを作成することはありません。
    - 可能な場合、プローブ認証用に設定済みの認証 SecretRef を解決します。必須の SecretRef を解決できず、プローブの接続性／認証が失敗した場合、`--json` は `rpc.authWarning` を報告します。`--token`/`--password` を明示的に渡すか、シークレットのソースを修正してください。プローブが成功すると、未解決の認証に関する警告は抑制されます。
    - 実行中の Gateway が `gateway.version` を報告する場合、JSON 出力にそれが含まれます。ハンドシェイクプローブからバージョンメタデータを取得できない場合、`--require-rpc` は `status.runtimeVersion` RPC ペイロードにフォールバックできます。
    - リッスン中のサービスだけでは不十分で、読み取りスコープの RPC も正常である必要があるスクリプト／自動化では、`--require-rpc` を使用します。
    - `--deep` は追加の launchd/systemd/schtasks インストールをスキャンします。Gateway に似たサービスが複数見つかった場合、人間向け出力にはクリーンアップのヒント（通常はマシンごとに 1 つの Gateway を実行）が表示され、該当する場合は最近のスーパーバイザー再起動の引き継ぎも報告されます。
    - `--deep` は Plugin 対応モード（`pluginValidation: "full"`）で設定検証も実行し、Plugin マニフェストの警告（例: チャンネル設定メタデータの欠落）を表示します。デフォルトの `gateway status` では、Plugin 検証をスキップする高速な読み取り専用パスが維持されます。
    - 人間向け出力には、プロファイルや状態ディレクトリのずれを診断しやすくするため、解決済みファイルログパスと、CLI／サービスそれぞれの設定パスおよび有効性が含まれます。
    - 人間向け出力には、適用された上限とその適応的な導出を示す `Gateway heap:` が含まれます。JSON 出力では、同じレポートが `service.gatewayHeap` として公開されます。

  </Accordion>
  <Accordion title="Linux systemd の認証ドリフトチェック">
    - サービス認証のドリフトチェックでは、ユニットから `Environment=` と `EnvironmentFile=` の両方を読み取ります（`%h`、引用符付きパス、複数ファイル、任意指定の `-` ファイルを含む）。
    - マージされたランタイム環境（最初にサービスコマンド環境、次にプロセス環境へのフォールバック）を使用して、`gateway.auth.token` SecretRef を解決します。
    - トークン認証が実質的に有効でない場合（`gateway.auth.mode` が明示的に `password`/`none`/`trusted-proxy` である場合、またはモードが未設定でパスワードが優先され、どのトークン候補も優先され得ない場合）、トークンドリフトチェックは設定トークンの解決をスキップします。

  </Accordion>
</AccordionGroup>

### `gateway probe`

「すべてをデバッグ」するコマンドです。常に次をプローブします。

- 設定済みのリモート Gateway（設定されている場合）、および
- localhost（ループバック）。**リモートが設定されている場合でも**プローブします。

`--url` を渡すと、その明示的な対象が両方より前に追加されます。人間向け出力では、対象に `URL (explicit)`、`Remote (configured)` / `Remote (configured, inactive)`、`Local loopback` というラベルが付けられます。

<Note>
複数のプローブ対象に到達できる場合、すべてが出力されます。SSH トンネル、TLS／プロキシ URL、設定済みのリモート URL は、転送ポートが異なっていても同じ Gateway を指すことがあります。`multiple_gateways` は、別個の Gateway または ID が曖昧で到達可能な Gateway のために予約されています。分離されたプロファイル（例: レスキューボット）では複数の Gateway の実行がサポートされていますが、ほとんどのインストールでは単一の Gateway を実行します。
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  local loopback プローブ対象と SSH トンネルのリモートポートにこのポートを使用します。`--url` を指定しない場合、設定済みの Gateway 環境 URL、環境ポート、またはリモート対象の代わりに、local loopback 対象のみが選択されます。
</ParamField>

<AccordionGroup>
  <Accordion title="解釈">
    - `Reachable: yes` は、少なくとも 1 つの対象が WebSocket 接続を受け入れたことを意味します。
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` は、到達可能性とは別に、プローブが認証について確認できた内容を報告します。
    - `Read probe: ok` は、読み取りスコープの詳細 RPC 呼び出し（`health`/`status`/`system-presence`/`config.get`）も成功したことを意味します。
    - `Read probe: limited - missing scope: operator.read` は、接続には成功したものの、読み取りスコープの RPC が制限されていることを意味します。完全な失敗ではなく、到達可能性が**低下**しているものとして報告されます。
    - `Connect: ok` の後の `Read probe: failed` は、WebSocket は接続できたものの、その後の読み取り診断がタイムアウトまたは失敗したことを意味します。これも到達不能ではなく**低下**として扱われます。
    - `gateway status` と同様に、プローブは既存のキャッシュ済みデバイス認証を再利用しますが、初回のデバイス ID やペアリング状態は作成しません。
    - プローブした対象に 1 つも到達できない場合にのみ、終了コードがゼロ以外になります。

  </Accordion>
  <Accordion title="JSON 出力">
    最上位:

    - `ok`: 少なくとも1つのターゲットに到達可能です。
    - `degraded`: 少なくとも1つのターゲットが接続を受け入れましたが、完全な詳細 RPC 診断は完了しませんでした。
    - `capability`: 到達可能なターゲット全体で確認された最良の機能（`read_only`、`write_capable`、`admin_capable`、`pairing_pending`、`connected_no_operator_scope`、または `unknown`）。
    - `primaryTargetId`: アクティブな優先ターゲットとして扱う最適なターゲット。優先順は、明示的な URL、SSH トンネル、設定済みリモート、local loopback です。
    - `warnings[]`: `code`、`message`、および任意の `targetIds` を含むベストエフォートの警告レコード。
    - `network`: 現在の設定とホストネットワークから導出された local loopback/tailnet URL のヒント。
    - `discovery.timeoutMs` / `discovery.count`: このプローブ処理で実際に使用された検出予算／結果数。

    ターゲットごと（`targets[].connect`）: `ok`（到達可能性 + 機能低下の分類）、`rpcOk`（完全な詳細 RPC の成功）、`scopeLimited`（operator スコープがないため詳細 RPC が失敗）。

    ターゲットごと（`targets[].auth`）: 利用可能な場合は `role` と `scopes` が `hello-ok` で報告され、さらに公開された `capability` の分類が含まれます。

  </Accordion>
  <Accordion title="一般的な警告コード">
    - `ssh_tunnel_failed`: SSH トンネルの設定に失敗しました。コマンドは直接プローブにフォールバックしました。
    - `multiple_gateways`: 異なる Gateway ID に到達可能だったか、到達可能なターゲットが同じ Gateway であることを OpenClaw が証明できませんでした。同じ Gateway への SSH トンネル、プロキシ URL、または設定済みリモート URL では、この警告は発生しません。
    - `auth_secretref_unresolved`: 失敗したターゲットについて、設定済みの認証 SecretRef を解決できませんでした。
    - `probe_scope_limited`: WebSocket 接続には成功しましたが、`operator.read` がないため読み取りプローブが制限されました。
    - `local_tls_runtime_unavailable`: ローカル Gateway の TLS は有効ですが、OpenClaw がローカル証明書のフィンガープリントを読み込めませんでした。

  </Accordion>
</AccordionGroup>

#### SSH 経由のリモート（Mac アプリと同等）

macOS アプリの「Remote over SSH」モードではローカルポートフォワーディングを使用し、loopback のみに制限されたリモート Gateway を `ws://127.0.0.1:<port>` で到達可能にします。

同等の CLI コマンド:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` または `user@host:port`（ポートのデフォルトは `22`）。
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  ID ファイル。
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  解決された検出エンドポイント（`local.` と、設定されている場合は広域ドメイン）から、最初に検出された Gateway ホストを SSH ターゲットとして選択します。TXT のみのヒントは無視されます。
</ParamField>

設定のデフォルト（任意）: `gateway.remote.sshTarget`、`gateway.remote.sshIdentity`。

### `gateway call <method>`

低レベル RPC ヘルパー。

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  パラメーター用の JSON オブジェクト文字列。
</ParamField>
<ParamField path="--url <url>" type="string">
  Gateway の WebSocket URL。
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway トークン。
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway パスワード。
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  タイムアウト予算。
</ParamField>
<ParamField path="--expect-final" type="boolean">
  主に、最終ペイロードの前に中間イベントをストリーミングするエージェント形式の RPC に使用します。
</ParamField>
<ParamField path="--json" type="boolean">
  機械可読な JSON 出力。
</ParamField>

<Note>
`--params` は有効な JSON である必要があり、各メソッドが独自のパラメーター形式を検証します（余分なフィールドや名前が誤っているフィールドは拒否されます）。
</Note>

## Gateway サービスの管理

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### ラッパーを使用したインストール

管理対象サービスを別の実行ファイル（シークレットマネージャーのシムや run-as ヘルパーなど）経由で起動する必要がある場合は、`--wrapper` を使用します。ラッパーは通常の Gateway 引数を受け取り、最終的にその引数を指定して `openclaw` または Node を exec する役割を担います。

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

環境を通じてラッパーを設定することもできます。`gateway install` は、パスが実行可能ファイルであることを検証し、ラッパーをサービスの `ProgramArguments` に書き込み、その後の強制再インストール、更新、doctor による修復に使用できるよう、サービス環境に `OPENCLAW_WRAPPER` を永続化します。

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

永続化されたラッパーを削除するには、再インストール時に `OPENCLAW_WRAPPER` を空にします。

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="コマンドオプション">
    - `gateway status`: `--url`、`--token`、`--password`、`--timeout`、`--no-probe`、`--require-rpc`、`--deep`、`--json`
    - `gateway install`: `--port`、`--runtime <node>`（デフォルト: `node`）、`--token`、`--wrapper <path>`、`--force`、`--json`
    - `gateway restart`: `--safe`、`--skip-deferral`、`--force`、`--wait <duration>`、`--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`、`--force`、`--json`

  </Accordion>
  <Accordion title="ライフサイクルの動作">
    - `gateway start` はべき等です。管理対象サービスがすでに実行中の場合は、実行中のプロセスを報告し、変更を加えません。読み込み済みで停止中のサービスは、従来どおり起動されます。
    - 管理対象サービスを再起動するには、`gateway restart` を使用します。再起動の代わりに `gateway stop` と `gateway start` を連続実行しないでください。
    - 非対話型シェルでは、`gateway stop` に `--force` が必要です。対話型ターミナルでは、既存のプロンプトなしの動作が維持されます。自動化とテストでは、`gateway run --dev`、または空いているポートを使用する分離された `--profile` を推奨します。
    - macOS では、`gateway stop` はデフォルトで `launchctl bootout` を使用します。これにより、無効化を永続化せずに現在の起動セッションから LaunchAgent が削除されます。KeepAlive による自動復旧は今後のクラッシュでも引き続き有効であり、`gateway start` は手動の `launchctl enable` なしで正常に再有効化されます。Gateway が次に明示的な `gateway start` を実行するまで再生成されないよう KeepAlive と RunAtLoad を永続的に抑制するには、`--disable` を指定します。手動停止を再起動後も維持する必要がある場合に使用してください。
    - Gateway のライフサイクル変更では、CLI の起動、停止、再起動操作、安全な再起動リクエスト、スーパーバイザーによる再起動、およびデタッチされた引き継ぎを含む、ベストエフォートのキーと値の監査レコードが `<state-dir>/logs/gateway-restart.log` に追記されます。
    - ライフサイクルコマンドは、スクリプト処理用の `--json` を受け付けます。

  </Accordion>
  <Accordion title="管理対象 Gateway のヒープサイズ設定">
    - `gateway install` は、管理対象 Gateway サービス向けにヒープのみに適用される `NODE_OPTIONS` の値を書き込みます。Node がコンテナまたはサービスの制限を報告する場合は制限メモリの 50%、それ以外の場合は物理メモリの 50% を目標とします。
    - 標準の目標範囲は 2048～8192 MiB で、さらにネイティブ用ヘッドルームを 75% 確保する上限が適用されます。小規模なホストでは、このヘッドルーム上限により、適用される制限が標準の下限である 2048 MiB を下回ることがあります。
    - インストール済みサービスにすでに保存されている有効な明示的 `--max-old-space-size` は、強制再インストールや doctor による修復後も維持されます。その他の `NODE_OPTIONS` フラグは管理対象サービスに引き継がれません。
    - シェル環境の `NODE_OPTIONS` は、このポリシーを上書きしません。インストール済みの値を確認するには `gateway status` または `doctor` を使用してください。管理対象ヒープ設定がない古いサービスメタデータを再生成するには、`openclaw gateway install --force` を実行します。
    - このポリシーは管理対象 Gateway サービスにのみ適用されます。フォアグラウンドの `gateway run`、Node サービス、および手動で作成されたスーパーバイザーユニットでは、それぞれ独自のランタイム設定が維持されます。

  </Accordion>
  <Accordion title="インストール時の認証と SecretRef">
    - トークン認証にトークンが必要で、`gateway.auth.token` が SecretRef で管理されている場合、`gateway install` は SecretRef を解決できることを検証しますが、解決したトークンをサービス環境のメタデータには永続化しません。
    - トークン認証にトークンが必要で、設定済みのトークン SecretRef を解決できない場合、フォールバックの平文を永続化せず、インストールはフェイルクローズします。
    - `gateway run` でのパスワード認証には、インラインの `--password` よりも、`OPENCLAW_GATEWAY_PASSWORD`、`--password-file`、または SecretRef を使用する `gateway.auth.password` を推奨します。
    - 推論された認証モードでは、シェルのみの `OPENCLAW_GATEWAY_PASSWORD` によってインストール時のトークン要件が緩和されることはありません。管理対象サービスのインストール時には、永続的な設定（`gateway.auth.password` または設定内の `env`）を使用してください。
    - `gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、モードが明示的に設定されるまでインストールはブロックされます。

  </Accordion>
</AccordionGroup>

## Gateway の検出（Bonjour）

`gateway discover` は Gateway ビーコン（`_openclaw-gw._tcp`）をスキャンします。

- マルチキャスト DNS-SD: `local.`
- ユニキャスト DNS-SD（広域 Bonjour）: ドメイン（例: `openclaw.internal.`）を選択し、スプリット DNS と DNS サーバーを設定します。詳細は [Bonjour](/ja-JP/gateway/bonjour) を参照してください。

Bonjour 検出が有効（デフォルト）な Gateway のみがビーコンをアドバタイズします。

各ビーコンの TXT ヒント: `role`（Gateway ロールのヒント）、`transport`（トランスポートのヒント、例: `gateway`）、`gatewayPort`（WebSocket ポート、通常は `18789`）、`tailnetDns`（利用可能な場合は MagicDNS ホスト名）、`gatewayTls` / `gatewayTlsSha256`（TLS の有効化状態 + 証明書フィンガープリント）。`sshPort` と `cliPath` は、完全検出モード（`discovery.mdns.mode: "full"`）でのみ公開されます。デフォルトは `"minimal"` で、これらは省略されます。この場合、クライアントは SSH ターゲットのデフォルトポートとして `22` を使用します。

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  コマンドごとのタイムアウト（ブラウズ／解決）。
</ParamField>
<ParamField path="--json" type="boolean">
  機械可読な出力（スタイル設定／スピナーも無効になります）。
</ParamField>

例:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- `local.` と、有効になっている場合は設定済みの広域ドメインをスキャンします。
- JSON 出力の `wsUrl` は、`lanHost` や `tailnetDns` などの TXT のみのヒントではなく、解決されたサービスエンドポイントから導出されます。
- `discovery.mdns.mode` は、`local.` mDNS と広域 DNS-SD の両方での `sshPort`/`cliPath` の公開を制御します（前述を参照）。

</Note>

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Gateway 運用手順書](/ja-JP/gateway)
