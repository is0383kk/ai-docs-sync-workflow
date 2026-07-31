---
read_when:
    - プラグインからコアヘルパー（TTS、STT、画像生成、ウェブ検索、Gateway、サブエージェント、ノード）を呼び出す必要がある場合
    - api.runtime が公開する内容を理解したい場合
    - Plugin コードから設定、エージェント、またはメディアのヘルパーにアクセスしている場合
sidebarTitle: Runtime helpers
summary: api.runtime -- Plugin で利用可能な注入済みランタイムヘルパー
title: Plugin ランタイムヘルパー
x-i18n:
    generated_at: "2026-07-26T09:45:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

登録時にすべての Plugin に注入される `api.runtime` オブジェクトのリファレンスです。ホスト内部を直接インポートする代わりに、これらのヘルパーを使用してください。

<CardGroup cols={2}>
  <Card title="チャンネル Plugin" href="/ja-JP/plugins/sdk-channel-plugins">
    チャンネル Plugin でこれらのヘルパーを実際のコンテキストに沿って使用するためのステップ形式のガイドです。
  </Card>
  <Card title="プロバイダー Plugin" href="/ja-JP/plugins/sdk-provider-plugins">
    プロバイダー Plugin でこれらのヘルパーを実際のコンテキストに沿って使用するためのステップ形式のガイドです。
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` は現在の OpenClaw 製品バージョンです。共有バージョンリゾルバーから取得されるため、Plugin には CLI が報告する値と同じ値が渡されます。

## 設定の読み込みと書き込み

登録時の `api.config` や、チャンネル／プロバイダーのコールバックに渡される `cfg` 引数など、アクティブな呼び出しパスにすでに渡されている設定を優先してください。これにより、ホットパスで設定を再解析するのではなく、単一のプロセススナップショットを処理全体で引き回せます。

`api.runtime.config.current()` は、長期間存続するハンドラーが現在のプロセススナップショットを必要とし、その関数に設定が渡されていない場合にのみ使用してください。返される値は読み取り専用です。編集する前に複製するか、変更ヘルパーを使用してください。

ツールファクトリーは `ctx.runtimeConfig` と `ctx.getRuntimeConfig()` を受け取ります。ツール定義の作成後に設定が変更される可能性がある場合は、長期間存続するツールの `execute` コールバック内でゲッターを使用してください。

変更は `api.runtime.config.mutateConfigFile(...)` または `api.runtime.config.replaceConfigFile(...)` で永続化します。各書き込みでは、明示的な `afterWrite` ポリシーを選択する必要があります。

- `afterWrite: { mode: "auto" }` は、Gateway のリロードプランナーに判断を委ねます。
- `afterWrite: { mode: "restart", reason: "..." }` は、ホットリロードが安全でないと書き込み側が判断している場合に、クリーンな再起動を強制します。
- `afterWrite: { mode: "none", reason: "..." }` は、呼び出し元が後続処理を担う場合にのみ、自動リロード／再起動を抑制します。

変更ヘルパーは `afterWrite` と、型付けされた `followUp` の概要を返すため、呼び出し元は再起動を要求したかどうかをログに記録したりテストしたりできます。その再起動を実際にいつ行うかは、引き続き Gateway が管理します。

実行時の設定アクセスと書き込みには、`current()`、渡された `cfg`、`mutateConfigFile(...)`、または
`replaceConfigFile(...)` を使用してください。

SDK から直接インポートする場合は、広範な `openclaw/plugin-sdk/config-runtime` 互換バレルよりも、用途別の設定サブパスを優先してください。型には `config-contracts`、現在のプロセススナップショットには `runtime-config-snapshot`、書き込みには `config-mutation` を使用します。エントリースコープの値は `api.pluginConfig` から読み取ってください。提供されたツールコンテキストは、実行環境全体の設定スナップショットにのみ使用し、Plugin 固有のマージはその境界で行ってください。バンドルされた Plugin のテストでは、広範な互換バレルではなく、これらの用途別サブパスを直接モックしてください。

OpenClaw の内部ランタイムコードも同じ方針に従います。CLI、Gateway、またはプロセス境界で設定を一度読み込み、その値を引き回します。変更の書き込みに成功すると、プロセスのランタイムスナップショットが更新され、内部リビジョンが進みます。長期間存続するキャッシュでは、設定をローカルでシリアライズするのではなく、ランタイムが所有するキャッシュキーを使用してください。長期間存続するランタイムモジュールには、暗黙的な `loadConfig()` 呼び出しを一切許容しないスキャナーがあります。渡された `cfg`、リクエストの `context.getRuntimeConfig()`、または明示的なプロセス境界の `getRuntimeConfig()` を使用してください。

プロバイダーとチャンネルの実行パスでは、設定の読み戻しや編集用に返されたファイルスナップショットではなく、アクティブなランタイム設定スナップショットを使用する必要があります。ファイルスナップショットは、UI と書き込みのために SecretRef マーカーなどのソース値を保持しますが、プロバイダーのコールバックには解決済みのランタイムビューが必要です。ヘルパーがアクティブなソーススナップショットまたはアクティブなランタイムスナップショットのどちらでも呼び出される可能性がある場合は、認証情報を読み取る前に `selectApplicableRuntimeConfig()` を経由してください。

## 再利用可能なランタイムユーティリティ

ボットが生成した受信メッセージには、受信した `botLoopProtection` の情報を使用してください。コアは、ポリシーを特定のチャンネルに結び付けることなく、セッションの記録とディスパッチの前に共有のインメモリ・スライディングウィンドウガードを適用します。このガードは `(scopeId, conversationId, participant pair)` キーを追跡し、ペアの両方向をまとめてカウントし、ウィンドウの上限を超えるとクールダウンを適用し、非アクティブなエントリを適宜削除します。

この動作を運用者に公開するチャンネル Plugin では、ベースラインの上限に共有の `channels.defaults.botLoopProtection` 形式を優先し、その上にチャンネル／プロバイダー固有のオーバーライドを重ねてください。共有設定では、ユーザー向けであるため秒単位を使用します。

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

正規化したボットペアの情報を、解決済みのターンとともに渡してください。コアがデフォルト、単位変換、および `enabled` のセマンティクスを解決します。

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

共有の受信返信ランナーを経由しない独自の
二者間イベントループでのみ、`openclaw/plugin-sdk/pair-loop-guard-runtime` を直接使用してください。

## ランタイム名前空間

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    エージェントのアイデンティティ、ディレクトリ、およびセッション管理です。

    ```typescript
    // エージェントの作業ディレクトリを解決する（agentId は必須）
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // エージェントのワークスペースを解決する
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // エージェントのアイデンティティを取得する
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // デフォルトの思考レベルを取得する
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // ユーザーが指定した思考レベルをアクティブなプロバイダープロファイルに照らして検証する
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // 埋め込み実行にレベルを渡す
    }

    // エージェントのタイムアウトを取得する
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // ワークスペースが存在することを確認する
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // 埋め込みエージェントのターンを実行する
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "最新の変更を要約してください",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` は、Plugin コードから通常の OpenClaw エージェントターンを開始するための中立的なヘルパーです。チャンネルからトリガーされる返信と同じプロバイダー／モデル解決およびエージェントハーネス選択を使用します。

    `runEmbeddedPiAgent(...)` は、既存の Plugin 向けの非推奨の互換エイリアスとして残されています。新しいコードでは `runEmbeddedAgent(...)` を使用してください。

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` は、埋め込みランナーによる CLI バックエンドへのディスパッチ判断（ルート、バックエンドが宣言する `subscriptionAuthDispatch` 機能、保存済みの認証情報モード。明示的に固定された `authProfileId` を尊重）を、埋め込み実行で `cliBackendDispatch: "subscription-auth"` を選択する呼び出し元と共有します。実行が CLI バックエンドを通じて行われる場合は `{ provider }`、直接パススルーに留まる場合は `undefined` を返すため、呼び出し元は実際に行われる実行に合わせてタイムアウトを見積もれます。

    `resolveThinkingPolicy(...)` は、プロバイダー／モデルがサポートする思考レベルと、任意のデフォルト値を返します。プロバイダー Plugin は思考フックを通じてモデル固有のプロファイルを所有するため、ツール Plugin ではプロバイダーのリストをインポートまたは複製するのではなく、このランタイムヘルパーを呼び出してください。

    `normalizeThinkingLevel(...)` は、`on`、`x-high`、`extra high` などのユーザーテキストを、解決済みのポリシーと照合する前に正規の保存レベルへ変換します。

    **セッションストアヘルパー**は `api.runtime.agent.session` 配下にあります。

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // 従来の sessions.json 形式に依存せず、セッション行を反復処理する。
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // セッションを作成または更新してから、許可されたエージェント実行に signal を渡す。
      },
    );
    ```

    セッションワークフローには、`getSessionEntry(...)`、`listSessionEntries(...)`、`patchSessionEntry(...)`、または `upsertSessionEntry(...)` を優先してください。これらのヘルパーはエージェント／セッションのアイデンティティによってセッションを指定するため、Plugin は従来の `sessions.json` ストレージ形式に依存しません。セッションのアクティビティを更新しないメタデータのみのパッチには `preserveActivity: true` を使用し、コールバックが完全なエントリを返し、削除されたフィールドを削除されたままにする必要がある場合にのみ `replaceEntry: true` を使用してください。Doctor と移行のパスでは、`fallbackEntry`、`skipMaintenance`、`requireWriteSuccess` を組み合わせて、正規ストアを1回のアトミック操作で修復できます。

    `createSessionEntry(...)` は、新しい正規セッション行とトランスクリプトを作成します。信頼済みの `initialEntry` サーフェスは意図的に限定されており、空でない `agentHarnessId`、任意の `modelSelectionLocked: true`、任意の `pluginExtensions` のみを受け付けます。注入されたランタイムは、`registerAgentHarness(...)` を通じて呼び出し元の Plugin が所有するハーネス ID のみを受け付けます。これは所有権の不変条件であり、同一プロセス内の Plugin 間を隔離するサンドボックスではありません。既存の行がある場合は拒否します。`label` と `spawnedCwd` は、信頼済みエントリへのパッチではなく、個別の作成フィールドです。

    作成処理では `afterCreate` を通じてセッションライフサイクルの変更フェンスを保持するため、新しい作業は Plugin が所有する初期化の完了を待ち、既存の許可済み作業がある場合は作成に失敗します。コールバックは作成済み状態の複製を受け取ります。コールバックがパッチを返す場合、そのパッチには `pluginExtensions` のみを含めることができ、その値が完全な最終 `pluginExtensions` フィールドになります。コールバックまたは最終永続化が失敗すると、変更されていない新しい行とトランスクリプトがロールバックされます。ガード付きロールバックは、同時に変更または取得された行を保持します。`recoverMatchingInitialEntry: true` は、永続化された信頼済みフィールドが完全に一致する場合に、中断された初期化を再試行するためだけに使用し、復旧には `afterCreate` が最終パッチを返す必要があります。

    Plugin が永続化済みセッションで作業を開始する場合は、`runWithWorkAdmission(...)` を使用してください。このコールバックは、アーカイブ済みまたは同時に置換されたセッションを拒否し、完了までアーカイブ／リセット／削除の変更を調整し、エージェント実行に渡す必要がある `AbortSignal` を受け取ります。ハーネスは、試験的な `delegatedExecutionPluginIds` 登録フィールドを通じて、信頼済みの実行デリゲートを明示的に指定できます。デリゲートが許可および実行できるのは、完全に一致する既存のモデル固定セッションのみです。すべてのセッション変更は引き続きハーネス所有者に制限されます。[エージェントハーネス Plugin](/ja-JP/plugins/sdk-agent-harness#delegated-execution)を参照してください。

    メンテナンスおよび修復用 Plugin は、スコープが限定された単一のセッションエントリには `deleteSessionEntry(...)`、ライフサイクルによって所有される一時セッションには `cleanupSessionLifecycleArtifacts(...)`、ストアを変更する前には `resolveSessionStoreBackupPaths(...)` を使用できます。削除が同時実行中のセッション更新と競合してはならない場合は `expectedSessionId` と `expectedUpdatedAt` を渡し、以前のスナップショットにセッション ID がなかった場合は `expectedSessionId: null` を使用します。これらのヘルパーは限定的な修復／ライフサイクル用のインターフェースであり、汎用的なストア削除 API ではありません。

    `resolveStorePath(...)` と `updateSessionStoreEntry(...)` は、セッションヘルパーを補完します。`resolveStorePath` は指定されたスコープのセッションストアパスを解決し、`updateSessionStoreEntry({ storePath, sessionKey, update })` は呼び出し元がそのパスをすでに把握している場合に、ストアパスを指定して 1 つのエントリを直接更新します。

    `loadTranscriptEventsSync(...)` は、非同期トランスクリプトランタイムを使用できない同期的な doctor および修復パスで利用できます。これは未加工の `SessionStoreTranscriptEvent` レコードを返します。通常の Plugin ランタイムコードでは `openclaw/plugin-sdk/session-transcript-runtime` を使用してください。

    `formatSqliteSessionFileMarker(...)`、`parseSqliteSessionFileMarker(...)`、`sqliteSessionFileMarkerMatchesSession(...)` は、`sessionFile` という名前のレガシーフィールドをまだ受け取るコード向けの移行用ヘルパーです。解析された SQLite マーカーは稼働中の SQLite トランスクリプトターゲットを識別するものであり、ファイルシステムパスではありません。新しい API では、マーカー文字列ではなく型付きのセッション ID を受け渡してください。

    トランスクリプトの読み書きには、`openclaw/plugin-sdk/session-transcript-runtime` をインポートし、`{ agentId, sessionKey, sessionId }` とともに `resolveSessionTranscriptIdentity(...)`、`resolveSessionTranscriptTarget(...)`、`readSessionTranscriptEvents(...)`、`readSessionTranscriptRawDelta(...)`、`readSessionTranscriptVisibleMessageDelta(...)`、`readVisibleSessionTranscriptMessageEntries(...)`、`appendSessionTranscriptMessageByIdentity(...)`、`publishSessionTranscriptUpdateByIdentity(...)`、または `withSessionTranscriptWriteLock(...)` を使用します。これらの API により、Plugin はアクティブなトランスクリプトファイルのパスに依存せずに、トランスクリプトの識別、未加工イベントまたはブランチセーフな可視メッセージエントリの読み取り、メッセージの追加、更新の公開、および同じトランスクリプト書き込みロック下での関連操作を実行できます。`readVisibleSessionTranscriptMessageEntries(...)` は順序付きの読み取りメタデータを返します。その `seq` フィールドは再開可能なカーソルではありません。

    `appendSessionTranscriptMessageByIdentity(...)` は、すでに正規化済みのメッセージを追加する低レベル操作です。Plugin は、トップレベルの `MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType`、または `MediaTypes` を含む、メディアを伴うユーザー行を合成してはなりません。チャネルの受信処理では、順序付きファクトを `MsgContext.media` 経由で渡し、ユーザーターンの永続化はホストに委ねてください。ホストが永続化用に準備したユーザーメッセージでは、正規化済みの順序付きファクトが `message.__openclaw.media` に格納されます。汎用追加 API は、レガシーな並列配列を推測または修復しません。

    `readSessionTranscriptRawDelta(...)` は、制限付きの `page`、`reset`、または `missing` の結果を返します。不透明な `page.cursor` を次の呼び出しに渡してください。単純な追加ではカーソルが維持されますが、トランスクリプトの置換では新しいブートストラップカーソルを伴う `reset` が返されます。ページのデフォルトは 1,000 イベントおよびシリアライズ後の 1,000,000 バイトで、呼び出し元は最大 10,000 イベントおよび 64 MiB まで要求できます。次のイベントだけで `maxBytes` を超える場合、ページは空になり、`requiredBytes` が報告されます。その値が 64 MiB 以下なら、少なくともそのバイト上限を指定して再試行してください。それより大きい個別イベントには完全読み取り API が必要です。カーソルは位置のみを識別し、別のセッションへのアクセスを許可することはありません。

    `readSessionTranscriptVisibleMessageDelta(...)` は、ホストが所有するアクティブメッセージプロジェクションに対して、同じ制限付きのブートストラップおよび再開形式を提供します。メッセージは古いものから新しいものの順で返されるため、コンテキストエンジンは初期履歴を順に処理し、不透明なカーソルをウォーターマークとして永続化できます。カーソルは変更せずに保存して返してください。これは継続のヒントであり、認可資格情報ではありません。線形な追加では、最後に返されたメッセージの後から再開します。トランスクリプトの置換、アンカーがアクティブブランチから外れたかブランチ内で移動したカーソル、不正なカーソル、およびセッションをまたぐカーソルの場合は、新しいブートストラップカーソルを伴う `reset` が返されます。件数とバイト数のデフォルトおよび上限は、未加工差分 API と同じです。ブランチ変更後にアクティブプロジェクションを再構築している間は、理由 `projection_rebuilding` を伴う `unavailable` が結果になります。アクティブなトランスクリプトファイルにフォールバックせず、後で再試行してください。

    レガシーなストア全体およびアクティブトランスクリプトファイルのヘルパーは、Plugin SDK からエクスポートされなくなりました。セッションメタデータにはスコープ付きエントリヘルパーを使用し、アクティブなトランスクリプト操作にはトランスクリプト ID ヘルパーを使用してください。ファイルアーティファクトを必要とするアーカイブ／サポートワークフローでは、アクティブセッションランタイム API ではなく、専用のアーカイブインターフェースを使用してください。

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    デフォルトのモデルおよびプロバイダー定数：

    ```typescript
    const model = api.runtime.agent.defaults.model; // 例："gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // 例："openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    プロバイダー内部をインポートしたり、OpenClaw のモデル／認証／ベース URL の準備を
    重複実装したりせずに、ホスト所有のテキスト補完を実行します。

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "このトランスクリプトを要約してください。" }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    プロバイダーのオーケストレーションでは、HTTP リクエストを発行する前に、設定済みローカルサービスの
    ライフサイクルを取得することもできます：

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // プロバイダーリクエストを送信し、完全に読み取ります。
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` は、安定した汎用プロバイダーサービス SDK
    コントラクトです。ホストは `models.providers.<providerId>.localService` から
    プロセス設定を解決します。呼び出し元は
    コマンド、引数、環境、またはライフサイクルポリシーを指定できません。プロセスの起動、
    準備完了判定、診断、およびアイドル停止ポリシーはホスト内部に保持されます。

    設定された正確なプロバイダー ID と、解決済みのリクエストベース URL を渡してください。
    エイリアスをアダプター ID に置き換えないでください。別々のエイリアスが別々の
    ローカル GPU ホストを指す場合があります。Ollama および LM
    Studio アダプターで使用される `/v1` の正規化を除き、設定済みの
    プロバイダーベース URL と一致しないエンドポイントはホストによって拒否されます。ホストは起動の直列化、準備完了プローブ、
    リクエストリース、中止処理、およびアイドルシャットダウンを所有します。

    このヘルパーは、OpenClaw の組み込みランタイムと同じ単純補完の準備パス、および
    ホスト所有のランタイム設定スナップショットを使用します。コンテキストエンジンは
    セッションにバインドされた `llm.complete` 機能を受け取るため、モデル呼び出しでは
    アクティブセッションのエージェントが使用され、デフォルトエージェントへ暗黙にフォールバックすることはありません。
    結果には、プロバイダー／モデル／エージェントの帰属情報に加え、利用可能な場合は正規化されたトークン、
    キャッシュ、および推定コストの使用量が含まれます。

    選択したモデルに推論強度を要求するには `reasoning` を設定します。
    ホストは補完をディスパッチする前に、正規の思考レベル（`off`、`minimal`、`low`、
    `medium`、`high`、`xhigh`、`adaptive`、`max`、`ultra`）を、選択された
    プロバイダーおよびモデル向けに正規化します。`adaptive` は
    `medium` になります。`max` と `ultra` は、対応している場合は `max` に、対応していない場合は `xhigh` になります。

    <Warning>
    モデルのオーバーライドには、設定内の `plugins.entries.<id>.llm.allowModelOverride: true` によるオペレーターのオプトインが必要です。信頼された Plugin を特定の正規 `provider/model` ターゲットに制限するには `plugins.entries.<id>.llm.allowedModels` を使用します。エージェントをまたぐ補完には `plugins.entries.<id>.llm.allowAgentIdOverride: true` が必要です。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    現在の Plugin の信頼されたランタイム ID を維持しながら、プロセス内で別の Gateway メソッドを呼び出します。
    これは、ループバック WebSocket 接続を開くことなく Plugin 所有の
    Gateway 機能を組み合わせる、バンドル済みまたは信頼された公式 Plugin を対象としています。

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    リクエストは `operator.write` スコープを使用し、管理者スコープを付与しません。任意の外部
    Plugin からの呼び出しは拒否されます。メソッドが失敗すると `GatewayClientRequestError` がスローされ、復旧フロー向けに構造化された
    `details`、再試行メタデータ、および Gateway エラーコードが維持されます。スタンドアロンのエージェントプロセスでも実行できるツールからこのパスを選択する前に、`isAvailable()`
    を使用してください。

  </Accordion>
  <Accordion title="api.runtime.subagent">
    バックグラウンドのサブエージェント実行を起動および管理します。

    ```typescript
    // サブエージェント実行を開始
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "このクエリを、焦点を絞った追加検索に展開してください。",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // オプションのオーバーライド
      model: "gpt-5.6-sol", // オプションのオーバーライド
      deliver: false,
    });

    // 完了を待機
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // セッションメッセージを読み取り
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // セッションを削除
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    モデルのオーバーライド（`provider`/`model`）には、設定内の `plugins.entries.<id>.subagent.allowModelOverride: true` によるオペレーターのオプトインが必要です。信頼されていない Plugin もサブエージェントを実行できますが、オーバーライド要求は拒否されます。
    </Warning>

    `toolsAlsoAllow` は、呼び出し元の Plugin によって登録され、一意に所有されている正確なツールを、ワーカーの通常のツールインターフェースに追加します。ランタイムは、コアツールおよび別の Plugin と共有されている名前を拒否します。明示的な許可リストおよび拒否を含め、プロファイルとオペレーターのツールポリシーは引き続き適用されます。

    `deleteSession(...)` は、同じ Plugin が `api.runtime.subagent.run(...)` を通じて作成したセッションを削除できます。任意のユーザーまたはオペレーターのセッションを削除するには、引き続き管理者スコープの Gateway リクエストが必要です。

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    エージェントセッションに対する実効的なサンドボックスワークスペース権限を調査します。

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    結果は、このセッションがサンドボックス化されているか、そのワークスペースが
    使用不可、読み取り専用、または書き込み可能のいずれであるかを報告します。また、実効的な Docker、ツール、セッション、ブラウザー、または昇格ポリシーによって
    そのワークスペースから脱出できる場合は、オプションの `confinementError` も報告します。これは、ワーカーに呼び出し元を超える権限を
    付与してはならない、ホスト所有の委任判断に使用します。これは証明用ヘルパーであり、
    呼び出し元自身の認可確認に代わるものではありません。

    `prepareWorkspaceAuthority(...)` は同じポリシーチェックを実行し、
    `workspaceDir` の Docker サンドボックスも準備します。実行中のコンテナの
    ライブ設定ハッシュが、要求されたマウントまたはポリシーと一致しない場合は拒否します。呼び出し元の Plugin が
    登録済みの実装を制限している、正確なツール名のみを渡してください。
    ワイルドカードプレフィックスではツールの所有権を証明できません。

  </Accordion>
  <Accordion title="api.runtime.nodes">
    接続済み Node を一覧表示し、Gateway によって読み込まれた Plugin コードまたは Plugin CLI コマンドから Node ホストコマンドを呼び出します。別の Mac 上のブラウザーやオーディオブリッジなど、ペアリング済みデバイスでのローカル作業を Plugin が所有する場合に使用します。

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` には、接続中の各 Node がエージェントに Plugin または MCP バックエンドのツールを公開している場合、その Node が通知した
    `nodePluginTools` 記述子が含まれます。これらの記述子はライブ接続状態です。Node が切断されると Gateway はそれらを破棄し、ローカルの Plugin/MCP インベントリが変更された後、Node はそれらを
    `node.pluginTools.update` に置き換えることができます。

    Gateway 内では、このランタイムはプロセス内で動作します。Plugin の CLI コマンドでは、設定済みの Gateway を RPC 経由で呼び出すため、`openclaw googlemeet recover-tab` などのコマンドでターミナルからペアリング済みの Node を検査できます。Node コマンドには、引き続き通常の Gateway の Node ペアリング、コマンド許可リスト、Plugin の Node 呼び出しポリシー、および Node ローカルのコマンド処理が適用されます。

    Node でホストされるエージェントツールを公開する Plugin は、デフォルトで許可リストに登録すべき危険性のないコマンドに `agentTool.defaultPlatforms` を設定できます。オペレーターが `gateway.nodes.commands.allow` で明示的に許可する必要がある場合は省略してください。危険な Node ホストコマンドでは、`api.registerNodeInvokePolicy(...)` を使用して Node 呼び出しポリシーを登録する必要があります。このポリシーは、コマンド許可リストのチェック後、コマンドが Node に転送される前に Gateway 内で実行されるため、`node.invoke` の直接呼び出し、Node でホストされる Plugin ツール、および上位レベルの Plugin ツールには同じ適用経路が使用されます。

    <Warning>
    オプションの `scopes` フィールドは、呼び出しに対する Gateway オペレータースコープを要求します。OpenClaw がこれを受け入れるのは、バンドルされた Plugin と、信頼済みの公式 Plugin インストールの場合のみです。その他の Plugin からの要求によって呼び出しの権限が昇格することはありません。信頼済みの Plugin が、`operator.admin` など、より厳格な Gateway スコープを持つ Node コマンドを呼び出す必要がある場合にのみ使用してください。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Task Flow と Task Run の状態を、既存の OpenClaw セッションキーまたは信頼済みツールコンテキストにバインドします。

    - `api.runtime.tasks.managedFlows` では変更操作が可能です。Task Flow の作成、進行、キャンセルを行えます。
    - `api.runtime.tasks.flows` と `api.runtime.tasks.runs` は、一覧表示とステータス検索用の読み取り専用 DTO ビューです。どちらも `bindSession(...)` / `fromToolContext(...)` に加えて、`get`、`list`、`findLatest`、`resolve` を公開します。

    Task Flow は、永続的な複数ステップのワークフロー状態を追跡します。スケジューラーではありません。
    将来のウェイクアップには Cron または `api.session.workflow.scheduleSessionTurn(...)` を使用し、
    その処理でフロー状態、子タスク、待機、またはキャンセルが必要な場合は、スケジュールされたターンから `managedFlows` を使用してください。

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Review new pull requests",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "Review PR #123",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    独自のバインディングレイヤーから信頼済みの OpenClaw セッションキーをすでに取得している場合は、`bindSession({ sessionKey, requesterOrigin })` を使用してください。生のユーザー入力からバインドしないでください。

  </Accordion>
  <Accordion title="api.runtime.tts">
    テキスト読み上げ合成。

    ```typescript
    // 標準 TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "OpenClaw からこんにちは",
      cfg: api.config,
    });

    // 電話向けに最適化された TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "OpenClaw からこんにちは",
      cfg: api.config,
    });

    // 利用可能な音声を一覧表示
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    コアの `tts` 設定とプロバイダー選択を使用します。PCM オーディオバッファとサンプルレートを返します。ストリーミング合成には `textToSpeechStream` も利用できます。

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    画像、音声、動画の分析。

    ```typescript
    // 画像を説明
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // 音声を文字起こし
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // MIME を推定できない場合に指定（任意）
    });

    // 動画を説明
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // 汎用ファイル分析
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // 特定のプロバイダー／モデルを介した構造化画像抽出。
    // 画像を少なくとも 1 つ含めます。テキスト入力は補足コンテキストです。
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "手書きのメモより、印刷された合計金額を優先してください。" },
      ],
      instructions: "販売者、合計金額、検索可能なタグを抽出してください。",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    出力が生成されなかった場合（入力がスキップされた場合など）は、`{ text: undefined }` を返します。

    `describeImageFileWithModel(...)` は、`describeImageFile(...)` が使用するデフォルトのアクティブモデル解決を迂回し、既知の画像を特定のプロバイダー／モデルで説明します。

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    画像生成。

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "夕焼けを描くロボット",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    画像生成と同じ形式の動画生成。

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "日の出の海岸線上空を飛行するドローン映像",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    画像生成と同じ形式の音楽生成。

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "コーディングセッション向けのアップテンポなローファイトラック",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Web 検索。

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw Plugin SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    低レベルのメディアユーティリティ。

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "画像"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    現在のランタイム設定スナップショットと、トランザクションによる設定書き込み。アクティブな呼び出し経路にすでに渡されている設定を優先し、ハンドラーがプロセスのスナップショットを直接必要とする場合にのみ
    `current()` を使用してください。

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` と `replaceConfigFile(...)` は、`{ mode: "restart", requiresRestart: true, reason }` などの `followUp`
    値を返します。これは Gateway から再起動制御を奪うことなく、書き込み側の意図を記録します。

  </Accordion>
  <Accordion title="api.runtime.system">
    システムレベルのユーティリティ。

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // 非推奨の互換性エイリアス。
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` は、通常の統合タイマーを迂回して、Heartbeat サイクルを 1 回即座に実行します。デフォルトの `target: "none"` 抑制を使用せず、最後にアクティブだったチャンネルへの配信を強制するには、`{ heartbeat: { target: "last" } }` を渡します。

    `runCommandWithTimeout(...)` は、キャプチャされた `stdout` と `stderr`、任意の
    切り詰め件数、`code`、`signal`、`killed`、`termination`、および
    `noOutputTimedOut` を返します。タイムアウトおよび無出力タイムアウトの結果では、子プロセスがゼロ以外の終了コードを返さない場合、`code: 124`
    が報告されます。タイムアウト以外のシグナル終了でも `code: null` が返されることがあるため、タイムアウトの理由を区別するには `termination` と
    `noOutputTimedOut` を使用してください。

  </Accordion>
  <Accordion title="api.runtime.events">
    イベント購読。

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    ロギング。

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    モデルとプロバイダーの認証解決。

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // プロバイダーのランタイム交換（OAuth 更新など）を含む、リクエストに使用可能な認証
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    状態ディレクトリの解決と、SQLite を基盤とするキー付きストレージ。

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    キー付きストアは再起動後も存続し、ランタイムにバインドされたプラグイン ID ごとに分離されます。アトミックな重複排除の取得には `registerIfAbsent(...)` を使用します。キーが存在しないか期限切れであり、登録された場合は `true` を返します。既存の有効な値が存在する場合は、その値、作成時刻、TTL を上書きせずに `false` を返します。クリーンアップで以前に確認した値のみを削除する必要がある場合は、`deleteIf(...)` を使用します。その同期述語と削除は、1 つの SQLite トランザクション内で実行されます。制限は、名前空間ごとに `maxEntries`、プラグインごとに 50,000 件の有効な行、64KB 未満の JSON 値、および任意の TTL 有効期限です。デフォルトでは、いずれかの行制限に達した状態で書き込むと、書き込み対象の名前空間から最も古い有効な行が破棄されます。その書き込みによって兄弟名前空間が削除されることはなく、名前空間で十分な行を解放できなければ、書き込みは引き続き失敗します。決して削除してはならない永続的な所有権レコードには `overflowPolicy: "reject-new"` を設定します。いずれかの制限に達すると新しいキーは失敗しますが、既存のキーは引き続き更新できます。

    `openSyncKeyedStore<T>(...)` は、待機できない呼び出し元向けに、同期メソッドを備えた同じストア形式を返します（`register`、`registerIfAbsent`、`deleteIf`、`lookup`、`consume`、`clear` はすべて、Promise ではなく値を直接返します）。

    `openBlobStore<TMetadata>(...)` は、base64 やファイルのサイドカーを使用せず、サイズ制限付きのバイナリペイロードを共有 SQLite に保存します。エントリごとのバイト数、名前空間ごとのバイト数、および行数の制限が必要です。また、API 境界でバイト配列をコピーし、すべての BLOB を読み込まずにメタデータを一覧表示します。`register(...)` は、期限切れのキーを含む明示的な upsert です。`registerIfAbsent(...)` は衝突に対して安全な作成を提供します。期限切れのキーは、その所有者が `deleteExpiredKey(key)` または `deleteExpired()` で取得するまで占有されたままになり、SQLite のコミット後に関連する名前付きアーティファクトを削除するために必要なメタデータが保持されます。TTL を持つ行はすべて一時的なものとして扱われ、期限切れになる前でもバックアップおよび復元から除外されます。永続的で復元可能な状態にするには TTL を省略します。ホストの安全制限により、各 BLOB は 100 MiB、各プラグインで物理的に保存される BLOB は 512 MiB、各プラグインで物理的に保存される行は 50,000 件に制限されます。これには、所有者によるクリーンアップを待つ期限切れの行も含まれます。外部の実体化データが置換や削除によって暗黙的に孤立してはならない場合は、`registerIfAbsent(...)` と `overflowPolicy: "reject-new"` を使用します。

    `openChannelIngressQueue<TPayload>(...)` は、再起動をまたいで最低 1 回の処理が必要な受信イベントをバッファリングするために、呼び出し元プラグインをスコープとする永続化された受信キューを開きます。古い取得状態の回復で `shouldRecover` を使用する際、取得済みの破損したペイロードを隔離する必要がある場合は、`shouldRecoverCorrupt` も指定します。そのペイロードに依存しない取得 ID により、キューが行をトゥームストーン化する前に、プラグインは有効な所有者とレーンのポリシーを保持できます。

    `withLease(...)` は、OpenClaw プロセス間で協調的なプラグイン処理を直列化します。グローバルな所有者を 1 つにするには `database: { scope: "shared" }`、エージェントごとに独立した所有権を持たせるには `{ scope: "agent", agentId }` を選択します。コールバックの `AbortSignal` を、失敗する可能性があるすべての処理に渡します。`assertOwned()` は、別の重要なステップを開始する前に使用する、その時点でのチェックポイントです。ホストはコールバックの後にも所有権を検証します。リースの喪失または呼び出し元によるキャンセルは、シグナルを中断します。取得待機と Heartbeat は、短い同期 SQLite トランザクションの外部で実行されます。プラグインがデータベースのパスやハンドルを受け取ることはありません。これは協調的キャンセルであり、フェンシングトークンでも、フェンシングされていない外部書き込みの認可でもありません。

    `openChannelIngressDrain(...)` は、そのキュー上でチャネルに依存しないコアワーカーを開きます（キューが指定されていない場合は作成します）。ドレインは、古い取得状態の回復、レーンごとの取得の直列化、採用時の完了またはディスパッチ復帰時の完了、再試行／デッドレターの処理、任意の採用前の置き換え、取得→採用の停滞タイムアウトを管理します。`turnAdoptionLifecycle`（`plugin-sdk/channel-outbound` の `bindIngressLifecycleToReplyOptions` 経由）を使用して、取得の所有権を返信生成に接続します。チャネルプラグインは、受け入れ側でのエンキュー、レーンの導出、再試行不可の分類、および置き換えの認可ポリシーを保持します。

    <Warning>
    `openBlobStore`、`openKeyedStore`、`openSyncKeyedStore`、`withLease`、`openChannelIngressQueue`、`openChannelIngressDrain` は、このリリースではバンドルされたプラグインと信頼済みの公式プラグインのインストールでのみ使用できます。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    チャネル固有のランタイムヘルパー（チャネルプラグインの読み込み時に使用可能）。目的別に分類されています。

    | グループ | 目的 |
    | --- | --- |
    | `text` | チャンク分割（`chunkText`、`chunkMarkdownText`、`resolveChunkMode`）、制御コマンドの検出、Markdown テーブルの変換。 |
    | `reply` | バッファリングされたブロック返信のディスパッチ、エンベロープの書式設定、有効なメッセージ／人間らしい遅延設定の解決。 |
    | `routing` | `buildAgentSessionKey`、`resolveAgentRoute`。 |
    | `pairing` | `buildPairingReply`、許可リストの読み取り／削除、ペアリング要求の upsert、要求から導出される承認エントリ。 |
    | `media` | リモートメディアのダウンロード／保存（以下を参照）。 |
    | `activity` | 最後のチャネルアクティビティを記録／読み取り。 |
    | `session` | 受信イベントからのセッションメタデータ、最終ルートの更新。 |
    | `mentions` | メンションポリシーのヘルパー（以下を参照）。 |
    | `reactions` | 処理中インジケーター用の確認リアクションハンドル。 |
    | `groups` | グループポリシーとメンション必須設定の解決。 |
    | `debounce` | 受信メッセージのデバウンス処理。 |
    | `commands` | コマンドの認可とテキストコマンドのゲーティング。 |
    | `outbound` | チャネルの送信アダプターを読み込み。 |
    | `inbound` | 受信イベントのコンテキストを構築し、共有受信イベント／返信カーネルを実行。 |
    | `threadBindings` | バインドされたセッションスレッドのアイドルタイムアウト／最大有効期間を調整。 |
    | `runtimeContexts` | プロセスローカルなチャネル／アカウント／ケイパビリティごとのコンテキストを登録、読み取り、監視。 |

    `api.runtime.channel.media` は、チャネルメディアのダウンロードと保存に推奨されるサーフェスです。

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    リモート URL を OpenClaw メディアに変換する場合は `saveRemoteMedia(...)` を使用します。プラグイン独自の認証、リダイレクト、または許可リスト処理を使用して、プラグインがすでに `Response` を取得している場合は、`saveResponseMedia(...)` を使用します。検査、変換、復号、または再アップロードのためにプラグインが生のバイト列を必要とする場合にのみ、`readRemoteMediaBuffer(...)` を使用します。`fetchRemoteMedia(...)` は、`readRemoteMediaBuffer(...)` の非推奨の互換性エイリアスとして残されています。

    `api.runtime.channel.mentions` は、ランタイム注入を使用するバンドル済みチャネルプラグイン向けの、共有受信メンションポリシーのサーフェスです。

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    使用可能なメンションヘルパー：

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    メンションの判定には、正規化された `{ facts, policy }` パスを使用します。

    `reply`、`session`、`inbound` の配下にあるいくつかのフィールドには、現在のチャネルターンカーネルまたはチャネル送信アダプターを示す、フィールドごとの `@deprecated` 注記があります。そのヘルパーに基づいて新しいコードを構築する前に、対象のヘルパーのインライン JSDoc を確認してください。

  </Accordion>
</AccordionGroup>

## ランタイム参照の保存

`register` コールバックの外部でランタイム参照を使用できるように保存するには、`createPluginRuntimeStore` を使用します。

<Steps>
  <Step title="ストアを作成する">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="エントリポイントに接続する">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="ほかのファイルからアクセスする">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // 初期化されていない場合は例外をスロー
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // 初期化されていない場合は null を返す
    }
    ```

  </Step>
