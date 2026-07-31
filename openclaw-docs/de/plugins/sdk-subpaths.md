---
read_when:
    - Den richtigen plugin-sdk-Unterpfad für einen Plugin-Import auswählen
    - Prüfung der Unterpfade gebündelter Plugins und der Hilfsoberflächen
summary: 'Plugin-SDK-Unterpfadkatalog: welche Importe wo liegen, nach Bereich gruppiert'
title: Unterpfade des Plugin SDK
x-i18n:
    generated_at: "2026-07-26T18:39:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

Das Plugin-SDK enthält schmale öffentliche Unterpfade und ausschließlich im Repository verfügbare gebündelte
Hilfsfunktionen unter `openclaw/plugin-sdk/`. Diese Seite katalogisiert beide und kennzeichnet
private-lokale Einträge ausdrücklich. Drei Dateien definieren die Grenze:

- `scripts/lib/plugin-sdk-entrypoints.json`: das gepflegte Inventar der Einstiegspunkte,
  das vom Build kompiliert wird.
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`: interne Unterpfade,
  die vom typisierten, dokumentierten SDK ausgeschlossen sind. Produktionseinträge bleiben
  als reine JavaScript-Exporte der Host-Laufzeit für separat veröffentlichte offizielle
  Plugins verfügbar; reine Testeinträge bleiben nicht exportiert.
- `src/plugin-sdk/entrypoints.ts`: Klassifizierungsmetadaten für veraltete
  Unterpfade, reservierte gebündelte Hilfsfunktionen, unterstützte gebündelte Fassaden und
  Plugin-eigene öffentliche Oberflächen.

Maintainer prüfen die Anzahl öffentlicher Exporte mit `pnpm plugin-sdk:surface` und
aktive reservierte Hilfsfunktions-Unterpfade mit `pnpm plugins:boundary-report:summary`;
ungenutzte reservierte Hilfsfunktionsexporte lassen den CI-Bericht fehlschlagen, statt als
inaktive Kompatibilitätsschuld im öffentlichen SDK zu verbleiben.

Den Leitfaden zur Plugin-Erstellung finden Sie unter [Plugin-SDK-Übersicht](/de/plugins/sdk-overview).

## Plugin-Einstieg

| Unterpfad                      | Wichtige Exporte                                                                                                                                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | Seit Juli 2026 private-lokal; `defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | Seit Juli 2026 private-lokal; Hilfsfunktionen für Migrations-Provider-Elemente wie `createMigrationItem`, Ursachenkonstanten, Elementstatusmarkierungen, Schwärzungshilfen und `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | Seit Juli 2026 private-lokal; Laufzeit-Migrationshilfen wie `copyMigrationFileItem`, `resolvePlannedMigrationTargets`, `withCachedMigrationConfigRuntime` und `writeMigrationReport`              |
| `plugin-sdk/health`            | Registrierung, Erkennung, Reparatur, Auswahl, Schweregrad und Befundtypen für Doctor-Systemprüfungen gebündelter Systemstatus-Verbraucher                                                                                |

### Kompatibilität und private-lokale Hilfsfunktionen

Nur die in einem späteren Zeitfenster veralteten Unterpfade bleiben exportiert. Aliasse vom Juli 2026 und
ungenutzte Unterpfade wurden gelöscht, während ausschließlich gebündelte Hilfsfunktionen aus dem
öffentlichen Paket entfernt wurden und nachfolgend als private-lokal gekennzeichnet sind. Die gepflegte Liste ist
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; die CI weist gebündelte
`plugin-sdk/text-runtime` zurück, die nur der Kompatibilität dienen, und `plugin-sdk/zod` ist ein
Kompatibilitäts-Reexport: Importieren Sie `zod` direkt aus `zod`. Die breiten Domänen-
Barrels `plugin-sdk/agent-runtime`, `plugin-sdk/channel-lifecycle`,
`plugin-sdk/conversation-runtime`, `plugin-sdk/hook-runtime`,
`plugin-sdk/media-runtime`, `plugin-sdk/plugin-runtime` und
`plugin-sdk/security-runtime` sind ebenfalls zugunsten fokussierter
Unterpfade veraltet.

Die Vitest-basierten Testhilfs-Unterpfade von OpenClaw sind ausschließlich repository-lokal und werden
nicht mehr als Paketexporte bereitgestellt: `agent-runtime-test-contracts`,
`channel-contract-testing`, `channel-target-testing`, `channel-test-helpers`,
`plugin-state-test-runtime`, `plugin-test-api`, `plugin-test-contracts`,
`plugin-test-runtime`, `provider-http-test-mocks`, `provider-test-contracts`,
`reply-payload-testing`, `sqlite-runtime-testing`, `test-env`, `test-fixtures`,
`test-live`, `test-live-auth`, `test-media-generation`,
`test-media-understanding`, `test-node-mocks` und `testing`. Die privaten gebündelten Hilfsoberflächen
`ssrf-runtime-internal` und `codex-native-task-runtime` sind ebenfalls ausschließlich
repository-lokal.

### Hilfs-Unterpfade gebündelter Plugins

Ausschließlich gebündelte Hilfsmodule sind seit der Bereinigung im Juli 2026 private-lokal. Eigentümerübergreifende Importe werden durch Schutzmechanismen des Paketvertrags blockiert. `src/plugin-sdk/entrypoints.ts` erfasst separat die unterstützten gebündelten Fassaden, die öffentlich bleiben, also SDK-
Einstiegspunkte, die von ihrem gebündelten Plugin unterstützt werden, bis generische Verträge
`plugin-sdk/qa-runner-runtime`, `plugin-sdk/telegram-account` ersetzen,
für neuen Code veraltet; beachten Sie die Hinweise in den jeweiligen Zeilen unten.

<AccordionGroup>
  <Accordion title="Kanal-Unterpfade">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | Seit Juli 2026 private-lokal; Hilfsfunktion zur zwischengespeicherten JSON-Schema-Validierung für Plugin-eigene Schemas |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`, kanaleigene Typen für Einrichtungsfelder und -eingaben, `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard` sowie `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Gemeinsame Hilfsfunktionen des Einrichtungsassistenten, Einrichtungsübersetzer, Zulassungslisten-Abfragen, Ersteller für Einrichtungsstatus |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`, `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Hilfsfunktionen für Mehrkontenkonfiguration und Aktionssperren, Hilfsfunktionen für den Rückgriff auf das Standardkonto |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, Hilfsfunktionen zur Normalisierung von Konto-IDs |
    | `plugin-sdk/account-resolution` | Hilfsfunktionen für Kontosuche und Rückgriff auf den Standardwert |
    | `plugin-sdk/account-helpers` | Schmale Hilfsfunktionen für Kontolisten und Kontoaktionen |
    | `plugin-sdk/access-groups` | Seit Juli 2026 private-lokal; Hilfsfunktionen zum Parsen von Zugriffsgruppen-Zulassungslisten und für geschwärzte Gruppendiagnosen |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | Veraltete Kompatibilitätsfassade. Verwenden Sie `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`, `resolveChannelDmAccess`, `resolveChannelDmAllowFrom`, `resolveChannelDmPolicy`, `normalizeChannelDmPolicy`, `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | Gemeinsame Grundelemente für Kanalkonfigurationsschemas sowie Zod- und direkte JSON-/TypeBox-Builder |
    | `plugin-sdk/bundled-channel-config-schema` | Seit Juli 2026 private-lokal; Konfigurationsschemas gebündelter OpenClaw-Kanäle ausschließlich für gepflegte gebündelte Plugins |
    | `plugin-sdk/chat-channel-ids` | Seit Juli 2026 private-lokal; `BUNDLED_CHAT_CHANNEL_IDS`, `BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`, `ChatChannelId`. Kanonische IDs gebündelter/offizieller Chatkanäle sowie Formatiererbezeichnungen/-aliase für Plugins, die Text mit vorangestelltem Envelope-Präfix erkennen müssen, ohne eine eigene Tabelle fest zu codieren. |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | Experimenteller, abstrakter Laufzeit-Resolver für den Kanaleingang, Resolver für Richtlinien zu impliziten Erwähnungen und Ersteller von Routing-Fakten für migrierte Empfangspfade von Kanälen. Verwenden Sie dies vorzugsweise, statt effektive Zulassungslisten, Befehlszulassungslisten und Legacy-Projektionen in jedem Plugin zusammenzustellen. Siehe [API für den Kanaleingang](/de/plugins/sdk-channel-ingress). |
    | `plugin-sdk/channel-lifecycle` | Veraltete Kompatibilitätsfassade. Verwenden Sie `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-outbound` | Verträge für den Nachrichtenlebenszyklus sowie Optionen der Antwort-Pipeline, Empfangsbestätigungen, Live-Vorschau/Streaming, Lebenszyklus-Hilfsfunktionen, ausgehende Identität, Nutzlastplanung, dauerhafte Sendungen und Hilfsfunktionen für den Nachrichtenversandkontext. Siehe [API für ausgehende Kanalnachrichten](/de/plugins/sdk-channel-outbound). |
    | `plugin-sdk/channel-message` | Veralteter Kompatibilitätsalias für `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/inbound-envelope` | Gemeinsame Hilfsfunktionen zum Erstellen eingehender Routen und Envelopes |
    | `plugin-sdk/inbound-reply-dispatch` | Veraltete Kompatibilitätsfassade. Verwenden Sie `plugin-sdk/channel-inbound` für Ausführungsfunktionen für eingehende Nachrichten und Dispatch-Prädikate sowie `plugin-sdk/channel-outbound` für Hilfsfunktionen zur Nachrichtenzustellung. |
    | `plugin-sdk/messaging-targets` | Veralteter Alias für das Parsen von Zielen; verwenden Sie `plugin-sdk/channel-targets` |
    | `plugin-sdk/outbound-media` | Seit Juli 2026 private-lokal; Gemeinsame Hilfsfunktionen zum Laden ausgehender Medien und für den Zustand gehosteter Medien |
    | `plugin-sdk/poll-runtime` | Seit Juli 2026 private-lokal; Schmale Hilfsfunktionen zur Umfragenormalisierung |
    | `plugin-sdk/thread-bindings-runtime` | Seit Juli 2026 private-lokal; Hilfsfunktionen für den Lebenszyklus und Adapter von Thread-Bindungen |
    | `plugin-sdk/agent-media-payload` | Veraltete Kompatibilitätsfassade für die Legacy-Nutzlastprojektion `Media*`. Übergeben Sie geordnete Fakten über `MsgContext.media` / `toInboundMediaFacts(...)`; importieren Sie die Richtlinie für lokale Wurzeln aus `plugin-sdk/media-local-roots`. |
    | `plugin-sdk/conversation-runtime` | Veraltetes breites Barrel für Unterhaltungs-/Thread-Bindung, Kopplung und Hilfsfunktionen für konfigurierte Bindungen; bevorzugen Sie fokussierte Bindungs-Unterpfade wie `plugin-sdk/thread-bindings-runtime` und `plugin-sdk/session-binding-runtime` |
    | `plugin-sdk/runtime-group-policy` | Hilfsfunktionen zur Laufzeitauflösung von Gruppenrichtlinien |
    | `plugin-sdk/channel-status` | Gemeinsame Hilfsfunktionen für Momentaufnahmen und Zusammenfassungen des Kanalstatus |
    | `plugin-sdk/channel-config-primitives` | Schmale Grundelemente für Kanalkonfigurationsschemas |
    | `plugin-sdk/channel-config-writes` | Seit Juli 2026 private-lokal; Hilfsfunktionen zur Autorisierung von Schreibvorgängen an der Kanalkonfiguration |
    | `plugin-sdk/channel-plugin-common` | Gemeinsame Prelude-Exporte für Kanal-Plugins |
    | `plugin-sdk/allowlist-config-edit` | Hilfsfunktionen zum Bearbeiten und Lesen der Zulassungslistenkonfiguration |
    | `plugin-sdk/group-access` | Veraltete Hilfsfunktionen für Entscheidungen zum Gruppenzugriff; verwenden Sie `resolveChannelMessageIngress` aus `plugin-sdk/channel-ingress-runtime` |
    | `plugin-sdk/direct-dm-guard-policy` | Seit Juli 2026 private-lokal; Schmale Richtlinienhilfen für direkte DMs vor der Kryptografieprüfung |
    | `plugin-sdk/discord` | Veraltete Discord-Kompatibilitätsfassade für veröffentlichtes `@openclaw/discord@2026.3.13` und nachverfolgte Eigentümerkompatibilität; neue Plugins sollten generische Kanal-SDK-Unterpfade verwenden |
    | `plugin-sdk/telegram-account` | Veraltete Telegram-Kompatibilitätsfassade zur Kontoauflösung für nachverfolgte Eigentümerkompatibilität; neue Plugins sollten injizierte Laufzeithilfen oder generische Kanal-SDK-Unterpfade verwenden |
    | `plugin-sdk/interactive-runtime` | Semantische Nachrichtendarstellung, -zustellung und Legacy-Hilfsfunktionen für interaktive Antworten. Siehe [Nachrichtendarstellung](/de/plugins/message-presentation) |
    | `plugin-sdk/question-gateway-runtime` | Von der Laufzeit erstellte `ask_user`-Auswahlmöglichkeiten über das Gateway aus Interaktionshandlern von Kanälen auflösen |
    | `plugin-sdk/channel-inbound` | Gemeinsame Hilfsfunktionen für eingehende Nachrichten zur Ereignisklassifizierung, Kontexterstellung, Formatierung, Wurzeln, Entprellung, Erwähnungsabgleich, Erwähnungsrichtlinie und Protokollierung eingehender Nachrichten |
    | `plugin-sdk/channel-inbound-debounce` | Schmale Entprellungshilfen für eingehende Nachrichten |
    | `plugin-sdk/channel-mention-gating` | Seit Juli 2026 private-lokal; Schmale Hilfsfunktionen für Erwähnungsrichtlinien, Erwähnungsmarkierungen und Erwähnungstext ohne die breitere Laufzeitoberfläche für eingehende Nachrichten |
    | `plugin-sdk/channel-streaming` | Veraltete Kompatibilitätsfassade. Verwenden Sie `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-send-result` | Antwortergebnistypen |
    | `plugin-sdk/channel-actions` | Hilfsfunktionen für Kanalnachrichtenaktionen sowie veraltete native Schema-Hilfsfunktionen, die für die Plugin-Kompatibilität beibehalten werden |
    | `plugin-sdk/channel-route` | Seit Juli 2026 private-lokal; Gemeinsame Routennormalisierung, parsergestützte Zielauflösung, Umwandlung von Thread-IDs in Zeichenfolgen, deduplizierte/kompakte Routenschlüssel, Typen geparster Ziele und Hilfsfunktionen zum Vergleichen von Routen/Zielen |
    | `plugin-sdk/channel-targets` | Seit Juli 2026 private-lokal; Hilfsfunktionen zum Parsen von Zielen; Aufrufer für Routenvergleiche sollten `plugin-sdk/channel-route` verwenden |
    | `plugin-sdk/channel-contract` | Typen für Kanalverträge |
    | `plugin-sdk/channel-feedback` | Verdrahtung von Feedback/Reaktionen |
  </Accordion>

