---
read_when:
    - Je hebt het ondersteuningscontract voor de Codex-harnessruntime nodig
    - Je debugt native Codex-tools, hooks, compaction of feedbackuploads
    - Je wijzigt het gedrag van de Plugin in OpenClaw- en Codex-harnessturns
summary: Runtimegrenzen, hooks, tools, machtigingen en diagnostiek voor de Codex-harness
title: Runtime van Codex-harnas
x-i18n:
    generated_at: "2026-07-27T05:06:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d18d42683df0d827b776547f7b45f60f572cf39410d00533f53f8fdcdccb0d2
    source_path: plugins/codex-harness-runtime.md
    workflow: 16
---

Runtimecontract voor Codex-harnassbeurten. Zie voor installatie en routering
[Codex-harnas](/nl/plugins/codex-harness). Zie voor configuratievelden de
[Codex-harnasreferentie](/nl/plugins/codex-harness-reference).

## Overzicht

Codex beheert de systeemeigen modellus, het systeemeigen hervatten van threads, de systeemeigen voortzetting van tools
en systeemeigen Compaction. OpenClaw beheert kanaalroutering, sessiebestanden,
zichtbare berichtbezorging, dynamische OpenClaw-tools, goedkeuringen, mediabezorging
en een transcriptspiegel rond die grens.

Promptroutering volgt de geselecteerde runtime, niet alleen de providertekenreeks. Een
systeemeigen Codex-beurt krijgt ontwikkelaarsinstructies van de Codex-app-server; een expliciete
OpenClaw-compatibiliteitsroute behoudt de normale OpenClaw-systeemprompt, zelfs wanneer
deze OpenAI-authenticatie of -transport in Codex-stijl gebruikt.

OpenClaw start en hervat systeemeigen Codex-threads met de ingebouwde
persoonlijkheid van Codex uitgeschakeld (`personality: "none"`), zodat persoonlijkheidsbestanden in de werkruimte
en de OpenClaw-agentidentiteit gezaghebbend blijven. Systeemeigen Codex behoudt verder de door Codex beheerde
basis-/modelinstructies en het laden van projectdocumentatie. Lichtgewicht
OpenClaw-runs (bijvoorbeeld Cron) onderdrukken nog steeds het laden van projectdocumentatie.

OpenClaw-ontwikkelaarsinstructies behandelen runtimeaspecten van OpenClaw: bezorging via het bronkanaal,
dynamische OpenClaw-tools, ACP-delegatie, adaptercontext en de
actieve werkruimteprofielbestanden van de agent. Skill-catalogi en via tools gerouteerde
`MEMORY.md`-verwijzingen worden geprojecteerd als samenwerkingsinstructies voor ontwikkelaars
die alleen voor de beurt gelden. Wanneer geheugentools niet beschikbaar zijn, vallen actieve `BOOTSTRAP.md`-inhoud
en de volledige `MEMORY.md` in plaats daarvan terug op platte invoercontext voor de beurt.

De meeste dynamische OpenClaw-tools gebruiken de doorzoekbare `openclaw`-naamruimte. Tools
die als `catalogMode: "direct-only"` zijn gemarkeerd, gebruiken `openclaw_direct`, dat Codex
rechtstreeks zichtbaar houdt voor het model als `DirectModelOnly`, in plaats van het beschikbaar te maken voor geneste
Code Mode-uitvoering.

## Threadkoppelingen en modelwijzigingen

Wanneer een OpenClaw-sessie aan een bestaande Codex-thread is gekoppeld, verzendt de volgende
beurt het momenteel geselecteerde model, goedkeuringsbeleid, de sandbox,
goedkeuringsbeoordelaar en servicelaag opnieuw naar de app-server. Overschakelen van
`openai/gpt-5.5` naar `openai/gpt-5.2` behoudt de threadkoppeling, maar vraagt Codex om
door te gaan met het nieuw geselecteerde model.

Koppelingen onder toezicht vormen de uitzondering. De OpenClaw-modelkiezer blijft vergrendeld
en bij hervatten worden model- en provideroverrides weggelaten, zodat Codex het persistente model
en de persistente provider van de canonieke thread herstelt. Een afzonderlijk systeemeigen Codex-besturingselement kan
dat persistente paar wijzigen en de eerste momentopname kan de normale
waarschuwing van Codex over modelverschillen opleveren; het buitenste OpenClaw-model en de fallbackketen
vervangen geen van beide ooit.