</Steps>

<Note>
ランタイムストアの ID には `pluginId` を推奨します。低レベルの `key` 形式は、1 つのプラグインで意図的に複数のランタイムスロットを必要とする、一般的ではないケース向けです。
</Note>

## その他のトップレベル `api` フィールド

`api.runtime` に加えて、API オブジェクトは次のものも提供します。

<ParamField path="api.id" type="string">
  Plugin ID。
</ParamField>
<ParamField path="api.name" type="string">
  Pluginの表示名。
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  現在の設定スナップショット（利用可能な場合は、アクティブなインメモリランタイムスナップショット）。
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  `plugins.entries.<id>.config`から取得したPlugin固有の設定。
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  スコープ付きロガー（`debug`、`info`、`warn`、`error`）。
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  現在の読み込みモード：`"full"`（ライブアクティベーション）、`"discovery"` / `"tool-discovery"`（読み取り専用のケイパビリティ検出）、`"setup-only"`（軽量なセットアップエントリ）、`"setup-runtime"`（ランタイムチャネルエントリも必要とするセットアップフロー）、または`"cli-metadata"`（CLIコマンドメタデータの収集）。
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Pluginルートを基準に相対パスを解決します。
</ParamField>

## 関連項目

- [Pluginの内部構造](/ja-JP/plugins/architecture) — ケイパビリティモデルとレジストリ
- [SDKエントリポイント](/ja-JP/plugins/sdk-entrypoints) — `definePluginEntry`のオプション
- [SDKの概要](/ja-JP/plugins/sdk-overview) — サブパスのリファレンス
