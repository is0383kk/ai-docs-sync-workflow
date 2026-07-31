---
read_when:
    - Het juiste plugin-sdk-subpad kiezen voor een Plugin-import
    - Subpaden en helperinterfaces van gebundelde plugins controleren
summary: 'Plugin-SDK-subpadcatalogus: welke imports waar staan, gegroepeerd per gebied'
title: Subpaden van de Plugin SDK
x-i18n:
    generated_at: "2026-07-27T06:04:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

De Plugin SDK bevat beperkte openbare subpaden en uitsluitend voor de repository bestemde gebundelde
helpers onder `openclaw/plugin-sdk/`. Deze pagina inventariseert beide en markeert
privé-lokale vermeldingen expliciet. Drie bestanden definiëren de grens:

- `scripts/lib/plugin-sdk-entrypoints.json`: de onderhouden inventaris van entrypoints
  die door de build wordt gecompileerd.
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`: interne subpaden
  die zijn uitgesloten van de getypeerde, gedocumenteerde SDK. Productie-entrypoints blijven beschikbaar
  als uitsluitend JavaScript-hostruntime-exports voor afzonderlijk gepubliceerde officiële
  plugins; entrypoints die alleen voor tests zijn bedoeld, blijven ongeëxporteerd.
- `src/plugin-sdk/entrypoints.ts`: classificatiemetadata voor verouderde
  subpaden, gereserveerde gebundelde helpers, ondersteunde gebundelde façades en
  openbare oppervlakken die eigendom zijn van plugins.

Onderhouders controleren het aantal openbare exports met `pnpm plugin-sdk:surface` en
actieve gereserveerde helpersubpaden met `pnpm plugins:boundary-report:summary`;
ongebruikte gereserveerde helperexports laten het CI-rapport mislukken in plaats van als
slapende compatibiliteitsschuld in de openbare SDK te blijven.

Zie voor de handleiding voor het schrijven van plugins het [overzicht van de Plugin SDK](/nl/plugins/sdk-overview).

## Plugin-entrypoint

| Subpad                        | Belangrijkste exports                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | Privé-lokaal na juli 2026; `defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | Privé-lokaal na juli 2026; helpers voor migratieprovideritems, zoals `createMigrationItem`, redenconstanten, itemstatusmarkeringen, redactieroutines en `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | Privé-lokaal na juli 2026; runtimemigratiehelpers, zoals `copyMigrationFileItem`, `resolvePlannedMigrationTargets`, `withCachedMigrationConfigRuntime` en `writeMigrationReport`              |
| `plugin-sdk/health`            | Registratie, detectie, reparatie, selectie, ernst- en bevindingstypen voor Doctor-gezondheidscontroles voor gebundelde gezondheidsconsumenten                                                                                |

### Compatibiliteits- en privé-lokale helpers

Alleen de verouderde subpaden uit het latere tijdvenster blijven geëxporteerd. Aliassen uit juli 2026 en
ongebruikte subpaden zijn verwijderd, terwijl helpers die uitsluitend gebundeld zijn uit het
openbare pakket zijn verwijderd en hieronder als privé-lokaal zijn gemarkeerd. De onderhouden lijst is
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; CI weigert gebundelde
`plugin-sdk/text-runtime` zijn uitsluitend voor compatibiliteit en `plugin-sdk/zod` is een
compatibiliteitsherexport: importeer `zod` rechtstreeks uit `zod`. De brede domein-
barrels `plugin-sdk/agent-runtime`, `plugin-sdk/channel-lifecycle`,
`plugin-sdk/conversation-runtime`, `plugin-sdk/hook-runtime`,
`plugin-sdk/media-runtime`, `plugin-sdk/plugin-runtime` en
`plugin-sdk/security-runtime` zijn eveneens verouderd ten gunste van gerichte
subpaden.

De op Vitest gebaseerde testhelpersubpaden van OpenClaw zijn uitsluitend repository-lokaal en zijn niet
langer pakketexports: `agent-runtime-test-contracts`,
`channel-contract-testing`, `channel-target-testing`, `channel-test-helpers`,
`plugin-state-test-runtime`, `plugin-test-api`, `plugin-test-contracts`,
`plugin-test-runtime`, `provider-http-test-mocks`, `provider-test-contracts`,
`reply-payload-testing`, `sqlite-runtime-testing`, `test-env`, `test-fixtures`,
`test-live`, `test-live-auth`, `test-media-generation`,
`test-media-understanding`, `test-node-mocks` en `testing`. De privé-oppervlakken voor gebundelde helpers
`ssrf-runtime-internal` en `codex-native-task-runtime` zijn eveneens uitsluitend repository-
lokaal.

### Helpersubpaden voor gebundelde plugins

Helpermodules die uitsluitend gebundeld zijn, zijn na de opschoning van juli 2026 privé-lokaal. Imports tussen verschillende eigenaren worden geblokkeerd door beschermingsregels voor pakketcontracten. `src/plugin-sdk/entrypoints.ts` houdt afzonderlijk de ondersteunde gebundelde façades bij die openbaar blijven: SDK-
entrypoints die door hun gebundelde plugin worden ondersteund totdat generieke contracten
`plugin-sdk/qa-runner-runtime`, `plugin-sdk/telegram-account` vervangen,
verouderd voor nieuwe code; zie de opmerkingen per rij hieronder.

<AccordionGroup>
  <Accordion title="Kanaalsubpaden">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | Privé-lokaal na juli 2026; gecachte helper voor JSON Schema-validatie van schema's die eigendom zijn van plugins |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`, kanaaleigen typen voor installatievelden/-invoer, `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, plus `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Gedeelde helpers voor de installatiewizard, installatievertaler, prompts voor toelatingslijsten, builders voor installatiestatussen |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`, `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Helpers voor configuratie/actiepoorten voor meerdere accounts, terugvalhelpers voor het standaardaccount |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, helpers voor normalisatie van account-id's |
    | `plugin-sdk/account-resolution` | Helpers voor het opzoeken van accounts en terugval naar de standaardwaarde |
    | `plugin-sdk/account-helpers` | Beperkte helpers voor accountlijsten/accountacties |
    | `plugin-sdk/access-groups` | Privé-lokaal na juli 2026; parsing van toelatingslijsten voor toegangsgroepen en helpers voor geredigeerde groepsdiagnostiek |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | Verouderde compatibiliteitsfaçade. Gebruik `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`, `resolveChannelDmAccess`, `resolveChannelDmAllowFrom`, `resolveChannelDmPolicy`, `normalizeChannelDmPolicy`, `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | Gedeelde primitieven voor kanaalconfiguratieschema's, plus Zod- en rechtstreekse JSON-/TypeBox-builders |
    | `plugin-sdk/bundled-channel-config-schema` | Privé-lokaal na juli 2026; gebundelde OpenClaw-kanaalconfiguratieschema's uitsluitend voor onderhouden gebundelde plugins |
    | `plugin-sdk/chat-channel-ids` | Privé-lokaal na juli 2026; `BUNDLED_CHAT_CHANNEL_IDS`, `BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`, `ChatChannelId`. Canonieke kanaal-id's voor gebundelde/officiële chatkanalen, plus formatterlabels/-aliassen voor plugins die tekst met een envelopvoorvoegsel moeten herkennen zonder hun eigen tabel hard te coderen. |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | Experimentele high-level runtime-resolver voor inkomend kanaalverkeer, resolver voor impliciet-vermeldingsbeleid en builders voor routefeiten voor gemigreerde ontvangstpaden van kanalen. Geef hieraan de voorkeur boven het in elke plugin samenstellen van effectieve toelatingslijsten, toelatingslijsten voor opdrachten en verouderde projecties. Zie [API voor inkomend kanaalverkeer](/nl/plugins/sdk-channel-ingress). |
    | `plugin-sdk/channel-lifecycle` | Verouderde compatibiliteitsfaçade. Gebruik `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-outbound` | Contracten voor de levenscyclus van berichten, plus opties voor de antwoordpijplijn, ontvangstbewijzen, livevoorbeelden/streaming, levenscyclushelpers, uitgaande identiteit, payloadplanning, duurzame verzendingen en helpers voor de context van berichtverzending. Zie [API voor uitgaande kanaalberichten](/nl/plugins/sdk-channel-outbound). |
    | `plugin-sdk/channel-message` | Verouderde compatibiliteitsalias voor `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/inbound-envelope` | Gedeelde helpers voor inkomende routes en envelopbuilders |
    | `plugin-sdk/inbound-reply-dispatch` | Verouderde compatibiliteitsfaçade. Gebruik `plugin-sdk/channel-inbound` voor inkomende runners en dispatchpredicaten, en `plugin-sdk/channel-outbound` voor helpers voor berichtbezorging. |
    | `plugin-sdk/messaging-targets` | Verouderde alias voor doelparsing; gebruik `plugin-sdk/channel-targets` |
    | `plugin-sdk/outbound-media` | Privé-lokaal na juli 2026; gedeelde helpers voor het laden van uitgaande media en statushelpers voor gehoste media |
    | `plugin-sdk/poll-runtime` | Privé-lokaal na juli 2026; beperkte helpers voor peilingnormalisatie |
    | `plugin-sdk/thread-bindings-runtime` | Privé-lokaal na juli 2026; levenscyclus- en adapterhelpers voor threadbinding |
    | `plugin-sdk/agent-media-payload` | Verouderde compatibiliteitsfaçade voor verouderde `Media*`-payloadprojectie. Geef geordende feiten door via `MsgContext.media` / `toInboundMediaFacts(...)`; importeer lokaal-rootbeleid uit `plugin-sdk/media-local-roots`. |
    | `plugin-sdk/conversation-runtime` | Verouderde brede barrel voor gespreks-/threadbinding, koppeling en helpers voor geconfigureerde binding; geef de voorkeur aan gerichte bindingsubpaden, zoals `plugin-sdk/thread-bindings-runtime` en `plugin-sdk/session-binding-runtime` |
    | `plugin-sdk/runtime-group-policy` | Helpers voor runtime-resolutie van groepsbeleid |
    | `plugin-sdk/channel-status` | Gedeelde helpers voor momentopnamen/samenvattingen van kanaalstatussen |
    | `plugin-sdk/channel-config-primitives` | Beperkte primitieven voor kanaalconfiguratieschema's |
    | `plugin-sdk/channel-config-writes` | Privé-lokaal na juli 2026; autorisatiehelpers voor het schrijven van kanaalconfiguratie |
    | `plugin-sdk/channel-plugin-common` | Gedeelde prelude-exports voor kanaalplugins |
    | `plugin-sdk/allowlist-config-edit` | Helpers voor het bewerken/lezen van toelatingslijstconfiguratie |
    | `plugin-sdk/group-access` | Verouderde helpers voor beslissingen over groepstoegang; gebruik `resolveChannelMessageIngress` uit `plugin-sdk/channel-ingress-runtime` |
    | `plugin-sdk/direct-dm-guard-policy` | Privé-lokaal na juli 2026; beperkte beleidshelpers voor de pre-cryptobescherming van rechtstreekse DM's |
    | `plugin-sdk/discord` | Verouderde Discord-compatibiliteitsfaçade voor gepubliceerde `@openclaw/discord@2026.3.13` en bijgehouden eigenaarscompatibiliteit; nieuwe plugins moeten generieke subpaden van de kanaal-SDK gebruiken |
    | `plugin-sdk/telegram-account` | Verouderde Telegram-compatibiliteitsfaçade voor accountresolutie en bijgehouden eigenaarscompatibiliteit; nieuwe plugins moeten geïnjecteerde runtimehelpers of generieke subpaden van de kanaal-SDK gebruiken |
    | `plugin-sdk/interactive-runtime` | Semantische berichtpresentatie, bezorging en verouderde helpers voor interactieve antwoorden. Zie [Berichtpresentatie](/nl/plugins/message-presentation) |
    | `plugin-sdk/question-gateway-runtime` | Los door de runtime geschreven `ask_user`-keuzes via de Gateway op vanuit handlers voor kanaalinteracties |
    | `plugin-sdk/channel-inbound` | Gedeelde inkomende helpers voor gebeurtenisclassificatie, contextopbouw, opmaak, roots, debounce, overeenkomsten met vermeldingen, vermeldingsbeleid en inkomende logboekregistratie |
    | `plugin-sdk/channel-inbound-debounce` | Beperkte inkomende debouncehelpers |
    | `plugin-sdk/channel-mention-gating` | Privé-lokaal na juli 2026; beperkte helpers voor vermeldingsbeleid, vermeldingsmarkeringen en vermeldingstekst zonder het bredere inkomende runtimeoppervlak |
    | `plugin-sdk/channel-streaming` | Verouderde compatibiliteitsfaçade. Gebruik `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/channel-send-result` | Typen voor antwoordresultaten |
    | `plugin-sdk/channel-actions` | Helpers voor kanaalberichtacties, plus verouderde systeemeigen schemahelpers die voor plugincompatibiliteit behouden blijven |
    | `plugin-sdk/channel-route` | Privé-lokaal na juli 2026; gedeelde routenormalisatie, parsergestuurde doelresolutie, omzetting van thread-id's naar tekenreeksen, ontdubbelde/compacte routesleutels, typen voor geparste doelen en helpers voor route-/doelvergelijking |
    | `plugin-sdk/channel-targets` | Privé-lokaal na juli 2026; helpers voor doelparsing; aanroepers voor routevergelijking moeten `plugin-sdk/channel-route` gebruiken |
    | `plugin-sdk/channel-contract` | Typen voor kanaalcontracten |
    | `plugin-sdk/channel-feedback` | Aansluiting voor feedback/reacties |
  </Accordion>

Kanaalcompatibiliteitssubpaden uit het latere tijdvenster blijven alleen openbaar tot hun
registerdatums. Juli-aliassen, zoals toegang tot rechtstreekse DM's, antwoordopties, koppelings-
paden en afgesplitste kanaalruntimes, zijn verwijderd; helpers die uitsluitend gebundeld zijn,
zijn privé-lokaal.

  <Accordion title="Providersubpaden">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | Privé-lokaal na juli 2026; `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Privé-lokaal na juli 2026; samengestelde helpers voor het instellen van lokale/zelfgehoste providers |
    | `plugin-sdk/cli-backend` | Privé-lokaal na juli 2026; standaardwaarden voor de CLI-backend + watchdog-constanten |
    | `plugin-sdk/provider-auth-runtime` | Privé-lokaal na juli 2026; runtimehelpers voor providerauthenticatie: OAuth-loopbackflow, tokenuitwisseling, opslag van authenticatie en resolutie van API-sleutels |
    | `plugin-sdk/provider-oauth-runtime` | Privé-lokaal na juli 2026; generieke OAuth-callbacktypen voor providers, rendering van callbackpagina's, PKCE-/state-helpers, parsing van autorisatie-invoer, helpers voor tokenverloop en afbreekhelpers |
    | `plugin-sdk/provider-auth-api-key` | Privé-lokaal na juli 2026; helpers voor onboarding met API-sleutels en het schrijven van profielen, zoals `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | Privé-lokaal na juli 2026; standaardbuilder voor OAuth-authenticatieresultaten |
    | `plugin-sdk/provider-env-vars` | Privé-lokaal na juli 2026; helpers voor het opzoeken van providerauthenticatie via omgevingsvariabelen |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials`, helpers voor het importeren van OpenAI Codex-authenticatie, verouderde compatibiliteitsexport `resolveOpenClawAgentDir` |
    | `plugin-sdk/provider-model-shared` | Privé-lokaal na juli 2026; `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `selectPreferredLocalModelId`, `normalizeModelCompat`, gedeelde builders voor replaybeleid, helpers voor providereindpunten en gedeelde helpers voor normalisatie van model-id's |
    | `plugin-sdk/provider-catalog-live-runtime` | Privé-lokaal na juli 2026; helpers voor live providermodelcatalogi voor beveiligde detectie in `/models`-stijl: `buildLiveModelProviderConfig`, `fetchLiveProviderModelRows`, `getCachedLiveProviderModelRows`, `fetchLiveProviderModelIds`, `LiveModelCatalogHttpError`, `clearLiveCatalogCacheForTests`, filtering van model-id's, TTL-cache en statische fallback |
    | `plugin-sdk/provider-catalog-runtime` | Runtimehook voor uitbreiding van de providercatalogus en registerkoppelingen voor pluginproviders voor contracttests |
    | `plugin-sdk/provider-catalog-shared` | Privé-lokaal na juli 2026; `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `buildManifestModelProviderConfig`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Privé-lokaal na juli 2026; generieke helpers voor HTTP-/eindpuntmogelijkheden van providers, HTTP-fouten van providers en multipart-formulierhelpers voor audiotranscriptie |
    | `plugin-sdk/provider-web-fetch-contract` | Privé-lokaal na juli 2026; beperkte contracthelpers voor web-fetchconfiguratie/-selectie, zoals `enablePluginInConfig` en `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | Privé-lokaal na juli 2026; helpers voor registratie/caching van web-fetchproviders |
    | `plugin-sdk/provider-web-search-config-contract` | Privé-lokaal na juli 2026; beperkte configuratie-/referentiehelpers voor webzoekproviders waarvoor geen koppeling voor het inschakelen van plugins nodig is |
    | `plugin-sdk/provider-web-search-contract` | Privé-lokaal na juli 2026; beperkte contracthelpers voor webzoekconfiguratie/-referenties, zoals `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, en scoped setters/getters voor referenties |
    | `plugin-sdk/provider-web-search` | Privé-lokaal na juli 2026; runtimehelpers voor registratie/caching van webzoekproviders |
    | `plugin-sdk/embedding-providers` | Privé-lokaal na juli 2026; algemene typen en leeshelpers voor embeddingproviders, waaronder `EmbeddingProviderAdapter`, `getEmbeddingProvider(...)` en `listEmbeddingProviders(...)`; plugins registreren providers via `api.registerEmbeddingProvider(...)`, zodat eigenaarschap van het manifest wordt afgedwongen |
    | `plugin-sdk/provider-tools` | Privé-lokaal na juli 2026; `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks` en opschoning + diagnostiek van DeepSeek-/Gemini-/OpenAI-schema's |
    | `plugin-sdk/provider-usage` | Privé-lokaal na juli 2026; snapshottypen voor providergebruik, gedeelde helpers voor het ophalen van gebruik en providerfetchers zoals `fetchClaudeUsage` |
    | `plugin-sdk/provider-stream` | Privé-lokaal na juli 2026; `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, typen voor streamwrappers, compatibiliteit voor toolaanroepen in platte tekst en gedeelde wrapperhelpers voor Anthropic/Google/Kilocode/MiniMax/Moonshot/OpenAI/OpenRouter/Z.AI |
    | `plugin-sdk/provider-stream-shared` | Privé-lokaal na juli 2026; openbare gedeelde wrapperhelpers voor providerstreams, waaronder `composeProviderStreamWrappers`, `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPlainTextToolCallCompatWrapper`, `createPayloadPatchStreamWrapper`, `createToolStreamWrapper`, `normalizeOpenAICompatibleReasoningPayload`, `setQwenChatTemplateThinking` en streamhulpprogramma's die compatibel zijn met Anthropic/DeepSeek/OpenAI |
    | `plugin-sdk/provider-transport-runtime` | Privé-lokaal na juli 2026; native providertransporthelpers, zoals beveiligd ophalen, tekstextractie uit toolresultaten, transformaties van transportberichten en beschrijfbare transportgebeurtenisstreams |
    | `plugin-sdk/provider-onboard` | Privé-lokaal na juli 2026; helpers voor configuratiepatches tijdens onboarding |
    | `plugin-sdk/global-singleton` | Privé-lokaal na juli 2026; proceslokale singleton-/map-/cachehelpers |
    | `plugin-sdk/group-activation` | Privé-lokaal na juli 2026; beperkte helpers voor groepsactiveringsmodi en opdrachtparsing |
  </Accordion>

Snapshots van providergebruik rapporteren normaal gesproken een of meer quota-`windows`, elk met
een label, gebruikt percentage en optionele resetdatum. Providers die saldo- of
accountstatustekst tonen in plaats van resetbare quotavensters, moeten
`summary` retourneren met een lege `windows`-array in plaats van percentages te verzinnen.
OpenClaw toont die samenvattingstekst in statusuitvoer; gebruik `error` alleen wanneer het
gebruikseindpunt is mislukt of geen bruikbare gebruiksgegevens heeft geretourneerd.

  <Accordion title="Subpaden voor authenticatie en beveiliging">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | Verouderd breed autorisatieoppervlak voor opdrachten (`resolveControlCommandGate`, helpers voor het opdrachtenregister, waaronder dynamische opmaak van menu's voor argumenten, helpers voor afzenderautorisatie); gebruik autorisatie bij kanaalingang/runtime of helpers voor opdrachtstatus |
    | `plugin-sdk/command-status` | Builders voor opdracht-/helpberichten, zoals `buildCommandsMessagePaginated` en `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | Helpers voor het bepalen van goedkeurders en actieautorisatie binnen dezelfde chat |
    | `plugin-sdk/approval-client-runtime` | Helpers voor profielen/filters voor native uitvoeringsgoedkeuring |
    | `plugin-sdk/approval-delivery-runtime` | Adapters voor mogelijkheden/levering van native goedkeuringen |
    | `plugin-sdk/approval-gateway-runtime` | Gedeelde resolver voor de goedkeuringsgateway |
    | `plugin-sdk/approval-reference-runtime` | Privé-lokaal na juli 2026; deterministische helper voor duurzame locators bij door transport beperkte goedkeuringscallbacks |
    | `plugin-sdk/approval-handler-adapter-runtime` | Lichtgewicht laadhelpers voor native goedkeuringsadapters voor intensief gebruikte kanaalingangspunten |
    | `plugin-sdk/approval-handler-runtime` | Bredere runtimehelpers voor goedkeuringshandlers; geef de voorkeur aan de beperktere adapter-/gatewaykoppelingen wanneer die volstaan |
    | `plugin-sdk/approval-native-runtime` | Helpers voor native goedkeuringsdoelen, accountbinding, routepoorten, doorstuurfallback en onderdrukking van lokale native uitvoeringsprompts |
    | `plugin-sdk/approval-reaction-runtime` | Privé-lokaal na juli 2026; hardgecodeerde bindingen voor goedkeuringsreacties, payloads voor reactieprompts, opslagplaatsen voor reactiedoelen, helpers voor reactiehinttekst en compatibiliteitsexport voor onderdrukking van lokale native uitvoeringsprompts |
    | `plugin-sdk/approval-reply-runtime` | Helpers voor antwoordpayloads voor uitvoerings-/plugingoedkeuringen |
    | `plugin-sdk/approval-runtime` | Payloadhelpers voor uitvoerings-/plugingoedkeuringen, builders voor goedkeuringsmogelijkheden, helpers voor goedkeuringsauthenticatie/-profielen, helpers voor routering/runtime van native goedkeuringen en helpers voor gestructureerde weergave van goedkeuringen, zoals `formatApprovalDisplayPath` |
    | `plugin-sdk/command-auth-native` | Native opdrachtautorisatie, dynamische opmaak van menu's voor argumenten en helpers voor native sessiedoelen |
    | `plugin-sdk/command-detection` | Gedeelde helpers voor opdrachtdetectie |
    | `plugin-sdk/command-primitives-runtime` | Lichtgewicht predicaten voor opdrachttekst voor intensief gebruikte kanaalpaden |
    | `plugin-sdk/command-surface` | Privé-lokaal na juli 2026; helpers voor normalisatie van opdrachtbody's en opdrachtoppervlakken |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | Privé-lokaal na juli 2026; helpers voor luie aanmeldflows voor providerauthenticatie voor koppeling via apparaatcodes in privékanalen en de Web-UI |
    | `plugin-sdk/channel-secret-runtime` | Verouderd breed oppervlak voor geheimcontracten (`collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, typen voor geheime doelen); geef de voorkeur aan de gerichte subpaden hieronder |
    | `plugin-sdk/channel-secret-basic-runtime` | Beperkte exports voor geheimcontracten en builders voor doelregisters voor niet-TTS-geheimoppervlakken van kanalen/plugins |
    | `plugin-sdk/channel-secret-tts-runtime` | Privé-lokaal na juli 2026; beperkte helpers voor toewijzing van geneste TTS-geheimen in kanalen |
    | `plugin-sdk/secret-ref-runtime` | Beperkte SecretRef-typering, resolutie en opzoeking van paden naar plandoelen voor parsing van geheimcontracten/configuratie |
    | `plugin-sdk/security-runtime` | Verouderde brede barrel voor vertrouwen, DM-poortcontrole, tot de root begrensde bestands-/padhelpers, waaronder uitsluitend aanmakende schrijfbewerkingen, synchrone/asynchrone atomische bestandsvervanging, tijdelijke schrijfbewerkingen naast het doelbestand, fallback voor verplaatsing tussen apparaten, helpers voor privébestandsopslag, beveiliging van bovenliggende symlinkpaden, externe inhoud, redactie van gevoelige tekst, constante-tijdvergelijking van geheimen en helpers voor het verzamelen van geheimen; geef de voorkeur aan gerichte subpaden voor beveiliging/SSRF/geheimen |
    | `plugin-sdk/ssrf-policy` | Helpers voor hosttoestaanlijsten en SSRF-beleid voor privénetwerken |
    | `plugin-sdk/ssrf-dispatcher` | Privé-lokaal na juli 2026; beperkte helpers voor vastgezette dispatchers zonder het brede infrastructuurruntimeoppervlak |
    | `plugin-sdk/ssrf-runtime` | Helpers voor vastgezette dispatchers, door SSRF beveiligd ophalen, SSRF-fouten en SSRF-beleid |
    | `plugin-sdk/secret-input` | Helpers voor parsing van geheiminvoer |
    | `plugin-sdk/webhook-ingress` | Helpers voor Webhook-verzoeken/-doelen en conversie van onbewerkte websockets/body's |
    | `plugin-sdk/webhook-request-guards` | Helpers voor grootte/time-out van aanvraagbody's en `runDetachedWebhookWork` voor getraceerde verwerking na bevestiging |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/runtime` | Helpers voor runtime, logging en back-ups, waarschuwingen voor installatiepaden van plugins en proceshelpers |
    | `plugin-sdk/runtime-env` | Gerichte helpers voor runtime-omgeving, logger, time-out, opnieuw proberen en back-off |
    | `plugin-sdk/browser-config` | Privé en lokaal na juli 2026; ondersteunde facade voor browserconfiguratie voor genormaliseerde profielen/standaardwaarden, het parseren van CDP-URL's en helpers voor browserbesturingsauthenticatie |
    | `plugin-sdk/agent-harness-task-runtime` | Privé en lokaal na juli 2026; generieke helpers voor de taaklevenscyclus en voltooiingslevering voor agents met een harness die een door de host uitgegeven taakbereik gebruiken |
    | `plugin-sdk/codex-mcp-projection` | Privé en lokaal na juli 2026; gereserveerde gebundelde Codex-helper om de configuratie van de MCP-server van de gebruiker te projecteren naar de Codex-threadconfiguratie; niet voor plugins van derden |
    | `plugin-sdk/codex-native-task-runtime` | Repo-lokale gebundelde Codex-helper voor native taakspiegeling/runtimebedrading; geen pakketexport |
    | `plugin-sdk/channel-runtime-context` | Generieke helpers voor registratie en opzoeken van de runtimecontext van kanalen |
    | `plugin-sdk/matrix` | Verouderde Matrix-compatibiliteitsfacade voor oudere kanaalpakketten van derden; nieuwe plugins moeten `plugin-sdk/run-command` rechtstreeks importeren |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Verouderde brede barrel voor helpers voor pluginopdrachten, hooks, HTTP en interactie; geef de voorkeur aan gerichte runtime-subpaden voor plugins |
    | `plugin-sdk/hook-runtime` | Verouderde brede barrel voor helpers voor de Webhook-/interne-hookpijplijn; geef de voorkeur aan gerichte subpaden voor hooks en de pluginruntime |
    | `plugin-sdk/lazy-runtime` | Helpers voor luie runtime-import en -binding, zoals `createLazyRuntimeModule`, `createLazyRuntimeMethod` en `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | Privé en lokaal na juli 2026; procesuitvoeringshelpers |
    | `plugin-sdk/node-host` | Privé en lokaal na juli 2026; helpers voor het oplossen van uitvoerbare bestanden op een Node-host en het hervatten van een PTY |
    | `plugin-sdk/cli-runtime` | Privé en lokaal na juli 2026; verouderde brede barrel voor CLI-opmaak, wachten, versiebeheer, aanroepen met argumenten en helpers voor luie opdrachtgroepen; geef de voorkeur aan gerichte CLI-/runtime-subpaden |
    | `plugin-sdk/qa-runner-runtime` | Privé en lokaal na juli 2026; ondersteunde facade die QA-scenario's voor plugins beschikbaar stelt via het CLI-opdrachtoppervlak |
    | `plugin-sdk/tts-runtime` | Privé en lokaal na juli 2026; ondersteunde facade voor configuratieschema's en runtimehelpers voor tekst-naar-spraak |
    | `plugin-sdk/gateway-method-runtime` | Gereserveerde helper voor verzending van Gateway-methoden voor HTTP-routes van plugins die `contracts.gatewayMethodDispatch: ["authenticated-request"]` declareren |
    | `plugin-sdk/gateway-runtime` | Gateway-client, helper voor het starten van een client wanneer de gebeurtenislus gereed is, Gateway-CLI-RPC, fouten in het Gateway-protocol, oplossing van geadverteerde LAN-hosts en helpers voor het bijwerken van de kanaalstatus |
    | `plugin-sdk/config-contracts` | Gericht configuratieoppervlak met alleen typen voor configuratievormen van plugins, zoals `OpenClawConfig`, en configuratietypen voor kanalen/providers |
    | `plugin-sdk/plugin-config-runtime` | Verouderde compatibiliteitsfacade voor runtimehelpers voor pluginconfiguratie; nieuwe plugins gebruiken `api.pluginConfig` plus gerichte configuratiecontracten, snapshots en mutatiehelpers |
    | `plugin-sdk/config-mutation` | Transactionele helpers voor configuratiemutatie, zoals `mutateConfigFile`, `replaceConfigFile` en `logConfigUpdated` |
    | `plugin-sdk/message-tool-delivery-hints` | Privé en lokaal na juli 2026; gedeelde hintteksten voor leveringsmetadata van berichttools |
    | `plugin-sdk/runtime-config-snapshot` | Helpers voor snapshots van de huidige procesconfiguratie, zoals `getRuntimeConfig`, `getRuntimeConfigSnapshot` en setters voor testsnapshots |
    | `plugin-sdk/text-autolink-runtime` | Privé en lokaal na juli 2026; detectie van automatische koppelingen voor bestandsverwijzingen zonder de brede tekstbarrel |
    | `plugin-sdk/reply-runtime` | Gedeelde runtimehelpers voor inkomende berichten en antwoorden, segmentering, verzending, Heartbeat en antwoordplanner |
    | `plugin-sdk/reply-dispatch-runtime` | Gerichte helpers voor het verzenden/afronden van antwoorden en gesprekslabels |
    | `plugin-sdk/reply-history` | Gedeelde helpers voor antwoordgeschiedenis binnen een kort tijdvenster. Nieuwe code voor berichtbeurten moet `createChannelHistoryWindow` gebruiken; helpers voor maps op lager niveau blijven uitsluitend verouderde compatibiliteitsexports |
    | `plugin-sdk/reply-reference` | Privé en lokaal na juli 2026; `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Gerichte helpers voor het segmenteren van tekst/Markdown |
    | `plugin-sdk/session-store-runtime` | Helpers voor sessieworkflows (`getSessionEntry`, `listSessionEntries`, `patchSessionEntry`, `upsertSessionEntry`), helpers voor herstel/levenscyclus (`deleteSessionEntry`, `cleanupSessionLifecycleArtifacts`, `resolveSessionStoreBackupPaths`), markeringshelpers voor tijdelijke `sessionFile`-waarden, begrensde uitlezingen van recente transcripttekst van gebruiker/assistent op basis van sessie-identiteit, helpers voor het pad van de sessieopslag/sessiesleutel en uitlezingen van het bijwerkingstijdstip, zonder brede imports voor configuratieschrijfacties/-onderhoud |
    | `plugin-sdk/session-transcript-runtime` | Privé en lokaal na juli 2026; transcriptidentiteit, begrensde onbewerkte en zichtbare cursors, bereikgebonden helpers voor doelen/lezen/schrijven, projectie van zichtbare berichtitems, publicatie van updates, schrijfvergrendelingen en sleutels voor treffers in het transcriptgeheugen |
    | `plugin-sdk/sqlite-runtime` | Privé en lokaal na juli 2026; gerichte helpers voor het SQLite-agentschema, paden en transacties voor de interne runtime, zonder besturing van de databaselevenscyclus |
    | `plugin-sdk/cron-store-runtime` | Privé en lokaal na juli 2026; helpers voor het pad, laden en opslaan van de Cron-opslag |
    | `plugin-sdk/state-paths` | Helpers voor paden naar status-/OAuth-mappen |
    | `plugin-sdk/plugin-state-runtime` | Privé en lokaal na juli 2026; plugingebonden contracten voor status met sleutels, BLOB's en coöperatieve SQLite-leases, plus verbindingspragma, geverifieerd WAL-onderhoud en atomaire migratiehelpers voor STRICT-schema's. Lease-callbacks ontvangen een afbreeksignaal en getypeerde fouten maken onderscheid tussen time-out, annulering, verloren eigenaarschap, ongeldige invoer en opslagfouten |
    | `plugin-sdk/routing` | Helpers voor bindingen van routes, sessiesleutels en accounts, zoals `resolveAgentRoute`, `buildAgentSessionKey` en `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | Gedeelde helpers voor statusoverzichten van kanalen/accounts, standaardwaarden voor runtimestatus en metadatahelpers voor problemen |
    | `plugin-sdk/target-resolver-runtime` | Privé en lokaal na juli 2026; gedeelde helpers voor doelresolutie |
    | `plugin-sdk/string-normalization-runtime` | Privé en lokaal na juli 2026; helpers voor normalisatie van slugs/tekenreeksen |
    | `plugin-sdk/request-url` | Privé en lokaal na juli 2026; tekenreeks-URL's extraheren uit fetch-/request-achtige invoer |
    | `plugin-sdk/run-command` | Opdrachtrunner met tijdslimiet en genormaliseerde stdout-/stderr-resultaten |
    | `plugin-sdk/param-readers` | Algemene readers voor tool-/CLI-parameters |
    | `plugin-sdk/tool-plugin` | Een eenvoudige getypeerde agenttool-plugin definiëren en statische metadata beschikbaar stellen voor het genereren van manifests |
    | `plugin-sdk/tool-payload` | Privé en lokaal na juli 2026; genormaliseerde payloads extraheren uit toolresultaatobjecten |
    | `plugin-sdk/tool-send` | Canonieke velden voor verzenddoelen extraheren uit toolargumenten |
    | `plugin-sdk/sandbox` | Privé en lokaal na juli 2026; typen voor sandboxbackends en helpers voor SSH-/OpenShell-opdrachten, inclusief een voorafgaande controle voor snel falende exec-opdrachten |
    | `plugin-sdk/temp-path` | Gedeelde helpers voor tijdelijke downloadpaden en privé, beveiligde tijdelijke werkruimten |
    | `plugin-sdk/logging-core` | Helpers voor subsysteemlogging en redactie |
    | `plugin-sdk/markdown-table-runtime` | Privé en lokaal na juli 2026; helpers voor de modus en conversie van Markdown-tabellen |
    | `plugin-sdk/model-session-runtime` | Helpers voor model-/sessieoverschrijvingen, zoals `applyModelOverrideToSessionEntry` en `resolveAgentMaxConcurrent` |
    | `plugin-sdk/talk-config-runtime` | Privé en lokaal na juli 2026; helpers voor het oplossen van de configuratie van Talk-providers |
    | `plugin-sdk/json-store` | Kleine helpers voor het lezen/schrijven van JSON-status |
    | `plugin-sdk/json-unsafe-integers` | Privé en lokaal na juli 2026; helpers voor het parseren van JSON die onveilige gehele getalliteralen als tekenreeksen behouden |
    | `plugin-sdk/file-lock` | Privé en lokaal na juli 2026; herintreedbare helpers voor bestandsvergrendeling plus Doctor-veilige terugwinning van aantoonbaar verouderde, ongewijzigde en buiten gebruik gestelde sidecars voor vergrendelingen |
    | `plugin-sdk/persistent-dedupe` | Helpers voor een schijfgebaseerde deduplicatiecache |
    | `plugin-sdk/ingress-effect-once` | Duurzame claim-/commitbeveiliging voor niet-idempotente neveneffecten bij binnenkomst |
    | `plugin-sdk/acp-runtime` | Privé en lokaal na juli 2026; helpers voor ACP-runtime/-sessies en antwoordverzending |
    | `plugin-sdk/acp-runtime-backend` | Privé en lokaal na juli 2026; lichtgewicht helpers voor ACP-backendregistratie en antwoordverzending voor bij het opstarten geladen plugins |
    | `plugin-sdk/acp-binding-resolve-runtime` | Privé en lokaal na juli 2026; alleen-lezenoplossing van ACP-bindingen zonder imports voor het starten van de levenscyclus |
    | `plugin-sdk/agent-config-primitives` | Verouderde configuratieschemaprimitieven voor de agentruntime; importeer schemaprimitieven vanuit een onderhouden, door een plugin beheerd oppervlak |
    | `plugin-sdk/boolean-param` | Flexibele reader voor booleaanse parameters |
    | `plugin-sdk/dangerous-name-runtime` | Privé en lokaal na juli 2026; resolutiehelpers voor overeenkomsten met gevaarlijke namen |
    | `plugin-sdk/device-bootstrap` | Helpers voor het initialiseren van apparaten en koppeltokens, inclusief `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` |
    | `plugin-sdk/extension-shared` | Gedeelde basishelpers voor passieve kanalen, status en omgevingsproxy's |
    | `plugin-sdk/models-provider-runtime` | Helpers voor antwoorden op `/models`-opdrachten/providers |
    | `plugin-sdk/skill-commands-runtime` | Helpers voor het weergeven van Skill-opdrachten |
    | `plugin-sdk/native-command-registry` | Helpers voor het register, bouwen en serialiseren van native opdrachten |
    | `plugin-sdk/agent-harness` | Experimenteel oppervlak voor vertrouwde plugins voor agent-harnesses op laag niveau: harnesstypen, helpers om actieve uitvoeringen bij te sturen/af te breken, helpers voor de OpenClaw-toolbridge, helpers voor toolbeleid in runtimeplannen, classificatie van terminalresultaten, helpers voor de opmaak/details van toolvoortgang en hulpprogramma's voor pogingresultaten |
    | `plugin-sdk/async-lock-runtime` | Privé en lokaal na juli 2026; proceslokale asynchrone vergrendelingshelper voor kleine runtimestatusbestanden |
    | `plugin-sdk/channel-activity-runtime` | Privé en lokaal na juli 2026; telemetriehelper voor kanaalactiviteit |
    | `plugin-sdk/concurrency-runtime` | Privé en lokaal na juli 2026; begrensde helper voor gelijktijdige asynchrone taken |
    | `plugin-sdk/dedupe-runtime` | Helpers voor deduplicatiecaches in het geheugen en met persistente opslag |
    | `plugin-sdk/delivery-queue-runtime` | Privé en lokaal na juli 2026; helper voor het leegmaken van wachtende uitgaande leveringen |
    | `plugin-sdk/file-access-runtime` | Privé en lokaal na juli 2026; veilige padhelpers voor lokale bestanden en mediabronnen |
    | `plugin-sdk/heartbeat-runtime` | Privé en lokaal na juli 2026; helpers voor het wekken, gebeurtenissen en zichtbaarheid van Heartbeat |
    | `plugin-sdk/expect-runtime` | Privé en lokaal na juli 2026; assertiehelper voor vereiste waarden bij aantoonbare runtime-invarianten |
    | `plugin-sdk/number-runtime` | Privé en lokaal na juli 2026; helper voor numerieke conversie |
    | `plugin-sdk/secure-random-runtime` | Privé en lokaal na juli 2026; helpers voor beveiligde tokens/UUID's |
    | `plugin-sdk/system-event-runtime` | Privé en lokaal na juli 2026; helpers voor de wachtrij van systeemgebeurtenissen |
    | `plugin-sdk/transport-ready-runtime` | Privé en lokaal na juli 2026; helper om te wachten tot het transport gereed is |
    | `plugin-sdk/exec-approvals-runtime` | Privé en lokaal na juli 2026; bestandshelpers voor het goedkeuringsbeleid voor exec zonder de brede infra-runtimebarrel |
    | `plugin-sdk/infra-runtime` | Verouderde compatibiliteitsshim; gebruik de gerichte runtime-subpaden hierboven |
    | `plugin-sdk/collection-runtime` | Kleine begrensde cachehelpers |
    | `plugin-sdk/diagnostic-runtime` | Helpers voor diagnostische vlaggen, gebeurtenissen en traceercontext |
    | `plugin-sdk/error-runtime` | Helpers voor foutgrafieken, opmaak, conversie van onbekende waarden en gedeelde foutclassificatie, `PlatformMessageNotDispatchedError`, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Privé en lokaal na juli 2026; helpers voor omhulde fetch, proxy's, EnvHttpProxyAgent-opties en vastgezette lookups |
    | `plugin-sdk/runtime-fetch` | Privé en lokaal na juli 2026; dispatcherbewuste runtime-fetch zonder imports voor proxy/beveiligde fetch |
    | `plugin-sdk/inline-image-data-url-runtime` | Privé en lokaal na juli 2026; helpers voor het opschonen van inline afbeeldingsgegevens-URL's en het detecteren van signatures zonder het brede mediaruntimeoppervlak |
    | `plugin-sdk/response-limit-runtime` | Privé en lokaal na juli 2026; readers voor responsebody's begrensd op bytes, inactiviteit en deadline, zonder het brede mediaruntimeoppervlak |
    | `plugin-sdk/session-binding-runtime` | Privé en lokaal na juli 2026; huidige status van gespreksbindingen zonder geconfigureerde routering van bindingen of koppelopslag |
    | `plugin-sdk/context-visibility-runtime` | Privé en lokaal na juli 2026; oplossing van contextzichtbaarheid en filtering van aanvullende context zonder brede configuratie-/beveiligingsimports |
    | `plugin-sdk/string-coerce-runtime` | Gerichte primitieve helpers voor conversie en normalisatie van records/tekenreeksen zonder imports voor Markdown/logging |
    | `plugin-sdk/html-entity-runtime` | Privé en lokaal na juli 2026; HTML5-entiteitsdecodering in één doorgang, beëindigd door puntkomma's, zonder brede teksthulpprogramma's |
    | `plugin-sdk/text-utility-runtime` | Privé-lokaal na juli 2026; laag-niveauhelpers voor tekst en paden, inclusief HTML-escaping van vijf entiteiten |
    | `plugin-sdk/widget-html` | Detectie van volledige documenten, groottevalidatie en fouten in toolinvoer voor zelfstandige HTML-widgets |
    | `plugin-sdk/host-runtime` | Privé-lokaal na juli 2026; helpers voor de normalisatie van hostnamen en SCP-hosts |
    | `plugin-sdk/retry-runtime` | Privé-lokaal na juli 2026; helpers voor configuratie en uitvoering van nieuwe pogingen |
    | `plugin-sdk/agent-runtime` | Verouderde brede barrel voor helpers voor agentmappen, -identiteiten en -werkruimten, inclusief `resolveAgentDir`, `resolveDefaultAgentDir` en de verouderde compatibiliteitsexport `resolveOpenClawAgentDir`; geef de voorkeur aan gerichte subpaden voor agents en runtimes |
    | `plugin-sdk/directory-runtime` | Door configuratie ondersteunde mapquery en deduplicatie |
    | `plugin-sdk/keyed-async-queue` | Privé-lokaal na juli 2026; `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Subpaden voor mogelijkheden en tests">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | Verouderde brede mediabarrel met onder meer `saveRemoteMedia`, `saveResponseMedia`, `readRemoteMediaBuffer` en de verouderde `fetchRemoteMedia`; geef de voorkeur aan `plugin-sdk/media-store`, `plugin-sdk/media-mime`, `plugin-sdk/outbound-media` en subpaden voor de runtime van mogelijkheden, en geef de voorkeur aan opslaghelpers boven het lezen van buffers wanneer een URL OpenClaw-media moet worden |
    | `plugin-sdk/media-local-roots` | Gerichte helpers voor `getAgentScopedMediaLocalRoots(...)` en beleidsbewuste helpers voor `getAgentScopedMediaLocalRootsForSources(...)` voor lokale medialeesbewerkingen die eigendom zijn van de plugin |
    | `plugin-sdk/media-mime` | Gerichte helpers voor MIME-normalisatie, toewijzing van bestandsextensies, MIME-detectie en mediasoorten |
    | `plugin-sdk/media-store` | Gerichte helpers voor mediaopslag, zoals `saveMediaBuffer` en `saveMediaStream` |
    | `plugin-sdk/media-generation-runtime` | Privé-lokaal na juli 2026; gedeelde helpers voor failover bij mediageneratie, selectie van kandidaten en meldingen over ontbrekende modellen |
    | `plugin-sdk/media-understanding` | Verouderde compatibiliteitsfacade voor providertypen en helpers voor mediabegrip; nieuwe providers registreren zich via de geïnjecteerde plugin-API en behouden aanvraaghelpers binnen de plugin |
    | `plugin-sdk/text-chunking` | Uitgaande tekst en bereiksegmentering met behoud van offsets, segmentering van Markdown/renderhelpers, tokenisatie van HTML-tags met inachtneming van citaten, conversie van Markdown-tabellen, verwijdering van instructietags en hulpprogramma's voor veilige tekst |
    | `plugin-sdk/speech` | Privé-lokaal na juli 2026; spraakprovidertypen plus providergerichte exports voor instructies, registers, validatie, een OpenAI-compatibele TTS-builder en spraakhelpers |
    | `plugin-sdk/speech-core` | Privé-lokaal na juli 2026; gedeelde exports voor spraakprovidertypen, registers, instructies, normalisatie en spraakhelpers |
    | `plugin-sdk/speech-settings` | Lichtgewicht primitieven voor het oplossen en normaliseren van TTS-configuratie, zonder providerregisters of syntheseruntime |
    | `plugin-sdk/realtime-transcription` | Privé-lokaal na juli 2026; providertypen voor realtime transcriptie, registerhelpers en een gedeelde WebSocket-sessiehelper |
    | `plugin-sdk/realtime-bootstrap-context` | Privé-lokaal na juli 2026; bootstraphelper voor realtime profielen voor begrensde contextinjectie van `IDENTITY.md`, `USER.md` en `SOUL.md` |
    | `plugin-sdk/realtime-voice` | Privé-lokaal na juli 2026; providertypen voor realtime spraak, registerhelpers, gedeelde poorten voor audio-energie/spraakaanvang en helpers voor realtime spraakgedrag, waaronder het transportonafhankelijke sessieharnas en het bijhouden van uitvoeractiviteit |
    | `plugin-sdk/meeting-runtime` | Sessieruntime voor browservergaderingen, realtime audio-engines/-transporten, `MeetingPlatformAdapter`, browser-/Node-besturing, agentconsultatie, delegatie van spraakoproepen, installatiecontroles en SoX-opdrachthelpers |
    | `plugin-sdk/image-generation` | Privé-lokaal na juli 2026; providertypen voor afbeeldingsgeneratie plus helpers voor afbeeldingsassets/data-URL's en de OpenAI-compatibele afbeeldingsproviderbuilder |
    | `plugin-sdk/image-generation-core` | Privé-lokaal na juli 2026; gedeelde typen, failover-, authenticatie- en registerhelpers voor afbeeldingsgeneratie |
    | `plugin-sdk/music-generation` | Privé-lokaal na juli 2026; provider-/aanvraag-/resultaattypen voor muziekgeneratie |
    | `plugin-sdk/video-generation` | Privé-lokaal na juli 2026; provider-/aanvraag-/resultaattypen voor videogeneratie |
    | `plugin-sdk/video-generation-core` | Privé-lokaal na juli 2026; gedeelde typen voor videogeneratie, failoverhelpers, provideropzoeking en verwerking van modelreferenties |
    | `plugin-sdk/transcripts` | Privé-lokaal na juli 2026; gedeelde providertypen voor transcriptbronnen, registerhelpers, een bridgefactory voor vergaderproviders, sessiebeschrijvingen en metadata van uitingen |
    | `plugin-sdk/webhook-targets` | Privé-lokaal na juli 2026; register voor Webhook-doelen en helpers voor het installeren van routes |
    | `plugin-sdk/web-media` | Gedeelde helpers voor het laden van externe/lokale media |
    | `plugin-sdk/zod` | Verouderde compatibiliteitsherexport; importeer `zod` rechtstreeks uit `zod` |
    | `plugin-sdk/plugin-test-api` | Minimale repo-lokale helper voor `createTestPluginApi` voor eenheidstests van directe pluginregistratie zonder bruggen naar repo-testhelpers te importeren |
    | `plugin-sdk/agent-runtime-test-contracts` | Repo-lokale fixtures voor het contract van de systeemeigen agent-runtimeadapter voor tests van authenticatie, aflevering, fallback, toolhooks, promptoverlays, schema's en transcriptprojectie |
    | `plugin-sdk/channel-test-helpers` | Repo-lokale kanaalgerichte testhelpers voor algemene contracten voor acties/installatie/status, directorycontroles, de opstartlevenscyclus van accounts, het doorgeven van verzendconfiguratie, runtimemocks, statusproblemen, uitgaande aflevering en hookregistratie |
    | `plugin-sdk/channel-target-testing` | Repo-lokale gedeelde suite met foutgevallen voor doelresolutie voor kanaaltests |
    | `plugin-sdk/channel-contract-testing` | Repo-lokale gerichte testhelpers voor kanaalcontracten zonder de brede testbarrel |
    | `plugin-sdk/plugin-test-contracts` | Repo-lokale helpers voor contracten rond pluginpakketten, registratie, openbare artefacten, directe imports, runtime-API's en neveneffecten van imports |
    | `plugin-sdk/plugin-state-test-runtime` | Repo-lokale testhelpers voor de pluginstatusopslag, ingress-wachtrij en statusdatabase |
    | `plugin-sdk/provider-test-contracts` | Repo-lokale helpers voor contracten rond providerruntime, authenticatie, ontdekking, onboarding, catalogi, wizards, mediamogelijkheden, herhalingsbeleid, live realtime STT-audio, zoeken/ophalen op het web en streams |
    | `plugin-sdk/provider-http-test-mocks` | Privé-lokaal na juli 2026; optionele repo-lokale Vitest-mocks voor HTTP/authenticatie voor providertests die `plugin-sdk/provider-http` uitvoeren |
    | `plugin-sdk/reply-payload-testing` | Repo-lokale helpers voor het koppelen van metadata aan fixtures voor antwoordpayloads |
    | `plugin-sdk/sqlite-runtime-testing` | Repo-lokale SQLite-levenscyclushelpers voor first-party-tests |
    | `plugin-sdk/test-fixtures` | Repo-lokale fixtures voor algemene vastlegging van de CLI-runtime, sandboxcontext, het schrijven van skills, agentberichten, systeemgebeurtenissen, het herladen van modules, paden van gebundelde plugins, terminaltekst, segmentering, authenticatietokens en getypeerde gevallen |
    | `plugin-sdk/test-node-mocks` | Repo-lokale gerichte mockhelpers voor ingebouwde Node-modules voor gebruik binnen Vitest-factory's voor `vi.mock("node:*")` |
  </Accordion>

  <Accordion title="Geheugensubpaden">
    | Subpad | Belangrijkste exports |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | Privé-lokaal na juli 2026; lichtgewicht registerhelpers voor providers van geheugenembeddings |
    | `plugin-sdk/memory-core-host-engine-foundation` | Exports van de basismotor voor de geheugenhost |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Privé-lokaal na juli 2026; contracten voor embeddings van de geheugenhost, registertoegang, lokale provider en algemene batch-/externe helpers. `registerMemoryEmbeddingProvider` op dit oppervlak is verouderd; gebruik voor nieuwe providers de algemene API voor embeddingproviders. |
    | `plugin-sdk/memory-core-host-engine-qmd` | Privé-lokaal na juli 2026; exports van de QMD-engine van de geheugenhost |
    | `plugin-sdk/memory-core-host-engine-storage` | Privé-lokaal na juli 2026; exports van de opslagengine van de geheugenhost |
    | `plugin-sdk/memory-core-host-secret` | Privé-lokaal na juli 2026; geheimhelpers van de geheugenhost |
    | `plugin-sdk/memory-core-host-status` | Privé-lokaal na juli 2026; statushelpers van de geheugenhost |
    | `plugin-sdk/memory-core-host-runtime-cli` | Privé-lokaal na juli 2026; CLI-runtimehelpers van de geheugenhost |
    | `plugin-sdk/memory-core-host-runtime-core` | Privé-lokaal na juli 2026; kernruntimehelpers van de geheugenhost |
    | `plugin-sdk/memory-core-host-runtime-files` | Privé-lokaal na juli 2026; bestands-/runtimehelpers van de geheugenhost |
    | `plugin-sdk/memory-host-core` | Verouderde compatibiliteitsfacade voor leveranciersonafhankelijke helpers van de geheugenhost. Nieuwe geheugenplugins gebruiken geïnjecteerde geheugenmogelijkheden en door de host voorbereide prompts; begeleidende plugins gebruiken de behouden facade nog voor het ontdekken van openbare artefacten totdat er een gerichte leesinterface bestaat. |
    | `plugin-sdk/memory-host-events` | Privé-lokaal na juli 2026; leveranciersonafhankelijke alias voor gebeurtenisjournaalhelpers van de geheugenhost |
    | `plugin-sdk/memory-host-markdown` | Privé-lokaal na juli 2026; gedeelde helpers voor beheerde Markdown voor geheugengerelateerde plugins |
    | `plugin-sdk/memory-host-search` | Privé-lokaal na juli 2026; runtimefacade voor Active Memory voor toegang tot de zoekmanager |
  </Accordion>

  <Accordion title="Gereserveerde subpaden voor gebundelde helpers">
    Gereserveerde SDK-subpaden voor gebundelde helpers zijn gerichte, eigenaarspecifieke oppervlakken voor
    gebundelde plugincode. Ze worden bijgehouden in de SDK-inventaris, zodat pakket-
    builds en aliasing deterministisch blijven, maar het zijn geen algemene API's
    voor het ontwikkelen van plugins. Nieuwe herbruikbare hostcontracten moeten algemene SDK-subpaden gebruiken,
    zoals `plugin-sdk/gateway-runtime` en `plugin-sdk/ssrf-runtime`.

    | Subpad | Eigenaar en doel |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | Privé-lokaal na juli 2026; helper voor de gebundelde Codex-plugin om de configuratie van de MCP-server van de gebruiker te projecteren naar de threadconfiguratie van de Codex-appserver (gereserveerde pakketexport) |
    | `plugin-sdk/codex-native-task-runtime` | Helper voor de gebundelde Codex-plugin om systeemeigen subagents van de Codex-appserver te spiegelen naar de taakstatus van OpenClaw (alleen repo-lokaal, geen pakketexport) |

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Overzicht van de plugin-SDK](/nl/plugins/sdk-overview)
- [Installatie van de plugin-SDK](/nl/plugins/sdk-setup)
- [Plugins bouwen](/nl/plugins/building-plugins)
