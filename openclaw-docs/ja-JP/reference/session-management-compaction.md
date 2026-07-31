---
read_when:
    - セッション ID、トランスクリプトイベント、またはセッション行のフィールドをデバッグする必要がある場合
    - 自動 Compaction の動作を変更するか、「Compaction 前」のハウスキーピングを追加する場合
    - メモリのフラッシュまたはサイレントなシステムターンを実装したい場合
summary: 詳細解説：セッションストアとトランスクリプト、ライフサイクル、（自動）Compaction の内部構造
title: セッション管理の詳細解説
x-i18n:
    generated_at: "2026-07-26T10:30:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

1 つの **Gateway プロセス**がセッション状態をエンドツーエンドで管理します。UI（macOS アプリ、Web Control UI、TUI）は、セッション一覧とトークン数を Gateway に問い合わせます。リモートモードでは、セッションファイルはリモートホストに存在するため、ローカル Mac のファイルを確認しても Gateway が使用している内容は反映されません。

まず概要ドキュメントを参照してください：[セッション管理](/ja-JP/concepts/session)、[Compaction](/ja-JP/concepts/compaction)、[メモリの概要](/ja-JP/concepts/memory)、[メモリ検索](/ja-JP/concepts/memory-search)、[セッションのプルーニング](/ja-JP/concepts/session-pruning)、[トランスクリプトの整理](/ja-JP/reference/transcript-hygiene)。完全な設定リファレンスは[エージェント設定](/ja-JP/gateway/config-agents)にあります。

## 2 つの永続化レイヤー

1. **セッション行（エージェントごとの SQLite）** - キー/値マップ `sessionKey -> SessionEntry`。Gateway が所有する可変のランタイム状態です。現在のセッション ID、最終アクティビティ、切り替え設定、トークンカウンターなどのメタデータを追跡します。
2. **トランスクリプトイベント（エージェントごとの SQLite）** - 追記専用のツリー構造（エントリには `id` + `parentId` があります）。会話、ツール呼び出し、Compaction の要約を保存し、以降のターン用にモデルコンテキストを再構築します。Compaction チェックポイントは、Compaction 後の後続トランスクリプトに付随するメタデータです。新しい Compaction で 2 つ目の `.checkpoint.*.jsonl` コピーが書き込まれることはありません。

