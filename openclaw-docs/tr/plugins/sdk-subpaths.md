---
read_when:
    - Bir Plugin içe aktarımı için doğru plugin-sdk alt yolunu seçme
    - Paketlenmiş Plugin alt yollarını ve yardımcı yüzeylerini denetleme
summary: 'Plugin SDK alt yol kataloğu: hangi içe aktarımların nerede bulunduğu, alana göre gruplandırılmış olarak'
title: Plugin SDK alt yolları
x-i18n:
    generated_at: "2026-07-27T00:09:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

Plugin SDK, dar kapsamlı herkese açık alt yolları ve `openclaw/plugin-sdk/` altındaki yalnızca depoya özgü paketlenmiş
yardımcıları içerir. Bu sayfa her ikisini de kataloglar ve
özel-yerel girdileri açıkça etiketler. Sınırı üç dosya tanımlar:

- `scripts/lib/plugin-sdk-entrypoints.json`: derlemenin derlediği, bakımı yapılan giriş noktası envanteri.
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`: türleri tanımlanmış ve belgelenmiş SDK'nın dışında tutulan
  dahili alt yollar. Üretim girdileri, ayrı olarak yayımlanan resmî
  pluginler için yalnızca JavaScript ana bilgisayar çalışma zamanı dışa aktarımları
  olarak kullanılabilir kalır; yalnızca test girdileri dışa aktarılmaz.
- `src/plugin-sdk/entrypoints.ts`: kullanımdan kaldırılmış
  alt yollar, ayrılmış paketlenmiş yardımcılar, desteklenen paketlenmiş cepheler ve
  plugine ait herkese açık yüzeyler için sınıflandırma meta verileri.

Bakımcılar, herkese açık dışa aktarım sayısını `pnpm plugin-sdk:surface` ile ve
etkin ayrılmış yardımcı alt yollarını `pnpm plugins:boundary-report:summary` ile denetler;
kullanılmayan ayrılmış yardımcı dışa aktarımlar, herkese açık SDK'da atıl uyumluluk
borcu olarak kalmak yerine CI raporunun başarısız olmasına neden olur.

Plugin yazma kılavuzu için [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview) bölümüne bakın.

## Plugin girişi

| Alt yol                        | Temel dışa aktarımlar                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | Temmuz 2026'dan sonra özel-yerel; `defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | Temmuz 2026'dan sonra özel-yerel; `createMigrationItem` gibi geçiş sağlayıcısı öğe yardımcıları, neden sabitleri, öğe durum işaretçileri, redaksiyon yardımcıları ve `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | Temmuz 2026'dan sonra özel-yerel; `copyMigrationFileItem`, `resolvePlannedMigrationTargets`, `withCachedMigrationConfigRuntime` ve `writeMigrationReport` gibi çalışma zamanı geçiş yardımcıları              |
| `plugin-sdk/health`            | Paketlenmiş sağlık durumu tüketicileri için Doctor sistem durumu denetimi kaydı, algılama, onarım, seçim, önem derecesi ve bulgu türleri                                                                                |

### Uyumluluk ve özel-yerel yardımcılar

Yalnızca daha sonraki dönem için kullanımdan kaldırılmış alt yollar dışa aktarılmaya devam eder. Temmuz 2026 takma adları ve
kullanılmayan alt yollar silinirken yalnızca paketlenmiş yardımcılar
herkese açık paketten kaldırılmış ve aşağıda özel-yerel olarak etiketlenmiştir. Bakımı yapılan liste
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; CI, paketlenmiş
`plugin-sdk/text-runtime` yalnızca uyumluluk içindir ve `plugin-sdk/zod` bir
uyumluluk yeniden dışa aktarımıdır: `zod` öğesini doğrudan `zod` konumundan içe aktarın. Geniş kapsamlı etki alanı
varilleri `plugin-sdk/agent-runtime`, `plugin-sdk/channel-lifecycle`,
`plugin-sdk/conversation-runtime`, `plugin-sdk/hook-runtime`,
`plugin-sdk/media-runtime`, `plugin-sdk/plugin-runtime` ve
`plugin-sdk/security-runtime` de odaklanmış
alt yollar lehine kullanımdan kaldırılmıştır.

OpenClaw'ın Vitest destekli test yardımcısı alt yolları yalnızca depoya özgüdür ve artık
paket dışa aktarımları değildir: `agent-runtime-test-contracts`,
`channel-contract-testing`, `channel-target-testing`, `channel-test-helpers`,
`plugin-state-test-runtime`, `plugin-test-api`, `plugin-test-contracts`,
`plugin-test-runtime`, `provider-http-test-mocks`, `provider-test-contracts`,
`reply-payload-testing`, `sqlite-runtime-testing`, `test-env`, `test-fixtures`,
`test-live`, `test-live-auth`, `test-media-generation`,
`test-media-understanding`, `test-node-mocks` ve `testing`. Özel paketlenmiş yardımcı yüzeyleri
`ssrf-runtime-internal` ve `codex-native-task-runtime` da yalnızca depoya özgüdür.

### Paketlenmiş plugin yardımcı alt yolları

Yalnızca paketlenmiş yardımcı modüller, Temmuz 2026 taramasından sonra özel-yereldir. Sahipler arası içe aktarımlar paket sözleşmesi korumalarıyla engellenir. `src/plugin-sdk/entrypoints.ts`, genel sözleşmeler
`plugin-sdk/qa-runner-runtime`, `plugin-sdk/telegram-account` öğelerinin yerini alana kadar paketlenmiş pluginleri tarafından desteklenen, herkese açık kalmaya devam eden SDK
giriş noktaları olan desteklenen paketlenmiş cepheleri ayrıca izler;
yeni kod için kullanımdan kaldırılmıştır; aşağıdaki satır bazındaki notlara bakın.

