---
read_when:
    - Je bouwt een nieuwe Plugin voor een berichtenkanaal
    - Je wilt OpenClaw verbinden met een berichtenplatform
    - Je moet het adapteroppervlak van ChannelPlugin begrijpen
sidebarTitle: Channel Plugins
summary: Stapsgewijze handleiding voor het bouwen van een berichtenkanaalplugin voor OpenClaw
title: Kanaalplugins bouwen
x-i18n:
    generated_at: "2026-07-27T06:01:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

Deze handleiding bouwt een channel-Plugin die OpenClaw verbindt met een
berichtenplatform: DM-beveiliging, koppeling, antwoordthreads en uitgaande berichten.

<Info>
  Nieuw met OpenClaw-plugins? Lees eerst [Aan de slag](/nl/plugins/building-plugins)
  voor de pakketstructuur en het instellen van het manifest.
</Info>

## Waar je plugin verantwoordelijk voor is

Channel-plugins implementeren geen tools voor verzenden/bewerken/reageren; de kern biedt één
gedeelde `message`-tool. Je plugin is verantwoordelijk voor:

- **Configuratie** - accountresolutie en installatiewizard
- **Beveiliging** - DM-beleid en toelatingslijsten
- **Koppeling** - DM-goedkeuringsflow
- **Sessiegrammatica** - hoe providerspecifieke gespreks-id's worden toegewezen aan basis-
  chats, thread-id's en terugvalopties voor bovenliggende items
- **Uitgaand** - tekst, media en peilingen naar het platform verzenden
- **Threading** - hoe antwoorden in threads worden geplaatst
- **Heartbeat-typindicatie** - optionele typ-/bezigsignalen voor Heartbeat-bezorgings-
  doelen

De kern is verantwoordelijk voor de gedeelde berichtentool, promptbedrading, de buitenste vorm van de sessiesleutel,
generieke `:thread:`-boekhouding en dispatch.

## Berichtadapter

Maak een `message`-adapter met `defineChannelMessageAdapter` uit
`openclaw/plugin-sdk/channel-outbound` beschikbaar. Declareer alleen de duurzame mogelijkheden voor definitieve verzending
die je native transport daadwerkelijk ondersteunt, onderbouwd door een contracttest
die het native neveneffect en het geretourneerde ontvangstbewijs aantoont. Laat tekst-/media-
verzendingen dezelfde transportfuncties gebruiken als de verouderde `outbound`-adapter. Zie voor
het volledige API-contract, de mogelijkhedenmatrix, regels voor ontvangstbewijzen, afronding van livevoorbeelden,
beleid voor ontvangstbevestigingen, tests en de migratietabel
[API voor uitgaande channel-berichten](/nl/plugins/sdk-channel-outbound).

Als je bestaande `outbound`-adapter al de juiste verzendmethoden en
mogelijkhedenmetadata heeft, leid dan de `message`-adapter af met
`createChannelMessageAdapterFromOutbound(...)` in plaats van handmatig nog een
brug te schrijven. Adapterverzendingen retourneren `MessageReceipt`-waarden. Leid verouderde id's
af met `listMessageReceiptPlatformIds(...)` of
`resolveMessageReceiptPrimaryId(...)` in plaats van parallelle `messageIds`-
velden te behouden.

Declareer live- en finalizermogelijkheden nauwkeurig - de kern gebruikt deze om te bepalen
wat een channel kan doen, en afwijkingen tussen het gedeclareerde en werkelijke gedrag leiden tot een
mislukte contracttest:

| Oppervlak                             | Waarden                                                                                          |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure`    |

Channels die een conceptvoorbeeld ter plaatse afronden, moeten de runtimelogica
via `defineFinalizableLivePreviewAdapter(...)` plus
`deliverWithFinalizableLivePreviewAdapter(...)` routeren en de gedeclareerde
mogelijkheden onderbouwen met `verifyChannelMessageLiveCapabilityAdapterProofs(...)`-
en `verifyChannelMessageLiveFinalizerProofs(...)`-tests, zodat native voorbeeld-,
voortgangs-, bewerkings-, terugval-/bewaar-, opschonings- en ontvangstbewijsgedrag niet
ongemerkt kunnen afwijken.

Inkomende ontvangers die platformbevestigingen uitstellen, moeten
`message.receive.defaultAckPolicy` en `supportedAckPolicies` declareren in plaats van
de timing van bevestigingen in lokale monitorstatus te verbergen. Dek elk gedeclareerd beleid af met
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)`.

Verouderde antwoordhelpers zoals `dispatchInboundReplyWithBase` en
`recordInboundSessionAndDispatchReply` blijven beschikbaar voor compatibiliteits-
dispatchers. Gebruik ze niet voor nieuwe channel-code; begin in plaats daarvan met de `message`-
adapter, ontvangstbewijzen en lifecyclehelpers voor ontvangst/verzending op
`openclaw/plugin-sdk/channel-outbound`.

### Inkomende toegang (experimenteel)

Channels die inkomende autorisatie migreren, kunnen het experimentele
`openclaw/plugin-sdk/channel-ingress-runtime`-subpad gebruiken vanuit runtime-ontvangst-
paden. Het accepteert platformfeiten, onbewerkte toelatingslijsten, routedescriptors, opdracht-
feiten en toegangsgroepconfiguratie, en retourneert vervolgens projecties voor afzender/route/opdracht/activering
plus de geordende toegangsgraaf, terwijl platformopzoekingen en neveneffecten
in de plugin blijven. Bewaar normalisatie van de pluginidentiteit in de
descriptor die je aan de resolver doorgeeft; serialiseer geen onbewerkte overeenkomende waarden uit
de opgeloste status of beslissing. Zie
[API voor inkomende channel-toegang](/nl/plugins/sdk-channel-ingress) voor het API-ontwerp,
de verantwoordelijkheidsgrens en testverwachtingen.

### Duurzame toegang en deduplicatie van herhalingen