## Toezicht en veilige voortzetting

Codex-toezicht is een optionele mogelijkheid van dezelfde `codex`-Plugin. Het detecteert
systeemeigen threads via een afzonderlijke verbinding en projecteert alleen niet-gearchiveerde
sessies in de Gateway-catalogus. Zonder expliciete `appServer`-verbindingsinstellingen
gebruikt die verbinding beheerde stdio vanuit de thuismap van de gebruiker, terwijl het gewone
harnas aan de agent gebonden blijft. Lijst- en metadata-lezingen zijn passief: ze
hervatten geen thread, abonneren OpenClaw niet op de livegebeurtenissen ervan en beantwoorden de
goedkeuringen ervan niet.

Voor een opgeslagen of inactieve sessie op de Gateway-computer maakt **Doorgaan als vertakking**
een normale, modelvergrendelde chat en spiegelt begrensde gebruikers- en assistentgeschiedenis
tot en met de laatste persistente, beëindigde beurt van de bron. De eerste normale
chatbeurt installeert de echte goedkeuringshandlers en gebruikt een tijdelijke systeemeigen fork
om de momentopname vast te zetten zonder model- of provideroverride. Codex App Server gebruikt
de huidige systeemeigen configuratie en retourneert het geselecteerde paar; deze geeft de
normale waarschuwing als dat model verschilt van het laatst geregistreerde model van de bron.
Op dezelfde toezichtsverbinding start OpenClaw de canonieke
Codex-harnass-thread met `appServer` als bron onder de cwd en het runtimebeleid ervan met
exact het geretourneerde model en de geretourneerde provider voor die eerste start, injecteert de
begrensde zichtbare geschiedenis en archiveert de tijdelijke fork. De bron wordt nooit
hervat. De canonieke thread beschikt over het volledige OpenClaw-harnastooloppervlak;
redeneringen, toolaanroepen en toolresultaten van de bron worden er niet naartoe gekloond.
Het bereik van de privéverbinding blijft behouden in wachtende en vastgelegde koppelingsstatussen, zodat
elke latere beurt op die verbinding blijft met systeemeigen authenticatie- en providerconfiguratie.
Uitgeschakeld toezicht of afwijking van koppeling/verbinding faalt gesloten
in plaats van over te schakelen naar het gewone harnas in de thuismap van de agent.

De oorspronkelijke CLI-, VS Code-, Atlas- of ChatGPT-bron blijft in aanmerking komen voor beide
catalogi. De canonieke vertakking is een systeemeigen Codex-thread, maar het brontype ervan is
`appServer`; systeemeigen clients kunnen dat brontype filteren, waardoor weergave ervan in
Codex Desktop niet is gegarandeerd.

Actieve bronnen kunnen geen nieuwe vertakking starten en niet worden gearchiveerd; een bestaande
chat onder toezicht kan nog steeds worden geopend. `notLoaded` betekent dat activiteit onbekend is, niet dat de bron inactief is;
OpenClaw staat archivering van een lokale `idle`- of `notLoaded`-rij alleen toe na expliciete
bevestiging dat er geen andere runner is en een nieuwe proceslokale statuslezing. Codex
serialiseert threadmutaties binnen één App Server-proces, maar biedt geen
exclusieve runner- of goedkeuringseigenaarslease tussen processen, waardoor die lezing niet kan
bewijzen dat een ander proces de thread niet gebruikt. OpenClaw blokkeert een bekende
actieve koppelingseigenaar voor het exacte doel of elke niet-gearchiveerde voortgebrachte afstammeling
die door de gepagineerde afstammelingenquery van Codex wordt geretourneerd. Enumeratiefouten, cycli en
uitputting van veiligheidslimieten falen gesloten. Systeemeigen archivering kan nog steeds wedijveren met een nieuwe beurt
in een ander proces, dus de bevestiging dekt onbekende clients en het gat tussen
statuslezing en archivering. Een modelvergrendelde chat onder toezicht kan niet worden verwijderd zolang
deze de systeemeigen koppeling beschermt.