Kanal-Kompatibilitäts-Unterpfade aus späteren Zeitfenstern bleiben nur bis zu ihren
Registry-Daten öffentlich. Juli-Aliasse wie direkter DM-Zugriff, Antwortoptionen, Kopplungs-
pfade und abgespaltene Kanallaufzeiten wurden entfernt; ausschließlich gebündelte Hilfsfunktionen
sind private-lokal.

  <Accordion title="Provider-Unterpfade">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/provider-entry` | Ab Juli 2026 nur noch privat-lokal; `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Ab Juli 2026 nur noch privat-lokal; kuratierte Hilfsfunktionen zur Einrichtung lokaler bzw. selbst gehosteter Provider |
    | `plugin-sdk/cli-backend` | Ab Juli 2026 nur noch privat-lokal; CLI-Backend-Standardwerte und Watchdog-Konstanten |
    | `plugin-sdk/provider-auth-runtime` | Ab Juli 2026 nur noch privat-lokal; Laufzeithilfen für die Provider-Authentifizierung: OAuth-Loopback-Ablauf, Token-Austausch, Authentifizierungspersistenz und API-Schlüsselauflösung |
    | `plugin-sdk/provider-oauth-runtime` | Ab Juli 2026 nur noch privat-lokal; generische OAuth-Callback-Typen für Provider, Rendering der Callback-Seite, PKCE-/State-Hilfen, Parsing der Autorisierungseingabe, Hilfen für den Token-Ablauf und Abbruchhilfen |
    | `plugin-sdk/provider-auth-api-key` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen für das Onboarding per API-Schlüssel und das Schreiben von Profilen wie `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Ab Juli 2026 nur noch privat-lokal; standardmäßiger Builder für OAuth-Authentifizierungsergebnisse |
    | `plugin-sdk/provider-env-vars` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen zum Nachschlagen von Umgebungsvariablen für die Provider-Authentifizierung |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials`, Hilfsfunktionen zum Importieren der OpenAI-Codex-Authentifizierung, veralteter Kompatibilitätsexport `resolveOpenClawAgentDir` |
    | `plugin-sdk/provider-model-shared` | Ab Juli 2026 nur noch privat-lokal; `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `selectPreferredLocalModelId`, `normalizeModelCompat`, gemeinsam genutzte Builder für Wiederholungsrichtlinien, Hilfsfunktionen für Provider-Endpunkte und gemeinsam genutzte Hilfsfunktionen zur Normalisierung von Modell-IDs |
    | `plugin-sdk/provider-catalog-live-runtime` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Live-Modellkataloge von Providern zur abgesicherten Erkennung im Stil von `/models`: `buildLiveModelProviderConfig`, `fetchLiveProviderModelRows`, `getCachedLiveProviderModelRows`, `fetchLiveProviderModelIds`, `LiveModelCatalogHttpError`, `clearLiveCatalogCacheForTests`, Modell-ID-Filterung, TTL-Cache und statischer Fallback |
    | `plugin-sdk/provider-catalog-runtime` | Laufzeit-Hook zur Erweiterung des Provider-Katalogs und Schnittstellen der Plugin-Provider-Registry für Vertragstests |
    | `plugin-sdk/provider-catalog-shared` | Ab Juli 2026 nur noch privat-lokal; `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `buildManifestModelProviderConfig`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Ab Juli 2026 nur noch privat-lokal; generische Hilfsfunktionen für HTTP-/Endpunktfähigkeiten von Providern, Provider-HTTP-Fehler und Hilfsfunktionen für Multipart-Formulare zur Audiotranskription |
    | `plugin-sdk/provider-web-fetch-contract` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Vertragshilfen für die Web-Fetch-Konfiguration und -Auswahl wie `enablePluginInConfig` und `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Registrierung und Cache von Web-Fetch-Providern |
    | `plugin-sdk/provider-web-search-config-contract` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Hilfsfunktionen für Websuche-Konfiguration und Anmeldedaten bei Providern, die keine Plugin-Aktivierungsverdrahtung benötigen |
    | `plugin-sdk/provider-web-search-contract` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Vertragshilfen für Websuche-Konfiguration und Anmeldedaten wie `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` sowie bereichsgebundene Setter/Getter für Anmeldedaten |
    | `plugin-sdk/provider-web-search` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Registrierung, Cache und Laufzeit von Websuche-Providern |
    | `plugin-sdk/embedding-providers` | Ab Juli 2026 nur noch privat-lokal; allgemeine Typen und Lesehilfen für Embedding-Provider, einschließlich `EmbeddingProviderAdapter`, `getEmbeddingProvider(...)` und `listEmbeddingProviders(...)`; Plugins registrieren Provider über `api.registerEmbeddingProvider(...)`, damit die Manifest-Zuständigkeit durchgesetzt wird |
    | `plugin-sdk/provider-tools` | Ab Juli 2026 nur noch privat-lokal; `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks` sowie Schemabereinigung und Diagnosen für DeepSeek/Gemini/OpenAI |
    | `plugin-sdk/provider-usage` | Ab Juli 2026 nur noch privat-lokal; Typen für Provider-Nutzungsmomentaufnahmen, gemeinsam genutzte Hilfsfunktionen zum Abrufen der Nutzung und Provider-Abrufroutinen wie `fetchClaudeUsage` |
    | `plugin-sdk/provider-stream` | Ab Juli 2026 nur noch privat-lokal; `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, Stream-Wrapper-Typen, Kompatibilität für Klartext-Tool-Aufrufe und gemeinsam genutzte Wrapper-Hilfsfunktionen für Anthropic/Google/Kilocode/MiniMax/Moonshot/OpenAI/OpenRouter/Z.AI |
    | `plugin-sdk/provider-stream-shared` | Ab Juli 2026 nur noch privat-lokal; öffentliche, gemeinsam genutzte Stream-Wrapper-Hilfsfunktionen für Provider, einschließlich `composeProviderStreamWrappers`, `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPlainTextToolCallCompatWrapper`, `createPayloadPatchStreamWrapper`, `createToolStreamWrapper`, `normalizeOpenAICompatibleReasoningPayload`, `setQwenChatTemplateThinking` sowie Anthropic-/DeepSeek-/OpenAI-kompatible Stream-Dienstprogramme |
    | `plugin-sdk/provider-transport-runtime` | Ab Juli 2026 nur noch privat-lokal; native Provider-Transporthilfen wie abgesicherter Abruf, Textextraktion aus Tool-Ergebnissen, Transformationen von Transportnachrichten und beschreibbare Transportereignis-Streams |
    | `plugin-sdk/provider-onboard` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen zum Patchen der Onboarding-Konfiguration |
    | `plugin-sdk/global-singleton` | Ab Juli 2026 nur noch privat-lokal; prozesslokale Singleton-/Map-/Cache-Hilfsfunktionen |
    | `plugin-sdk/group-activation` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Hilfsfunktionen für den Gruppenaktivierungsmodus und das Befehlsparsing |
  </Accordion>

