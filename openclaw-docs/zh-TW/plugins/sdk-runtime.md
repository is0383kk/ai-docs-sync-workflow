---
read_when:
    - 你需要從外掛呼叫核心輔助功能（TTS、STT、影像生成、網路搜尋、閘道、子代理、節點）
    - 你想了解 `api.runtime` 公開了哪些內容
    - 你正從外掛程式碼存取設定、代理程式或媒體輔助工具
sidebarTitle: Runtime helpers
summary: api.runtime -- 可供外掛使用的注入式執行階段輔助函式
title: 外掛執行階段輔助工具
x-i18n:
    generated_at: "2026-07-26T08:00:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

注入至每個外掛並在註冊期間使用的 `api.runtime` 物件參考。請使用這些輔助工具，而不要直接匯入主機內部項目。

<CardGroup cols={2}>
  <Card title="頻道外掛" href="/zh-TW/plugins/sdk-channel-plugins">
    逐步指南，說明頻道外掛如何在實際情境中使用這些輔助工具。
  </Card>
  <Card title="提供者外掛" href="/zh-TW/plugins/sdk-provider-plugins">
    逐步指南，說明提供者外掛如何在實際情境中使用這些輔助工具。
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` 是目前的 OpenClaw 產品版本，來源為共用版本解析器，因此外掛看到的值會與命令列介面回報的值相同。

## 設定載入與寫入

優先使用已傳入作用中呼叫路徑的設定，例如註冊期間的 `api.config`，或頻道／提供者回呼的 `cfg` 引數。如此可讓單一處理程序快照貫穿整個工作流程，而不必在熱路徑上重新剖析設定。

只有在長期存續的處理常式需要目前的處理程序快照，且該函式未收到設定時，才使用 `api.runtime.config.current()`。傳回的值是唯讀的；編輯前請複製它或使用變更輔助工具。

工具工廠會收到 `ctx.runtimeConfig` 以及 `ctx.getRuntimeConfig()`。如果建立工具定義後設定仍可能變更，請在長期存續工具的 `execute` 回呼內使用 getter。

使用 `api.runtime.config.mutateConfigFile(...)` 或 `api.runtime.config.replaceConfigFile(...)` 持久化變更。每次寫入都必須明確選擇 `afterWrite` 原則：

- `afterWrite: { mode: "auto" }` 讓閘道重新載入規劃器決定。
- `afterWrite: { mode: "restart", reason: "..." }` 在寫入端確知熱重新載入不安全時，強制執行乾淨重新啟動。
- `afterWrite: { mode: "none", reason: "..." }` 僅在呼叫端負責後續處理時，抑制自動重新載入／重新啟動。

變更輔助工具會傳回 `afterWrite`，以及具型別的 `followUp` 摘要，讓呼叫端可以記錄或測試是否要求重新啟動。實際何時重新啟動仍由閘道負責。

使用 `current()`、傳入的 `cfg`、`mutateConfigFile(...)` 或
`replaceConfigFile(...)` 進行執行階段設定存取與寫入。

若直接從 SDK 匯入，請優先使用專用設定子路徑，而非廣泛的 `openclaw/plugin-sdk/config-runtime` 相容性彙整匯出：`config-contracts` 用於型別、`runtime-config-snapshot` 用於目前的處理程序快照，而 `config-mutation` 用於寫入。請從 `api.pluginConfig` 讀取項目範圍的值；提供的工具情境僅用於其執行階段全域設定快照，並在該邊界進行外掛專屬合併。內建外掛測試應直接模擬這些專用子路徑，而非模擬廣泛的相容性彙整匯出。

OpenClaw 內部執行階段程式碼也遵循相同方向：在命令列介面、閘道或處理程序邊界載入設定一次，之後持續傳遞該值。成功的變更寫入會重新整理處理程序執行階段快照並遞增其內部修訂版本；長期存續的快取應以執行階段擁有的快取鍵為依據，而非在本機序列化設定。長期存續的執行階段模組設有零容忍掃描器，禁止環境式 `loadConfig()` 呼叫；請使用傳入的 `cfg`、請求的 `context.getRuntimeConfig()`，或在明確的處理程序邊界使用 `getRuntimeConfig()`。

提供者與頻道執行路徑必須使用作用中的執行階段設定快照，而非為設定讀回或編輯所傳回的檔案快照。檔案快照會保留 SecretRef 標記等來源值，以供 UI 與寫入使用；提供者回呼則需要解析後的執行階段檢視。當輔助工具可能收到作用中的來源快照或作用中的執行階段快照時，請先透過 `selectApplicableRuntimeConfig()` 路由，再讀取認證資訊。

## 可重複使用的執行階段公用工具

針對由機器人撰寫的傳入訊息，請使用傳入的 `botLoopProtection` 事實。核心會在工作階段記錄與分派前套用共用的記憶體內滑動視窗防護，而不將原則綁定至單一頻道。此防護會追蹤 `(scopeId, conversationId, participant pair)` 鍵、合併計算配對雙向的事件數、在超過視窗預算後套用冷卻期，並伺機清除非作用中的項目。

向操作人員公開此行為的頻道外掛，應優先使用共用的 `channels.defaults.botLoopProtection` 結構作為基準預算，再於其上套用頻道／提供者專屬覆寫。共用設定使用秒為單位，因為這是面向使用者的設定：

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

請將正規化後的機器人配對事實與解析後的回合一併傳入。核心會解析預設值、單位轉換，以及 `enabled` 語意：

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

只有對不經過共用傳入回覆執行器的自訂
雙方事件迴圈，才直接使用 `openclaw/plugin-sdk/pair-loop-guard-runtime`。

## 執行階段命名空間

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    代理程式身分、目錄與工作階段管理。

    ```typescript
    // 解析代理程式的工作目錄（必須提供 agentId）
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // 解析代理程式工作區
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // 取得代理程式身分
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // 取得預設思考層級
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // 依作用中的提供者設定檔驗證使用者提供的思考層級
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // 將層級傳遞至內嵌執行
    }

    // 取得代理程式逾時
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // 確保工作區存在
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // 執行內嵌代理程式回合
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "摘要最新變更",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` 是從外掛程式碼啟動一般 OpenClaw 代理程式回合的中立輔助工具。它使用與頻道觸發回覆相同的提供者／模型解析及代理程式框架選擇。

    `runEmbeddedPiAgent(...)` 仍保留為現有外掛的已棄用相容性別名。新程式碼應使用 `runEmbeddedAgent(...)`。

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` 會與選擇讓內嵌執行使用 `cliBackendDispatch: "subscription-auth"` 的呼叫端，共用內嵌執行器的命令列介面後端分派決策（路由、後端宣告的 `subscriptionAuthDispatch` 功能、儲存的認證資訊模式——並遵循明確固定的 `authProfileId`）。如果該執行會透過命令列介面後端執行，它會傳回 `{ provider }`；如果維持直接傳遞，則傳回 `undefined`，讓呼叫端能針對實際執行的作業配置逾時預算。

    `resolveThinkingPolicy(...)` 會傳回提供者／模型支援的思考層級及選用的預設值。提供者外掛透過其思考鉤點擁有模型專屬設定檔，因此工具外掛應呼叫此執行階段輔助工具，而非匯入或複製提供者清單。

    `normalizeThinkingLevel(...)` 會將 `on`、`x-high` 或 `extra high` 等使用者文字轉換為標準儲存層級，再依解析後的原則進行檢查。

    **工作階段儲存區輔助工具**位於 `api.runtime.agent.session`：

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // 逐一處理工作階段資料列，而不依賴舊版 sessions.json 結構。
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
        // 建立或更新工作階段，然後將 signal 傳給已准入的代理程式執行。
      },
    );
    ```

    工作階段工作流程請優先使用 `getSessionEntry(...)`、`listSessionEntries(...)`、`patchSessionEntry(...)` 或 `upsertSessionEntry(...)`。這些輔助工具會透過代理程式／工作階段身分定位工作階段，讓外掛不必依賴舊版 `sessions.json` 儲存結構。若僅修補中繼資料且不應重新整理工作階段活動，請使用 `preserveActivity: true`；只有在回呼傳回完整項目且已刪除欄位必須維持刪除時，才使用 `replaceEntry: true`。Doctor 與移轉路徑可以結合 `fallbackEntry`、`skipMaintenance` 和 `requireWriteSuccess`，以原子方式修復標準儲存區。

    `createSessionEntry(...)` 會建立新的標準工作階段資料列與對話記錄。其受信任的 `initialEntry` 介面刻意保持精簡：非空的 `agentHarnessId`、選用的 `modelSelectionLocked: true`，以及選用的 `pluginExtensions`。注入的執行階段僅接受呼叫外掛透過 `registerAgentHarness(...)` 擁有的框架 ID；這是所有權不變條件，而非處理程序內外掛之間的沙箱。若資料列已存在，操作會遭拒；`label` 與 `spawnedCwd` 是獨立的建立欄位，而非受信任項目修補。

    建立期間會透過 `afterCreate` 持有工作階段生命週期變更柵欄，因此新工作會等待外掛擁有的初始化完成，而先前已准入的工作則會使建立失敗。回呼會收到已建立狀態的複本。如果回呼傳回修補內容，該修補只能包含 `pluginExtensions`，且其值是完整的最終 `pluginExtensions` 欄位。回呼或最終持久化失敗時，會回復未變更的新資料列與對話記錄；受防護的回復會保留已遭並行變更或認領的資料列。`recoverMatchingInitialEntry: true` 僅用於在持久化的受信任欄位完全相符時，重試中斷的初始化；復原要求 `afterCreate` 傳回最終修補內容。

    外掛開始處理持久化工作階段時，請使用 `runWithWorkAdmission(...)`。回呼會拒絕已封存或遭並行取代的工作階段，在作業完成前持續協調封存／重設／刪除變更，並接收必須轉傳給代理程式執行的 `AbortSignal`。框架可透過其實驗性的 `delegatedExecutionPluginIds` 註冊欄位，明確指定受信任的執行委派者。委派者只能准入並執行完全相符、已存在且模型已鎖定的工作階段；所有工作階段變更仍僅限框架擁有者執行。請參閱[代理程式框架外掛](/zh-TW/plugins/sdk-agent-harness#delegated-execution)。

    維護與修復外掛可將 `deleteSessionEntry(...)` 用於單一限定範圍的工作階段項目、將 `cleanupSessionLifecycleArtifacts(...)` 用於由生命週期管理的暫存工作階段，並在變更儲存區前使用 `resolveSessionStoreBackupPaths(...)`。當刪除作業不得與並行的工作階段更新發生競爭時，請傳入 `expectedSessionId` 和 `expectedUpdatedAt`；若較早的快照沒有工作階段 ID，請使用 `expectedSessionId: null`。這些輔助函式是範圍有限的修復／生命週期介面，不是通用的儲存區刪除 API。

    `resolveStorePath(...)` 和 `updateSessionStoreEntry(...)` 補全了工作階段輔助函式：`resolveStorePath` 會解析指定範圍的工作階段儲存區路徑，而當呼叫端已知儲存區路徑時，`updateSessionStoreEntry({ storePath, sessionKey, update })` 可直接修補其中一個項目。

    `loadTranscriptEventsSync(...)` 可用於無法使用非同步逐字稿執行階段的同步 doctor 與修復路徑。它會傳回原始 `SessionStoreTranscriptEvent` 記錄。一般外掛執行階段程式碼應優先使用 `openclaw/plugin-sdk/session-transcript-runtime`。

    `formatSqliteSessionFileMarker(...)`、`parseSqliteSessionFileMarker(...)` 和 `sqliteSessionFileMarkerMatchesSession(...)` 是過渡性輔助函式，供仍會收到名為 `sessionFile` 的舊版欄位之程式碼使用。經剖析的 SQLite 標記代表作用中的 SQLite 逐字稿目標；它不是檔案系統路徑。新的 API 應攜帶具型別的工作階段身分，而非標記字串。

    若要讀寫逐字稿，請匯入 `openclaw/plugin-sdk/session-transcript-runtime`，並搭配 `{ agentId, sessionKey, sessionId }` 使用 `resolveSessionTranscriptIdentity(...)`、`resolveSessionTranscriptTarget(...)`、`readSessionTranscriptEvents(...)`、`readSessionTranscriptRawDelta(...)`、`readSessionTranscriptVisibleMessageDelta(...)`、`readVisibleSessionTranscriptMessageEntries(...)`、`appendSessionTranscriptMessageByIdentity(...)`、`publishSessionTranscriptUpdateByIdentity(...)` 或 `withSessionTranscriptWriteLock(...)`。這些 API 可讓外掛識別逐字稿、讀取原始事件或分支安全的可見訊息項目、附加訊息、發布更新，以及在相同的逐字稿寫入鎖定下執行相關作業，而無須依賴作用中逐字稿的檔案路徑。`readVisibleSessionTranscriptMessageEntries(...)` 會傳回已排序的讀取中繼資料；其 `seq` 欄位不是可續接的游標。

    `appendSessionTranscriptMessageByIdentity(...)` 是用於附加已符合標準格式訊息的低階介面。外掛不得使用頂層 `MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType` 或 `MediaTypes` 合成含媒體的使用者資料列。頻道輸入端應透過 `MsgContext.media` 傳遞已排序的事實，並由主機負責保存使用者回合。由主機準備且已保存的使用者訊息會在 `message.__openclaw.media` 下攜帶標準化的已排序事實；通用附加 API 不會推斷或修復舊版平行陣列。

    `readSessionTranscriptRawDelta(...)` 會傳回有界的 `page`、`reset` 或 `missing` 結果。請將不透明的 `page.cursor` 傳入下一次呼叫。單純附加會保留游標，而逐字稿取代則會傳回 `reset` 與新的啟動游標。每頁預設上限為 1,000 個事件和 1,000,000 個序列化位元組；呼叫端最多可要求 10,000 個事件和 64 MiB。若僅下一個事件就超過 `maxBytes`，該頁將為空，並回報 `requiredBytes`；若所需的位元組限制不超過 64 MiB，請至少使用該限制重試。更大的單一事件需要使用完整讀取 API。游標只識別位置，絕不授予存取其他工作階段的權限。

    `readSessionTranscriptVisibleMessageDelta(...)` 在由主機管理的作用中訊息投影上提供相同的有界啟動與續接形式。它會依最舊到最新的順序傳回訊息，讓內容脈絡引擎能清空初始歷程，並將不透明游標保存為其水位標記。請原樣儲存並傳回游標；它是續接提示，而非授權認證資訊。線性附加會從最後一則已傳回訊息之後續接。逐字稿取代、錨點已離開作用中分支或在其中移動的游標、格式錯誤的游標，以及跨工作階段游標，都會傳回 `reset` 與新的啟動游標。計數與位元組的預設值及上限皆與原始差異 API 相同。分支變更後重建作用中投影時，結果會是 `unavailable`，原因為 `projection_rebuilding`；請稍後重試，而不要改用作用中逐字稿檔案。

    外掛 SDK 不再匯出舊版的完整儲存區及作用中逐字稿檔案輔助函式。工作階段中繼資料請使用範圍限定的項目輔助函式；作用中逐字稿作業請使用逐字稿身分輔助函式。需要檔案成品的封存／支援工作流程應使用其專用封存介面，而非作用中工作階段執行階段 API。

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    預設模型與提供者常數：

    ```typescript
    const model = api.runtime.agent.defaults.model; // 例如 "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // 例如 "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    執行由主機管理的文字補全，而無須匯入提供者內部元件或
    重複 OpenClaw 的模型／驗證／基礎 URL 準備作業。

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "摘要這份逐字稿。" }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    提供者協調層也可先取得已設定之本機服務的
    生命週期，再發出 HTTP 要求：

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
      // 傳送並完整取用提供者要求。
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` 是穩定且通用的提供者服務 SDK
    合約。主機會從 `models.providers.<providerId>.localService` 解析程序設定；
    呼叫端無法提供命令、引數、環境或生命週期原則。
    程序啟動、就緒檢查、診斷及閒置停止原則仍由主機內部管理。

    請傳入設定中確切的提供者 ID 與已解析的要求基礎 URL。請勿
    將別名替換為配接器 ID：不同的別名可能指向不同的
    本機 GPU 主機。除了 Ollama 和 LM
    Studio 配接器使用的 `/v1` 正規化外，主機會拒絕與所設定
    提供者基礎 URL 不相符的端點。主機負責啟動序列化、就緒探測、
    要求租約、中止處理及閒置關閉。

    此輔助函式使用與 OpenClaw
    內建執行階段相同的簡易補全準備路徑，以及由主機管理的執行階段設定快照。內容脈絡引擎
    會收到與工作階段繫結的 `llm.complete` 能力，因此模型呼叫會使用
    作用中工作階段的代理程式，而不會在未提示的情況下改用預設代理程式。結果
    會包含提供者／模型／代理程式的歸屬資訊，以及可用時經正規化的權杖、
    快取與估算成本用量。

    設定 `reasoning` 可要求所選模型使用特定推理強度。主機
    會先為所選提供者與模型正規化標準思考層級（`off`、`minimal`、`low`、
    `medium`、`high`、`xhigh`、`adaptive`、`max` 和 `ultra`），
    再分派補全。`adaptive` 會變成
    `medium`；若支援，`max` 和 `ultra` 會變成 `max`，否則變成 `xhigh`。

    <Warning>
    模型覆寫需要操作人員透過設定中的 `plugins.entries.<id>.llm.allowModelOverride: true` 明確選擇啟用。使用 `plugins.entries.<id>.llm.allowedModels` 可將受信任外掛限制為特定的標準 `provider/model` 目標。跨代理程式補全需要 `plugins.entries.<id>.llm.allowAgentIdOverride: true`。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    在程序內呼叫另一個閘道方法，同時保留目前外掛的受信任執行階段
    身分。這適用於組合外掛自有
    閘道功能的內附或受信任官方外掛，且無須開啟回送 WebSocket 連線。

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    要求使用 `operator.write` 範圍，且不會授予管理員範圍。任意外部
    外掛的呼叫會遭拒絕。失敗的方法會擲回 `GatewayClientRequestError`，並保留結構化
    `details`、重試中繼資料及供復原流程使用的閘道錯誤代碼。若工具也可在獨立代理程式程序中執行，
    請先使用 `isAvailable()`，再選擇此路徑。

  </Accordion>
  <Accordion title="api.runtime.subagent">
    啟動及管理背景子代理程式執行。

    ```typescript
    // 啟動子代理程式執行
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "將此查詢展開為聚焦的後續搜尋。",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // 選用覆寫
      model: "gpt-5.6-sol", // 選用覆寫
      deliver: false,
    });

    // 等待完成
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // 讀取工作階段訊息
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // 刪除工作階段
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    模型覆寫（`provider`/`model`）需要操作人員透過設定中的 `plugins.entries.<id>.subagent.allowModelOverride: true` 明確選擇啟用。不受信任的外掛仍可執行子代理程式，但覆寫要求會遭拒絕。
    </Warning>

    `toolsAlsoAllow` 會將呼叫外掛所註冊、具有確切名稱且由其唯一擁有的工具，加入工作程式的一般工具介面。執行階段會拒絕核心工具及與其他外掛共用名稱的工具。設定檔與操作人員工具原則仍會套用，包括明確的允許清單與拒絕規則。

    `deleteSession(...)` 可刪除同一外掛透過 `api.runtime.subagent.run(...)` 建立的工作階段。刪除任意使用者或操作人員工作階段仍需要具有管理員範圍的閘道要求。

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    檢查代理程式工作階段對沙箱工作區的有效權限。

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

    結果會回報此工作階段是否位於沙箱中、其工作區是否
    無法使用、唯讀或可寫入；若有效的 Docker、工具、工作階段、瀏覽器或提升權限原則可
    逸出該工作區，也會提供選用的 `confinementError`。
    請將此用於由主機管理的委派決策，這類決策不得授予工作程式高於其呼叫端的權限。它是證明
    輔助函式，不能取代檢查呼叫端本身的授權。

    `prepareWorkspaceAuthority(...)` 會執行相同的原則檢查，並且
    為 `workspaceDir` 準備 Docker 沙箱。若熱容器的
    即時設定雜湊與所要求的掛載或原則不相符，它會拒絕該容器。請僅傳入呼叫外掛
    確實限制其已註冊實作的工具名稱；萬用字元前綴無法證明工具擁有權。

  </Accordion>
  <Accordion title="api.runtime.nodes">
    從閘道載入的外掛程式碼或外掛命令列介面命令中，列出已連線的節點並叫用節點主機命令。當外掛負責已配對裝置上的本機工作時，請使用此功能，例如另一台 Mac 上的瀏覽器或音訊橋接器。

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` 包含每個已連線節點所公告的
    `nodePluginTools` 描述元，前提是該節點向代理程式公開由外掛或 MCP 支援的
    工具。這些描述元是即時連線狀態：當節點中斷連線時，閘道會
    移除它們；而當本機外掛／MCP 清單變更後，節點可用
    `node.pluginTools.update` 取代它們。

    在閘道內，此執行階段於處理程序內運作。在外掛命令列介面命令中，它會透過 RPC 呼叫已設定的閘道，因此 `openclaw googlemeet recover-tab` 等命令可從終端機檢查已配對的節點。節點命令仍會經過一般的閘道節點配對、命令允許清單、外掛節點叫用原則，以及節點本機命令處理。

    公開由節點代管之代理程式工具的外掛，可針對預設應列入允許清單的非危險命令設定 `agentTool.defaultPlatforms`。若必須由操作人員透過 `gateway.nodes.commands.allow` 選擇啟用，請省略此設定。危險的節點主機命令應使用 `api.registerNodeInvokePolicy(...)` 註冊節點叫用原則；此原則會在命令允許清單檢查之後、命令轉送至節點之前於閘道中執行，因此直接的 `node.invoke` 呼叫、由節點代管的外掛工具，以及更高階的外掛工具會共用相同的強制執行路徑。

    <Warning>
    選用的 `scopes` 欄位會為此叫用要求閘道操作人員範圍。OpenClaw 僅會針對內建外掛和受信任的官方外掛安裝接受此欄位；其他外掛的要求不會提升呼叫權限。僅當受信任的外掛必須以更嚴格的閘道範圍（例如 `operator.admin`）叫用節點命令時才使用此欄位。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    將 Task Flow 和 Task Run 狀態繫結至現有的 OpenClaw 工作階段金鑰或受信任的工具情境。

    - `api.runtime.tasks.managedFlows` 可執行變更：建立、推進及取消 Task Flow。
    - `api.runtime.tasks.flows` 和 `api.runtime.tasks.runs` 是唯讀的 DTO 檢視，用於列出項目和查詢狀態；兩者都會公開 `bindSession(...)` / `fromToolContext(...)`，以及 `get`、`list`、`findLatest` 和 `resolve`。

    Task Flow 會追蹤持久的多步驟工作流程狀態。它不是排程器：
    請使用排程或 `api.session.workflow.scheduleSessionTurn(...)` 進行未來的
    喚醒，接著，如果該工作需要流程狀態、子任務、等待或取消，
    請在排定的回合中使用 `managedFlows`。

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

    當你已從自己的繫結層取得受信任的 OpenClaw 工作階段金鑰時，請使用 `bindSession({ sessionKey, requesterOrigin })`。請勿從原始使用者輸入進行繫結。

  </Accordion>
  <Accordion title="api.runtime.tts">
    文字轉語音合成。

    ```typescript
    // 標準 TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "Hello from OpenClaw",
      cfg: api.config,
    });

    // 針對電話通訊最佳化的 TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "Hello from OpenClaw",
      cfg: api.config,
    });

    // 列出可用語音
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    使用核心 `tts` 設定和供應商選擇。傳回 PCM 音訊緩衝區與取樣率。`textToSpeechStream` 也可用於串流合成。

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    影像、音訊和影片分析。

    ```typescript
    // 描述影像
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // 轉錄音訊
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // 選用，適用於無法推斷 MIME 時
    });

    // 描述影片
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // 一般檔案分析
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // 透過特定供應商／模型進行結構化影像擷取。
    // 至少包含一張影像；文字輸入是補充情境。
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
        { type: "text", text: "Prefer the printed total over handwritten notes." },
      ],
      instructions: "Extract vendor, total, and searchable tags.",
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

    未產生任何輸出時（例如略過輸入），會傳回 `{ text: undefined }`。

    `describeImageFileWithModel(...)` 會透過特定供應商／模型描述已知影像，略過 `describeImageFile(...)` 所使用的預設作用中模型解析。

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    影像生成。

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "A robot painting a sunset",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    影片生成，其形式與影像生成相同。

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "A drone shot flying over a coastline at sunrise",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    音樂生成，其形式與影像生成相同。

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "An upbeat lo-fi track for a coding session",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    網路搜尋。

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw plugin SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    低階媒體公用工具。

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
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
    目前的執行階段設定快照和交易式設定寫入。請優先使用
    已傳入作用中呼叫路徑的設定；僅在處理常式需要直接取得處理程序快照時，
    才使用 `current()`。

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` 和 `replaceConfigFile(...)` 會傳回 `followUp`
    值，例如 `{ mode: "restart", requiresRestart: true, reason }`；
    此值會記錄寫入者的意圖，而不會從閘道奪走重新啟動的控制權。

  </Accordion>
  <Accordion title="api.runtime.system">
    系統層級公用工具。

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // 已淘汰的相容性別名。
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` 會立即執行單次心跳偵測週期，略過一般的合併計時器。傳入 `{ heartbeat: { target: "last" } }` 可強制傳送至最後一個作用中的頻道，而不採用預設的 `target: "none"` 抑制。

    `runCommandWithTimeout(...)` 會傳回擷取的 `stdout` 和 `stderr`、選用的
    截斷計數、`code`、`signal`、`killed`、`termination`，以及
    `noOutputTimedOut`。當子處理程序未提供非零結束代碼時，逾時和無輸出逾時結果會回報 `code: 124`。
    非逾時的訊號結束仍可能傳回 `code: null`，因此請使用 `termination` 和
    `noOutputTimedOut` 區分逾時原因。

  </Accordion>
  <Accordion title="api.runtime.events">
    事件訂閱。

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
    記錄。

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    模型與供應商的驗證解析。

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // 可供請求使用的驗證資訊，包括供應商執行階段交換（例如 OAuth 重新整理）
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    狀態目錄解析與以 SQLite 為基礎的鍵值儲存空間。

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

    鍵值儲存空間可在重新啟動後繼續保留，並依執行階段繫結的外掛 ID 隔離。使用 `registerIfAbsent(...)` 進行不可分割的重複資料刪除宣告：當鍵不存在或已到期並完成註冊時，會傳回 `true`；若已有有效值存在，則傳回 `false`，且不覆寫其值、建立時間或 TTL。當清理作業只能移除先前觀察到的值時，請使用 `deleteIf(...)`；其同步述詞與刪除作業會在同一個 SQLite 交易中執行。限制：每個命名空間 `maxEntries`、每個外掛 50,000 個有效資料列、JSON 值小於 64KB，以及選用的 TTL 到期機制。依預設，當寫入時達到任一資料列限制，系統會從正在寫入的命名空間移除最舊的有效資料列；同層命名空間不會因該次寫入而遭到淘汰，而若命名空間無法釋出足夠的資料列，寫入仍會失敗。對於絕不可遭到淘汰的持久擁有權記錄，請設定 `overflowPolicy: "reject-new"`：在達到任一限制時，新鍵會寫入失敗，但現有鍵仍可更新。

    `openSyncKeyedStore<T>(...)` 會傳回具有同步方法的相同儲存空間形式（`register`、`registerIfAbsent`、`deleteIf`、`lookup`、`consume`、`clear` 都會直接傳回值，而非 Promise），供無法使用 await 的呼叫端使用。

    `openBlobStore<TMetadata>(...)` 會將有界限的二進位承載資料儲存在共用 SQLite 中，不使用 base64 或檔案附屬檔。它要求設定每個項目、每個命名空間的位元組限制及資料列限制；在 API 邊界複製位元組陣列；並可列出中繼資料，而無須載入每個 BLOB。`register(...)` 是明確的 upsert，也適用於已到期的鍵。`registerIfAbsent(...)` 提供可避免衝突的建立方式：已到期的鍵仍會維持占用狀態，直到其擁有者使用 `deleteExpiredKey(key)` 或 `deleteExpired()` 宣告該鍵，以保留 SQLite 提交後移除相關具名成品所需的中繼資料。任何具有 TTL 的資料列都是暫時性的，即使尚未到期，也不會納入備份／還原；若是持久且可還原的狀態，請省略 TTL。主機保險絲會將每個 BLOB 的上限設為 100 MiB、每個外掛實際儲存的 BLOB 上限設為 512 MiB，並將每個外掛實際儲存的資料列上限設為 50,000，其中包括等待擁有者清理的已到期資料列。當外部具體化項目絕不可因取代或淘汰而在未告知的情況下成為孤立項目時，請搭配 `overflowPolicy: "reject-new"` 使用 `registerIfAbsent(...)`。

    `openChannelIngressQueue<TPayload>(...)` 會開啟限定於呼叫外掛的持久化輸入佇列，用於緩衝需要在重新啟動後至少處理一次的傳入事件。使用 `shouldRecover` 進行過期宣告復原時，若損毀的已宣告承載資料應予隔離，也請提供 `shouldRecoverCorrupt`：其不依賴承載資料的宣告識別資訊，可讓外掛在佇列將資料列標記為刪除前，保留有效的擁有者與通道政策。

    `withLease(...)` 會在各個 OpenClaw 程序間依序執行協作式外掛工作。選擇 `database: { scope: "shared" }` 以使用單一全域擁有者，或選擇 `{ scope: "agent", agentId }` 以使用各自獨立的每代理程式擁有權。將回呼的 `AbortSignal` 轉送至每個可能失敗的作業。`assertOwned()` 是開始另一個重要步驟前的時間點檢查；主機也會在回呼後驗證擁有權。租約遺失或呼叫端取消時，訊號會中止。取得等待與心跳偵測會在短暫的同步 SQLite 交易外進行；外掛絕不會收到資料庫路徑或控制代碼。這是協作式取消機制，而非隔離權杖，也不代表已獲授權可進行不受隔離保護的外部寫入。

    `openChannelIngressDrain(...)` 會在該佇列上開啟與核心通道無關的工作執行器（若未提供佇列，則建立一個）。排空程序負責過期宣告復原、每通道宣告序列化、採用時完成或分派傳回時完成、重試／無法傳遞處置、選用的採用前取代，以及宣告→採用停滯逾時。使用 `turnAdoptionLifecycle` 將宣告擁有權連接至回覆產生作業（透過來自 `plugin-sdk/channel-outbound` 的 `bindIngressLifecycleToReplyOptions`）。通道外掛負責接收端入列、通道衍生、不可重試分類，以及任何取代授權政策。

    <Warning>
    在此版本中，`openBlobStore`、`openKeyedStore`、`openSyncKeyedStore`、`withLease`、`openChannelIngressQueue` 和 `openChannelIngressDrain` 僅適用於隨附外掛與受信任的官方外掛安裝項目。
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    通道專用的執行階段輔助工具（載入通道外掛時可用）。依用途分組如下：

    | 群組 | 用途 |
    | --- | --- |
    | `text` | 分塊（`chunkText`、`chunkMarkdownText`、`resolveChunkMode`）、控制命令偵測、Markdown 表格轉換。 |
    | `reply` | 緩衝區塊回覆分派、信封格式化、有效訊息／人工延遲設定解析。 |
    | `routing` | `buildAgentSessionKey`、`resolveAgentRoute`。 |
    | `pairing` | `buildPairingReply`、允許清單讀取／移除、配對要求 upsert，以及衍生自要求的核准項目。 |
    | `media` | 遠端媒體下載／儲存（請參閱下文）。 |
    | `activity` | 記錄／讀取上次通道活動。 |
    | `session` | 來自傳入事件的工作階段中繼資料、上次路由更新。 |
    | `mentions` | 提及政策輔助工具（請參閱下文）。 |
    | `reactions` | 用於處理中指示器的確認回應控制代碼。 |
    | `groups` | 群組政策與必須提及設定解析。 |
    | `debounce` | 傳入訊息防彈跳。 |
    | `commands` | 命令授權與文字命令閘控。 |
    | `outbound` | 載入通道的輸出配接器。 |
    | `inbound` | 建立傳入事件內容，並執行共用傳入事件／回覆核心。 |
    | `threadBindings` | 調整繫結工作階段討論串的閒置逾時／存留時間上限。 |
    | `runtimeContexts` | 註冊、讀取及監看程序本機的每通道／帳號／功能內容。 |

    `api.runtime.channel.media` 是通道媒體下載與儲存的建議介面：

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    當遠端 URL 應轉換為 OpenClaw 媒體時，請使用 `saveRemoteMedia(...)`。當外掛已使用外掛自行管理的驗證資訊、重新導向或允許清單處理取得 `Response` 時，請使用 `saveResponseMedia(...)`。僅當外掛需要原始位元組以進行檢查、轉換、解密或重新上傳時，才使用 `readRemoteMediaBuffer(...)`。`fetchRemoteMedia(...)` 仍是 `readRemoteMediaBuffer(...)` 的已淘汰相容性別名。

    `api.runtime.channel.mentions` 是使用執行階段注入之隨附通道外掛的共用傳入提及政策介面：

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

    可用的提及輔助工具：

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    請使用正規化的 `{ facts, policy }` 路徑進行提及決策。

    `reply`、`session` 和 `inbound` 下的數個欄位包含各欄位的 `@deprecated` 附註，指向目前的通道輪次核心或通道輸出配接器；以特定輔助工具建置新程式碼前，請先查閱其內嵌 JSDoc。

  </Accordion>
</AccordionGroup>

## 儲存執行階段參照

使用 `createPluginRuntimeStore` 儲存執行階段參照，以便在 `register` 回呼外使用：

<Steps>
  <Step title="建立儲存空間">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="連接至進入點">
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
  <Step title="從其他檔案存取">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // 若未初始化則擲回錯誤
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // 若未初始化則傳回 null
    }
    ```

  </Step>