Catalogi van gekoppelde Nodes blijven in de eerste release uitsluitend metadata bevatten. De huidige
Node-aanroepgrens werkt volgens verzoek/antwoord en kan de langlopende beurtgebeurtenissen,
goedkeuringsverzoeken of streaminguitvoer die vereist zijn voor een echte Codex-harnaskoppeling
niet overdragen. **Doorgaan** en **Archiveren** op afstand blijven daarom niet beschikbaar, zelfs
wanneer de rij inactief is.

Zie [Codex-toezicht](/plugins/codex-supervision) voor de installatie voor operators en het
zichtbare gedrag van de Control UI.

## Zichtbare antwoorden en Heartbeats

Directe chatbeurten of chatbeurten via de bron door het Codex-harnas gebruiken standaard automatische bezorging van het definitieve
assistentantwoord voor interne WebChat-oppervlakken, overeenkomstig het Pi-harnascontract:
de agent antwoordt normaal en OpenClaw plaatst de definitieve tekst in het
brongesprek. Stel `messages.visibleReplies: "message_tool"` in om
definitieve assistenttekst privé te houden, tenzij de agent `message(action="send")` aanroept.

Codex-Heartbeat-beurten krijgen standaard `heartbeat_respond` in de doorzoekbare OpenClaw-toolcatalogus,
zodat de agent kan vastleggen of de activering stil moet blijven
of een melding moet geven. Heartbeat-richtlijnen voor initiatief worden als ontwikkelaarsinstructie voor de samenwerkingsmodus van Codex
verzonden en gelden alleen voor de Heartbeat-beurt; gewone chatbeurten blijven
in de Codex Default-modus. Wanneer `HEARTBEAT.md` niet leeg is, verwijzen de Heartbeat-
instructies Codex naar het bestand in plaats van de inhoud ervan inline op te nemen.

## Hookgrenzen

| Laag                                  | Eigenaar                 | Doel                                                                |
| ------------------------------------- | ------------------------ | ------------------------------------------------------------------- |
| OpenClaw-pluginhooks                  | OpenClaw                 | Product-/plugincompatibiliteit tussen OpenClaw- en Codex-harnassen. |
| Codex-app-serverextensiemiddleware    | Gebundelde OpenClaw-plugins | Adaptergedrag per beurt rond dynamische OpenClaw-tools.           |
| Systeemeigen Codex-hooks              | Codex                    | Codex-levenscyclus op laag niveau en systeemeigen toolbeleid vanuit Codex-configuratie. |

OpenClaw gebruikt geen project- of globale Codex-`hooks.json`-bestanden om
plugingedrag te routeren. Voor de brug voor systeemeigen tools en machtigingen injecteert OpenClaw
Codex-configuratie per thread voor `PreToolUse`, `PostToolUse`, `PermissionRequest`
en `Stop`.

Wanneer Codex-app-servergoedkeuringen zijn ingeschakeld (`approvalPolicy` is niet
`"never"`), laat de standaard geïnjecteerde configuratie voor systeemeigen hooks `PermissionRequest`
weg, zodat de app-serverbeoordelaar van Codex en de goedkeuringsbrug van OpenClaw echte
escalaties na beoordeling afhandelen. Voeg `permission_request` toe aan
`nativeHookRelay.events` om het compatibiliteitsrelais toch af te dwingen. Andere Codex-
hooks, zoals `SessionStart` en `UserPromptSubmit`, blijven besturingselementen op Codex-niveau;
ze worden in het v1-contract niet beschikbaar gemaakt als OpenClaw-pluginhooks.

Voor dynamische OpenClaw-tools voert OpenClaw de tool uit nadat Codex om
de aanroep vraagt, zodat plugin- en middlewaregedrag in de harnasadapter wordt uitgevoerd. Codex
Code Mode ontvangt generieke dynamische resultaten als tekst en serialiseert geneste
dynamische aanroepen; aanroepers moeten op JSON lijkende resultaten parseren en kunnen niet vertrouwen op
`Promise.all` voor gelijktijdige indiening. Voor systeemeigen Codex-tools beheert Codex het
canonieke toolrecord; OpenClaw kan geselecteerde gebeurtenissen spiegelen, maar kan de
systeemeigen thread niet herschrijven, tenzij Codex dat beschikbaar maakt via app-server- of systeemeigen hook-
callbacks.