Provider-Nutzungsmomentaufnahmen melden normalerweise ein oder mehrere Kontingent-`windows`, jeweils mit
einer Bezeichnung, dem verwendeten Prozentsatz und einer optionalen Rücksetzzeit. Provider, die anstelle
zurücksetzbarer Kontingentfenster einen Saldo- oder Kontostatus-Text bereitstellen, sollten
`summary` mit einem leeren `windows`-Array zurückgeben, statt Prozentsätze zu erfinden.
OpenClaw zeigt diesen Zusammenfassungstext in der Statusausgabe an; verwenden Sie `error` nur, wenn der
Nutzungsendpunkt fehlgeschlagen ist oder keine nutzbaren Nutzungsdaten zurückgegeben hat.

  <Accordion title="Unterpfade für Authentifizierung und Sicherheit">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/command-auth` | Veraltete, breit gefasste Oberfläche für die Befehlsautorisierung (`resolveControlCommandGate`, Hilfsfunktionen der Befehls-Registry einschließlich dynamischer Formatierung von Argumentmenüs, Hilfsfunktionen zur Absenderautorisierung); verwenden Sie stattdessen die Autorisierung beim Kanaleingang bzw. zur Laufzeit oder Hilfsfunktionen für den Befehlsstatus |
    | `plugin-sdk/command-status` | Builder für Befehls-/Hilfenachrichten wie `buildCommandsMessagePaginated` und `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Hilfsfunktionen zur Auflösung von Genehmigenden und zur Aktionsauthentifizierung innerhalb desselben Chats |
    | `plugin-sdk/approval-client-runtime` | Hilfsfunktionen für native Ausführungsgenehmigungsprofile und -filter |
    | `plugin-sdk/approval-delivery-runtime` | Native Adapter für Genehmigungsfähigkeiten und -zustellung |
    | `plugin-sdk/approval-gateway-runtime` | Gemeinsam genutzter Resolver für das Genehmigungs-Gateway |
    | `plugin-sdk/approval-reference-runtime` | Ab Juli 2026 nur noch privat-lokal; deterministische Hilfsfunktion für dauerhafte Locators bei transportbeschränkten Genehmigungs-Callbacks |
    | `plugin-sdk/approval-handler-adapter-runtime` | Leichtgewichtige Hilfsfunktionen zum Laden nativer Genehmigungsadapter für häufig durchlaufene Kanaleinstiegspunkte |
    | `plugin-sdk/approval-handler-runtime` | Breiter gefasste Laufzeithilfen für Genehmigungs-Handler; bevorzugen Sie die enger gefassten Adapter-/Gateway-Schnittstellen, wenn diese ausreichen |
    | `plugin-sdk/approval-native-runtime` | Hilfsfunktionen für native Genehmigungsziele, Kontobindung, Routing-Gates, Weiterleitungs-Fallbacks und die Unterdrückung lokaler nativer Ausführungsaufforderungen |
    | `plugin-sdk/approval-reaction-runtime` | Ab Juli 2026 nur noch privat-lokal; fest codierte Bindungen für Genehmigungsreaktionen, Nutzlasten für Reaktionsaufforderungen, Speicher für Reaktionsziele, Hilfsfunktionen für Reaktionshinweistexte und Kompatibilitätsexport zur Unterdrückung lokaler nativer Ausführungsaufforderungen |
    | `plugin-sdk/approval-reply-runtime` | Hilfsfunktionen für Antwortnutzlasten bei Ausführungs-/Plugin-Genehmigungen |
    | `plugin-sdk/approval-runtime` | Hilfsfunktionen für Nutzlasten bei Ausführungs-/Plugin-Genehmigungen, Builder für Genehmigungsfähigkeiten, Hilfsfunktionen für Genehmigungsauthentifizierung und -profile, Hilfsfunktionen für natives Genehmigungs-Routing und dessen Laufzeit sowie strukturierte Hilfsfunktionen zur Genehmigungsanzeige wie `formatApprovalDisplayPath` |
    | `plugin-sdk/command-auth-native` | Native Befehlsauthentifizierung, dynamische Formatierung von Argumentmenüs und Hilfsfunktionen für native Sitzungsziele |
    | `plugin-sdk/command-detection` | Gemeinsam genutzte Hilfsfunktionen zur Befehlserkennung |
    | `plugin-sdk/command-primitives-runtime` | Leichtgewichtige Prädikate für Befehlstext in häufig durchlaufenen Kanalpfaden |
    | `plugin-sdk/command-surface` | Ab Juli 2026 nur noch privat-lokal; Normalisierung von Befehlstextkörpern und Hilfsfunktionen für Befehlsoberflächen |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | Ab Juli 2026 nur noch privat-lokal; Hilfsfunktionen für verzögert geladene Provider-Authentifizierungsabläufe zur Kopplung privater Kanäle und der Web-UI per Gerätecode |
    | `plugin-sdk/channel-secret-runtime` | Veraltete, breit gefasste Oberfläche für Secret-Verträge (`collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, Secret-Zieltypen); bevorzugen Sie die nachstehenden fokussierten Unterpfade |
    | `plugin-sdk/channel-secret-basic-runtime` | Eng gefasste Exporte für Secret-Verträge und Builder für Ziel-Registries bei Nicht-TTS-Secret-Oberflächen von Kanälen/Plugins |
    | `plugin-sdk/channel-secret-tts-runtime` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Hilfsfunktionen zur Zuweisung verschachtelter Kanal-TTS-Secrets |
    | `plugin-sdk/secret-ref-runtime` | Eng gefasste Typisierung und Auflösung von SecretRef sowie Nachschlagen von Planzielpfaden für das Parsing von Secret-Verträgen und Konfigurationen |
    | `plugin-sdk/security-runtime` | Veraltetes, breit gefasstes Barrel für Vertrauensprüfung, DM-Gating, auf das Stammverzeichnis begrenzte Datei-/Pfadhilfen einschließlich ausschließlich erstellender Schreibvorgänge, synchrone/asynchrone atomare Dateiersetzung, Schreiben temporärer Geschwisterdateien, Fallback für das Verschieben über Gerätegrenzen hinweg, Hilfsfunktionen für private Dateispeicher, Schutzprüfungen für Symlink-Elternverzeichnisse, externe Inhalte, Schwärzung vertraulicher Texte, Secret-Vergleich mit konstanter Laufzeit und Hilfsfunktionen zur Secret-Sammlung; bevorzugen Sie fokussierte Unterpfade für Sicherheit/SSRF/Secrets |
    | `plugin-sdk/ssrf-policy` | Hilfsfunktionen für Host-Zulassungslisten und SSRF-Richtlinien für private Netzwerke |
    | `plugin-sdk/ssrf-dispatcher` | Ab Juli 2026 nur noch privat-lokal; eng gefasste Hilfsfunktionen für angeheftete Dispatcher ohne die breit gefasste Infrastruktur-Laufzeitoberfläche |
    | `plugin-sdk/ssrf-runtime` | Hilfsfunktionen für angeheftete Dispatcher, SSRF-abgesicherte Abrufe, SSRF-Fehler und SSRF-Richtlinien |
    | `plugin-sdk/secret-input` | Hilfsfunktionen zum Parsen von Secret-Eingaben |
    | `plugin-sdk/webhook-ingress` | Hilfsfunktionen für Webhook-Anfragen/-Ziele und Roh-WebSocket-/Body-Koersion |
    | `plugin-sdk/webhook-request-guards` | Hilfsfunktionen für Größe und Zeitüberschreitung von Anfragekörpern sowie `runDetachedWebhookWork` für nachverfolgte Verarbeitung nach der Bestätigung |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/runtime` | Hilfsfunktionen für Runtime, Protokollierung und Sicherung, Warnungen zu Plugin-Installationspfaden sowie Prozess-Hilfsfunktionen |
    | `plugin-sdk/runtime-env` | Eng gefasste Hilfsfunktionen für Runtime-Umgebung, Logger, Zeitüberschreitung, Wiederholungsversuche und Backoff |
    | `plugin-sdk/browser-config` | Nach Juli 2026 nur noch privat-lokal; unterstützte Browserkonfigurationsfassade für normalisierte Profile/Standardwerte, das Parsen von CDP-URLs und Hilfsfunktionen zur Authentifizierung der Browsersteuerung |
    | `plugin-sdk/agent-harness-task-runtime` | Nach Juli 2026 nur noch privat-lokal; generische Hilfsfunktionen für Aufgabenlebenszyklus und Abschlusszustellung für Harness-gestützte Agenten mit einem vom Host ausgegebenen Aufgabenbereich |
    | `plugin-sdk/codex-mcp-projection` | Nach Juli 2026 nur noch privat-lokal; reservierte gebündelte Codex-Hilfsfunktion zur Übertragung der MCP-Serverkonfiguration des Benutzers in die Codex-Threadkonfiguration; nicht für Drittanbieter-Plugins |
    | `plugin-sdk/codex-native-task-runtime` | Repo-lokale gebündelte Codex-Hilfsfunktion für die native Aufgabenpiegelung/Runtime-Verdrahtung; kein Paketexport |
    | `plugin-sdk/channel-runtime-context` | Generische Hilfsfunktionen zur Registrierung und Abfrage des Runtime-Kontexts von Kanälen |
    | `plugin-sdk/matrix` | Veraltete Matrix-Kompatibilitätsfassade für ältere Kanalpakete von Drittanbietern; neue Plugins sollten `plugin-sdk/run-command` direkt importieren |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Veralteter breiter Barrel-Export für Plugin-Befehls-, Hook-, HTTP- und interaktive Hilfsfunktionen; bevorzugen Sie fokussierte Plugin-Runtime-Unterpfade |
    | `plugin-sdk/hook-runtime` | Veralteter breiter Barrel-Export für Webhook-/interne Hook-Pipeline-Hilfsfunktionen; bevorzugen Sie fokussierte Hook-/Plugin-Runtime-Unterpfade |
    | `plugin-sdk/lazy-runtime` | Hilfsfunktionen für verzögerten Runtime-Import und -Binding wie `createLazyRuntimeModule`, `createLazyRuntimeMethod` und `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Prozessausführung |
    | `plugin-sdk/node-host` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Auflösung ausführbarer Dateien auf dem Node-Host und zur PTY-Fortsetzung |
    | `plugin-sdk/cli-runtime` | Nach Juli 2026 nur noch privat-lokal; veralteter breiter Barrel-Export für CLI-Formatierung, Warten, Versionierung, Argumentaufruf und verzögert geladene Befehlsgruppen; bevorzugen Sie fokussierte CLI-/Runtime-Unterpfade |
    | `plugin-sdk/qa-runner-runtime` | Nach Juli 2026 nur noch privat-lokal; unterstützte Fassade, die Plugin-QA-Szenarien über die CLI-Befehlsoberfläche bereitstellt |
    | `plugin-sdk/tts-runtime` | Nach Juli 2026 nur noch privat-lokal; unterstützte Fassade für Text-zu-Sprache-Konfigurationsschemas und Runtime-Hilfsfunktionen |
    | `plugin-sdk/gateway-method-runtime` | Reservierte Hilfsfunktion zur Gateway-Methodenweiterleitung für Plugin-HTTP-Routen, die `contracts.gatewayMethodDispatch: ["authenticated-request"]` deklarieren |
    | `plugin-sdk/gateway-runtime` | Gateway-Client, Hilfsfunktion zum Starten eines ereignisschleifenbereiten Clients, Gateway-CLI-RPC, Gateway-Protokollfehler, Auflösung angekündigter LAN-Hosts und Hilfsfunktionen für Kanalstatus-Patches |
    | `plugin-sdk/config-contracts` | Fokussierte reine Typ-Konfigurationsoberfläche für Plugin-Konfigurationsformen wie `OpenClawConfig` sowie Kanal-/Provider-Konfigurationstypen |
    | `plugin-sdk/plugin-config-runtime` | Veraltete Kompatibilitätsfassade für Runtime-Hilfsfunktionen der Plugin-Konfiguration; neue Plugins verwenden `api.pluginConfig` zusammen mit fokussierten Konfigurationsverträgen, Snapshots und Mutationshilfsfunktionen |
    | `plugin-sdk/config-mutation` | Transaktionale Hilfsfunktionen für Konfigurationsänderungen wie `mutateConfigFile`, `replaceConfigFile` und `logConfigUpdated` |
    | `plugin-sdk/message-tool-delivery-hints` | Nach Juli 2026 nur noch privat-lokal; gemeinsam genutzte Hinweiszeichenfolgen für Zustellungsmetadaten von Nachrichtenwerkzeugen |
    | `plugin-sdk/runtime-config-snapshot` | Hilfsfunktionen für Snapshots der aktuellen Prozesskonfiguration wie `getRuntimeConfig`, `getRuntimeConfigSnapshot` und Test-Snapshot-Setter |
    | `plugin-sdk/text-autolink-runtime` | Nach Juli 2026 nur noch privat-lokal; Erkennung automatischer Links für Dateiverweise ohne den breiten Text-Barrel-Export |
    | `plugin-sdk/reply-runtime` | Gemeinsam genutzte Runtime-Hilfsfunktionen für eingehende Nachrichten/Antworten, Aufteilung, Weiterleitung, Heartbeat und Antwortplanung |
    | `plugin-sdk/reply-dispatch-runtime` | Eng gefasste Hilfsfunktionen für Antwortweiterleitung/-abschluss und Konversationsbezeichnungen |
    | `plugin-sdk/reply-history` | Gemeinsam genutzte Hilfsfunktionen für den kurzfristigen Antwortverlauf. Neuer Code für Nachrichtendurchläufe sollte `createChannelHistoryWindow` verwenden; untergeordnete Map-Hilfsfunktionen bleiben ausschließlich veraltete Kompatibilitätsexporte |
    | `plugin-sdk/reply-reference` | Nach Juli 2026 nur noch privat-lokal; `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Eng gefasste Hilfsfunktionen zur Aufteilung von Text/Markdown |
    | `plugin-sdk/session-store-runtime` | Hilfsfunktionen für Sitzungsworkflows (`getSessionEntry`, `listSessionEntries`, `patchSessionEntry`, `upsertSessionEntry`), Reparatur-/Lebenszyklus-Hilfsfunktionen (`deleteSessionEntry`, `cleanupSessionLifecycleArtifacts`, `resolveSessionStoreBackupPaths`), Marker-Hilfsfunktionen für übergangsweise `sessionFile`-Werte, begrenztes Lesen kürzlich erfasster Benutzer-/Assistenten-Transkripttexte anhand der Sitzungsidentität, Hilfsfunktionen für Sitzungsspeicherpfade/Sitzungsschlüssel und Lesen des Aktualisierungszeitpunkts, ohne breite Importe für Konfigurationsschreibvorgänge/-wartung |
    | `plugin-sdk/session-transcript-runtime` | Nach Juli 2026 nur noch privat-lokal; Transkriptidentität, begrenzte rohe und sichtbare Cursor, bereichsgebundene Ziel-/Lese-/Schreibhilfsfunktionen, Projektion sichtbarer Nachrichteneinträge, Veröffentlichung von Aktualisierungen, Schreibsperren und Trefferschlüssel für den Transkriptspeicher |
    | `plugin-sdk/sqlite-runtime` | Nach Juli 2026 nur noch privat-lokal; fokussierte SQLite-Hilfsfunktionen für Agentenschema, Pfade und Transaktionen der Erstanbieter-Runtime, ohne Steuerung des Datenbanklebenszyklus |
    | `plugin-sdk/cron-store-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Pfad, Laden und Speichern des Cron-Speichers |
    | `plugin-sdk/state-paths` | Hilfsfunktionen für Pfade von Status-/OAuth-Verzeichnissen |
    | `plugin-sdk/plugin-state-runtime` | Nach Juli 2026 nur noch privat-lokal; Plugin-bereichsgebundene Verträge für schlüsselbasierten Zustand, BLOBs und kooperative SQLite-Leases sowie Hilfsfunktionen für Verbindungs-Pragmas, verifizierte WAL-Wartung und atomare Migrationen von STRICT-Schemas. Lease-Callbacks erhalten ein Abbruchsignal, und typisierte Fehler unterscheiden zwischen Zeitüberschreitung, Abbruch, verlorenem Besitz, ungültiger Eingabe und Speicherfehler |
    | `plugin-sdk/routing` | Hilfsfunktionen für Routen-/Sitzungsschlüssel-/Kontobindungen wie `resolveAgentRoute`, `buildAgentSessionKey` und `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Gemeinsam genutzte Hilfsfunktionen für Kanal-/Kontostatuszusammenfassungen, Standardwerte des Runtime-Zustands und Problemmetadaten |
    | `plugin-sdk/target-resolver-runtime` | Nach Juli 2026 nur noch privat-lokal; gemeinsam genutzte Hilfsfunktionen zur Zielauflösung |
    | `plugin-sdk/string-normalization-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Normalisierung von Slugs/Zeichenfolgen |
    | `plugin-sdk/request-url` | Nach Juli 2026 nur noch privat-lokal; Extrahieren von URL-Zeichenfolgen aus Fetch-/Request-ähnlichen Eingaben |
    | `plugin-sdk/run-command` | Zeitgesteuerter Befehls-Runner mit normalisierten stdout-/stderr-Ergebnissen |
    | `plugin-sdk/param-readers` | Allgemeine Leser für Werkzeug-/CLI-Parameter |
    | `plugin-sdk/tool-plugin` | Definiert ein einfaches typisiertes Agentenwerkzeug-Plugin und stellt statische Metadaten für die Manifestgenerierung bereit |
    | `plugin-sdk/tool-payload` | Nach Juli 2026 nur noch privat-lokal; Extrahieren normalisierter Nutzlasten aus Werkzeugergebnisobjekten |
    | `plugin-sdk/tool-send` | Extrahieren kanonischer Sendeziel-Felder aus Werkzeugargumenten |
    | `plugin-sdk/sandbox` | Nach Juli 2026 nur noch privat-lokal; Typen für Sandbox-Backends und SSH-/OpenShell-Befehlshilfsfunktionen, einschließlich Fail-Fast-Vorprüfung für Ausführungsbefehle |
    | `plugin-sdk/temp-path` | Gemeinsam genutzte Hilfsfunktionen für temporäre Downloadpfade und private sichere temporäre Arbeitsbereiche |
    | `plugin-sdk/logging-core` | Hilfsfunktionen für Subsystem-Logger und Schwärzung |
    | `plugin-sdk/markdown-table-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Markdown-Tabellenmodus und -konvertierung |
    | `plugin-sdk/model-session-runtime` | Hilfsfunktionen für Modell-/Sitzungsüberschreibungen wie `applyModelOverrideToSessionEntry` und `resolveAgentMaxConcurrent` |
    | `plugin-sdk/talk-config-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Auflösung der Konfiguration des Talk-Providers |
    | `plugin-sdk/json-store` | Kleine Hilfsfunktionen zum Lesen/Schreiben von JSON-Zuständen |
    | `plugin-sdk/json-unsafe-integers` | Nach Juli 2026 nur noch privat-lokal; JSON-Parsing-Hilfsfunktionen, die unsichere Ganzzahlliterale als Zeichenfolgen erhalten |
    | `plugin-sdk/file-lock` | Nach Juli 2026 nur noch privat-lokal; wiedereintrittsfähige Dateisperr-Hilfsfunktionen sowie Doctor-sichere Rückgewinnung eindeutig veralteter, unveränderter ausgemusterter Sperr-Sidecars |
    | `plugin-sdk/persistent-dedupe` | Hilfsfunktionen für einen datenträgergestützten Deduplizierungs-Cache |
    | `plugin-sdk/ingress-effect-once` | Dauerhafte Anspruchs-/Commit-Sicherung für nicht idempotente Nebeneffekte eingehender Daten |
    | `plugin-sdk/acp-runtime` | Nach Juli 2026 nur noch privat-lokal; ACP-Runtime-/Sitzungs- und Antwortweiterleitungs-Hilfsfunktionen |
    | `plugin-sdk/acp-runtime-backend` | Nach Juli 2026 nur noch privat-lokal; schlanke ACP-Backend-Registrierungs- und Antwortweiterleitungs-Hilfsfunktionen für beim Start geladene Plugins |
    | `plugin-sdk/acp-binding-resolve-runtime` | Nach Juli 2026 nur noch privat-lokal; schreibgeschützte Auflösung von ACP-Bindungen ohne Importe für den Lebenszyklusstart |
    | `plugin-sdk/agent-config-primitives` | Veraltete Primitive für Agenten-Runtime-Konfigurationsschemas; importieren Sie Schema-Primitive aus einer gepflegten Plugin-eigenen Oberfläche |
    | `plugin-sdk/boolean-param` | Lockerer Leser für boolesche Parameter |
    | `plugin-sdk/dangerous-name-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Auflösung von Übereinstimmungen gefährlicher Namen |
    | `plugin-sdk/device-bootstrap` | Hilfsfunktionen für Geräte-Bootstrap und Kopplungstoken, einschließlich `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` |
    | `plugin-sdk/extension-shared` | Gemeinsam genutzte Hilfsprimitive für passive Kanäle, Status und Umgebungs-Proxys |
    | `plugin-sdk/models-provider-runtime` | Hilfsfunktionen für `/models`-Befehls-/Provider-Antworten |
    | `plugin-sdk/skill-commands-runtime` | Hilfsfunktionen zum Auflisten von Skill-Befehlen |
    | `plugin-sdk/native-command-registry` | Hilfsfunktionen für Registrierung, Erstellung und Serialisierung nativer Befehle |
    | `plugin-sdk/agent-harness` | Experimentelle Oberfläche für vertrauenswürdige Plugins und Low-Level-Agenten-Harnesses: Harness-Typen, Hilfsfunktionen zum Steuern/Abbrechen aktiver Ausführungen, OpenClaw-Werkzeugbrücken-Hilfsfunktionen, Hilfsfunktionen für Runtime-Plan-Werkzeugrichtlinien, Klassifizierung terminaler Ergebnisse, Hilfsfunktionen für Formatierung/Details des Werkzeugfortschritts und Dienstprogramme für Versuchsergebnisse |
    | `plugin-sdk/async-lock-runtime` | Nach Juli 2026 nur noch privat-lokal; prozesslokale asynchrone Sperr-Hilfsfunktion für kleine Runtime-Zustandsdateien |
    | `plugin-sdk/channel-activity-runtime` | Nach Juli 2026 nur noch privat-lokal; Telemetrie-Hilfsfunktion für Kanalaktivität |
    | `plugin-sdk/concurrency-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktion für begrenzte Nebenläufigkeit asynchroner Aufgaben |
    | `plugin-sdk/dedupe-runtime` | Hilfsfunktionen für speicherinterne und persistent gestützte Deduplizierungs-Caches |
    | `plugin-sdk/delivery-queue-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktion zum Leeren ausstehender ausgehender Zustellungen |
    | `plugin-sdk/file-access-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für sichere Pfade lokaler Dateien und Medienquellen |
    | `plugin-sdk/heartbeat-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Heartbeat-Aufwecken, -Ereignisse und -Sichtbarkeit |
    | `plugin-sdk/expect-runtime` | Nach Juli 2026 nur noch privat-lokal; Assertions-Hilfsfunktion für erforderliche Werte bei beweisbaren Runtime-Invarianten |
    | `plugin-sdk/number-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktion für numerische Typumwandlung |
    | `plugin-sdk/secure-random-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für sichere Token/UUIDs |
    | `plugin-sdk/system-event-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für die Systemereigniswarteschlange |
    | `plugin-sdk/transport-ready-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktion zum Warten auf Transportbereitschaft |
    | `plugin-sdk/exec-approvals-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Richtliniendateien zur Ausführungsgenehmigung ohne den breiten Infrastruktur-Runtime-Barrel-Export |
    | `plugin-sdk/infra-runtime` | Veralteter Kompatibilitäts-Shim; verwenden Sie die fokussierten Runtime-Unterpfade oben |
    | `plugin-sdk/collection-runtime` | Kleine begrenzte Cache-Hilfsfunktionen |
    | `plugin-sdk/diagnostic-runtime` | Hilfsfunktionen für Diagnose-Flags, Ereignisse und Trace-Kontexte |
    | `plugin-sdk/error-runtime` | Hilfsfunktionen für Fehlergraphen, Formatierung, Typumwandlung unbekannter Werte und gemeinsame Fehlerklassifizierung, `PlatformMessageNotDispatchedError`, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für umschlossenes Fetch, Proxy, EnvHttpProxyAgent-Optionen und festgelegte Lookups |
    | `plugin-sdk/runtime-fetch` | Nach Juli 2026 nur noch privat-lokal; Dispatcher-fähiges Runtime-Fetch ohne Proxy-/Guarded-Fetch-Importe |
    | `plugin-sdk/inline-image-data-url-runtime` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Bereinigung von Inline-Bilddaten-URLs und Signaturerkennung ohne die breite Medien-Runtime-Oberfläche |
    | `plugin-sdk/response-limit-runtime` | Nach Juli 2026 nur noch privat-lokal; nach Byteanzahl, Leerlauf und Frist begrenzte Leser für Antwortinhalte ohne die breite Medien-Runtime-Oberfläche |
    | `plugin-sdk/session-binding-runtime` | Nach Juli 2026 nur noch privat-lokal; aktueller Zustand der Konversationsbindung ohne konfigurierte Bindungsweiterleitung oder Kopplungsspeicher |
    | `plugin-sdk/context-visibility-runtime` | Nach Juli 2026 nur noch privat-lokal; Auflösung der Kontextsichtbarkeit und Filterung ergänzender Kontexte ohne breite Konfigurations-/Sicherheitsimporte |
    | `plugin-sdk/string-coerce-runtime` | Eng gefasste primitive Hilfsfunktionen für Typumwandlung und Normalisierung von Datensätzen/Zeichenfolgen ohne Markdown-/Protokollierungsimporte |
    | `plugin-sdk/html-entity-runtime` | Nach Juli 2026 nur noch privat-lokal; einmalige Dekodierung durch Semikolon abgeschlossener HTML5-Entitäten ohne breite Textdienstprogramme |
    | `plugin-sdk/text-utility-runtime` | Seit Juli 2026 nur noch privat-lokal; grundlegende Text- und Pfad-Hilfsfunktionen, einschließlich HTML-Escaping für fünf Entitäten |
    | `plugin-sdk/widget-html` | Erkennung vollständiger Dokumente, Größenvalidierung und Tool-Eingabefehler für eigenständige HTML-Widgets |
    | `plugin-sdk/host-runtime` | Seit Juli 2026 nur noch privat-lokal; Hilfsfunktionen zur Normalisierung von Hostnamen und SCP-Hosts |
    | `plugin-sdk/retry-runtime` | Seit Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Wiederholungskonfiguration und Wiederholungsausführung |
    | `plugin-sdk/agent-runtime` | Veralteter, breit gefasster Barrel-Export für Hilfsfunktionen zu Agentenverzeichnis, -identität und -Arbeitsbereich, einschließlich `resolveAgentDir`, `resolveDefaultAgentDir` und des veralteten Kompatibilitätsexports `resolveOpenClawAgentDir`; bevorzugen Sie gezielte Agenten-/Runtime-Unterpfade |
    | `plugin-sdk/directory-runtime` | Konfigurationsgestützte Verzeichnisabfrage und Deduplizierung |
    | `plugin-sdk/keyed-async-queue` | Seit Juli 2026 nur noch privat-lokal; `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Unterpfade für Fähigkeiten und Tests">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Veraltetes umfassendes Medien-Barrel einschließlich `saveRemoteMedia`, `saveResponseMedia`, `readRemoteMediaBuffer` und des veralteten `fetchRemoteMedia`; bevorzugen Sie `plugin-sdk/media-store`, `plugin-sdk/media-mime`, `plugin-sdk/outbound-media` und Unterpfade der Fähigkeiten-Runtime sowie Store-Hilfsfunktionen vor Pufferlesevorgängen, wenn eine URL in OpenClaw-Medien umgewandelt werden soll |
    | `plugin-sdk/media-local-roots` | Fokussierte `getAgentScopedMediaLocalRoots(...)`- und richtlinienbewusste `getAgentScopedMediaLocalRootsForSources(...)`-Hilfsfunktionen für Plugin-eigene Lesevorgänge lokaler Medien |
    | `plugin-sdk/media-mime` | Eng begrenzte MIME-Normalisierung, Dateierweiterungszuordnung, MIME-Erkennung und Hilfsfunktionen für Medienarten |
    | `plugin-sdk/media-store` | Eng begrenzte Medienspeicher-Hilfsfunktionen wie `saveMediaBuffer` und `saveMediaStream` |
    | `plugin-sdk/media-generation-runtime` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Failover-Hilfsfunktionen für die Mediengenerierung, Kandidatenauswahl und Meldungen zu fehlenden Modellen |
    | `plugin-sdk/media-understanding` | Veraltete Kompatibilitätsfassade für Provider-Typen und Hilfsfunktionen zum Medienverständnis; neue Provider registrieren sich über die injizierte Plugin-API und behalten Anfrage-Hilfsfunktionen im Besitz des Plugins |
    | `plugin-sdk/text-chunking` | Ausgehender Text und bereichsweise Segmentierung unter Beibehaltung von Offsets, Markdown-Segmentierung und Rendering-Hilfsfunktionen, anführungszeichenbewusste Tokenisierung von HTML-Tags, Konvertierung von Markdown-Tabellen, Entfernung von Direktiven-Tags und Hilfsfunktionen für sicheren Text |
    | `plugin-sdk/speech` | Nach Juli 2026 nur noch privat-lokal; Typen für Sprach-Provider sowie Provider-seitige Direktiven-, Registry-, Validierungs-, OpenAI-kompatible TTS-Builder- und Sprach-Hilfsexporte |
    | `plugin-sdk/speech-core` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Typen für Sprach-Provider sowie Registry-, Direktiven-, Normalisierungs- und Sprach-Hilfsexporte |
    | `plugin-sdk/speech-settings` | Leichtgewichtige Primitive zur Auflösung und Normalisierung der TTS-Konfiguration ohne Provider-Registrys oder Synthese-Runtime |
    | `plugin-sdk/realtime-transcription` | Nach Juli 2026 nur noch privat-lokal; Typen für Echtzeit-Transkriptions-Provider, Registry-Hilfsfunktionen und gemeinsame WebSocket-Sitzungs-Hilfsfunktion |
    | `plugin-sdk/realtime-bootstrap-context` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktion zum Initialisieren von Echtzeitprofilen für die begrenzte Kontextinjektion von `IDENTITY.md`, `USER.md` und `SOUL.md` |
    | `plugin-sdk/realtime-voice` | Nach Juli 2026 nur noch privat-lokal; Typen für Echtzeit-Sprach-Provider, Registry-Hilfsfunktionen, gemeinsame Gates für Audioenergie und Sprachbeginn sowie Hilfsfunktionen für das Echtzeit-Sprachverhalten, einschließlich des transportunabhängigen Sitzungsharnesses und der Nachverfolgung der Ausgabeaktivität |
    | `plugin-sdk/meeting-runtime` | Sitzungs-Runtime für Browser-Meetings, Echtzeit-Audio-Engines und -Transporte, `MeetingPlatformAdapter`, Browser-/Node-Steuerung, Agentenkonsultation, Delegierung von Sprachanrufen, Einrichtungsprüfungen und SoX-Befehlshilfen |
    | `plugin-sdk/image-generation` | Nach Juli 2026 nur noch privat-lokal; Typen für Bildgenerierungs-Provider sowie Hilfsfunktionen für Bild-Assets und Daten-URLs und der OpenAI-kompatible Bild-Provider-Builder |
    | `plugin-sdk/image-generation-core` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Typen sowie Failover-, Authentifizierungs- und Registry-Hilfsfunktionen für die Bildgenerierung |
    | `plugin-sdk/music-generation` | Nach Juli 2026 nur noch privat-lokal; Provider-, Anfrage- und Ergebnistypen für die Musikgenerierung |
    | `plugin-sdk/video-generation` | Nach Juli 2026 nur noch privat-lokal; Provider-, Anfrage- und Ergebnistypen für die Videogenerierung |
    | `plugin-sdk/video-generation-core` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Typen für die Videogenerierung, Failover-Hilfsfunktionen, Provider-Suche und Analyse von Modellreferenzen |
    | `plugin-sdk/transcripts` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Provider-Typen für Transkriptquellen, Registry-Hilfsfunktionen, Bridge-Factory für Meeting-Provider, Sitzungsdeskriptoren und Äußerungsmetadaten |
    | `plugin-sdk/webhook-targets` | Nach Juli 2026 nur noch privat-lokal; Registry für Webhook-Ziele und Hilfsfunktionen zur Routeninstallation |
    | `plugin-sdk/web-media` | Gemeinsame Hilfsfunktionen zum Laden entfernter/lokaler Medien |
    | `plugin-sdk/zod` | Veralteter Kompatibilitäts-Reexport; importieren Sie `zod` direkt aus `zod` |
    | `plugin-sdk/plugin-test-api` | Repo-lokale minimale `createTestPluginApi`-Hilfsfunktion für Unit-Tests der direkten Plugin-Registrierung, ohne Bridges zu Repo-Testhilfsfunktionen zu importieren |
    | `plugin-sdk/agent-runtime-test-contracts` | Repo-lokale Fixtures für Verträge nativer Agenten-Runtime-Adapter für Authentifizierungs-, Zustellungs-, Fallback-, Tool-Hook-, Prompt-Overlay-, Schema- und Transkriptionsprojektionstests |
    | `plugin-sdk/channel-test-helpers` | Repo-lokale kanalorientierte Testhilfsfunktionen für generische Aktions-, Einrichtungs- und Statusverträge, Verzeichnisassertionen, den Startlebenszyklus von Konten, die Weitergabe der Sendekonfiguration, Runtime-Mocks, Statusprobleme, ausgehende Zustellung und Hook-Registrierung |
    | `plugin-sdk/channel-target-testing` | Repo-lokale gemeinsame Suite mit Fehlerfällen der Zielauflösung für Kanaltests |
    | `plugin-sdk/channel-contract-testing` | Repo-lokale eng begrenzte Testhilfsfunktionen für Kanalverträge ohne das umfassende Test-Barrel |
    | `plugin-sdk/plugin-test-contracts` | Repo-lokale Hilfsfunktionen für Verträge zu Plugin-Paketen, Registrierung, öffentlichen Artefakten, direkten Importen, der Runtime-API und Import-Nebeneffekten |
    | `plugin-sdk/plugin-state-test-runtime` | Repo-lokale Testhilfsfunktionen für Plugin-Zustandsspeicher, Eingangs-Warteschlangen und Zustandsdatenbanken |
    | `plugin-sdk/provider-test-contracts` | Repo-lokale Hilfsfunktionen für Verträge zu Provider-Runtime, Authentifizierung, Erkennung, Onboarding, Katalog, Assistenten, Medienfähigkeiten, Wiederholungsrichtlinien, Echtzeit-STT-Live-Audio, Websuche/-abruf und Streams |
    | `plugin-sdk/provider-http-test-mocks` | Nach Juli 2026 nur noch privat-lokal; repo-lokale optionale Vitest-HTTP-/Authentifizierungs-Mocks für Provider-Tests, die `plugin-sdk/provider-http` ausführen |
    | `plugin-sdk/reply-payload-testing` | Repo-lokale Hilfsfunktionen zum Anhängen von Metadaten an Fixtures für Antwort-Payloads |
    | `plugin-sdk/sqlite-runtime-testing` | Repo-lokale SQLite-Lebenszyklus-Hilfsfunktionen für Erstanbieter-Tests |
    | `plugin-sdk/test-fixtures` | Repo-lokale generische Fixtures für CLI-Runtime-Erfassung, Sandbox-Kontext, Skill-Writer, Agentennachrichten, Systemereignisse, Modulneuladen, gebündelte Plugin-Pfade, Terminaltext, Segmentierung, Authentifizierungs-Token und typisierte Fälle |
    | `plugin-sdk/test-node-mocks` | Repo-lokale fokussierte Mock-Hilfsfunktionen für integrierte Node-Module zur Verwendung innerhalb von Vitest-`vi.mock("node:*")`-Factories |
  </Accordion>

  <Accordion title="Speicher-Unterpfade">
    | Unterpfad | Wichtige Exporte |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | Nach Juli 2026 nur noch privat-lokal; leichtgewichtige Registry-Hilfsfunktionen für Provider von Speicher-Embeddings |
    | `plugin-sdk/memory-core-host-engine-foundation` | Engine-Exporte der Speicher-Host-Grundlage |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Nach Juli 2026 nur noch privat-lokal; Embedding-Verträge des Speicher-Hosts, Registry-Zugriff, lokaler Provider und generische Batch-/Remote-Hilfsfunktionen. `registerMemoryEmbeddingProvider` auf dieser Oberfläche ist veraltet; verwenden Sie für neue Provider die generische Embedding-Provider-API. |
    | `plugin-sdk/memory-core-host-engine-qmd` | Nach Juli 2026 nur noch privat-lokal; Exporte der QMD-Engine des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-engine-storage` | Nach Juli 2026 nur noch privat-lokal; Exporte der Speicher-Engine des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-secret` | Nach Juli 2026 nur noch privat-lokal; Hilfsfunktionen für Geheimnisse des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-status` | Nach Juli 2026 nur noch privat-lokal; Status-Hilfsfunktionen des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-runtime-cli` | Nach Juli 2026 nur noch privat-lokal; CLI-Runtime-Hilfsfunktionen des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-runtime-core` | Nach Juli 2026 nur noch privat-lokal; zentrale Runtime-Hilfsfunktionen des Speicher-Hosts |
    | `plugin-sdk/memory-core-host-runtime-files` | Nach Juli 2026 nur noch privat-lokal; Datei-/Runtime-Hilfsfunktionen des Speicher-Hosts |
    | `plugin-sdk/memory-host-core` | Veraltete Kompatibilitätsfassade für herstellerneutrale Hilfsfunktionen des Speicher-Hosts. Neue Speicher-Plugins verwenden injizierte Speicherfähigkeiten und vom Host vorbereitete Prompts; begleitende Plugins verwenden die beibehaltene Fassade weiterhin zur Erkennung öffentlicher Artefakte, bis eine fokussierte Leseschnittstelle vorhanden ist. |
    | `plugin-sdk/memory-host-events` | Nach Juli 2026 nur noch privat-lokal; herstellerneutraler Alias für Ereignisjournal-Hilfsfunktionen des Speicher-Hosts |
    | `plugin-sdk/memory-host-markdown` | Nach Juli 2026 nur noch privat-lokal; gemeinsame Hilfsfunktionen für verwaltetes Markdown für speichernahe Plugins |
    | `plugin-sdk/memory-host-search` | Nach Juli 2026 nur noch privat-lokal; Active-Memory-Runtime-Fassade für den Zugriff auf den Suchmanager |
  </Accordion>

  <Accordion title="Reservierte Unterpfade für gebündelte Hilfsfunktionen">
    Reservierte SDK-Unterpfade für gebündelte Hilfsfunktionen sind eng begrenzte, inhaberspezifische Oberflächen für
    gebündelten Plugin-Code. Sie werden im SDK-Inventar erfasst, damit Paket-
    Builds und Aliasing deterministisch bleiben, sind jedoch keine allgemeinen APIs
    für die Plugin-Entwicklung. Neue wiederverwendbare Host-Verträge sollten generische SDK-Unterpfade
    wie `plugin-sdk/gateway-runtime` und `plugin-sdk/ssrf-runtime` verwenden.

    | Unterpfad | Inhaber und Zweck |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | Nach Juli 2026 nur noch privat-lokal; gebündelte Hilfsfunktion des Codex-Plugins zur Projektion der MCP-Serverkonfiguration des Benutzers in die Thread-Konfiguration des Codex-App-Servers (reservierter Paketexport) |
    | `plugin-sdk/codex-native-task-runtime` | Gebündelte Hilfsfunktion des Codex-Plugins zum Spiegeln nativer Subagenten des Codex-App-Servers in den OpenClaw-Aufgabenstatus (nur repo-lokal, kein Paketexport) |

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [Übersicht über das Plugin SDK](/de/plugins/sdk-overview)
- [Einrichtung des Plugin SDK](/de/plugins/sdk-setup)
- [Plugins erstellen](/de/plugins/building-plugins)