</Steps>

<Note>
執行階段儲存空間的識別資訊最好使用 `pluginId`。較低階的 `key` 形式適用於單一外掛刻意需要多個執行階段插槽的少見情況。
</Note>

## 其他頂層 `api` 欄位

除了 `api.runtime` 之外，API 物件也提供：

<ParamField path="api.id" type="string">
  外掛 ID。
</ParamField>
<ParamField path="api.name" type="string">
  外掛顯示名稱。
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  目前的設定快照（若可用，則為作用中的記憶體內執行階段快照）。
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  來自 `plugins.entries.<id>.config` 的外掛專屬設定。
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  限定範圍的記錄器（`debug`、`info`、`warn`、`error`）。
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  目前的載入模式：`"full"`（即時啟用）、`"discovery"` / `"tool-discovery"`（唯讀功能探索）、`"setup-only"`（輕量設定進入點）、`"setup-runtime"`（同時需要執行階段頻道進入點的設定流程），或 `"cli-metadata"`（命令列介面命令中繼資料收集）。
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  解析相對於外掛根目錄的路徑。
</ParamField>

## 相關內容

- [外掛內部機制](/zh-TW/plugins/architecture) — 功能模型與登錄檔
- [SDK 進入點](/zh-TW/plugins/sdk-entrypoints) — `definePluginEntry` 選項
- [SDK 概覽](/zh-TW/plugins/sdk-overview) — 子路徑參考資料