<AccordionGroup>
  <Accordion title="Kanal alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | Temmuz 2026'dan sonra özel-yerel; Plugine ait şemalar için önbelleğe alınmış JSON Schema doğrulama yardımcısı |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`, kanala ait kurulum alanı/girdi türleri, `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard` ile `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | Paylaşılan kurulum sihirbazı yardımcıları, kurulum çevirmeni, izin verilenler listesi istemleri, kurulum durumu oluşturucuları |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`, `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | Çok hesaplı yapılandırma/eylem kapısı yardımcıları, varsayılan hesap geri dönüş yardımcıları |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, hesap kimliği normalleştirme yardımcıları |
    | `plugin-sdk/account-resolution` | Hesap arama + varsayılana geri dönüş yardımcıları |
    | `plugin-sdk/account-helpers` | Dar kapsamlı hesap listesi/hesap eylemi yardımcıları |
    | `plugin-sdk/access-groups` | Temmuz 2026'dan sonra özel-yerel; Erişim grubu izin verilenler listesi ayrıştırma ve redakte edilmiş grup tanılama yardımcıları |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | Kullanımdan kaldırılmış uyumluluk cephesi. `plugin-sdk/channel-outbound` kullanın. |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`, `resolveChannelDmAccess`, `resolveChannelDmAllowFrom`, `resolveChannelDmPolicy`, `normalizeChannelDmPolicy`, `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | Paylaşılan kanal yapılandırma şeması temel öğelerinin yanı sıra Zod ve doğrudan JSON/TypeBox oluşturucuları |
    | `plugin-sdk/bundled-channel-config-schema` | Temmuz 2026'dan sonra özel-yerel; Yalnızca bakımı yapılan paketlenmiş pluginler için paketlenmiş OpenClaw kanal yapılandırma şemaları |
    | `plugin-sdk/chat-channel-ids` | Temmuz 2026'dan sonra özel-yerel; `BUNDLED_CHAT_CHANNEL_IDS`, `BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`, `ChatChannelId`. Kendi tablolarını sabit kodlamadan zarf önekli metni tanıması gereken pluginler için biçimlendirici etiketleri/takma adlarının yanı sıra standart paketlenmiş/resmî sohbet kanalı kimlikleri. |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | Taşınmış kanal alma yolları için deneysel üst düzey kanal giriş çalışma zamanı çözümleyicisi, örtük bahsetme ilkesi çözümleyicisi ve yol olgusu oluşturucuları. Her pluginde etkin izin verilenler listeleri, komut izin verilenler listeleri ve eski projeksiyonları bir araya getirmek yerine bunu tercih edin. [Kanal giriş API'si](/tr/plugins/sdk-channel-ingress) bölümüne bakın. |
    | `plugin-sdk/channel-lifecycle` | Kullanımdan kaldırılmış uyumluluk cephesi. `plugin-sdk/channel-outbound` kullanın. |
    | `plugin-sdk/channel-outbound` | Yanıt işlem hattı seçenekleri, alındı bildirimleri, canlı önizleme/akış, yaşam döngüsü yardımcıları, giden kimliği, yük planlaması, kalıcı gönderimler ve ileti gönderme bağlamı yardımcılarının yanı sıra ileti yaşam döngüsü sözleşmeleri. [Kanal giden API'si](/tr/plugins/sdk-channel-outbound) bölümüne bakın. |
    | `plugin-sdk/channel-message` | `plugin-sdk/channel-outbound` için kullanımdan kaldırılmış uyumluluk takma adı. |
    | `plugin-sdk/inbound-envelope` | Paylaşılan gelen yol + zarf oluşturucu yardımcıları |
    | `plugin-sdk/inbound-reply-dispatch` | Kullanımdan kaldırılmış uyumluluk cephesi. Gelen çalıştırıcılar ve yönlendirme koşulları için `plugin-sdk/channel-inbound`, ileti teslimi yardımcıları için ise `plugin-sdk/channel-outbound` kullanın. |
    | `plugin-sdk/messaging-targets` | Kullanımdan kaldırılmış hedef ayrıştırma takma adı; `plugin-sdk/channel-targets` kullanın |
    | `plugin-sdk/outbound-media` | Temmuz 2026'dan sonra özel-yerel; Paylaşılan giden medya yükleme ve barındırılan medya durumu yardımcıları |
    | `plugin-sdk/poll-runtime` | Temmuz 2026'dan sonra özel-yerel; Dar kapsamlı anket normalleştirme yardımcıları |
    | `plugin-sdk/thread-bindings-runtime` | Temmuz 2026'dan sonra özel-yerel; İş parçacığı bağlama yaşam döngüsü ve bağdaştırıcı yardımcıları |
    | `plugin-sdk/agent-media-payload` | Eski `Media*` yük projeksiyonu için kullanımdan kaldırılmış uyumluluk cephesi. Sıralı olguları `MsgContext.media` / `toInboundMediaFacts(...)` üzerinden geçirin; yerel kök ilkesini `plugin-sdk/media-local-roots` konumundan içe aktarın. |
    | `plugin-sdk/conversation-runtime` | Konuşma/iş parçacığı bağlama, eşleştirme ve yapılandırılmış bağlama yardımcıları için kullanımdan kaldırılmış geniş kapsamlı varil; `plugin-sdk/thread-bindings-runtime` ve `plugin-sdk/session-binding-runtime` gibi odaklanmış bağlama alt yollarını tercih edin |
    | `plugin-sdk/runtime-group-policy` | Çalışma zamanı grup ilkesi çözümleme yardımcıları |
    | `plugin-sdk/channel-status` | Paylaşılan kanal durumu anlık görüntü/özet yardımcıları |
    | `plugin-sdk/channel-config-primitives` | Dar kapsamlı kanal yapılandırma şeması temel öğeleri |
    | `plugin-sdk/channel-config-writes` | Temmuz 2026'dan sonra özel-yerel; Kanal yapılandırma yazma yetkilendirme yardımcıları |
    | `plugin-sdk/channel-plugin-common` | Paylaşılan kanal plugini başlangıç dışa aktarımları |
    | `plugin-sdk/allowlist-config-edit` | İzin verilenler listesi yapılandırma düzenleme/okuma yardımcıları |
    | `plugin-sdk/group-access` | Kullanımdan kaldırılmış grup erişimi karar yardımcıları; `plugin-sdk/channel-ingress-runtime` konumundaki `resolveChannelMessageIngress` öğesini kullanın |
    | `plugin-sdk/direct-dm-guard-policy` | Temmuz 2026'dan sonra özel-yerel; Dar kapsamlı doğrudan DM kriptografi öncesi koruma ilkesi yardımcıları |
    | `plugin-sdk/discord` | Yayımlanmış `@openclaw/discord@2026.3.13` ve izlenen sahip uyumluluğu için kullanımdan kaldırılmış Discord uyumluluk cephesi; yeni pluginler genel kanal SDK alt yollarını kullanmalıdır |
    | `plugin-sdk/telegram-account` | İzlenen sahip uyumluluğu için kullanımdan kaldırılmış Telegram hesap çözümleme uyumluluk cephesi; yeni pluginler eklenmiş çalışma zamanı yardımcılarını veya genel kanal SDK alt yollarını kullanmalıdır |
    | `plugin-sdk/interactive-runtime` | Anlamsal ileti sunumu, teslimi ve eski etkileşimli yanıt yardımcıları. [İleti Sunumu](/tr/plugins/message-presentation) bölümüne bakın |
    | `plugin-sdk/question-gateway-runtime` | Çalışma zamanı tarafından oluşturulan `ask_user` seçimlerini kanal etkileşimi işleyicilerinden Gateway aracılığıyla çözümleyin |
    | `plugin-sdk/channel-inbound` | Olay sınıflandırması, bağlam oluşturma, biçimlendirme, kökler, gecikmeli birleştirme, bahsetme eşleştirme, bahsetme ilkesi ve gelen günlük kaydı için paylaşılan gelen yardımcıları |
    | `plugin-sdk/channel-inbound-debounce` | Dar kapsamlı gelen gecikmeli birleştirme yardımcıları |
    | `plugin-sdk/channel-mention-gating` | Temmuz 2026'dan sonra özel-yerel; Daha geniş gelen çalışma zamanı yüzeyi olmadan dar kapsamlı bahsetme ilkesi, bahsetme işaretçisi ve bahsetme metni yardımcıları |
    | `plugin-sdk/channel-streaming` | Kullanımdan kaldırılmış uyumluluk cephesi. `plugin-sdk/channel-outbound` kullanın. |
    | `plugin-sdk/channel-send-result` | Yanıt sonuç türleri |
    | `plugin-sdk/channel-actions` | Kanal ileti eylemi yardımcılarının yanı sıra plugin uyumluluğu için tutulan kullanımdan kaldırılmış yerel şema yardımcıları |
    | `plugin-sdk/channel-route` | Temmuz 2026'dan sonra özel-yerel; Paylaşılan yol normalleştirme, ayrıştırıcı güdümlü hedef çözümleme, iş parçacığı kimliğini dizgeleştirme, yinelenenleri kaldırılmış/kompakt yol anahtarları, ayrıştırılmış hedef türleri ve yol/hedef karşılaştırma yardımcıları |
    | `plugin-sdk/channel-targets` | Temmuz 2026'dan sonra özel-yerel; Hedef ayrıştırma yardımcıları; yol karşılaştırma çağıranları `plugin-sdk/channel-route` kullanmalıdır |
    | `plugin-sdk/channel-contract` | Kanal sözleşmesi türleri |
    | `plugin-sdk/channel-feedback` | Geri bildirim/tepki bağlantıları |
  </Accordion>

Daha sonraki dönem kanal uyumluluk alt yolları yalnızca kayıt tarihlerine kadar
herkese açık kalır. Doğrudan DM erişimi, yanıt seçenekleri, eşleştirme
yolları ve kanal çalışma zamanı parçaları gibi Temmuz takma adları kaldırılmıştır; yalnızca paketlenmiş yardımcılar
özel-yereldir.

  <Accordion title="Sağlayıcı alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/provider-entry` | Temmuz 2026'dan sonra özel-yerel; `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | Temmuz 2026'dan sonra özel-yerel; Seçilmiş yerel/kendi barındırdığınız sağlayıcı kurulum yardımcıları |
    | `plugin-sdk/cli-backend` | Temmuz 2026'dan sonra özel-yerel; CLI arka uç varsayılanları + watchdog sabitleri |
    | `plugin-sdk/provider-auth-runtime` | Temmuz 2026'dan sonra özel-yerel; Sağlayıcı kimlik doğrulama çalışma zamanı yardımcıları: OAuth geri döngü akışı, belirteç değişimi, kimlik doğrulama kalıcılığı ve API anahtarı çözümleme |
    | `plugin-sdk/provider-oauth-runtime` | Temmuz 2026'dan sonra özel-yerel; Genel sağlayıcı OAuth geri çağırma türleri, geri çağırma sayfası işleme, PKCE/durum yardımcıları, yetkilendirme girdisi ayrıştırma, belirteç süre sonu yardımcıları ve iptal yardımcıları |
    | `plugin-sdk/provider-auth-api-key` | Temmuz 2026'dan sonra özel-yerel; `upsertApiKeyProfile` gibi API anahtarı ilk katılım/profil yazma yardımcıları |
    | `plugin-sdk/provider-auth-result` | Temmuz 2026'dan sonra özel-yerel; Standart OAuth kimlik doğrulama sonucu oluşturucu |
    | `plugin-sdk/provider-env-vars` | Temmuz 2026'dan sonra özel-yerel; Sağlayıcı kimlik doğrulama ortam değişkeni arama yardımcıları |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials`, OpenAI Codex kimlik doğrulama içe aktarma yardımcıları, kullanımdan kaldırılmış `resolveOpenClawAgentDir` uyumluluk dışa aktarımı |
    | `plugin-sdk/provider-model-shared` | Temmuz 2026'dan sonra özel-yerel; `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `selectPreferredLocalModelId`, `normalizeModelCompat`, paylaşılan yeniden oynatma politikası oluşturucuları, sağlayıcı uç noktası yardımcıları ve paylaşılan model kimliği normalleştirme yardımcıları |
    | `plugin-sdk/provider-catalog-live-runtime` | Temmuz 2026'dan sonra özel-yerel; Korumalı `/models` tarzı keşif için canlı sağlayıcı model kataloğu yardımcıları: `buildLiveModelProviderConfig`, `fetchLiveProviderModelRows`, `getCachedLiveProviderModelRows`, `fetchLiveProviderModelIds`, `LiveModelCatalogHttpError`, `clearLiveCatalogCacheForTests`, model kimliği filtreleme, TTL önbelleği ve statik yedek |
    | `plugin-sdk/provider-catalog-runtime` | Sözleşme testleri için sağlayıcı kataloğu genişletme çalışma zamanı kancası ve plugin sağlayıcı kayıt defteri bağlantı noktaları |
    | `plugin-sdk/provider-catalog-shared` | Temmuz 2026'dan sonra özel-yerel; `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `buildManifestModelProviderConfig`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | Temmuz 2026'dan sonra özel-yerel; Genel sağlayıcı HTTP/uç nokta yetenek yardımcıları, sağlayıcı HTTP hataları ve ses transkripsiyonu çok parçalı form yardımcıları |
    | `plugin-sdk/provider-web-fetch-contract` | Temmuz 2026'dan sonra özel-yerel; `enablePluginInConfig` ve `WebFetchProviderPlugin` gibi dar kapsamlı web getirme yapılandırma/seçim sözleşmesi yardımcıları |
    | `plugin-sdk/provider-web-fetch` | Temmuz 2026'dan sonra özel-yerel; Web getirme sağlayıcısı kayıt/önbellek yardımcıları |
    | `plugin-sdk/provider-web-search-config-contract` | Temmuz 2026'dan sonra özel-yerel; Plugin etkinleştirme bağlantısına ihtiyaç duymayan sağlayıcılar için dar kapsamlı web arama yapılandırma/kimlik bilgisi yardımcıları |
    | `plugin-sdk/provider-web-search-contract` | Temmuz 2026'dan sonra özel-yerel; `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig` gibi dar kapsamlı web arama yapılandırma/kimlik bilgisi sözleşmesi yardımcıları ve kapsamlı kimlik bilgisi ayarlayıcıları/alıcıları |
    | `plugin-sdk/provider-web-search` | Temmuz 2026'dan sonra özel-yerel; Web arama sağlayıcısı kayıt/önbellek/çalışma zamanı yardımcıları |
    | `plugin-sdk/embedding-providers` | Temmuz 2026'dan sonra özel-yerel; `EmbeddingProviderAdapter`, `getEmbeddingProvider(...)` ve `listEmbeddingProviders(...)` dâhil genel gömme sağlayıcısı türleri ve okuma yardımcıları; bildirim sahipliğinin uygulanması için pluginler sağlayıcıları `api.registerEmbeddingProvider(...)` aracılığıyla kaydeder |
    | `plugin-sdk/provider-tools` | Temmuz 2026'dan sonra özel-yerel; `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks` ve DeepSeek/Gemini/OpenAI şema temizliği + tanılama |
    | `plugin-sdk/provider-usage` | Temmuz 2026'dan sonra özel-yerel; Sağlayıcı kullanım anlık görüntüsü türleri, paylaşılan kullanım getirme yardımcıları ve `fetchClaudeUsage` gibi sağlayıcı getiricileri |
    | `plugin-sdk/provider-stream` | Temmuz 2026'dan sonra özel-yerel; `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, akış sarmalayıcı türleri, düz metin araç çağrısı uyumluluğu ve paylaşılan Anthropic/Google/Kilocode/MiniMax/Moonshot/OpenAI/OpenRouter/Z.AI sarmalayıcı yardımcıları |
    | `plugin-sdk/provider-stream-shared` | Temmuz 2026'dan sonra özel-yerel; `composeProviderStreamWrappers`, `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPlainTextToolCallCompatWrapper`, `createPayloadPatchStreamWrapper`, `createToolStreamWrapper`, `normalizeOpenAICompatibleReasoningPayload`, `setQwenChatTemplateThinking` ve Anthropic/DeepSeek/OpenAI uyumlu akış yardımcı programları dâhil herkese açık paylaşılan sağlayıcı akış sarmalayıcı yardımcıları |
    | `plugin-sdk/provider-transport-runtime` | Temmuz 2026'dan sonra özel-yerel; Korumalı getirme, araç sonucu metin çıkarma, aktarım mesajı dönüşümleri ve yazılabilir aktarım olay akışları gibi yerel sağlayıcı aktarım yardımcıları |
    | `plugin-sdk/provider-onboard` | Temmuz 2026'dan sonra özel-yerel; İlk katılım yapılandırma yaması yardımcıları |
    | `plugin-sdk/global-singleton` | Temmuz 2026'dan sonra özel-yerel; İşlem-yerel tekil örnek/harita/önbellek yardımcıları |
    | `plugin-sdk/group-activation` | Temmuz 2026'dan sonra özel-yerel; Dar kapsamlı grup etkinleştirme modu ve komut ayrıştırma yardımcıları |
  </Accordion>

Sağlayıcı kullanım anlık görüntüleri normalde her biri bir etiket, kullanılan yüzde ve
isteğe bağlı sıfırlama zamanı içeren bir veya daha fazla kota `windows` bildirir.
Sıfırlanabilir kota pencereleri yerine bakiye veya hesap durumu metni sunan sağlayıcılar,
yüzdeler uydurmak yerine boş bir `windows` dizisiyle `summary` döndürmelidir.
OpenClaw bu özet metnini durum çıktısında gösterir; `error` yalnızca kullanım
uç noktası başarısız olduğunda veya kullanılabilir kullanım verisi döndürmediğinde kullanılmalıdır.

  <Accordion title="Kimlik doğrulama ve güvenlik alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/command-auth` | Kullanımdan kaldırılmış geniş komut yetkilendirme yüzeyi (`resolveControlCommandGate`, dinamik bağımsız değişken menüsü biçimlendirmesi dâhil komut kayıt defteri yardımcıları, gönderen yetkilendirme yardımcıları); kanal girişi/çalışma zamanı yetkilendirmesi veya komut durumu yardımcılarını kullanın |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated` ve `buildHelpMessage` gibi komut/yardım mesajı oluşturucuları |
    | `plugin-sdk/approval-auth-runtime` | Onaylayan çözümleme ve aynı sohbet eylemi kimlik doğrulama yardımcıları |
    | `plugin-sdk/approval-client-runtime` | Yerel çalıştırma onayı profil/filtre yardımcıları |
    | `plugin-sdk/approval-delivery-runtime` | Yerel onay yeteneği/teslim adaptörleri |
    | `plugin-sdk/approval-gateway-runtime` | Paylaşılan onay Gateway çözümleyicisi |
    | `plugin-sdk/approval-reference-runtime` | Temmuz 2026'dan sonra özel-yerel; Aktarım sınırlı onay geri çağırmaları için belirlenimci kalıcı konum belirleyici yardımcısı |
    | `plugin-sdk/approval-handler-adapter-runtime` | Yoğun kanal giriş noktaları için hafif yerel onay adaptörü yükleme yardımcıları |
    | `plugin-sdk/approval-handler-runtime` | Daha geniş onay işleyicisi çalışma zamanı yardımcıları; yeterli olduklarında daha dar adaptör/Gateway bağlantı noktalarını tercih edin |
    | `plugin-sdk/approval-native-runtime` | Yerel onay hedefi, hesap bağlama, rota geçidi, iletme yedeği ve yerel yerel çalıştırma istemi bastırma yardımcıları |
    | `plugin-sdk/approval-reaction-runtime` | Temmuz 2026'dan sonra özel-yerel; Sabit kodlanmış onay tepki bağlamaları, tepki istemi yükleri, tepki hedefi depoları, tepki ipucu metni yardımcıları ve yerel yerel çalıştırma istemi bastırma için uyumluluk dışa aktarımı |
    | `plugin-sdk/approval-reply-runtime` | Çalıştırma/plugin onayı yanıt yükü yardımcıları |
    | `plugin-sdk/approval-runtime` | Çalıştırma/plugin onayı yükü yardımcıları, onay yeteneği oluşturucuları, onay kimlik doğrulama/profil yardımcıları, yerel onay yönlendirme/çalışma zamanı yardımcıları ve `formatApprovalDisplayPath` gibi yapılandırılmış onay görüntüleme yardımcıları |
    | `plugin-sdk/command-auth-native` | Yerel komut kimlik doğrulaması, dinamik bağımsız değişken menüsü biçimlendirmesi ve yerel oturum hedefi yardımcıları |
    | `plugin-sdk/command-detection` | Paylaşılan komut algılama yardımcıları |
    | `plugin-sdk/command-primitives-runtime` | Yoğun kanal yolları için hafif komut metni yüklemleri |
    | `plugin-sdk/command-surface` | Temmuz 2026'dan sonra özel-yerel; Komut gövdesi normalleştirme ve komut yüzeyi yardımcıları |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | Temmuz 2026'dan sonra özel-yerel; Özel kanal ve Web kullanıcı arayüzü cihaz kodu eşleştirmesi için gecikmeli sağlayıcı kimlik doğrulama oturum açma akışı yardımcıları |
    | `plugin-sdk/channel-secret-runtime` | Kullanımdan kaldırılmış geniş gizli bilgi sözleşmesi yüzeyi (`collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment`, gizli bilgi hedefi türleri); aşağıdaki odaklı alt yolları tercih edin |
    | `plugin-sdk/channel-secret-basic-runtime` | TTS olmayan kanal/plugin gizli bilgi yüzeyleri için dar kapsamlı gizli bilgi sözleşmesi dışa aktarımları ve hedef kayıt defteri oluşturucuları |
    | `plugin-sdk/channel-secret-tts-runtime` | Temmuz 2026'dan sonra özel-yerel; Dar kapsamlı iç içe kanal TTS gizli bilgi atama yardımcıları |
    | `plugin-sdk/secret-ref-runtime` | Gizli bilgi sözleşmesi/yapılandırma ayrıştırması için dar kapsamlı SecretRef tür belirleme, çözümleme ve plan hedefi yolu arama |
    | `plugin-sdk/security-runtime` | Güven, DM geçitleme, yalnızca oluşturma yazmaları, eşzamanlı/eşzamansız atomik dosya değiştirme, kardeş geçici yazmalar, aygıtlar arası taşıma yedeği, özel dosya deposu yardımcıları, sembolik bağlantı üst dizin korumaları, harici içerik, hassas metin sansürleme, sabit zamanlı gizli bilgi karşılaştırması ve gizli bilgi toplama yardımcılarını içeren kökle sınırlı dosya/yol yardımcıları için kullanımdan kaldırılmış geniş varil; odaklı güvenlik/SSRF/gizli bilgi alt yollarını tercih edin |
    | `plugin-sdk/ssrf-policy` | Ana makine izin listesi ve özel ağ SSRF politikası yardımcıları |
    | `plugin-sdk/ssrf-dispatcher` | Temmuz 2026'dan sonra özel-yerel; Geniş altyapı çalışma zamanı yüzeyi olmadan dar kapsamlı sabitlenmiş dağıtıcı yardımcıları |
    | `plugin-sdk/ssrf-runtime` | Sabitlenmiş dağıtıcı, SSRF korumalı getirme, SSRF hatası ve SSRF politikası yardımcıları |
    | `plugin-sdk/secret-input` | Gizli bilgi girdisi ayrıştırma yardımcıları |
    | `plugin-sdk/webhook-ingress` | Webhook istek/hedef yardımcıları ve ham websocket/gövde tür dönüştürmesi |
    | `plugin-sdk/webhook-request-guards` | İstek gövdesi boyutu/zaman aşımı yardımcıları ve izlenen onay sonrası işleme için `runDetachedWebhookWork` |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/runtime` | Çalışma zamanı/günlükleme/yedekleme yardımcıları, plugin yükleme yolu uyarıları ve süreç yardımcıları |
    | `plugin-sdk/runtime-env` | Dar kapsamlı çalışma zamanı ortamı, günlükleyici, zaman aşımı, yeniden deneme ve geri çekilme yardımcıları |
    | `plugin-sdk/browser-config` | Temmuz 2026'dan sonra özel-yerel; normalleştirilmiş profil/varsayılanlar, CDP URL ayrıştırma ve tarayıcı denetimi kimlik doğrulama yardımcıları için desteklenen tarayıcı yapılandırma cephesi |
    | `plugin-sdk/agent-harness-task-runtime` | Temmuz 2026'dan sonra özel-yerel; ana makine tarafından verilen görev kapsamını kullanan harness destekli aracılar için genel görev yaşam döngüsü ve tamamlanma teslimatı yardımcıları |
    | `plugin-sdk/codex-mcp-projection` | Temmuz 2026'dan sonra özel-yerel; kullanıcı MCP sunucusu yapılandırmasını Codex iş parçacığı yapılandırmasına yansıtmak için ayrılmış paketlenmiş Codex yardımcısı; üçüncü taraf plugin'ler için değildir |
    | `plugin-sdk/codex-native-task-runtime` | Yerel görev yansıtma/çalışma zamanı bağlantıları için depoya yerel paketlenmiş Codex yardımcısı; paket dışa aktarımı değildir |
    | `plugin-sdk/channel-runtime-context` | Genel kanal çalışma zamanı bağlamı kaydı ve arama yardımcıları |
    | `plugin-sdk/matrix` | Eski üçüncü taraf kanal paketleri için kullanımdan kaldırılmış Matrix uyumluluk cephesi; yeni plugin'ler doğrudan `plugin-sdk/run-command` içe aktarmalıdır |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | Plugin komutu/hook/http/etkileşim yardımcıları için kullanımdan kaldırılmış geniş barrel; odaklanmış plugin çalışma zamanı alt yollarını tercih edin |
    | `plugin-sdk/hook-runtime` | Webhook/dahili hook işlem hattı yardımcıları için kullanımdan kaldırılmış geniş barrel; odaklanmış hook/plugin çalışma zamanı alt yollarını tercih edin |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`, `createLazyRuntimeMethod` ve `createLazyRuntimeSurface` gibi tembel çalışma zamanı içe aktarma/bağlama yardımcıları |
    | `plugin-sdk/process-runtime` | Temmuz 2026'dan sonra özel-yerel; süreç çalıştırma yardımcıları |
    | `plugin-sdk/node-host` | Temmuz 2026'dan sonra özel-yerel; Node ana makinesi yürütülebilir dosya çözümleme ve PTY sürdürme yardımcıları |
    | `plugin-sdk/cli-runtime` | Temmuz 2026'dan sonra özel-yerel; CLI biçimlendirme, bekleme, sürüm, bağımsız değişken çağırma ve tembel komut grubu yardımcıları için kullanımdan kaldırılmış geniş barrel; odaklanmış CLI/çalışma zamanı alt yollarını tercih edin |
    | `plugin-sdk/qa-runner-runtime` | Temmuz 2026'dan sonra özel-yerel; plugin QA senaryolarını CLI komut yüzeyi üzerinden sunan desteklenen cephe |
    | `plugin-sdk/tts-runtime` | Temmuz 2026'dan sonra özel-yerel; metinden konuşmaya yapılandırma şemaları ve çalışma zamanı yardımcıları için desteklenen cephe |
    | `plugin-sdk/gateway-method-runtime` | `contracts.gatewayMethodDispatch: ["authenticated-request"]` bildiren plugin HTTP rotaları için ayrılmış Gateway yöntem yönlendirme yardımcısı |
    | `plugin-sdk/gateway-runtime` | Gateway istemcisi, olay döngüsüne hazır istemci başlatma yardımcısı, gateway CLI RPC'si, gateway protokol hataları, ilan edilen LAN ana makinesi çözümleme ve kanal durumu yama yardımcıları |
    | `plugin-sdk/config-contracts` | `OpenClawConfig` gibi plugin yapılandırma biçimleri ile kanal/sağlayıcı yapılandırma türleri için yalnızca türe yönelik odaklanmış yapılandırma yüzeyi |
    | `plugin-sdk/plugin-config-runtime` | Çalışma zamanı plugin yapılandırma yardımcıları için kullanımdan kaldırılmış uyumluluk cephesi; yeni plugin'ler `api.pluginConfig` ile birlikte odaklanmış yapılandırma sözleşmelerini, anlık görüntüleri ve değiştirme yardımcılarını kullanır |
    | `plugin-sdk/config-mutation` | `mutateConfigFile`, `replaceConfigFile` ve `logConfigUpdated` gibi işlemsel yapılandırma değiştirme yardımcıları |
    | `plugin-sdk/message-tool-delivery-hints` | Temmuz 2026'dan sonra özel-yerel; paylaşılan mesaj aracı teslimat meta verisi ipucu dizeleri |
    | `plugin-sdk/runtime-config-snapshot` | `getRuntimeConfig`, `getRuntimeConfigSnapshot` ve test anlık görüntüsü ayarlayıcıları gibi geçerli süreç yapılandırması anlık görüntü yardımcıları |
    | `plugin-sdk/text-autolink-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş metin barrel'i olmadan dosya başvurusu otomatik bağlantı algılama |
    | `plugin-sdk/reply-runtime` | Paylaşılan gelen/yanıt çalışma zamanı yardımcıları, parçalara ayırma, yönlendirme, Heartbeat, yanıt planlayıcı |
    | `plugin-sdk/reply-dispatch-runtime` | Dar kapsamlı yanıt yönlendirme/sonlandırma ve konuşma etiketi yardımcıları |
    | `plugin-sdk/reply-history` | Paylaşılan kısa zaman aralıklı yanıt geçmişi yardımcıları. Yeni mesaj sırası kodu `createChannelHistoryWindow` kullanmalıdır; alt düzey eşleme yardımcıları yalnızca kullanımdan kaldırılmış uyumluluk dışa aktarımları olarak kalır |
    | `plugin-sdk/reply-reference` | Temmuz 2026'dan sonra özel-yerel; `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | Dar kapsamlı metin/markdown parçalara ayırma yardımcıları |
    | `plugin-sdk/session-store-runtime` | Geniş yapılandırma yazma/bakım içe aktarımları olmadan oturum iş akışı yardımcıları (`getSessionEntry`, `listSessionEntries`, `patchSessionEntry`, `upsertSessionEntry`), onarım/yaşam döngüsü yardımcıları (`deleteSessionEntry`, `cleanupSessionLifecycleArtifacts`, `resolveSessionStoreBackupPaths`), geçiş niteliğindeki `sessionFile` değerleri için işaretçi yardımcıları, oturum kimliğine göre sınırlı yakın tarihli kullanıcı/asistan transkript metni okumaları, oturum deposu yolu/oturum anahtarı yardımcıları ve güncellenme zamanı okumaları |
    | `plugin-sdk/session-transcript-runtime` | Temmuz 2026'dan sonra özel-yerel; transkript kimliği, sınırlı ham ve görünür imleçler, kapsamlı hedef/okuma/yazma yardımcıları, görünür mesaj girdisi yansıtma, güncelleme yayımlama, yazma kilitleri ve transkript bellek isabet anahtarları |
    | `plugin-sdk/sqlite-runtime` | Temmuz 2026'dan sonra özel-yerel; veritabanı yaşam döngüsü denetimleri olmadan birinci taraf çalışma zamanı için odaklanmış SQLite aracı şeması, yolu ve işlem yardımcıları |
    | `plugin-sdk/cron-store-runtime` | Temmuz 2026'dan sonra özel-yerel; Cron deposu yolu/yükleme/kaydetme yardımcıları |
    | `plugin-sdk/state-paths` | Durum/OAuth dizini yolu yardımcıları |
    | `plugin-sdk/plugin-state-runtime` | Temmuz 2026'dan sonra özel-yerel; plugin kapsamlı anahtarlı durum, BLOB ve işbirlikçi SQLite kiralama sözleşmelerinin yanı sıra bağlantı pragma'sı, doğrulanmış WAL bakımı ve atomik STRICT şema geçişi yardımcıları. Kiralama geri çağrıları bir iptal sinyali alır ve türü belirlenmiş hatalar zaman aşımı, iptal, sahiplik kaybı, geçersiz girdi ve depolama hatasını birbirinden ayırır |
    | `plugin-sdk/routing` | `resolveAgentRoute`, `buildAgentSessionKey` ve `resolveDefaultAgentBoundAccountId` gibi rota/oturum anahtarı/hesap bağlama yardımcıları |
    | `plugin-sdk/status-helpers` | Paylaşılan kanal/hesap durumu özeti yardımcıları, çalışma zamanı durumu varsayılanları ve sorun meta verisi yardımcıları |
    | `plugin-sdk/target-resolver-runtime` | Temmuz 2026'dan sonra özel-yerel; paylaşılan hedef çözümleyici yardımcıları |
    | `plugin-sdk/string-normalization-runtime` | Temmuz 2026'dan sonra özel-yerel; kısa ad/dize normalleştirme yardımcıları |
    | `plugin-sdk/request-url` | Temmuz 2026'dan sonra özel-yerel; fetch/request benzeri girdilerden dize URL'lerini ayıklama |
    | `plugin-sdk/run-command` | Normalleştirilmiş stdout/stderr sonuçlarına sahip zaman sınırlı komut çalıştırıcı |
    | `plugin-sdk/param-readers` | Ortak araç/CLI parametre okuyucuları |
    | `plugin-sdk/tool-plugin` | Basit, türü belirlenmiş bir aracı-aracı plugin'i tanımlama ve manifest oluşturma için statik meta verileri sunma |
    | `plugin-sdk/tool-payload` | Temmuz 2026'dan sonra özel-yerel; araç sonucu nesnelerinden normalleştirilmiş yükleri ayıklama |
    | `plugin-sdk/tool-send` | Araç bağımsız değişkenlerinden kurallı gönderim hedefi alanlarını ayıklama |
    | `plugin-sdk/sandbox` | Temmuz 2026'dan sonra özel-yerel; hızlı hata veren yürütme komutu ön kontrolü dâhil korumalı alan arka ucu türleri ve SSH/OpenShell komut yardımcıları |
    | `plugin-sdk/temp-path` | Paylaşılan geçici indirme yolu yardımcıları ve özel güvenli geçici çalışma alanları |
    | `plugin-sdk/logging-core` | Alt sistem günlükleyicisi ve sansürleme yardımcıları |
    | `plugin-sdk/markdown-table-runtime` | Temmuz 2026'dan sonra özel-yerel; Markdown tablo modu ve dönüştürme yardımcıları |
    | `plugin-sdk/model-session-runtime` | `applyModelOverrideToSessionEntry` ve `resolveAgentMaxConcurrent` gibi model/oturum geçersiz kılma yardımcıları |
    | `plugin-sdk/talk-config-runtime` | Temmuz 2026'dan sonra özel-yerel; konuşma sağlayıcısı yapılandırma çözümleme yardımcıları |
    | `plugin-sdk/json-store` | Küçük JSON durum okuma/yazma yardımcıları |
    | `plugin-sdk/json-unsafe-integers` | Temmuz 2026'dan sonra özel-yerel; güvenli olmayan tamsayı değişmezlerini dize olarak koruyan JSON ayrıştırma yardımcıları |
    | `plugin-sdk/file-lock` | Temmuz 2026'dan sonra özel-yerel; yeniden girişli dosya kilidi yardımcılarının yanı sıra kesinlikle eski, değişmemiş ve kullanım dışı kilit yan dosyalarının Doctor tarafından güvenli biçimde geri alınması |
    | `plugin-sdk/persistent-dedupe` | Disk destekli yinelenenleri ayıklama önbelleği yardımcıları |
    | `plugin-sdk/ingress-effect-once` | İdempotent olmayan giriş yan etkileri için kalıcı talep/işleme koruması |
    | `plugin-sdk/acp-runtime` | Temmuz 2026'dan sonra özel-yerel; ACP çalışma zamanı/oturum ve yanıt yönlendirme yardımcıları |
    | `plugin-sdk/acp-runtime-backend` | Temmuz 2026'dan sonra özel-yerel; başlangıçta yüklenen plugin'ler için hafif ACP arka ucu kaydı ve yanıt yönlendirme yardımcıları |
    | `plugin-sdk/acp-binding-resolve-runtime` | Temmuz 2026'dan sonra özel-yerel; yaşam döngüsü başlatma içe aktarımları olmadan salt okunur ACP bağlama çözümlemesi |
    | `plugin-sdk/agent-config-primitives` | Kullanımdan kaldırılmış aracı çalışma zamanı yapılandırma şeması temel öğeleri; şema temel öğelerini bakımı yapılan, plugin'e ait bir yüzeyden içe aktarın |
    | `plugin-sdk/boolean-param` | Gevşek boole parametresi okuyucusu |
    | `plugin-sdk/dangerous-name-runtime` | Temmuz 2026'dan sonra özel-yerel; tehlikeli ad eşleştirmesi çözümleme yardımcıları |
    | `plugin-sdk/device-bootstrap` | `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` dâhil cihaz önyükleme ve eşleştirme belirteci yardımcıları |
    | `plugin-sdk/extension-shared` | Paylaşılan pasif kanal, durum ve ortam proxy yardımcısı temel öğeleri |
    | `plugin-sdk/models-provider-runtime` | `/models` komut/sağlayıcı yanıt yardımcıları |
    | `plugin-sdk/skill-commands-runtime` | Skill komutu listeleme yardımcıları |
    | `plugin-sdk/native-command-registry` | Yerel komut kayıt defteri/oluşturma/serileştirme yardımcıları |
    | `plugin-sdk/agent-harness` | Alt düzey aracı harness'ları için deneysel güvenilir plugin yüzeyi: harness türleri, etkin çalıştırmayı yönlendirme/iptal etme yardımcıları, OpenClaw araç köprüsü yardımcıları, çalışma zamanı planı araç ilkesi yardımcıları, terminal sonucu sınıflandırması, araç ilerlemesi biçimlendirme/ayrıntı yardımcıları ve deneme sonucu yardımcı programları |
    | `plugin-sdk/async-lock-runtime` | Temmuz 2026'dan sonra özel-yerel; küçük çalışma zamanı durum dosyaları için süreç içi eşzamansız kilit yardımcısı |
    | `plugin-sdk/channel-activity-runtime` | Temmuz 2026'dan sonra özel-yerel; kanal etkinliği telemetri yardımcısı |
    | `plugin-sdk/concurrency-runtime` | Temmuz 2026'dan sonra özel-yerel; sınırlı eşzamansız görev eşzamanlılığı yardımcısı |
    | `plugin-sdk/dedupe-runtime` | Bellek içi ve kalıcı depolama destekli yinelenenleri ayıklama önbelleği yardımcıları |
    | `plugin-sdk/delivery-queue-runtime` | Temmuz 2026'dan sonra özel-yerel; giden bekleyen teslimatları boşaltma yardımcısı |
    | `plugin-sdk/file-access-runtime` | Temmuz 2026'dan sonra özel-yerel; güvenli yerel dosya ve medya kaynağı yolu yardımcıları |
    | `plugin-sdk/heartbeat-runtime` | Temmuz 2026'dan sonra özel-yerel; Heartbeat uyandırma, olay ve görünürlük yardımcıları |
    | `plugin-sdk/expect-runtime` | Temmuz 2026'dan sonra özel-yerel; kanıtlanabilir çalışma zamanı değişmezleri için gerekli değer doğrulama yardımcısı |
    | `plugin-sdk/number-runtime` | Temmuz 2026'dan sonra özel-yerel; sayısal tür dönüştürme yardımcısı |
    | `plugin-sdk/secure-random-runtime` | Temmuz 2026'dan sonra özel-yerel; güvenli belirteç/UUID yardımcıları |
    | `plugin-sdk/system-event-runtime` | Temmuz 2026'dan sonra özel-yerel; sistem olayı kuyruğu yardımcıları |
    | `plugin-sdk/transport-ready-runtime` | Temmuz 2026'dan sonra özel-yerel; aktarım hazır olma durumu bekleme yardımcısı |
    | `plugin-sdk/exec-approvals-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş altyapı çalışma zamanı barrel'i olmadan yürütme onay ilkesi dosyası yardımcıları |
    | `plugin-sdk/infra-runtime` | Kullanımdan kaldırılmış uyumluluk shim'i; yukarıdaki odaklanmış çalışma zamanı alt yollarını kullanın |
    | `plugin-sdk/collection-runtime` | Küçük sınırlı önbellek yardımcıları |
    | `plugin-sdk/diagnostic-runtime` | Tanılama bayrağı, olay ve izleme bağlamı yardımcıları |
    | `plugin-sdk/error-runtime` | Hata grafiği, biçimlendirme, bilinmeyen değer tür dönüştürme, paylaşılan hata sınıflandırma yardımcıları, `PlatformMessageNotDispatchedError`, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | Temmuz 2026'dan sonra özel-yerel; sarmalanmış fetch, proxy, EnvHttpProxyAgent seçeneği ve sabitlenmiş arama yardımcıları |
    | `plugin-sdk/runtime-fetch` | Temmuz 2026'dan sonra özel-yerel; proxy/korumalı fetch içe aktarımları olmadan yönlendiriciyi dikkate alan çalışma zamanı fetch'i |
    | `plugin-sdk/inline-image-data-url-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş medya çalışma zamanı yüzeyi olmadan satır içi görüntü veri URL'si temizleyicisi ve imza algılama yardımcıları |
    | `plugin-sdk/response-limit-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş medya çalışma zamanı yüzeyi olmadan bayt, boşta kalma ve son tarih sınırlı yanıt gövdesi okuyucuları |
    | `plugin-sdk/session-binding-runtime` | Temmuz 2026'dan sonra özel-yerel; yapılandırılmış bağlama yönlendirmesi veya eşleştirme depoları olmadan geçerli konuşma bağlama durumu |
    | `plugin-sdk/context-visibility-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş yapılandırma/güvenlik içe aktarımları olmadan bağlam görünürlüğü çözümleme ve ek bağlam filtreleme |
    | `plugin-sdk/string-coerce-runtime` | Markdown/günlükleme içe aktarımları olmadan dar kapsamlı temel kayıt/dize tür dönüştürme ve normalleştirme yardımcıları |
    | `plugin-sdk/html-entity-runtime` | Temmuz 2026'dan sonra özel-yerel; geniş metin yardımcı programları olmadan tek geçişli, noktalı virgülle sonlandırılan HTML5 varlık kodu çözme |
    | `plugin-sdk/text-utility-runtime` | Temmuz 2026'dan sonra özel-yerel; beş varlıklı HTML kaçışını içeren düşük seviyeli metin ve yol yardımcıları |
    | `plugin-sdk/widget-html` | Bağımsız HTML widget'ları için tam belge algılama, boyut doğrulama ve araç girdisi hataları |
    | `plugin-sdk/host-runtime` | Temmuz 2026'dan sonra özel-yerel; ana makine adı ve SCP ana makinesi normalleştirme yardımcıları |
    | `plugin-sdk/retry-runtime` | Temmuz 2026'dan sonra özel-yerel; yeniden deneme yapılandırması ve yeniden deneme çalıştırıcısı yardımcıları |
    | `plugin-sdk/agent-runtime` | `resolveAgentDir`, `resolveDefaultAgentDir` ve kullanımdan kaldırılmış `resolveOpenClawAgentDir` uyumluluk dışa aktarımı dahil olmak üzere agent dizini/kimliği/çalışma alanı yardımcıları için kullanımdan kaldırılmış geniş kapsamlı barrel; odaklanmış agent/runtime alt yollarını tercih edin |
    | `plugin-sdk/directory-runtime` | Yapılandırma destekli dizin sorgulama/yinelenenleri kaldırma |
    | `plugin-sdk/keyed-async-queue` | Temmuz 2026'dan sonra özel-yerel; `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Yetenek ve test alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/media-runtime` | `saveRemoteMedia`, `saveResponseMedia`, `readRemoteMediaBuffer` ve kullanımdan kaldırılmış `fetchRemoteMedia` öğelerini içeren, kullanımdan kaldırılmış geniş medya barrel'ı; `plugin-sdk/media-store`, `plugin-sdk/media-mime`, `plugin-sdk/outbound-media` ve yetenek çalışma zamanı alt yollarını tercih edin; ayrıca bir URL'nin OpenClaw medyasına dönüşmesi gerektiğinde arabellek okumalarından önce depo yardımcılarını tercih edin |
    | `plugin-sdk/media-local-roots` | Plugin'e ait yerel medya okumaları için odaklanmış `getAgentScopedMediaLocalRoots(...)` ve ilke duyarlı `getAgentScopedMediaLocalRootsForSources(...)` yardımcıları |
    | `plugin-sdk/media-mime` | Dar kapsamlı MIME normalleştirme, dosya uzantısı eşleme, MIME algılama ve medya türü yardımcıları |
    | `plugin-sdk/media-store` | `saveMediaBuffer` ve `saveMediaStream` gibi dar kapsamlı medya deposu yardımcıları |
    | `plugin-sdk/media-generation-runtime` | Temmuz 2026'dan sonra özel-yerel; paylaşılan medya oluşturma yük devretme yardımcıları, aday seçimi ve eksik model mesajları |
    | `plugin-sdk/media-understanding` | Medya anlama sağlayıcı türleri ve yardımcıları için kullanımdan kaldırılmış uyumluluk cephesi; yeni sağlayıcılar, eklenen Plugin API'si üzerinden kaydolur ve istek yardımcılarının sahipliği Plugin'de kalır |
    | `plugin-sdk/text-chunking` | Giden metin ve ofsetleri koruyan aralık parçalama, Markdown parçalama/işleme yardımcıları, alıntı duyarlı HTML etiketi tokenleştirme, Markdown tablosu dönüştürme, yönerge etiketi kaldırma ve güvenli metin yardımcı programları |
    | `plugin-sdk/speech` | Temmuz 2026'dan sonra özel-yerel; konuşma sağlayıcı türlerinin yanı sıra sağlayıcıya yönelik yönerge, kayıt defteri, doğrulama, OpenAI uyumlu TTS oluşturucu ve konuşma yardımcısı dışa aktarımları |
    | `plugin-sdk/speech-core` | Temmuz 2026'dan sonra özel-yerel; paylaşılan konuşma sağlayıcı türleri, kayıt defteri, yönerge, normalleştirme ve konuşma yardımcısı dışa aktarımları |
    | `plugin-sdk/speech-settings` | Sağlayıcı kayıt defterleri veya sentez çalışma zamanı olmadan hafif TTS yapılandırma çözümleme ve normalleştirme temel öğeleri |
    | `plugin-sdk/realtime-transcription` | Temmuz 2026'dan sonra özel-yerel; gerçek zamanlı transkripsiyon sağlayıcı türleri, kayıt defteri yardımcıları ve paylaşılan WebSocket oturum yardımcısı |
    | `plugin-sdk/realtime-bootstrap-context` | Temmuz 2026'dan sonra özel-yerel; sınırlı `IDENTITY.md`, `USER.md` ve `SOUL.md` bağlam ekleme için gerçek zamanlı profil önyükleme yardımcısı |
    | `plugin-sdk/realtime-voice` | Temmuz 2026'dan sonra özel-yerel; gerçek zamanlı ses sağlayıcı türleri, kayıt defteri yardımcıları, paylaşılan ses enerjisi/konuşma başlangıcı geçitleri ve taşıma bağımsız oturum test donanımı ile çıktı etkinliği takibi dâhil gerçek zamanlı ses davranışı yardımcıları |
    | `plugin-sdk/meeting-runtime` | Tarayıcı toplantısı oturum çalışma zamanı, gerçek zamanlı ses motorları/taşımaları, `MeetingPlatformAdapter`, tarayıcı/Node denetimi, agent danışması, sesli arama devri, kurulum kontrolleri ve SoX komut yardımcıları |
    | `plugin-sdk/image-generation` | Temmuz 2026'dan sonra özel-yerel; görüntü oluşturma sağlayıcı türlerinin yanı sıra görüntü varlığı/veri URL'si yardımcıları ve OpenAI uyumlu görüntü sağlayıcı oluşturucu |
    | `plugin-sdk/image-generation-core` | Temmuz 2026'dan sonra özel-yerel; paylaşılan görüntü oluşturma türleri, yük devretme, kimlik doğrulama ve kayıt defteri yardımcıları |
    | `plugin-sdk/music-generation` | Temmuz 2026'dan sonra özel-yerel; müzik oluşturma sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/video-generation` | Temmuz 2026'dan sonra özel-yerel; video oluşturma sağlayıcı/istek/sonuç türleri |
    | `plugin-sdk/video-generation-core` | Temmuz 2026'dan sonra özel-yerel; paylaşılan video oluşturma türleri, yük devretme yardımcıları, sağlayıcı arama ve model başvurusu ayrıştırma |
    | `plugin-sdk/transcripts` | Temmuz 2026'dan sonra özel-yerel; paylaşılan transkript kaynağı sağlayıcı türleri, kayıt defteri yardımcıları, toplantı sağlayıcısı köprü fabrikası, oturum tanımlayıcıları ve sözce meta verileri |
    | `plugin-sdk/webhook-targets` | Temmuz 2026'dan sonra özel-yerel; Webhook hedef kayıt defteri ve rota yükleme yardımcıları |
    | `plugin-sdk/web-media` | Paylaşılan uzak/yerel medya yükleme yardımcıları |
    | `plugin-sdk/zod` | Kullanımdan kaldırılmış uyumluluk yeniden dışa aktarımı; `zod` öğesini doğrudan `zod` üzerinden içe aktarın |
    | `plugin-sdk/plugin-test-api` | Depo test yardımcısı köprülerini içe aktarmadan doğrudan Plugin kaydı birim testleri için depo-yerel, asgari `createTestPluginApi` yardımcısı |
    | `plugin-sdk/agent-runtime-test-contracts` | Kimlik doğrulama, teslimat, geri dönüş, araç kancası, istem katmanı, şema ve transkript izdüşümü testleri için depo-yerel yerel agent çalışma zamanı bağdaştırıcı sözleşmesi fikstürleri |
    | `plugin-sdk/channel-test-helpers` | Genel eylem/kurulum/durum sözleşmeleri, dizin doğrulamaları, hesap başlatma yaşam döngüsü, gönderme yapılandırması iş parçacığı oluşturma, çalışma zamanı taklitleri, durum sorunları, giden teslimat ve kanca kaydı için depo-yerel kanal odaklı test yardımcıları |
    | `plugin-sdk/channel-target-testing` | Kanal testleri için depo-yerel paylaşılan hedef çözümleme hata durumu paketi |
    | `plugin-sdk/channel-contract-testing` | Geniş test barrel'ı olmadan depo-yerel dar kapsamlı kanal sözleşmesi test yardımcıları |
    | `plugin-sdk/plugin-test-contracts` | Depo-yerel Plugin paketi, kayıt, genel yapıt, doğrudan içe aktarma, çalışma zamanı API'si ve içe aktarma yan etkisi sözleşmesi yardımcıları |
    | `plugin-sdk/plugin-state-test-runtime` | Depo-yerel Plugin durum deposu, giriş kuyruğu ve durum veritabanı test yardımcıları |
    | `plugin-sdk/provider-test-contracts` | Depo-yerel sağlayıcı çalışma zamanı, kimlik doğrulama, keşif, başlangıç kurulumu, katalog, sihirbaz, medya yeteneği, yeniden oynatma ilkesi, gerçek zamanlı STT canlı ses, web arama/getirme ve akış sözleşmesi yardımcıları |
    | `plugin-sdk/provider-http-test-mocks` | Temmuz 2026'dan sonra özel-yerel; `plugin-sdk/provider-http` çalıştıran sağlayıcı testleri için depo-yerel, isteğe bağlı Vitest HTTP/kimlik doğrulama taklitleri |
    | `plugin-sdk/reply-payload-testing` | Yanıt yükü fikstürlerine meta veri eklemek için depo-yerel yardımcılar |
    | `plugin-sdk/sqlite-runtime-testing` | Birinci taraf testleri için depo-yerel SQLite yaşam döngüsü yardımcıları |
    | `plugin-sdk/test-fixtures` | Depo-yerel genel CLI çalışma zamanı yakalama, korumalı alan bağlamı, Skills yazıcısı, agent mesajı, sistem olayı, modül yeniden yükleme, paketlenmiş Plugin yolu, terminal metni, parçalama, kimlik doğrulama token'ı ve türü belirlenmiş durum fikstürleri |
    | `plugin-sdk/test-node-mocks` | Vitest `vi.mock("node:*")` fabrikalarında kullanılmak üzere depo-yerel odaklanmış Node yerleşik modül taklit yardımcıları |
  </Accordion>

  <Accordion title="Bellek alt yolları">
    | Alt yol | Temel dışa aktarımlar |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | Temmuz 2026'dan sonra özel-yerel; hafif bellek gömme sağlayıcısı kayıt defteri yardımcıları |
    | `plugin-sdk/memory-core-host-engine-foundation` | Bellek ana bilgisayarı temel motoru dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı gömme sözleşmeleri, kayıt defteri erişimi, yerel sağlayıcı ve genel toplu/uzak yardımcılar. Bu yüzeydeki `registerMemoryEmbeddingProvider` kullanımdan kaldırılmıştır; yeni sağlayıcılar için genel gömme sağlayıcısı API'sini kullanın. |
    | `plugin-sdk/memory-core-host-engine-qmd` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı QMD motoru dışa aktarımları |
    | `plugin-sdk/memory-core-host-engine-storage` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı depolama motoru dışa aktarımları |
    | `plugin-sdk/memory-core-host-secret` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı gizli bilgi yardımcıları |
    | `plugin-sdk/memory-core-host-status` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı durum yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-cli` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı CLI çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-core` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı çekirdek çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-core-host-runtime-files` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı dosya/çalışma zamanı yardımcıları |
    | `plugin-sdk/memory-host-core` | Sağlayıcıdan bağımsız bellek ana bilgisayarı yardımcıları için kullanımdan kaldırılmış uyumluluk cephesi. Yeni bellek Plugin'leri eklenen bellek yeteneklerini ve ana bilgisayar tarafından hazırlanan istemleri kullanır; eşlikçi Plugin'ler ise odaklanmış bir okuma bağlantısı bulunana kadar genel yapıt keşfi için korunan cepheyi kullanmaya devam eder. |
    | `plugin-sdk/memory-host-events` | Temmuz 2026'dan sonra özel-yerel; bellek ana bilgisayarı olay günlüğü yardımcıları için sağlayıcıdan bağımsız takma ad |
    | `plugin-sdk/memory-host-markdown` | Temmuz 2026'dan sonra özel-yerel; bellekle ilişkili Plugin'ler için paylaşılan yönetilen Markdown yardımcıları |
    | `plugin-sdk/memory-host-search` | Temmuz 2026'dan sonra özel-yerel; arama yöneticisi erişimi için Active Memory çalışma zamanı cephesi |
  </Accordion>

  <Accordion title="Ayrılmış paketlenmiş yardımcı alt yolları">
    Ayrılmış paketlenmiş yardımcı SDK alt yolları, paketlenmiş Plugin kodu için
    dar kapsamlı, sahibe özgü yüzeylerdir. Paket derlemelerinin ve
    takma adlandırmanın belirlenimci kalması için SDK envanterinde izlenirler,
    ancak genel Plugin geliştirme API'leri değildirler. Yeniden kullanılabilir
    yeni ana bilgisayar sözleşmeleri, `plugin-sdk/gateway-runtime` ve
    `plugin-sdk/ssrf-runtime` gibi genel SDK alt yollarını kullanmalıdır.

    | Alt yol | Sahip ve amaç |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | Temmuz 2026'dan sonra özel-yerel; kullanıcı MCP sunucusu yapılandırmasını Codex uygulama sunucusu iş parçacığı yapılandırmasına yansıtmak için paketlenmiş Codex Plugin yardımcısı (ayrılmış paket dışa aktarımı) |
    | `plugin-sdk/codex-native-task-runtime` | Codex uygulama sunucusunun yerel alt agent'larını OpenClaw görev durumuna yansıtmak için paketlenmiş Codex Plugin yardımcısı (yalnızca depo-yerel, paket dışa aktarımı değil) |

  </Accordion>
</AccordionGroup>

## İlgili

- [Plugin SDK'ye genel bakış](/tr/plugins/sdk-overview)
- [Plugin SDK kurulumu](/tr/plugins/sdk-setup)
- [Plugin oluşturma](/tr/plugins/building-plugins)