Codex-app-server-`PreToolUse`-gebeurtenissen in rapportagemodus stellen plugingoedkeuring uit tot de
overeenkomende app-servergoedkeuring. Als een OpenClaw-`before_tool_call`-hook
`requireApproval` retourneert terwijl de systeemeigen payload `openclaw_approval_mode:
"report"` instelt, registreert het systeemeigen hookrelais de vereiste plugingoedkeuring en
retourneert het geen systeemeigen beslissing. Wanneer Codex later het app-servergoedkeuringsverzoek
voor hetzelfde toolgebruik verzendt, opent OpenClaw de prompt voor plugingoedkeuring en
koppelt de beslissing terug aan Codex. Codex-`PermissionRequest`-gebeurtenissen vormen een
afzonderlijk goedkeuringstraject en kunnen nog steeds via OpenClaw-goedkeuringen worden gerouteerd wanneer
ze voor die brug zijn geconfigureerd.

Meldingen over Codex-app-serveritems bieden ook asynchrone `after_tool_call`-
waarnemingen voor voltooiingen van systeemeigen tools die nog niet door het systeemeigen
`PostToolUse`-relais worden gedekt. Deze dienen alleen voor telemetrie/compatibiliteit; ze kunnen de
systeemeigen toolaanroep niet blokkeren, vertragen of wijzigen.

Projecties van Compaction en de LLM-levenscyclus komen uit Codex-app-servermeldingen
en de status van de OpenClaw-adapter, niet uit systeemeigen Codex-hookopdrachten.
`before_compaction`, `after_compaction`, `llm_input` en `llm_output` zijn
waarnemingen op adapterniveau, geen byte-voor-byte vastleggingen van de interne
verzoek- of Compaction-payloads van Codex.

Systeemeigen Codex-`hook/started`- en `hook/completed`-app-servermeldingen worden
geprojecteerd als `codex_app_server.hook`-agentgebeurtenissen voor trajectregistratie en
foutopsporing. Ze roepen geen OpenClaw-pluginhooks aan.

## V1-ondersteuningscontract

Ondersteund in Codex-runtime v1:

| Oppervlak                                       | Ondersteuning                                                                          | Waarom                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenAI-modellus via Codex               | Ondersteund                                                                        | Codex app-server beheert de OpenAI-beurt, het native hervatten van threads en de native voortzetting van tools.                                                                                                                                                                                                                                                                                                                                                                                          |
| OpenClaw-kanaalroutering en -aflevering         | Ondersteund                                                                        | Telegram, Discord, Slack, WhatsApp, iMessage en andere kanalen blijven buiten de modelruntime.                                                                                                                                                                                                                                                                                                                                                                                    |
| Dynamische tools van OpenClaw                        | Ondersteund                                                                        | Codex vraagt OpenClaw om deze tools uit te voeren, zodat OpenClaw deel blijft uitmaken van het uitvoeringspad.                                                                                                                                                                                                                                                                                                                                                                                                |
| Prompt- en contextplugins                    | Ondersteund                                                                        | OpenClaw projecteert OpenClaw-specifieke prompts/context in de Codex-beurt, terwijl de door Codex beheerde basis-, model- en geconfigureerde projectdocumentprompts in het native Codex-pad blijven. OpenClaw schakelt de ingebouwde persoonlijkheid van Codex uit voor native threads, zodat persoonlijkheidsbestanden in de agentwerkruimte gezaghebbend blijven. Native Codex-ontwikkelaarsinstructies accepteren alleen opdrachtbegeleiding die expliciet is beperkt tot `codex_app_server`; verouderde globale opdrachthints blijven behouden voor niet-Codex-promptoppervlakken. |
| Levenscyclus van de contextengine                      | Ondersteund                                                                        | Samenstelling, opname en onderhoud na de beurt worden rondom Codex-beurten uitgevoerd. Contextengines vervangen native Codex-compaction niet.                                                                                                                                                                                                                                                                                                                                                        |
| Dynamische toolhooks                            | Ondersteund                                                                        | `before_tool_call`, `after_tool_call` en middleware voor toolresultaten worden rondom dynamische tools van OpenClaw uitgevoerd.                                                                                                                                                                                                                                                                                                                                                                          |
| Levenscyclushooks                               | Ondersteund als adapterwaarnemingen                                                | `llm_input`, `llm_output`, `agent_end`, `before_compaction` en `after_compaction` worden geactiveerd met waarheidsgetrouwe payloads voor de Codex-modus.                                                                                                                                                                                                                                                                                                                                                           |
| Revisiepoort voor het definitieve antwoord                    | Ondersteund via native hookdoorgifte                                              | Codex `Stop` wordt doorgegeven aan `before_agent_finalize`; `revise` vraagt Codex om nog één modeldoorgang vóór de afronding.                                                                                                                                                                                                                                                                                                                                                                |
| Native shell, patches en MCP blokkeren of observeren | Ondersteund via native hookdoorgifte                                              | Codex `PreToolUse` en `PostToolUse` worden doorgegeven voor vastgelegde native tooloppervlakken, inclusief MCP-payloads op Codex app-server `0.142.0` of nieuwer. Blokkeren wordt ondersteund; het herschrijven van argumenten niet.                                                                                                                                                                                                                                                                               |
| Native machtigingsbeleid                      | Ondersteund via Codex app-server-goedkeuringen en compatibele native hookdoorgifte | Goedkeuringsverzoeken van Codex app-server worden na Codex-beoordeling via OpenClaw gerouteerd. De native hookdoorgifte `PermissionRequest` is opt-in voor native goedkeuringsmodi, omdat Codex deze vóór de guardian-beoordeling uitzendt.                                                                                                                                                                                                                                                                          |
| Vastlegging van app-servertrajecten                 | Ondersteund                                                                        | OpenClaw registreert het verzoek dat het naar app-server heeft verzonden en de app-servermeldingen die het ontvangt.                                                                                                                                                                                                                                                                                                                                                                                    |