古いインストールでは、エージェントの `sessions/`
ディレクトリ配下に `sessions.json` ファイルが残っている場合があります。
これらのファイルは、レガシーなセッション行の移行入力、または明示的な
オフラインメンテナンス対象として扱ってください。Gateway の起動時と
`openclaw doctor --fix` は、使用中のレガシー行とトランスクリプト履歴を
エージェントごとの SQLite ストアへ自動的にインポートします。明示的な
検査や検証証拠が必要な場合は、`openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` を実行してから
[Doctor の移行
手順](/ja-JP/cli/doctor#session-sqlite-migration)に従ってください。レガシーなトランスクリプト
アーティファクトのアーカイブ後に移行が失敗した場合は、その手順にある
Doctor の復旧モードを使用してください。復旧では移行マニフェストを使用し、
影響を受けたアーカイブ済みサポートアーティファクトだけを復元し、要求された場合は
サニタイズ済みの GitHub issue レポートを準備します。アクティブなランタイムが
JSONL ファイルを再び読み取るようになることはありません。

Gateway の履歴リーダーは、任意の過去データへのアクセスが必要な画面でない限り、トランスクリプト全体を実体化しません。履歴の最初のページ、埋め込みチャット履歴、再起動時の復旧、トークンおよび使用量の確認では、SQLite から末尾を制限付きで読み取ります。トランスクリプト全体のスキャンは非同期トランスクリプトインデックスを経由し、同時実行されるリーダー間で共有されます。

## ディスク上の場所

Gateway ホスト上で、エージェントごとに配置されます（`src/config/sessions.ts` により解決）：

- ランタイムセッション行ストア：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- ランタイムトランスクリプト行：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- レガシー/アーカイブのトランスクリプトアーティファクト：`~/.openclaw/agents/<agentId>/sessions/`
- レガシー行の移行入力：`~/.openclaw/agents/<agentId>/sessions/sessions.json`

## ストアのメンテナンスとディスク制御

`session.maintenance` は、SQLite セッション行、SQLite トランスクリプト行、アーカイブアーティファクト、軌跡サイドカーの自動メンテナンスを制御します：

| キー                     | デフォルト               | 注記                                                                                       |
| ----------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | または `"warn"`（報告のみ、変更なし）                                                      |
| `pruneAfter`            | `"30d"`               | 古いエントリを判定する経過時間のしきい値                                                                      |
| `maxEntries`            | `500`                 | セッションエントリ数の上限                                                                      |
| `resetArchiveRetention` | 維持（経過時間のしきい値なし）  | `*.reset.*`/`*.deleted.*` トランスクリプトアーカイブの経過時間のしきい値。期間を指定すると削除が有効になります |
| `maxDiskBytes`          | `10gb`                | エージェントごとのセッションディスク容量。`false` で無効化                                            |
| `highWaterBytes`        | `maxDiskBytes` の 80% | 容量クリーンアップ後の目標                                                                 |

リセットすると使用中の `sessionKey -> sessionId` マッピングは更新されますが、以前の SQLite セッション、トランスクリプト、軌跡、検索行は保持されます。その履歴は同じセッションキーで引き続き検索できますが、通常のエントリ一覧とセッション一覧には新しい使用中のマッピングだけが表示されます。保持されたリセット履歴はディスク容量によって制限されます。`resetArchiveRetention` はアーカイブアーティファクトの経過時間だけを制御するため、これによって制限されることはありません。明示的な削除は異なります。削除対象セッションの行を削除する前に、圧縮されたトランスクリプトアーカイブ（zstd が使用可能な場合は `*.jsonl.deleted.<timestamp>.zst`）を書き込み、検証します。

`maxDiskBytes` の適用には物理バイト数を使用します。対象は、エージェントごとの SQLite メインファイル、その `-wal` ファイル、およびエージェントのセッションディレクトリ内で集計対象となるファイルです。行の JSON サイズを推定したり、その合計から論理的な行サイズを差し引いたりすることはありません。

Gateway のモデル実行プローブセッション（`agent:*:explicit:model-run-<uuid>` に一致するキー）には、固定された別個の `24h` 保持期間が適用されます。このプルーニングは負荷条件付きです。セッションエントリのメンテナンスまたは上限による負荷に達した場合に限り、グローバルな古いエントリのクリーンアップ/上限制御ステップの直前にのみ実行されます。その他の明示的なセッションには、この保持期間は適用されません。

物理使用量の合計が `maxDiskBytes` を超えると、`mode: "enforce"` はまずチェックポイント処理可能なデータベース領域を回収し、次に保持されているリセット/削除アーカイブを古いものから削除します。それでも使用量が `highWaterBytes` を超えている場合は、履歴 SQLite セッションを `sessions.updated_at` の古い順に処理します。「履歴」とは、そのセッション ID が使用中のセッションエントリ、ルートターゲット、または許可済み/実行中の処理から参照されていないことを意味します。各削除対象について、クリーンアップは圧縮アーカイブを書き込み、fsync を実行し、読み戻した後、書き込みトランザクションでセッション行とそのトランスクリプト、軌跡、アクティブ、インデックス、FTS プロジェクションを削除します。これには、軌跡イベントは含むもののトランスクリプトイベントを含まないセッションも含まれます。クリーンアップは削除時にルート、エントリ、許可状態の参照を再確認し、アーカイブまたはセッションを 1 件削除するたびに物理使用量を再計測し、`highWaterBytes` に達すると停止します。

コミット済みの書き込みと削除は、まず WAL に記録されます。クリーンアップは WAL を直ちに縮小できるようチェックポイントを実行し、その後インクリメンタル vacuum を使用して、解放可能な末尾ページをメインファイルから返却します。まだ回収できないページはメインファイルに残るため、次回の物理計測でも引き続き集計されます。`mode: "warn"` は、チェックポイント、アーカイブの書き込み、行の削除を行わずに、現在の物理的な超過量を報告します。

必要に応じてメンテナンスを実行します：

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

メンテナンスでは、グループセッションやスレッド単位のチャットセッションなど、永続的な外部会話ポインターを保持します。一方、合成ランタイムエントリ（cron、フック、heartbeat、ACP、サブエージェント）は、設定された経過時間、件数、またはディスク容量を超えると削除される場合があります。分離された cron 実行では、モデル実行プローブの保持期間とは独立した `cron.sessionRetention` 制御を使用します。

通常の Gateway 書き込みはセッションアクセサーを経由し、ランタイムライターパスを通じてエージェントごとの SQLite 変更を直列化します。ランタイムコードでは `src/config/sessions/session-accessor.ts` のアクセサーヘルパーを優先してください。レガシーな `sessions.json` ヘルパーは、移行およびオフラインメンテナンス用のツールです。Gateway に到達可能な場合、dry-run ではない `openclaw sessions cleanup` と `openclaw agents delete` はストア変更を Gateway に委譲し、クリーンアップを同じライターキューに参加させます。`--store <path>` は選択したレガシーストア向けの明示的なオフライン修復パスであり、常にローカルで実行されます（`--dry-run` も同様です）。`maxEntries` のクリーンアップは本番規模のストア向けにバッチ処理されるため、次の高水位クリーンアップで上限以下に書き換えられるまで、ストアが設定上限を一時的に超える場合があります。Gateway の起動中、読み取りによってエントリがプルーニングされたり上限が適用されたりすることはありません。これを行うのは書き込みまたは `openclaw sessions cleanup --enforce` だけです。後者は上限も即座に適用し、ディスク容量が設定されていない場合でも、参照されていない古いレガシートランスクリプト、チェックポイント、軌跡アーティファクトをプルーニングします。

OpenClaw は Gateway の書き込み中に自動的な `sessions.json.bak.*` ローテーションバックアップを作成しなくなりました。現在のスキーマはレガシーな `session.maintenance.rotateBytes` キーを拒否し、`openclaw doctor --fix` は古い設定からこのキーを削除します。

トランスクリプトの変更では、SQLite トランスクリプトターゲット用のセッション書き込みキューを使用します：

セッション書き込みロックでは、本番環境向けの固定デフォルト値を使用します。対応する
`OPENCLAW_SESSION_WRITE_LOCK_*` 環境変数は、プロセスレベルの診断および
緊急時の上書き用として引き続き利用できます。

### SQLite への切り替え後のダウングレード

古いファイルベースの OpenClaw バージョンを実行する前に、
アーカイブされたレガシートランスクリプトアーティファクトを復元してください：

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

移行では、サポートおよびロールバック用としてレガシーな `sessions.json` ファイルを
そのまま残しますが、SQLite にインポートされた使用中のトランスクリプト JSONL ファイルは
`session-sqlite-import-archive/` に名前変更されます。古いファイルベースのランタイムは
`sessions.json` 内の `sessionFile` パスを使用するため、起動前にこれらの
アーティファクトを復元する必要があります。復元では移行マニフェストを使用し、元のパスが
存在しない、記録済みのアーカイブアーティファクトだけを移動します。また、再アップグレード時の
復旧に備えて SQLite データベースはそのまま残します。

SQLite への切り替え後に作成されたセッションは SQLite 専用であり、
古いファイルベースのランタイムには表示されません。ダウングレード後に再アップグレードする場合は、
Doctor の検査および検証手順を再度実行し、復元されたレガシーアーティファクトを
インポートする前に OpenClaw が検証できるようにしてください。

## Cron セッションと実行ログ

分離された cron 実行では、専用の保持設定を持つ独自のセッションエントリ/トランスクリプトが作成されます：

- `cron.sessionRetention`（デフォルト `"24h"`）は、分離された古い cron 実行セッションをストアからプルーニングします。`false` で無効化できます。
- 実行履歴では、cron ジョブごとに最新の 2000 件の終了行を保持します。失われた行には、24 時間のクリーンアップ期間が引き続き適用されます。

cron が新しい分離実行セッションを強制作成する場合、新しい行を書き込む前に、以前の `cron:<jobId>` セッションエントリをサニタイズします。安全な設定（thinking/fast/verbose/reasoning 設定、ラベル、表示名）と、ユーザーが明示的に選択したモデル/認証の上書きは引き継ぎますが、周辺的な会話コンテキスト（チャンネル/グループのルーティング、送信/キューポリシー、権限昇格、オリジン、ACP ランタイムバインディング）は破棄します。これにより、新しい分離実行が古い実行から陳腐化した配信権限やランタイム権限を継承することを防ぎます。

## セッションキー（`sessionKey`）

`sessionKey` は、現在どの会話バケットにいるか（ルーティング + 分離）を識別します。標準ルール：[/concepts/session](/ja-JP/concepts/session)。

| パターン                      | 例                                                     |
| ---------------------------- | ----------------------------------------------------------- |
| メイン/ダイレクトチャット（エージェントごと） | `agent:<agentId>:<mainKey>`（デフォルト `main`）                |
| グループ                        | `agent:<agentId>:<channel>:group:<id>`                      |
| ルーム/チャンネル（Discord/Slack） | `agent:<agentId>:<channel>:channel:<id>` または `...:room:<id>` |
| Cron                         | `cron:<job.id>`                                             |
| Webhook                      | `hook:<uuid>`（上書きされていない場合）                           |

## セッション ID（`sessionId`）

各 `sessionKey` は、現在の `sessionId`（会話を継続する SQLite トランスクリプト ID）を指します。判定ロジックは `src/auto-reply/reply/session.ts` 内の `initSessionState()` にあります。

- **リセット**（`/new`、`/reset`）は、その`sessionKey`用に新しい`sessionId`を作成します。
- デフォルトは**自動リセットなし**です。Compaction によってアクティブなモデルコンテキストを制限内に保ちながら、現在の`sessionId`が継続します。
- **日次リセット**（`session.reset.mode: "daily"`）は、設定されたローカル時刻の境界（`session.reset.atHour`、デフォルトは`4`）を越えた後の次のメッセージで、新しい`sessionId`を作成します。
- **アイドル期限切れ**（`session.reset.idleMinutes`を伴う`session.reset.mode: "idle"`、または従来の`session.idleMinutes`）は、アイドル期間後にメッセージが到着すると新しい`sessionId`を作成します。日次とアイドルの両方が設定されている場合は、先に期限切れになった方が優先されます。
- **Control UI 再接続の再開**では、Gateway がオペレーター UI クライアントから一致する`sessionId`を受信した場合、再接続後の1回の送信について現在表示中のセッションを維持します。これは一度限りのシグナルです。通常の古い送信では、引き続き新しい`sessionId`が作成されます。
- **システムイベント**（Heartbeat、Cron のウェイクアップ、exec 通知、Gateway の管理処理）はセッション行を変更する場合がありますが、日次／アイドルリセットの有効期間を延長することはありません。リセットによる切り替えでは、新しいプロンプトを構築する前に、前のセッション用にキューに入っているシステムイベント通知を破棄します。
- **親フォークポリシー**では、スレッドまたはサブエージェントのフォークを作成するときに OpenClaw のアクティブなブランチを使用します。そのブランチが大きすぎる場合（固定された内部上限を超える場合。現在は100Kトークン）、OpenClaw は失敗させたり使用不能な履歴を継承させたりせず、分離されたコンテキストで子を開始します。サイズ判定は自動であり、設定できません。従来の`session.parentForkMaxTokens`設定は`openclaw doctor --fix`によって削除されます。
- **オペレーターフォーク**：`sessions.create { parentSessionKey, fork: true }`は、親の現在の状態からトランスクリプトを分岐させた新しいセッションを作成します（上記のサイズ上限を含め、サブエージェント生成と同じフォーク機構）。親で実行がアクティブな間はフォークが拒否されます。明示的に指定されない限り親のモデル選択を継承し、新しいトークンカウンターを持つ子を`forkedFromParent`としてマークします。

## セッションストアのスキーマ

ランタイムストアは、エージェントごとの SQLite に`SessionEntry`値を保持します。値の型は`src/config/sessions.ts`内の`SessionEntry`です。主なフィールド（網羅的ではありません）：

- `sessionId`：SQLite のトランスクリプト行を指定するために使用される現在のトランスクリプト ID
- `sessionStartedAt`：現在の`sessionId`の開始タイムスタンプ。日次リセットの有効期間判定で使用されます。従来の行では、JSONL セッションヘッダーから導出される場合があります。
- `lastInteractionAt`：最後の実際のユーザー／チャネル操作のタイムスタンプ。Heartbeat、Cron、exec イベントによってセッションが維持されないよう、アイドルリセットの有効期間判定で使用されます。このフィールドがない従来の行では、復元されたセッション開始時刻にフォールバックします。
- `updatedAt`：ストア行が最後に変更されたタイムスタンプ。一覧表示、整理、管理処理に使用されますが、日次／アイドルの有効期間を決定する基準ではありません。
- `archivedAt`：任意のアーカイブタイムスタンプ。アーカイブされたセッションはトランスクリプトを維持したままストアに残り、通常のアクティブ一覧から除外されます。
- `pinnedAt`：任意のピン留めタイムスタンプ。アクティブなピン留め済みセッションは、ピン留めされていないセッションより前に並びます。セッションをアーカイブするとピン留めは解除されます。
- Codex スレッド相互運用：両方のフィールドは Codex のスレッド管理形式に従います。通信上の`archived`/`pinned`ブール値は常にタイムスタンプから導出され、Codex の`threads.archived_at`セマンティクスおよび camelCase シリアル化に合わせてサーバー側で設定されます。OpenClaw のタイムスタンプはエポックミリ秒、Codex はエポック秒を使用するため、ブリッジは`codex` Plugin 境界で変換します。Codex にはまだピン留め API がなく（`thread/archive`/`thread/unarchive`のみ）、API が提供されるまではピン留め状態が OpenClaw 側に保持されます。提供後は対応する形式により、バインドされたセッションのピン留め状態を機械的に往復できます。
- Codex の監視一覧には、アーカイブされていないネイティブスレッドのみが表示されます。Gateway ローカルの`idle`または`notLoaded`でアクティビティ不明のスレッドは、他の Codex プロセスが所有していないことをオペレーターが明示的に確認した後に限り、ネイティブの`thread/archive`を介してアーカイブできます。Plugin は最初にプロセスローカルのステータスを新たに読み取り、その後スレッドはカタログから消えます。この読み取りでは、別の App Server プロセスがそのスレッドを使用していないことまでは証明できません。OpenClaw はアクティブな行とエラー行のアーカイブを拒否します。また、Node ブリッジがストリーミングされるスレッドのライフサイクル全体を管理できるようになるまで、ペアリングされた Node のアーカイブは利用できません。ネイティブ Codex クライアントでアーカイブを解除すると、そのスレッドは再び表示対象になります。
- `lastReadAt` / `markedUnreadAt`：`sessions.patch { unread }`によってサーバー側で設定される既読状態のタイムスタンプ。`unread: false`は既読を記録し（`lastReadAt`を設定して`markedUnreadAt`をクリア）、`unread: true`は次に既読になるまでセッションを未読としてマークします。セッション行は導出された`unread`ブール値を公開します。明示的に未読とマークされているか、最新のアクティビティより前に既読になっている場合です。一度も既読とマークされていないセッションは`unread: false`のままなので、既存のインストール環境でアップグレード時に未読表示が一斉に点灯することはありません。
- `lastActivityAt`：未読に値するアクティビティとして扱われる、最後に完了したエージェント実行（ユーザー、チャネル、Cron 実行）のタイムスタンプ。Heartbeat と内部イベントのターン、およびメタデータのパッチでは更新されません。`updatedAt`はアクティビティシグナルではありません。
- `sessionFile`：移行／アーカイブ互換性のために保持される従来のマーカー。アクティブなランタイムは SQLite の識別子を使用します
- `chatType`：`direct | group | room`
- `provider`、`subject`、`room`、`space`、`displayName`：グループ／チャネルのラベル付けメタデータ
- 切り替え設定：`thinkingLevel`、`verboseLevel`、`reasoningLevel`、`elevatedLevel`、`sendPolicy`（セッションごとのオーバーライド）
- モデル選択：`providerOverride`、`modelOverride`、`authProfileOverride`
- トークンカウンター（ベストエフォート／プロバイダー依存）：`inputTokens`、`outputTokens`、`totalTokens`、`contextTokens`
- `compactionCount`：このセッションキーで自動 Compaction が完了した回数
- `memoryFlushAt` / `memoryFlushCompactionCount`：直近の Compaction 前メモリフラッシュのタイムスタンプと Compaction 回数

Gateway が正式な情報源です。セッションの実行に伴い、エントリを書き換えたり再構築したりする場合があります。従来のファイルベースのインストール環境では、
`sessions.json`を編集してランタイムがそのファイルを読み続けると期待するのではなく、
`openclaw doctor --session-sqlite import --session-sqlite-all-agents`を使用して移行してください。

## トランスクリプトイベントの構造

トランスクリプトは OpenClaw のセッションアクセサーによって管理され、識別子ベースのヘルパーを通じてランタイムコードに公開されます。イベントストリームは追記専用です：

- 最初のエントリ：セッションヘッダー — `type: "session"`、`id`、`cwd`、`timestamp`、任意の`parentSession`。
- 以降：`id` + `parentId`を持つエントリ（ツリー構造）。

主なエントリタイプ：

- `message`：ユーザー／アシスタント／toolResult メッセージ
- `custom_message`：拡張機能によって挿入され、モデルコンテキストに_入る_メッセージ（`display: true`の場合は TUI に表示され、`display: false`の場合は完全に非表示）
- `custom`：モデルコンテキストに_入らない_拡張機能の状態（再読み込みをまたいで拡張機能の状態を永続化するため）
- `compaction`：`firstKeptEntryId`と`tokensBefore`を持つ永続化された Compaction 要約
- `branch_summary`：ツリーブランチを移動するときに永続化される要約

OpenClaw は意図的にトランスクリプトを「修正」しません。Gateway は`SessionManager`を使用して読み書きします。

## コンテキストウィンドウと追跡トークン

異なる2つの概念があります：

1. **モデルコンテキストウィンドウ**：モデルごとの厳格な上限（モデルから見えるトークン）。モデルカタログから取得され、設定でオーバーライドできます。
2. **セッションストアのカウンター**：セッション行に書き込まれるローリング統計（`/status`およびダッシュボードで使用）。`contextTokens`はランタイムによる推定／レポート値であり、厳格な保証として扱わないでください。

上限の詳細：[/reference/token-use](/ja-JP/reference/token-use)。

## Compaction とは

Compaction は古い会話を要約して、トランスクリプト内の永続化された`compaction`エントリにし、最近のメッセージはそのまま維持します。Compaction 後、以降のターンでは Compaction 要約と`firstKeptEntryId`より後のメッセージが参照されます。セッションのプルーニングとは異なり、Compaction は**永続的**です。[/concepts/session-pruning](/ja-JP/concepts/session-pruning)を参照してください。

組み込み OpenClaw の Compaction は、デフォルトでセッションの思考レベルを継承します。要約呼び出しに別のレベルを使用するには`agents.defaults.compaction.thinkingLevel`を設定します。ランタイムは各具体的な Compaction モデルまたはフォールバックに合わせてその値を制限します。ネイティブ Codex App Server の Compaction は自身で compact リクエストを管理し、Compaction ごとの思考レベルのオーバーライドを受け付けられないため、OpenClaw は警告を表示し、その設定を Codex に委ねます。

Compaction 後の AGENTS.md セクションの再挿入は、引き続き`agents.defaults.compaction.postCompactionSections`によるオプトインです。Plugins は`before_prompt_build`を通じて他のプロンプトコンテキストを追加できます。

### チャンク境界とツールの対応付け

長いトランスクリプトを Compaction 用のチャンクに分割する際、OpenClaw はアシスタントのツール呼び出しと、それに対応する`toolResult`エントリの組み合わせを維持します：

- トークン比率による分割位置がツール呼び出しとその結果の間になる場合、OpenClaw は両者を分離せず、境界をアシスタントのツール呼び出しメッセージまで移動します。
- 末尾のツール結果ブロックによってチャンクが目標を超える場合、OpenClaw はその保留中のツールブロックを保持し、要約されていない末尾部分をそのまま維持します。
- 中止／エラーになったツール呼び出しブロックによって、保留中の分割が開いたままになることはありません。

## 自動 Compaction が発生するタイミング

組み込み OpenClaw エージェントには2つのトリガーがあります：

1. **オーバーフロー復旧**：モデルがコンテキストオーバーフローエラー（`request_too_large`、`context length exceeded`、`input exceeds the maximum number of tokens`、`input token count exceeds the maximum number of input tokens`、`input is too long for the model`、`ollama error: context length exceeded`、およびその他のプロバイダー固有の形式）を返した場合、Compaction を実行してから再試行します。プロバイダーが試行時のトークン数を報告した場合、OpenClaw はその実測値をオーバーフロー復旧用 Compaction に渡します。プロバイダーがオーバーフローを確認しても解析可能な数値を公開しない場合、OpenClaw は予算を最小限だけ超過する合成値を Compaction エンジンと診断機能に渡します。オーバーフロー復旧が引き続き失敗する場合、OpenClaw は明示的なガイダンスを表示し、新しいセッション ID に暗黙的に切り替えるのではなく、現在のセッションマッピングを維持します。メッセージを再試行するか、`/compact`または`/new`を実行してください。
2. **しきい値メンテナンス**：ターンが正常に完了した後、現在のコンテキストが、モデルウィンドウからプロンプトおよび次のモデル出力用に OpenClaw が組み込んでいる余裕を差し引いた値を超えた場合に実行されます。

これら2つのトリガーとは別に、追加で2つのガードが実行されます：

- **事前ローカル Compaction**: アクティブなトランスクリプトが指定サイズに達した後、次の実行を開始する前にローカル Compaction をトリガーするには、`agents.defaults.compaction.maxActiveTranscriptBytes`（バイト数、または `"20mb"` のような文字列）を設定します。これはローカルで再オープンするコストに対するサイズガードであり、生のアーカイブ処理ではありません。通常のセマンティック Compaction は引き続き実行され、圧縮された要約を新しい後継トランスクリプトにするには `truncateAfterCompaction` が必要です。
- **ターン途中の事前チェック**: ツールループガードを追加するには、`agents.defaults.compaction.midTurnPrecheck.enabled: true`（デフォルトは `false`）を設定します。ツール結果が追加された後、次のモデル呼び出しの前に、OpenClaw はターン開始時と同じ事前予算ロジックを使用してプロンプトの逼迫度を推定します。コンテキストが収まらなくなった場合、ガードはその場で Compaction を行いません。構造化されたターン途中の事前チェックシグナルを発生させ、現在のプロンプト送信を停止し、外側の実行ループに既存の復旧パスを使用させます（それで十分な場合は大きすぎるツール結果を切り詰め、そうでなければ設定された Compaction モードをトリガーして再試行します）。プロバイダーを利用するセーフガード Compaction を含め、`default` と `safeguard` の両方の Compaction モードで動作します。`maxActiveTranscriptBytes` とは独立しています。バイトサイズガードはターンの開始前に実行され、ターン途中の事前チェックはその後、新しいツール結果が追加された後に実行されます。

## Compaction 設定

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw は埋め込み実行用の組み込み予約領域を適用し、プロンプト予算全体を消費しないよう、アクティブなモデルのコンテキストウィンドウに基づいて上限を設定します。これにより、コンテキストが小さいローカルモデルが最初のトークンから Compaction に入ることを防ぎつつ、メモリーフラッシュなどの複数ターンにわたる保守処理に十分な余裕を確保します。

手動の `/compact` は明示的な `agents.defaults.compaction.keepRecentTokens` に従い、ランタイムの最新末尾の切り分け位置を維持します。維持予算を明示しない場合、手動 Compaction はハードチェックポイントとなり、再構築されたコンテキストは新しい要約から開始されます。

`truncateAfterCompaction` が有効な場合、OpenClaw は Compaction 後にアクティブなトランスクリプトを圧縮済みの後継トランスクリプトへローテーションします。分岐／復元のチェックポイント操作では、その圧縮済み後継トランスクリプトが使用されます。Compaction 前の従来のチェックポイントファイルは、参照されている間は引き続き読み取れます。

## 差し替え可能な Compaction プロバイダー

Plugin は Plugin API の `registerCompactionProvider()` を使用して Compaction プロバイダーを登録します。`agents.defaults.compaction.provider` に登録済みプロバイダー ID を設定すると、セーフガード拡張機能は組み込みの `summarizeInStages` パイプラインではなく、そのプロバイダーに要約処理を委任します。

- `provider`: 登録済み Compaction プロバイダー Plugin の ID。デフォルトの LLM 要約を使用する場合は未設定のままにします。`provider` を設定すると、`mode: "safeguard"` が強制されます。
- プロバイダーは組み込みパスと同じ Compaction 指示および識別子保持ポリシーを受け取り、セーフガードはプロバイダーの出力後も直近ターンおよび分割ターンの接尾コンテキストを保持します。
- 組み込みのセーフガード要約では、以前の要約全体をそのまま保持するのではなく、新しいメッセージと合わせて以前の要約を再抽出します。
- セーフガードモードでは、要約品質の監査がデフォルトで有効になります。不正な形式の出力に対する再試行動作を省略するには、`qualityGuard.enabled: false` を設定します。
- プロバイダーが失敗した場合、または空の結果を返した場合、OpenClaw は自動的に組み込みの LLM 要約へフォールバックします。呼び出し元が明示的に発生させた中断／タイムアウトシグナルは握りつぶされず、再スローされるため、キャンセルは常に尊重されます。

ソース: `src/plugins/compaction-provider.ts`、`src/agents/agent-hooks/compaction-safeguard.ts`。

## ユーザーに表示される箇所

- 任意のチャットセッション内の `/status`
- `openclaw status`（CLI）
- `openclaw sessions` / `openclaw sessions --json`
- Gateway ログ（`pnpm gateway:watch` または `openclaw logs --follow`）: `embedded run auto-compaction start` + `complete`
- 詳細モード: `🧹 Auto-compaction complete` と Compaction 回数

## サイレント保守処理（`NO_REPLY`）

OpenClaw は、ユーザーに途中の出力を表示すべきでないバックグラウンドタスク向けに「サイレント」ターンをサポートします。

- アシスタントは出力の先頭に正確なサイレントトークン `NO_REPLY` / `no_reply` を付け、「ユーザーに応答を配信しない」ことを示します。OpenClaw は配信レイヤーでこれを除去／抑制します。
- 正確なサイレントトークンの抑制では大文字と小文字を区別しません。ペイロード全体がサイレントトークンのみである場合、`NO_REPLY` と `no_reply` はどちらも該当します。
- `2026.1.10` 以降、OpenClaw は部分チャンクが `NO_REPLY` で始まる場合、下書き／入力中のストリーミングも抑制するため、サイレント操作の部分的な出力がターン途中で漏れることはありません。
- これは完全なバックグラウンド／非配信ターン専用であり、通常の対応可能なユーザーリクエストに対するショートカットではありません。

## Compaction 前のメモリーフラッシュ

自動 Compaction が行われる前に、OpenClaw は永続的な状態をディスク（たとえば、エージェントワークスペース内の `memory/YYYY-MM-DD.md`）へ書き込むサイレントなエージェントターンを実行できます。これにより、Compaction によって重要なコンテキストが消去されることを防ぎます。セッションのコンテキスト使用量を監視し、それが Compaction しきい値より低いソフトしきい値を超えると、正確なサイレントトークン `NO_REPLY` / `no_reply` を使用して「今すぐメモリーを書き込む」というサイレント指示を送信するため、ユーザーには何も表示されません。

設定（`agents.defaults.compaction.memoryFlush`）。完全なリファレンスは [/gateway/config-agents](/ja-JP/gateway/config-agents#agentsdefaultscompaction) を参照してください。

| キー                         | デフォルト          | 備考                                                                                                                                  |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | 未設定            | フラッシュターン専用の正確なプロバイダー／モデルのオーバーライド。例: `ollama/qwen3:8b`                                                   |
| `softThresholdTokens`       | `4000`           | フラッシュをトリガーする、Compaction しきい値からの差                                                                               |
| `forceFlushTranscriptBytes` | 未設定（無効） | トークンカウンターが古い場合でも、トランスクリプトファイルがこのバイトサイズ（または `"2mb"` のような文字列）に達した時点でフラッシュを強制します。`0` で無効化します |

注:

- 組み込みプロンプトとシステムプロンプトには、配信を抑制するための `NO_REPLY` ヒントが含まれます。
- `model` が設定されている場合、フラッシュターンはアクティブなセッションのフォールバックチェーンを継承せず、そのモデルを使用します。そのため、ローカル専用の保守処理が失敗時に有料の会話モデルへ暗黙的にフォールバックすることはありません。
- フラッシュは Compaction サイクルごとに 1 回実行されます（セッション行で追跡されます）。
- フラッシュは埋め込み OpenClaw セッションでのみ実行されます。CLI バックエンドと Heartbeat ターンではスキップされます。
- セッションワークスペースが読み取り専用（`workspaceAccess: "ro"` または `"none"`）の場合、フラッシュはスキップされます。
- ワークスペースのファイルレイアウトと書き込みパターンについては、[メモリー](/ja-JP/concepts/memory)を参照してください。

OpenClaw は拡張 API で `session_before_compact` フックを公開していますが、前述のフラッシュロジックはそのフック上ではなく、Gateway 側（`src/auto-reply/reply/memory-flush.ts`、`src/auto-reply/reply/agent-runner-memory.ts`）にあります。

## トラブルシューティングのチェックリスト

- **セッションキーが間違っている場合**: [/concepts/session](/ja-JP/concepts/session)から始め、`/status` 内の `sessionKey` を確認します。
- **ストアとトランスクリプトが一致しない場合**: `openclaw status` で Gateway ホストとストアパスを確認します。
- **Compaction が頻発する場合**: モデルのコンテキストウィンドウ（小さすぎると頻繁な Compaction が発生します）とツール結果の肥大化（セッション枝刈りを調整します）を確認します。
- **小さいローカルモデルですべてのプロンプトがオーバーフローするように見える場合**: プロバイダーが正しいモデルのコンテキストウィンドウを報告していることを確認します。OpenClaw が有効な予約領域に上限を設定できるのは、そのウィンドウが既知の場合のみです。
- **サイレントターンが漏れる場合**: 応答が正確なサイレントトークン `NO_REPLY`（大文字と小文字を区別しません）で始まること、およびストリーミング抑制の修正を含むビルド（`2026.1.10` 以降）を使用していることを確認します。

## 関連項目

- [セッション管理](/ja-JP/concepts/session)
- [セッション枝刈り](/ja-JP/concepts/session-pruning)
- [コンテキストエンジン](/ja-JP/concepts/context-engine)
- [エージェント設定リファレンス](/ja-JP/gateway/config-agents)