Channels die duurzame toegang invoeren, moeten `createChannelIngressMonitor`
uit `openclaw/plugin-sdk/channel-outbound` gebruiken, tenzij ze een wezenlijk
ander toelatings- of pompcontract nodig hebben. Plaats de onbewerkte transportenvelop op één
centraal ontvangstpunt in de wachtrij (geen normalisatie tijdens ontvangst), laat voor Webhook-transporten
de transportbevestiging afhangen van de duurzame toevoeging, leid één
geserialiseerde baan per gesprek af en markeer de gebeurtenis bij overname door dispatch
als voltooid. De primaire sleutel van de wachtrij is `(queue_name, event_id)` en voltooiing
plaatst een tombstone op de rij in plaats van deze te verwijderen, zodat een late herbezorging door het platform van
dezelfde `event_id` duurzaam wordt geweigerd tijdens de bewaartermijn van de tombstone.
Zie [API voor uitgaande channel-berichten](/nl/plugins/sdk-channel-outbound#durable-ingress-monitors)
voor de monitor-API en het afsluitcontract.

Die tombstone is de gelaagdheidsregel voor replaybeveiligingen
(`openclaw/plugin-sdk/persistent-dedupe`): een geleegde channel behoudt alleen een afzonderlijke
replaybeveiliging wanneer de identiteit of bewaartermijn van de beveiliging die van de wachtrij
overtreft — een logische berichtsleutel die verschilt van het transportbezorgings-id (Telegram
dedupliceert `chat_id:message_id` omdat debounce-samenvoegingen een bericht opnieuw kunnen laten verschijnen
onder een nieuwe `update_id`), of een langer venster dan de tombstone-
bewaartermijn van de channel. Als je beveiligingssleutel gelijk zou zijn aan de `event_id` van de drain, verwijder dan de
beveiliging bij het invoeren van de drain en dimensioneer in plaats daarvan `completedTtlMs`/`completedMaxEntries`
zodat ze het oude beveiligingsvenster dekken. Beschermingen die geen deduplicatie uitvoeren, zoals leeftijds-
grenzen, vallen niet onder deze regel. Stabiele id's voor uitgaande berichten gebruiken het gedeelde
register voor uitgaande echo's uit `openclaw/plugin-sdk/channel-outbound` in plaats van een
channel-lokale TTL-cache.

#### Transportklassen en bewaring

Classificeer een transport aan de hand van de herstelgarantie bij de ontvangstgrens:

- **Webhook- of gebeurtenisbezorging met bevestigingsvoorwaarde:** bevestig of retourneer alleen succes
  na de duurzame toevoeging. Bij een mislukte toevoeging moet de bezorging in aanmerking blijven komen
  voor een nieuwe poging of moet de ontvangstgrens mislukken. Deze klasse omvat Slack, SMS, Zalo,
  Microsoft Teams, Google Chat, LINE en Synology Chat.
- **Afgewachte polling- of streambezorging:** verplaats de externe cursor of verzend de
  transportbevestiging pas na de toevoeging. Als er geen expliciete cursor bestaat, houd de
  ontvangstcallback dan geserialiseerd en afgewacht, zodat een mislukte toevoeging er niet toe kan leiden dat de
  ontvangstlus vooruitloopt. Telegram-polling, Signal en Tlon gebruiken deze klasse;
  Telegram-bezorging via een Webhook volgt de bovenstaande regel met bevestigingsvoorwaarde.
- **Sockets zonder herhalingsmogelijkheid:** IRC, Mattermost, Twitch en Zalo Personal kunnen het
  platform niet vragen een geaccepteerde gebeurtenis opnieuw te bezorgen. Hun duurzame wachtrij beschermt het
  venster voor procescrashes en ondersteunt lokaal herstel na herstart; voltooiings-
  tombstones zijn vrijwel inert tegen herhaling door het platform.

Gebruik 30 dagen als conventie voor de tombstone-TTL binnen de gehele vloot, niet als SDK-standaard. Een
herbezorgingsvenster met hoog volume gebruikt normaal een limiet van 20,000 voltooide items;
afgewachte transporten en transporten zonder herhalingsmogelijkheid met een lager volume gebruiken normaal 1,000-2,000.
Huidige uitzonderingen zijn onder meer de limieten van 4,096 items van LINE, de voltooide
TTL van 24 uur voor SMS en de uitsluitend op een limiet gebaseerde bewaring van voltooide items voor Tlon. Limieten voor mislukte rijen kunnen ook lager
zijn dan limieten voor voltooide rijen. Zowel TTL als limiet verwijderen rijen, dus effectieve bewaring eindigt
wanneer de eerste grens wordt bereikt. Wijk alleen af vanwege een gedocumenteerde horizon voor nieuwe platformpogingen,
een behouden uitgebracht venster van replaybeveiliging, verwacht volume of schijfbudget,
of een transport zonder herhalingsmogelijkheid, en dek het bewaarcontract af met tests.

#### Neveneffecten met minstens-eenmaalgarantie

Drain-dispatch voert neveneffecten van opdrachten uit voordat de toegangsrij zijn
voltooiingstombstone bereikt. Een procescrash tussen deze stappen herhaalt de rij en
kan het neveneffect opnieuw uitvoeren. Dit crashvenster met minstens-eenmaalgarantie is het
standaardcontract. Gebruik voor niet-idempotent werk, zoals configuratieschrijfacties, het wissen van
opslag of zichtbare bevestigingen buiten de antwoordbaan,
`createIngressEffectOnce(...)` uit
`openclaw/plugin-sdk/ingress-effect-once`. Geef elke aanroep de stabiele inkomende
`eventId` plus een effectnaam. Maak één helper per toegangswachtrij/account en
gebruik een stabiele, unieke `namespacePrefix` voor dat bereik, omdat transportgebeurtenis-
id's lokaal voor een wachtrij kunnen zijn. De helper commit zijn duurzame claim pas nadat het
effect is geslaagd; een effect dat een fout genereert, geeft de claim vrij zodat een nieuwe drainpoging
het opnieuw kan uitvoeren, terwijl gelijktijdige aanroepers op de actieve claim wachten. Fouten in duurzame
status roepen `onDiskError` aan wanneer deze is opgegeven en wijzen af in plaats van terug te
vallen op procesgeheugen.

Stel de `ttlMs` van de helper in op minstens de bewaartermijn van de toegangstombstone van de channel,
plus de maximale vertraging tussen het committen van het effect en het voltooien van de rij, inclusief
begrensde uitvaltijd en nieuwe drainpogingen. De TTL van de effectrecord begint bij de commit,
terwijl de bewaring van de tombstone later bij voltooiing begint; als de levensduur van een openstaande rij
onbegrensd is, dekt geen eindige TTL willekeurig lange uitvaltijd. Nadat de tombstone
de rij niet meer kan herhalen, zijn oudere effectrecords overbodig. Dimensioneer
`stateMaxEntries` voor elke afzonderlijke gebeurtenis-/effectsleutel die binnen dat
bewaarvenster kan bestaan, rekening houdend met de limiet voor voltooide items van de wachtrij en het
maximale aantal effecten per gebeurtenis. Een lagere limiet verwijdert de oudste record vóór het einde van de TTL
en maakt het mogelijk dat het effect opnieuw wordt uitgevoerd. Resterende vensters met minstens-eenmaalgarantie blijven
bestaan als het proces stopt of persistentie mislukt nadat het effect is geslaagd maar voordat
de claim wordt gecommit, of als de record verloopt terwijl de toegangsrij nog
openstaat.

#### Herstartcontract per account

Wijzigingen in de channel-configuratie herstarten standaard de volledige channel. Een channel met meerdere accounts
mag `reload.accountScopedRestart: true` alleen instellen wanneer configuratie-
resolutie gedeelde velden voor de hele channel plus het geselecteerde account leest, maar nooit een
naastgelegen account, en de Gateway één `(channel, accountId)`-
runtime kan stoppen en starten zonder naastgelegen runtimes te vervangen.

Het bereikgebonden pad is alleen van toepassing op wijzigingen onder
`channels.<channel>.accounts.<non-default-id>.*`. Wijzigingen aan gedeelde channel-
velden, `accounts.default`, verwijderde of niet-oplosbare accounts en gemengde wijzigingen
die overerving kunnen beïnvloeden, worden opgewaardeerd naar een herstart van de volledige channel. Plugins
die hier niet expliciet voor kiezen, gebruiken altijd het pad voor de volledige channel.

Voor channels die de duurzame toegangsdrain gebruiken, moet het stoppad van de accountmonitor
eerst alle geaccepteerde transporttoelatingen afhandelen en daarna de
drain verwijderen en afwachten. Bij het starten van het account wordt dezelfde accountgebonden wachtrij geopend, waarvan de eerste
drain niet-verzonden duurzame rijen herstelt. Voeg geen tweede herlaadspecifieke
herhalingsronde toe; wachtrijherstel is het canonieke herstartpad.

Behandel deze vlag als een claim op een mogelijkheid, niet als een prestatievoorkeur. Contract-
tests moeten aantonen dat het toevoegen en bewerken van één benoemd account de opgeloste
configuratie van een naastgelegen account ongewijzigd laat, dat het stoppen van één account alleen de
monitor en drain van dat account afhandelt en dat een nieuwe monitor de rijen van dat account precies
één keer herstelt. Als een garantie niet kan worden aangetoond, laat de vlag dan weg.

### Typindicatoren

Als je channel typindicatoren buiten inkomende antwoorden ondersteunt, maak dan
`heartbeat.sendTyping(...)` beschikbaar op de channel-plugin. De kern roept deze aan met het
opgeloste Heartbeat-bezorgingsdoel voordat de Heartbeat-modelrun begint en
gebruikt de gedeelde lifecycle voor het actief houden/opschonen van de typindicatie. Voeg
`heartbeat.clearTyping(...)` toe wanneer het platform een expliciet stopsignaal nodig heeft.

### Parameters voor mediabronnen

Als je channel parameters aan de berichtentool toevoegt die mediabronnen bevatten, maak dan
die parameternamen beschikbaar via `plugin.actions.describeMessageTool(...).mediaSourceParams`.
De kern gebruikt die expliciete lijst voor normalisatie van sandboxpaden en het beleid voor uitgaande
mediatoegang, zodat plugins geen speciale gevallen in de gedeelde kern nodig hebben voor
providerspecifieke parameters voor avatars, bijlagen of omslagafbeeldingen.

Geef de voorkeur aan een op acties gebaseerde map zoals `{ "set-profile": ["avatarUrl", "avatarPath"] }`,
zodat niet-gerelateerde acties de media-argumenten van een andere actie niet overnemen. Een platte array
werkt nog steeds voor parameters die bewust door elke beschikbare actie worden gedeeld.

Kanalen die een tijdelijke openbare URL beschikbaar moeten stellen voor het ophalen
van media aan de platformzijde, kunnen `createHostedOutboundMediaStore(...)` uit
`openclaw/plugin-sdk/outbound-media` gebruiken met Plugin-statusopslagen. Houd het parseren van
platformroutes en de handhaving van tokens in de kanaal-Plugin; de gedeelde helper
beheert alleen het laden van media, vervalmetadata, chunkrijen en opschoning.

Inkomende bijlagen gebruiken geordende feiten, geen parallelle `Media*`-velden. Normaliseer
kanaalrecords met `toInboundMediaFacts(...)` uit
`openclaw/plugin-sdk/channel-inbound` en geef ze door als `media` bij het opbouwen van de
inkomende context. Wanneer een Plugin lokale medialeesbewerkingen moet autoriseren, importeer je
`getAgentScopedMediaLocalRoots(...)` of
`getAgentScopedMediaLocalRootsForSources(...)` uit het gerichte
`openclaw/plugin-sdk/media-local-roots`-subpad. De oude
`agent-media-payload`-builder/rootfacade is verouderde compatibiliteit.

### Vormgeving van systeemeigen payloads

Als je kanaal providerspecifieke vormgeving nodig heeft voor `message(action="send")`,
geef je de voorkeur aan `actions.prepareSendPayload(...)`. Plaats systeemeigen kaarten, blokken, insluitingen of
andere duurzame gegevens onder `payload.channelData.<channel>` en laat de kern deze verzenden
via de adapter voor uitgaande berichten. Gebruik `actions.handleAction(...)` voor verzenden
alleen als compatibiliteitsterugval voor payloads die niet kunnen worden geserialiseerd en
opnieuw geprobeerd.

### Grammatica voor sessiegesprekken

Als je platform extra bereik opslaat in gespreks-id's, houd je die parsering
in de Plugin met `messaging.resolveSessionConversation(...)`. Dat is de
canonieke hook voor het toewijzen van `rawId` aan de basisgespreks-id, een optionele
thread-id, expliciete `baseConversationId` en eventuele
`parentConversationCandidates`. Wanneer je `parentConversationCandidates` retourneert,
orden je deze van de meest specifieke bovenliggende conversatie naar de breedste/basisconversatie.

`messaging.resolveParentConversationCandidates(...)` is een verouderde
compatibiliteitsterugval voor Plugins die alleen terugval naar bovenliggende conversaties nodig hebben boven op
de generieke/onbewerkte id. Als beide hooks bestaan, gebruikt de kern
eerst `resolveSessionConversation(...).parentConversationCandidates` en valt alleen
terug op `resolveParentConversationCandidates(...)` wanneer de canonieke
hook deze weglaat.

Gebundelde Plugins die dezelfde parsering nodig hebben voordat het kanaalregister opstart,
kunnen een `session-key-api.ts`-bestand op het hoogste niveau beschikbaar stellen met een overeenkomende
`resolveSessionConversation(...)`-export (zie de Feishu- en Telegram-
Plugins). De kern gebruikt dat opstartveilige oppervlak alleen wanneer het runtime-Pluginregister
nog niet beschikbaar is.

Gebruik `openclaw/plugin-sdk/channel-route` wanneer Plugincode routeachtige
velden moet normaliseren, een onderliggende thread met de bovenliggende route moet vergelijken of een
stabiele deduplicatiesleutel uit `{ channel, to, accountId, threadId }` moet opbouwen. De helper
normaliseert numerieke thread-id's op dezelfde manier als de kern, dus geef hieraan de voorkeur boven ad-hoc
`String(threadId)`-vergelijkingen. Plugins met providerspecifieke doelsyntaxis
moeten `messaging.resolveOutboundSessionRoute(...)` beschikbaar stellen, zodat de kern
providerspecifieke sessie- en threadidentiteit krijgt zonder parsershims.

### Ondersteuning voor accountgebonden gesprekskoppelingen

Stel `conversationBindings.supportsCurrentConversationBinding` in wanneer het kanaal
generieke koppelingen voor de huidige conversatie ondersteunt. `createChatChannelPlugin(...)`
stelt deze statische mogelijkheid standaard in op `true`.

Als de ondersteuning per geconfigureerd account verschilt, implementeer je ook
`conversationBindings.isCurrentConversationBindingSupported({ accountId })`.
De kern evalueert deze synchrone hook pas nadat de statische mogelijkheid is
ingeschakeld. Door `false` te retourneren, worden generieke bewerkingen voor mogelijkheden,
koppelen, opzoeken, weergeven, bijwerken en ontkoppelen van de huidige conversatie niet beschikbaar voor dat account.
Als je de hook weglaat, geldt de statische mogelijkheid voor elk account.

Leid het antwoord af uit reeds geladen accountconfiguratie of runtimestatus. Deze
hook regelt alleen generieke koppelingen voor de huidige conversatie; deze vervangt geen
geconfigureerde koppelingsregels of sessieroutering die eigendom is van de Plugin. Contracttests
moeten ten minste één ondersteund en één niet-ondersteund account behandelen via het
`ChannelPlugin["conversationBindings"]`-contract dat wordt geëxporteerd door
`openclaw/plugin-sdk/channel-core`.

## Goedkeuringen en kanaalmogelijkheden

De meeste kanaal-Plugins hebben geen goedkeuringsspecifieke code nodig. De kern beheert dezelfde-chat-
`/approve`, gedeelde payloads voor goedkeuringsknoppen en generieke terugvalbezorging.
`ChannelPlugin.approvals` is verwijderd; plaats feiten over goedkeuringsbezorging, systeemeigen gedrag, rendering en autorisatie
in plaats daarvan op één `approvalCapability`-object. `plugin.auth` is alleen voor inloggen/uitloggen
— de kern leest geen goedkeuringsautorisatiehooks meer uit dat object.

Gebruik `approvalCapability.delivery` alleen voor systeemeigen goedkeuringsroutering of het
onderdrukken van terugval, en `approvalCapability.render` alleen wanneer een kanaal werkelijk
aangepaste goedkeuringspayloads nodig heeft in plaats van de gedeelde renderer.

### Goedkeuringsautorisatie

- `approvalCapability.authorizeActorAction` en
  `approvalCapability.getActionAvailabilityState` vormen het canonieke
  raakvlak voor goedkeuringsautorisatie.
- Gebruik `getActionAvailabilityState` voor de beschikbaarheid van goedkeuringsautorisatie in dezelfde chat.
  Houd geconfigureerde goedkeurders beschikbaar voor `/approve`, zelfs wanneer systeemeigen bezorging
  is uitgeschakeld; gebruik in plaats daarvan de status van het systeemeigen initiërende oppervlak voor richtlijnen over bezorging/installatie.
- Als je kanaal systeemeigen uitvoeringsgoedkeuringen beschikbaar stelt, gebruik je
  `approvalCapability.getExecInitiatingSurfaceState` voor de
  status van het initiërende oppervlak/de systeemeigen client wanneer deze afwijkt van goedkeuringsautorisatie
  in dezelfde chat. De kern gebruikt die uitvoeringsspecifieke hook om onderscheid te maken tussen `enabled` en
  `disabled`, te bepalen of het initiërende kanaal systeemeigen uitvoeringsgoedkeuringen
  ondersteunt en het kanaal op te nemen in terugvalrichtlijnen voor systeemeigen clients.
  `createApproverRestrictedNativeApprovalCapability(...)` vult dit in voor
  het gebruikelijke geval.
- Als een kanaal stabiele eigenaarachtige DM-identiteiten uit bestaande configuratie kan afleiden,
  gebruik je `createResolvedApproverActionAuthAdapter` uit
  `openclaw/plugin-sdk/approval-runtime` om dezelfde-chat-`/approve`
  te beperken zonder goedkeuringsspecifieke kernlogica toe te voegen.
- Als aangepaste goedkeuringsautorisatie bewust alleen terugval binnen dezelfde chat toestaat, retourneer je
  `markImplicitSameChatApprovalAuthorization({ authorized: true })` uit
  `openclaw/plugin-sdk/approval-auth-runtime`; anders behandelt de kern het
  resultaat als expliciete autorisatie van de goedkeurder.
- Als een systeemeigen callback die eigendom is van het kanaal goedkeuringen rechtstreeks afhandelt, gebruik je
  `isImplicitSameChatApprovalAuthorization(...)` vóór het afhandelen, zodat impliciete
  terugval nog steeds via de normale actorautorisatie van het kanaal verloopt.

### Levenscyclus van payloads en installatierichtlijnen

- Gebruik `outbound.shouldSuppressLocalPayloadPrompt` of
  `outbound.beforeDeliverPayload` voor kanaalspecifiek gedrag van de payloadlevenscyclus,
  zoals het verbergen van dubbele lokale goedkeuringsprompts of het verzenden van typindicatoren
  vóór bezorging.
- Gebruik `approvalCapability.describeExecApprovalSetup` wanneer het kanaal wil
  dat het antwoord voor het uitgeschakelde pad precies uitlegt welke configuratie-instellingen nodig zijn om
  systeemeigen uitvoeringsgoedkeuringen in te schakelen. De hook ontvangt `{ channel, channelLabel, accountId }`;
  kanalen met benoemde accounts moeten accountgebonden paden weergeven, zoals
  `channels.<channel>.accounts.<id>.execApprovals.*`, in plaats van standaardwaarden
  op het hoogste niveau.
- Gebruik `approvalCapability.describePluginApprovalSetup` wanneer richtlijnen bij mislukte Plugin-
  goedkeuringen veilig kunnen worden weergegeven voor geen-route- en time-outfouten bij Plugin-goedkeuringen.
  `createApproverRestrictedNativeApprovalCapability(...)` leidt
  dit niet af uit `describeExecApprovalSetup`; geef dezelfde helper alleen expliciet door
  wanneer Plugin- en uitvoeringsgoedkeuringen werkelijk dezelfde systeemeigen installatie gebruiken.

### Systeemeigen goedkeuringsbezorging

Als een kanaal systeemeigen goedkeuringsbezorging nodig heeft, houd je de kanaalcode gericht op
doelnormalisatie plus transport-/presentatiefeiten. Gebruik
`createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`,
`createChannelApproverDmTargetResolver` en
`createApproverRestrictedNativeApprovalCapability` uit
`openclaw/plugin-sdk/approval-runtime`. Plaats de kanaalspecifieke feiten achter
`approvalCapability.nativeRuntime`, bij voorkeur via
`createChannelApprovalNativeRuntimeAdapter(...)` of
`createLazyChannelApprovalNativeRuntimeAdapter(...)`, zodat de kern de
handler kan samenstellen en aanvraagfiltering, routering, deduplicatie, verval, Gateway-
abonnementen en meldingen over routering naar elders kan beheren.

`nativeRuntime` is opgesplitst in enkele kleinere raakvlakken:

- `availability` - of het account is geconfigureerd en of een aanvraag
  moet worden afgehandeld
- `presentation` - het gedeelde goedkeuringsweergavemodel omzetten in
  wachtende/afgehandelde/verlopen systeemeigen payloads of definitieve acties
- `transport` - doelen voorbereiden en systeemeigen goedkeuringsberichten verzenden/bijwerken/verwijderen
- `interactions` - optionele hooks voor koppelen/ontkoppelen/acties wissen voor systeemeigen knoppen
  of reacties, plus een optionele `cancelDelivered`-hook. Implementeer
  `cancelDelivered` wanneer `deliverPending` status binnen het proces of persistente
  status registreert (zoals een opslag voor reactiedoelen), zodat die status kan worden vrijgegeven als het
  stoppen van een handler de bezorging annuleert voordat `bindPending` wordt uitgevoerd, of wanneer
  `bindPending` geen handle retourneert
- `observe` - optionele hooks voor bezorgingsdiagnostiek

Andere goedkeuringshelpers:

- Gebruik `createNativeApprovalChannelRouteGates` uit
  `openclaw/plugin-sdk/approval-native-runtime` wanneer een kanaal zowel
  sessiegebonden systeemeigen bezorging als expliciete doorstuurdoelen voor goedkeuring ondersteunt. De
  helper centraliseert de selectie van goedkeuringsconfiguratie, afhandeling van `mode`, agent-/sessie-
  filters, accountkoppeling, overeenkomsten met sessiedoelen en overeenkomsten met doellijsten,
  terwijl aanroepers verantwoordelijk blijven voor de kanaal-id, standaarddoorstuurmodus, het
  opzoeken van accounts, controle of transport is ingeschakeld, doelnormalisatie en het bepalen van
  het doel uit de beurtbron. Gebruik deze niet om kanaalbeleidsstandaarden te maken die eigendom zijn van de kern;
  geef de gedocumenteerde standaardmodus van het kanaal expliciet door.
- `createChannelNativeOriginTargetResolver` gebruikt standaard de gedeelde matcher voor kanaalroutes
  voor `{ to, accountId, threadId }`-doelen. Geef
  `targetsMatch` alleen door wanneer een kanaal providerspecifieke equivalentiegels heeft,
  zoals prefixvergelijking voor Slack-tijdstempels. Geef `normalizeTargetForMatch` door wanneer
  het kanaal provider-id's moet canonicaliseren voordat de standaardroutematcher
  of een aangepaste `targetsMatch`-callback wordt uitgevoerd, terwijl het
  oorspronkelijke doel voor bezorging behouden blijft. Gebruik `normalizeTarget` alleen wanneer het opgeloste
  bezorgingsdoel zelf moet worden gecanonicaliseerd.
- Als het kanaal objecten nodig heeft die door de runtime worden beheerd, zoals een client, token, Bolt-
  app of Webhook-ontvanger, registreer je deze via
  `openclaw/plugin-sdk/channel-runtime-context`. Met het generieke runtimecontext-
  register kan de kern handlers op basis van mogelijkheden opstarten vanuit de opstartstatus van het kanaal,
  zonder goedkeuringsspecifieke wrapperlijm toe te voegen.
- Gebruik de lagere `createChannelApprovalHandler` of
  `createChannelNativeApprovalRuntime` alleen wanneer het op mogelijkheden gebaseerde raakvlak
  nog niet expressief genoeg is.
- Kanalen voor systeemeigen goedkeuringen moeten zowel `accountId` als `approvalKind`
  via die helpers routeren. `accountId` houdt goedkeuringsbeleid voor meerdere accounts
  beperkt tot het juiste botaccount, en `approvalKind` houdt het gedrag van uitvoerings- versus Plugin-
  goedkeuringen beschikbaar voor het kanaal zonder hardgecodeerde vertakkingen in
  de kern.
- De kern beheert ook meldingen over omgeleide goedkeuringen. Kanaal-Plugins mogen
  niet hun eigen vervolgberichten met "goedkeuring ging naar DM's / een ander kanaal" verzenden vanuit
  `createChannelNativeApprovalRuntime`; stel in plaats daarvan nauwkeurige routering voor oorsprong +
  goedkeurder-DM's beschikbaar via de gedeelde helpers voor goedkeuringsmogelijkheden en laat
  de kern daadwerkelijke bezorgingen samenvoegen voordat een melding terug naar de
  initiërende chat wordt geplaatst.
- Behoud het id-type van de bezorgde goedkeuring van begin tot eind. Systeemeigen clients mogen
  de routering van uitvoerings- versus Plugin-goedkeuringen niet raden of herschrijven vanuit kanaallokale
  status.
- Geef die expliciete `approvalKind` door aan `resolveApprovalOverGateway`. Dit gebruikt
  de canonieke `approval.resolve`-service en retourneert de geregistreerde winnaar wanneer
  een ander oppervlak als eerste antwoordt. De oudere expliciete `resolveMethod`-invoer
  blijft bestaan voor opdrachtgestuurde bedieningselementen; nieuwe systeemeigen acties mogen deze niet gebruiken of
  het type uit een id afleiden.
- Verschillende goedkeuringstypen kunnen bewust verschillende systeemeigen
  oppervlakken aanbieden. Huidige gebundelde voorbeelden: Matrix behoudt dezelfde systeemeigen DM-/kanaal-
  routering en reactie-UX voor uitvoerings- en Plugin-goedkeuringen, terwijl autorisatie
  nog steeds per goedkeuringstype kan verschillen; Slack houdt systeemeigen goedkeuringsroutering beschikbaar
  voor zowel uitvoerings- als Plugin-id's.
- `createApproverRestrictedNativeApprovalAdapter` bestaat nog steeds als
  compatibiliteitswrapper, maar nieuwe code moet de voorkeur geven aan de builder voor mogelijkheden
  en `approvalCapability` beschikbaar stellen op de Plugin.

### Smallere subpaden voor de goedkeuringsruntime

Geef voor veelgebruikte kanaalingangspunten de voorkeur aan deze smallere subpaden boven de bredere
`approval-runtime`-barrel wanneer je slechts één onderdeel van die familie nodig hebt:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Geef ook de voorkeur aan `openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` en
`openclaw/plugin-sdk/reply-chunking` boven bredere overkoepelende oppervlakken wanneer je
ze niet allemaal nodig hebt.

### Subpaden voor installatie

- `openclaw/plugin-sdk/setup-runtime` omvat de runtime-veilige installatiehelpers:
  `createSetupTranslator`, importveilige adapters voor installatiepatches
  (`createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), uitvoer van opzoeknotities,
  `promptResolvedAllowFrom`, `splitSetupEntries` en de gedelegeerde
  builders voor installatieproxy's.
- `openclaw/plugin-sdk/channel-setup` omvat de installatiebuilders voor optionele installaties
  plus enkele installatieveilige primitieven: `createOptionalChannelSetupSurface`,
  `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`,
  `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`,
  `setSetupChannelEnabled` en `splitSetupEntries`.
- Gebruik de bredere `openclaw/plugin-sdk/setup`-naad alleen wanneer je ook
  de zwaardere gedeelde installatie-/configuratiehelpers nodig hebt, zoals
  `moveSingleAccountChannelSectionToDefaultAccount(...)`.

Als je kanaal in installatieoppervlakken alleen „installeer eerst deze plugin” wil
vermelden, geef dan de voorkeur aan `createOptionalChannelSetupSurface(...)`. De gegenereerde
adapter/wizard weigert veilig configuratieschrijfbewerkingen en voltooiing, en hergebruikt
dezelfde melding over de vereiste installatie voor validatie, voltooiing en tekst
bij de documentatielink.

Als je kanaal omgevingsgestuurde installatie of authenticatie ondersteunt, stel je dit beschikbaar via het
configuratieschema en de installatiebeschrijvingen van het kanaal. Houd `envVars` van de kanaalruntime of
lokale constanten uitsluitend voor tekst die voor operators bestemd is.

Als je kanaal in `status`, `channels list`, `channels status` of
SecretRef-scans kan verschijnen voordat de pluginruntime start, voeg dan `openclaw.setupEntry` toe in
`package.json`. Dat toegangspunt moet veilig te importeren zijn in alleen-lezen opdrachtpaden
en moet de kanaalmetadata, de installatieveilige configuratieadapter,
statusadapter en metadata van kanaalgeheimdoelen retourneren die voor deze
samenvattingen nodig zijn. Start geen clients, listeners of transportruntimes vanuit het
installatietoegangspunt.

Houd ook het importpad van het hoofdkanaaltoegangspunt beperkt. Detectie kan
het toegangspunt en de kanaalpluginmodule evalueren om mogelijkheden te registreren zonder
het kanaal te activeren. Bestanden zoals `channel-plugin-api.ts` moeten
het kanaalpluginobject exporteren zonder installatiewizards, transportclients,
socketlisteners, starters van sub-processen of modules voor het starten van services te importeren.
Plaats die runtimeonderdelen in modules die vanuit `registerFull(...)`, runtime-
setters of luie mogelijkheidadapters worden geladen.

### Andere beperkte kanaalsubpaden

Geef voor andere intensief gebruikte kanaalpaden de voorkeur aan beperkte helpers boven bredere verouderde
oppervlakken:

- `openclaw/plugin-sdk/account-core`, `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` en
  `openclaw/plugin-sdk/account-helpers` voor configuratie met meerdere accounts en
  terugval naar het standaardaccount
- `openclaw/plugin-sdk/inbound-envelope` en
  `openclaw/plugin-sdk/channel-inbound` voor bedrading van inkomende routes/enveloppen en
  registratie en verzending
- `openclaw/plugin-sdk/channel-targets` voor helpers voor het parseren van doelen
- `openclaw/plugin-sdk/channel-outbound` voor uitgaande identiteits-/verzenddelegates
  en planning van getypeerde payloads
- `buildThreadAwareOutboundSessionRoute(...)` uit
  `openclaw/plugin-sdk/channel-core` wanneer een uitgaande route een expliciete
  `replyToId`/`threadId` moet behouden of de huidige `:thread:`-
  sessie moet herstellen nadat de basissessiesleutel nog steeds overeenkomt. Providerplugins kunnen
  de prioriteit, het achtervoegselgedrag en de normalisatie van thread-id's overschrijven wanneer
  hun platform systeemeigen semantiek voor levering aan threads heeft.
- `openclaw/plugin-sdk/thread-bindings-runtime` voor de levenscyclus van threadbindingen
  en adapterregistratie

Kanalen die alleen authenticatie bieden, kunnen doorgaans bij het standaardpad stoppen: de kern verwerkt
goedkeuringen en de plugin stelt alleen uitgaande en authenticatiemogelijkheden beschikbaar. Kanalen met
systeemeigen goedkeuringen, zoals Matrix, Slack, Telegram en aangepaste chattransporten,
moeten de gedeelde systeemeigen helpers gebruiken in plaats van hun eigen levenscyclus voor
goedkeuringen te bouwen.

## Beleid voor inkomende vermeldingen

Houd de verwerking van inkomende vermeldingen verdeeld over twee lagen:

- verzameling van bewijs door de plugin
- evaluatie van gedeeld beleid

Gebruik `openclaw/plugin-sdk/channel-mention-gating` voor beslissingen over vermeldingsbeleid.
Gebruik `openclaw/plugin-sdk/channel-inbound` alleen wanneer je de bredere
barrel met inkomende helpers nodig hebt.

Geschikt voor lokale pluginlogica:

- detectie van antwoorden aan de bot
- detectie van geciteerde botberichten
- controles op deelname aan threads
- uitsluitingen van service-/systeemberichten
- platformspecifieke caches die nodig zijn om deelname van de bot aan te tonen

Geschikt voor de gedeelde helper:

- `requireMention`
- expliciet vermeldingsresultaat
- toelatingslijst voor impliciete vermeldingen
- opdrachtomzeiling
- definitieve beslissing om over te slaan

Voorkeursstroom:

1. Bereken lokale vermeldingsfeiten.
2. Geef deze feiten door aan `resolveInboundMentionDecision({ facts, policy })`.
3. Gebruik `decision.effectiveWasMentioned`, `decision.shouldBypassMention` en
   `decision.shouldSkip` in je inkomende poort.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` retourneert een booleaanse waarde. `hasAnyMention`,
`isExplicitlyMentioned` en `canResolveExplicit` zijn afkomstig uit de eigen
systeemeigen vermeldingsmetadata van het kanaal (berichtentiteiten, antwoord-aan-bot-vlaggen en vergelijkbare gegevens);
geef `false`/`undefined`-waarden op wanneer je platform ze niet kan detecteren.

`api.runtime.channel.mentions` stelt dezelfde gedeelde vermeldingshelpers beschikbaar voor
meegeleverde kanaalplugins die al afhankelijk zijn van runtime-injectie:
`buildMentionRegexes`, `matchesMentionPatterns`, `matchesMentionWithExplicit`,
`implicitMentionKindWhen`, `resolveInboundMentionDecision`.

Als je alleen `implicitMentionKindWhen` en `resolveInboundMentionDecision` nodig hebt,
importeer je deze uit `openclaw/plugin-sdk/channel-mention-gating` om te voorkomen dat
ongerelateerde inkomende runtimehelpers worden geladen.

## Stapsgewijze uitleg

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Pakket en manifest">
    Maak de standaardpluginbestanden. Het veld `channels` in
    `openclaw.plugin.json` (niet een veld `kind`) bepaalt dat een manifest
    eigenaar is van een kanaal. Zie voor het volledige oppervlak van pakketmetadata
    [Plugininstallatie en -configuratie](/nl/plugins/sdk-setup#openclaw-channel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Verbind OpenClaw met Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Kanaalplugin voor Acme Chat",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "Bottoken",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema` valideert `plugins.entries.acme-chat.config`. Gebruik dit voor
    instellingen waarvan de plugin eigenaar is en die niet de kanaalaccountconfiguratie zijn.
    `channelConfigs.acme-chat.schema` valideert `channels.acme-chat` en is de
    bron voor niet-kritieke paden die door configuratieschema-, installatie- en UI-oppervlakken wordt gebruikt voordat de
    pluginruntime wordt geladen. Zie [Pluginmanifest](/nl/plugins/manifest) voor de volledige
    referentie voor velden op het hoogste niveau.

  </Step>

  <Step title="Bouw het kanaalpluginobject">
    De interface `ChannelPlugin` heeft veel optionele adapteroppervlakken. Begin met
    het minimum: `id`, `config` en `setup`, en voeg adapters toe wanneer je ze
    nodig hebt.

    Maak `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    Gebruik voor kanalen die zowel canonieke DM-sleutels op het hoogste niveau als verouderde geneste sleutels accepteren, de helpers uit `plugin-sdk/channel-config-helpers`: `resolveChannelDmAccess`, `resolveChannelDmPolicy`, `resolveChannelDmAllowFrom` en `normalizeChannelDmPolicy` zorgen ervoor dat accountspecifieke waarden voorrang houden op overgenomen hoofdwaarden. Koppel dezelfde resolver via `normalizeLegacyDmAliases` aan doctor-reparatie, zodat de runtime en migratie hetzelfde contract lezen.

    <Accordion title="Wat createChatChannelPlugin voor je doet">
      In plaats van adapterinterfaces op laag niveau handmatig te implementeren, geef je
      declaratieve opties door en stelt de builder ze samen:

      | Optie | Wat ermee wordt gekoppeld |
      | --- | --- |
      | `security.dm` | Resolver voor DM-beveiliging met bereik op basis van configuratievelden |
      | `pairing.text` | Tekstgebaseerde DM-koppelingsflow met code-uitwisseling |
      | `threading` | Resolver voor de antwoordmodus (vast, accountgebonden of aangepast) |
      | `outbound.attachedResults` | Verzendfuncties die resultaatmetadata (bericht-ID's) retourneren; vereist een naastgelegen `channel`-id zodat de kern het geretourneerde bezorgingsresultaat kan vastleggen |

      Je kunt in plaats van de declaratieve opties ook onbewerkte adapterobjecten doorgeven
      als je volledige controle nodig hebt.

      Onbewerkte uitgaande adapters kunnen een `chunker(text, limit, ctx)`-functie definiëren.
      De optionele `ctx.formatting` bevat opmaaktbeslissingen voor het bezorgmoment,
      zoals `maxLinesPerMessage`; pas deze vóór het verzenden toe, zodat antwoordthreads
      en segmentgrenzen eenmaal door de gedeelde uitgaande bezorging worden bepaald.
      Verzendcontexten bevatten ook `replyToIdSource` (`implicit` of `explicit`)
      wanneer een systeemeigen antwoorddoel is bepaald, zodat payloadhelpers expliciete
      antwoordtags kunnen behouden zonder een impliciet, eenmalig te gebruiken antwoordslot te verbruiken.
    </Accordion>

    ### Adapters voor groepstoolbeleid

    Een kanaal dat `group.resolveToolPolicy` implementeert en
    `toolsBySender` ondersteunt, moet de volledige `ChannelGroupContext` doorsturen naar de
    gedeelde beleidsresolver. Respecteer in het bijzonder `senderPolicyMode: "never"`
    door afzenderspecifieke overlays over te slaan, zowel binnen het bereik van de overeenkomende groep als binnen het wildcardbereik,
    terwijl het basisbeleid `tools` nog steeds wordt toegepast.

    OpenClaw stelt deze modus alleen in voor vertrouwde uitvoering buiten de ingress om, waarbij de
    bevoegdheid van de afzender al is vastgelegd in een envelop die door de server wordt beheerd, zoals een
    expliciet begrensde geplande uitvoering. Plugins mogen de modus niet afleiden uit
    inkomende metagegevens, deze opslaan als kanaalstatus of beschikbaar stellen als configuratie. Voeg
    een adaptertest toe die bewijst dat de modus een wildcardvermelding `toolsBySender` overslaat
    zonder de overeenkomende basisbeperking `tools` te laten vervallen.

  </Step>

  <Step title="Koppel het toegangspunt">
    Maak `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat-kanaalplugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat-beheer");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat-beheer",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Plaats CLI-descriptors die eigendom zijn van het kanaal in `registerCliMetadata(...)`, zodat OpenClaw
    ze in de hoofdhulp kan tonen zonder de volledige kanaalruntime te activeren,
    terwijl normale volledige laadbewerkingen dezelfde descriptors blijven gebruiken voor de daadwerkelijke registratie van
    opdrachten. Reserveer `registerFull(...)` voor werk dat alleen tijdens runtime plaatsvindt.
    `defineChannelPluginEntry` verwerkt de splitsing van de registratiemodus automatisch.
    Als `registerFull(...)` Gateway-RPC-methoden registreert, gebruik dan een
    Plugin-specifiek voorvoegsel. De beheerdersnaamruimten van de kern (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) blijven gereserveerd en worden altijd
    omgezet naar `operator.admin`. Zie
    [Toegangspunten](/nl/plugins/sdk-entrypoints#definechannelpluginentry) voor alle
    opties.

  </Step>

  <Step title="Voeg een setup-toegangspunt toe">
    Maak `setup-entry.ts` voor lichtgewicht laden tijdens de onboarding:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw laadt dit in plaats van het volledige toegangspunt wanneer het kanaal is uitgeschakeld
    of niet is geconfigureerd. Zo wordt voorkomen dat zware runtimecode tijdens configuratieflows
    wordt geladen. Zie [Configuratie en instellingen](/nl/plugins/sdk-setup#setup-entry) voor meer informatie.

    Meegeleverde werkruimtekanalen die configuratieveilige exports opsplitsen in aanvullende
    modules, kunnen `defineBundledChannelSetupEntry(...)` uit
    `openclaw/plugin-sdk/channel-entry-contract` gebruiken wanneer ze ook een
    expliciete runtime-setter voor de configuratiefase nodig hebben.

  </Step>

  <Step title="Inkomende berichten verwerken">
    Je Plugin moet berichten van het platform ontvangen en doorsturen naar
    OpenClaw. Het gebruikelijke patroon is een Webhook die het verzoek verifieert en
    dit doorstuurt via de handler voor inkomende berichten van je kanaal:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // door de plugin beheerde authenticatie (verifieer zelf de handtekeningen)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Je handler voor inkomende berichten stuurt het bericht door naar OpenClaw.
          // De exacte koppeling hangt af van je platform-SDK -
          // bekijk een echt voorbeeld in het meegeleverde pluginpakket voor Microsoft Teams of Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      De verwerking van inkomende berichten is kanaalspecifiek. Elke kanaalplugin beheert
      zijn eigen pijplijn voor inkomende berichten. Bekijk de meegeleverde kanaalplugins
      (bijvoorbeeld het pluginpakket voor Microsoft Teams of Google Chat) voor praktijkvoorbeelden.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Testen">
Schrijf tests naast de broncode in `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    Zie [Testen](/nl/plugins/sdk-testing) voor gedeelde testhulpmiddelen.

</Step>
</Steps>

## Bestandsstructuur

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel-metagegevens
├── openclaw.plugin.json      # Manifest met configuratieschema
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Openbare exports (optioneel)
├── runtime-api.ts            # Interne runtime-exports (optioneel)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # API-client van het platform
    └── runtime.ts            # Runtime-opslag (indien nodig)
```

## Geavanceerde onderwerpen

<CardGroup cols={2}>
  <Card title="Opties voor threads" icon="git-branch" href="/nl/plugins/sdk-entrypoints#registration-mode">
    Vaste, accountgebonden of aangepaste antwoordmodi
  </Card>
  <Card title="Integratie van de berichtentool" icon="puzzle" href="/nl/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool en actiedetectie
  </Card>
  <Card title="Doelbepaling" icon="crosshair" href="/nl/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType, looksLikeId, reservedLiterals, resolveTarget
  </Card>
  <Card title="Runtimehelpers" icon="settings" href="/nl/plugins/sdk-runtime">
    TTS, STT, media en subagent via api.runtime
  </Card>
  <Card title="API voor inkomende kanaalgebeurtenissen" icon="bolt" href="/nl/plugins/sdk-channel-inbound">
    Gedeelde levenscyclus van inkomende gebeurtenissen: ontvangen, bepalen, vastleggen, doorsturen, voltooien
  </Card>
</CardGroup>

<Note>
Er bestaan nog enkele gebundelde helperinterfaces voor het onderhoud en de
compatibiliteit van gebundelde plugins. Dit is niet het aanbevolen patroon voor
nieuwe kanaalplugins; geef de voorkeur aan de generieke subpaden voor kanalen,
configuratie, antwoorden en de runtime van het gemeenschappelijke SDK-oppervlak,
tenzij je die gebundelde pluginfamilie rechtstreeks onderhoudt.
</Note>

## Volgende stappen

- [Providerplugins](/nl/plugins/sdk-provider-plugins) - als je plugin ook modellen aanbiedt
- [SDK-overzicht](/nl/plugins/sdk-overview) - volledig overzicht van imports via subpaden
- [SDK-tests](/nl/plugins/sdk-testing) - testhulpmiddelen en contracttests
- [Pluginmanifest](/nl/plugins/manifest) - volledig manifestschema

## Gerelateerd

- [Plugin SDK-configuratie](/nl/plugins/sdk-setup)
- [Plugins bouwen](/nl/plugins/building-plugins)
- [Plugins voor de agentharnas](/nl/plugins/sdk-agent-harness)