Niet ondersteund in Codex-runtime v1:

| Oppervlak                                             | V1-grens                                                                                                                                     | Toekomstig pad                                                                               |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Mutatie van native toolargumenten                       | Native Codex-hooks vóór tools kunnen blokkeren, maar OpenClaw herschrijft geen argumenten van Codex-native tools.                                               | Vereist ondersteuning in Codex-hooks/schema's voor vervangende toolinvoer.                            |
| Bewerkbare Codex-native transcriptgeschiedenis            | Codex beheert de canonieke native threadgeschiedenis. OpenClaw beheert een spiegel en kan toekomstige context projecteren, maar hoort niet-ondersteunde interne onderdelen niet te wijzigen. | Voeg expliciete Codex app-server-API's toe als ingrepen in native threads nodig zijn.                    |
| `tool_result_persist` voor Codex-native toolrecords | Die hook transformeert door OpenClaw beheerde transcriptwrites, niet Codex-native toolrecords.                                                           | Getransformeerde records kunnen worden gespiegeld, maar voor canoniek herschrijven is Codex-ondersteuning nodig.              |
| Uitgebreide native compactionmetadata                     | OpenClaw kan native compaction aanvragen, maar ontvangt geen stabiele lijst van behouden/verwijderde items, tokendelta, voltooiingssamenvatting of samenvattingspayload.   | Vereist uitgebreidere Codex-compactiongebeurtenissen.                                                     |
| Ingrijpen in compaction                             | OpenClaw laat plugins of contextengines native Codex-compaction niet blokkeren, herschrijven of vervangen.                                             | Voeg Codex-hooks vóór/na compaction toe als plugins native compaction moeten kunnen blokkeren of herschrijven. |
| Byte-voor-byte-vastlegging van model-API-verzoeken             | OpenClaw kan app-serververzoeken en -meldingen vastleggen, maar Codex core bouwt intern het uiteindelijke OpenAI-API-verzoek op.                      | Vereist een tracinggebeurtenis voor Codex-modelverzoeken of een debug-API.                                   |

## Native machtigingen en MCP-uitvragen

Voor `PermissionRequest` retourneert OpenClaw alleen expliciete toestaan- of weigerenbeslissingen
wanneer het beleid een beslissing neemt. Geen beslissing betekent niet toestaan: Codex
behandelt dit alsof de hook geen beslissing heeft genomen en valt terug op het eigen guardian- of
gebruikersgoedkeuringspad.

In de goedkeuringsmodi van Codex app-server wordt deze native hook standaard weggelaten. Dit
geldt tenzij `permission_request` expliciet is opgenomen in
`nativeHookRelay.events` of een compatibiliteitsruntime deze installeert.

Wanneer een operator `allow-always` kiest voor een native Codex-machtigingsverzoek,
onthoudt OpenClaw die exacte vingerafdruk van provider/sessie/toolinvoer/cwd
gedurende een begrensd sessievenster. De onthouden beslissing geldt
bewust alleen bij een exacte overeenkomst: een gewijzigde opdracht, gewijzigde argumenten, toolpayload of
cwd leidt tot een nieuwe goedkeuring.

Goedkeuringsuitvragen voor Codex MCP-tools worden via de plugingoedkeuringsflow
van OpenClaw gerouteerd wanneer Codex `_meta.codex_approval_kind` markeert als `"mcp_tool_call"`. Codex
`request_user_input` registreert een providerneutrale gatewayvraag voor de
sessie van oorsprong. De Control UI toont de gatewayvraagkaart en voor één
niet-geheime keuze worden getypeerde kanaalknoppen gebruikt wanneer het kanaal deze ondersteunt.
Tikacties op knoppen, antwoorden in de Control UI en het volgende antwoord in platte tekst in de wachtrij
lossen allemaal hetzelfde gatewayrecord op voordat OpenClaw het app-serverantwoord retourneert.
Automatische afhandeling door Codex en afgebroken pogingen begrenzen de wachttijd en annuleren het record.
Geheime vragen blijven volledig binnen het gewaarschuwde tekstantwoordpad. Andere MCP-
uitvraagverzoeken worden standaard geweigerd.

Zie voor de algemene plugingoedkeuringsflow die deze prompts verwerkt
[Pluginmachtigingsverzoeken](/nl/plugins/plugin-permission-requests).

## Wachtrijsturing

Sturing van de wachtrij voor actieve runs wordt gekoppeld aan Codex app-server `turn/steer`. Met de
standaardinstelling `messages.queue.mode: "steer"` bundelt OpenClaw chatberichten in
stuurmodus gedurende het geconfigureerde stille venster en verzendt deze als één
`turn/steer`-verzoek in volgorde van binnenkomst.

Codex-review- en handmatige Compaction-beurten kunnen sturing tijdens dezelfde
beurt weigeren. In dat geval wacht OpenClaw totdat de actieve run is voltooid
voordat de prompt wordt gestart. Gebruik `/queue followup` of
`/queue collect` wanneer berichten standaard in de wachtrij moeten worden
geplaatst in plaats van gestuurd. Zie [Sturingswachtrij](/nl/concepts/queue-steering).

## Codex-feedback uploaden

Wanneer `/diagnostics [note]` voor een sessie in de native Codex-harness is
goedgekeurd, roept OpenClaw ook Codex app-server `feedback/upload` aan voor
relevante Codex-threads, inclusief logboeken voor elke vermelde thread en
aangemaakte Codex-subthreads wanneer deze beschikbaar zijn.

De upload verloopt via het normale feedbackpad van Codex naar OpenAI-servers.
Als Codex-feedback in die app-server is uitgeschakeld, retourneert de opdracht
de app-serverfout. Het voltooide diagnostische antwoord vermeldt de kanalen,
OpenClaw-sessie-id's, Codex-thread-id's en lokale `codex resume <thread-id>`-opdrachten
voor de verzonden threads.

Als je de goedkeuring weigert of negeert, drukt OpenClaw die Codex-id's niet af
en verzendt het geen Codex-feedback. De upload vervangt de lokale export van
Gateway-diagnostiek niet. Zie [Diagnostiek exporteren](/nl/gateway/diagnostics) voor
het gedrag rond goedkeuring, privacy, de lokale bundel en groepschats.

Gebruik `/codex diagnostics [note]` alleen wanneer je de Codex-feedbackupload wilt
uitvoeren voor de momenteel gekoppelde thread zonder de volledige
Gateway-diagnostiekbundel.

## Compaction en transcriptspiegel

Wanneer het geselecteerde model de Codex-harness gebruikt, valt native
thread-Compaction onder Codex app-server. OpenClaw voert geen voorbereidende
Compaction uit voor Codex-beurten, vervangt Codex-Compaction niet door
Compaction van de contextengine en valt niet terug op samenvatting door
OpenClaw of de openbare OpenAI-service wanneer native Compaction niet kan
worden gestart. OpenClaw bewaart een transcriptspiegel voor kanaalgeschiedenis,
zoeken, `/new`, `/reset` en toekomstige wisselingen van
model of harness.

Expliciete Compaction-verzoeken, zoals `/compact` of een door een Plugin
aangevraagde handmatige compact-bewerking, starten native Codex-Compaction met
`thread/compact/start`. OpenClaw houdt het verzoek en de lease van de gedeelde
client open totdat Codex het overeenkomende voltooiingsitem
`contextCompaction` uitzendt en rapporteert de Compaction-beurt vervolgens als
voltooid. Als die afsluitende beurt de geconfigureerde Compaction-time-out
overschrijdt, vraagt OpenClaw om een native onderbreking van de beurt. De lease
en de Compaction-afscherming per thread blijven behouden totdat Codex een
eindstatus rapporteert of de onderbrekings-RPC bevestigt. Als Codex niet binnen
de respijtperiode voor de onderbreking bevestigt, stelt OpenClaw de verbinding
buiten gebruik voordat de afscherming wordt vrijgegeven. Externe verbindingen
ontkoppelen ook de bijbehorende threadbinding, zodat later werk niet kan
overlappen met een onbevestigde externe beurt. Andere beurten op een buiten
gebruik gestelde verbinding mislukken en kunnen het opnieuw proberen met een
nieuwe client. Het sluiten van de client, annuleren van het verzoek of een
mislukte Compaction-beurt retourneert een mislukte bewerking. Automatische
Compaction bij contextdruk is de taak van Codex; OpenClaw start native
Compaction alleen voor handmatig aangevraagde triggers.

Wanneer een contextengine om een Codex-projectie voor het initialiseren van
een thread vraagt, projecteert OpenClaw namen en id's van toolaanroepen,
invoervormen en geredigeerde inhoud van toolresultaten naar de nieuwe
Codex-thread. Het kopieert geen onbewerkte argumentwaarden van toolaanroepen
naar die projectie.

De spiegel bevat de gebruikersprompt, de definitieve assistenttekst en
lichtgewicht Codex-redenerings- of planrecords wanneer de app-server deze
uitzendt. OpenClaw registreert de start en eindstatus van de native Compaction,
maar stelt geen voor mensen leesbare Compaction-samenvatting of controleerbare
lijst beschikbaar van de vermeldingen die Codex na Compaction heeft behouden.

Omdat Codex eigenaar is van de canonieke native thread, herschrijft
`tool_result_persist` geen Codex-native toolresultaatrecords. Dit is alleen van
toepassing wanneer OpenClaw een toolresultaat schrijft naar een
sessietranscript waarvan OpenClaw eigenaar is.

## Media en aflevering

OpenClaw blijft verantwoordelijk voor media-aflevering en de selectie van de
mediaprovider. Afbeeldingen, video, muziek, PDF, TTS en mediabegrip gebruiken
overeenkomende provider-/modelinstellingen, zoals `agents.defaults.mediaModels.image`,
`agents.defaults.mediaModels.video`, `pdfModel` en `tts`.

Tekst, afbeeldingen, video, muziek, TTS, goedkeuringen en uitvoer van
berichtentools blijven via het normale afleverpad van OpenClaw verlopen;
mediageneratie vereist de verouderde runtime niet. Wanneer Codex een native
item voor afbeeldingsgeneratie uitzendt met een `savedPath`, stuurt
OpenClaw exact dat bestand door via het normale antwoordmediapad, zelfs als de
Codex-beurt geen assistenttekst bevat.

## Gerelateerd

- [Codex-harness](/nl/plugins/codex-harness)
- [Naslaginformatie voor de Codex-harness](/nl/plugins/codex-harness-reference)
- [Codex-supervisie](/plugins/codex-supervision)
- [Native Codex-plugins](/nl/plugins/codex-native-plugins)
- [Plugin-hooks](/nl/plugins/hooks)
- [Plugins voor agent-harnesses](/nl/plugins/sdk-agent-harness)
- [Diagnostiek exporteren](/nl/gateway/diagnostics)
- [Traject exporteren](/nl/tools/trajectory)
