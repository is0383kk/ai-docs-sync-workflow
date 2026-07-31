---
read_when:
    - Je hebt elk configuratieveld van de Codex-harness nodig
    - Je wijzigt het transport-, authenticatie-, detectie- of time-outgedrag van de app-server
    - Je debugt het opstarten van de Codex-harness, modeldetectie of omgevingsisolatie
summary: Referentie voor configuratie, authenticatie, detectie en app-server van de Codex-harness
title: Codex-harnasreferentie
x-i18n:
    generated_at: "2026-07-27T06:23:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

Deze referentie behandelt de gedetailleerde configuratie voor de officiële `codex`-plugin.
Begin voor installatie- en routeringsbeslissingen bij
[Codex-harnas](/nl/plugins/codex-harness).

## Configuratieoppervlak van de plugin

Alle instellingen voor het Codex-harnas bevinden zich onder `plugins.entries.codex.config`.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

Velden op het hoogste niveau:

| Veld                       | Standaard                | Betekenis                                                                                                                                          |
| -------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`         | ingeschakeld             | Instellingen voor modeldetectie voor Codex app-server `model/list`.                                                                           |
| `appServer`         | beheerde stdio-app-server | Instellingen voor transport, opdracht, authenticatie, goedkeuring, sandbox en time-out. Het gewone harnas gebruikt standaard agentspecifieke status. |
| `codexDynamicToolsLoading`         | `"searchable"`       | Gebruik `"direct"` om dynamische OpenClaw-tools rechtstreeks in de initiële Codex-toolcontext te plaatsen.                                  |
| `codexDynamicToolsExclude`         | `[]`       | Aanvullende namen van dynamische OpenClaw-tools die uit Codex app-server-beurten moeten worden weggelaten.                                          |
| `codexPlugins`         | uitgeschakeld            | Ondersteuning voor native Codex-plugins/apps, inclusief opt-in-toegang tot apps van verbonden accounts. Zie [Native Codex-plugins](/nl/plugins/codex-native-plugins). |
| `computerUse`         | uitgeschakeld            | Configuratie van Codex Computer Use. Zie [Codex Computer Use](/nl/plugins/codex-computer-use).                                                         |
| `sessionCatalog`         | ingeschakeld             | Detectie van native Codex-sessies voor de zijbalk. Stel `enabled: false` in om detectie uit te schakelen zonder de provider of het harnas uit te schakelen. |
| `supervision`         | uitgeschakeld            | Beleid voor transcripten en schrijfbeheer van native sessies dat zichtbaar is voor de agent. Zie [Codex-supervisie](/plugins/codex-supervision).    |

## Supervisie

Detectie van native sessies toont standaard niet-gearchiveerde Codex-sessies van de Gateway-
computer en aangemelde gekoppelde nodes. Schakel alleen die catalogus hiermee uit:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` beheert afzonderlijk de tools die zichtbaar zijn voor agents:

| Veld                  | Standaard                | Betekenis                                                                                                                                                                                                                                     |
| --------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`    | `false`       | Schakel Codex-supervisietools in die zichtbaar zijn voor agents. Dit beheert niet de catalogus voor geauthenticeerde operatorsessies.                                                                                                         |
| `endpoints`    | ingebouwd lokaal eindpunt | Compatibiliteits- en geavanceerde eindpuntdoelen voor de behouden Codex-supervisieagent en zelfstandige MCP-tools. De menselijke catalogus en branchflow negeren deze doelen en gebruiken de supervisie-App Server die vanuit `appServer` wordt opgelost. |
| `allowRawTranscripts`    | `false`       | Sta bij ingeschakelde supervisie autonome transcriptlezingen door agents of zelfstandige MCP en van transcripten afgeleide lijstvelden toe. Alleen-metadata-lezingen van `codex_threads` blijven beschikbaar. Beheert niet het geauthenticeerd voortzetten via de Control UI. |
| `allowWriteControls`    | `false`       | Sta bij ingeschakelde supervisie autonome `codex_threads`-mutaties voor forks, hernoemen, archiveren en dearchiveren toe, plus zelfstandige MCP-bewerkingen voor verzenden, bijsturen en onderbreken. Omzeilt geen andere controles voor binding, host, status of bevestiging. |

Eindpuntvermeldingen accepteren deze velden:

| Veld                   | Van toepassing op | Betekenis                                                               |
| ---------------------- | ------------------ | ----------------------------------------------------------------------- |
| `id`     | alle               | Stabiele eindpunt-id.                                                    |
| `label`     | alle               | Optioneel weergavelabel.                                                 |
| `transport`     | alle               | `"stdio-proxy"` of `"websocket"`.                                |
| `command`     | `stdio-proxy` | Optionele App Server-opdracht.                                           |
| `args`     | `stdio-proxy` | Optionele opdrachtargumenten.                                            |
| `cwd`     | `stdio-proxy` | Optionele werkmap voor het onderliggende proces.                         |
| `url`     | `websocket` | Vereiste WebSocket-URL of ondersteunde URL voor een lokale socket.       |
| `authTokenEnv`     | `websocket` | Optionele omgevingsvariabele waarvan de waarde het eindpunt authenticeert. |

De pagina **Codex-sessies** gebruikt de supervisie-App Server van de plugin en toont
alleen niet-gearchiveerde sessies. Zonder expliciete verbindingsinstellingen voor
`appServer` is die verbinding beheerde stdio vanuit de gebruikershome. Opgeslagen
of inactieve lokale rijen kunnen een modelvergrendelde Chat maken met een begrensde
geschiedenis van gebruiker en assistent tot en met de laatst opgeslagen afsluitende
bronbeurt. De privébinding houdt de snapshotfork, de canonieke branch uit de
`appServer`-bron, de geschiedenisinjectie en latere beurten op die verbinding.
Bij de eerste canonieke start wordt het door de fork geretourneerde paar gebruikt.
Bij latere hervattingen worden OpenClaw-model- en provideroverrides weggelaten, zodat
Codex het opgeslagen paar van de canonieke thread herstelt; een afzonderlijke native
wijziging kan dat paar bijwerken, maar het buitenste model en de fallbackketen vervangen
het nooit. Opgeslagen en inactieve rijen kunnen worden gearchiveerd na bevestiging dat er
geen andere runner is, tenzij een andere actieve OpenClaw-binding eigenaar is van het
exacte doel of van een van de niet-gearchiveerde voortgebrachte afstammelingen ervan.
OpenClaw volgt de paginering van afstammelingen van Codex en sluit bij opsommingsfouten,
cycli of uitputting van de veiligheidslimiet af met een fout. Bevestiging dekt nog steeds
onbekende native clients en de race tussen status en archivering. Een modelvergrendelde
Chat onder supervisie kan niet worden verwijderd zolang deze de native binding beschermt.
Actieve bronnen kunnen geen branch maken of worden gearchiveerd, maar een bestaande Chat
onder supervisie kan nog steeds worden geopend. Elke rij van een gekoppelde node blijft
alleen-lezen; het nodetransport biedt nog niet de streaminglevenscyclus die het harnas nodig heeft.

Alleen `appServer.homeScope: "user"` wijzigt welke Codex-home een beheerd harnasproces
gebruikt; dit publiceert de vlootcatalogus niet. Het inschakelen van supervisie wijzigt
de standaardinstelling van het harnas niet. In plaats daarvan gebruikt de afzonderlijke
supervisieverbinding standaard beheerde stdio vanuit de gebruikershome wanneer er geen
expliciete verbindingsinstellingen voor `appServer` bestaan. Expliciete instellingen
worden voor die verbinding gerespecteerd. Wachtende en vastgelegde bindingen onder supervisie
behouden die verbinding voor elke beurt; uitgeschakelde supervisie of afwijkingen in verbinding
of levenscyclus sluiten af met een fout in plaats van terug te vallen op het harnas vanuit de
agent-home. De standaardverbinding deelt opgeslagen sessies met native Codex-clients, niet hun
proceslokale activiteitsstatus.

Verouderde instellingen voor `plugins.entries.codex-supervisor` zijn ingetrokken. Voer
`openclaw doctor --fix` uit om de oude vermelding, eindpuntdefinities, beleidsvlaggen
en allow/deny-verwijzingen voor plugins naar dit blok te migreren. Expliciete canonieke
waarden voor `codex.config.supervision` hebben voorrang bij conflicten.

## App-servertransport

Voor gewone harnasbeurten start OpenClaw het beheerde Codex-binaire bestand dat
met de officiële plugin wordt meegeleverd (momenteel `@openai/codex` `0.145.0`):

```bash
codex app-server --listen stdio://
```

Hierdoor blijft de app-serverversie gekoppeld aan de officiële `codex`-plugin in plaats van
aan een afzonderlijke Codex CLI die toevallig lokaal is geïnstalleerd. Stel
`appServer.command` alleen in wanneer je bewust een ander uitvoerbaar bestand wilt gebruiken.
Gewone beheerde beurten met de standaard geïsoleerde agent-home geven de voorkeur aan dit
vastgezette pakket, zelfs wanneer een macOS-desktopbundel is geïnstalleerd. Wanneer
[Computer Use](/nl/plugins/codex-computer-use) is ingeschakeld, of wanneer `homeScope`
`"user"` is en native Computer Use-status kan laden, geeft beheerd opstarten in plaats
daarvan de voorkeur aan het binaire bestand van de desktopapp dat eigenaar is van de vereiste
macOS-machtigingen. Dezelfde desktop-eerst-regel geldt wanneer de effectieve Codex-configuratie
van een geïsoleerde agent-home native Computer Use inschakelt. Als er geen desktopappbundel is
geïnstalleerd, valt OpenClaw terug op het binaire bestand van het vastgezette pakket.

De overdracht van uitvoerbare bestanden en de afscherming van native configuratie coördineren
clients binnen één actief Gateway-proces. Start de Gateway opnieuw nadat een ander proces de
configuratie van de native Codex-plugin heeft gewijzigd.

Supervisie lost een afzonderlijke verbinding op. Zonder expliciete verbindingsinstellingen
voor `appServer` gebruikt deze beheerde stdio met `homeScope: "user"`;
het gewone harnas blijft beheerde stdio met `homeScope: "agent"`. Expliciete
verbindingsinstellingen worden door beide paden gerespecteerd. Stel `homeScope: "user"`
expliciet in wanneer het gewone harnas `$CODEX_HOME` (of `~/.codex`)
met native clients moet delen. Een privébinding onder supervisie gebruikt de
supervisieverbinding, ongeacht de standaardinstelling van het gewone harnas. Onafhankelijke
App Server-processen behouden afzonderlijke live status- en goedkeuringsstatussen.

Voor niet-productietests met een al actieve app-server is WebSocket-
transport beschikbaar:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex classificeert WebSocket-transport als experimenteel en niet ondersteund. Geef voor
productieworkloads de voorkeur aan beheerde stdio of de lokale Unix-besturingssocket.

Velden van `appServer`:

| Veld                                          | Standaard                                              | Betekenis                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                            | `"stdio"`                                     | `"stdio"` start Codex; expliciete `"unix"` maakt verbinding met de lokale besturingssocket; `"websocket"` maakt verbinding met `url`.                                                                                                                                                                                                                         |
| `homeScope`                            | `"agent"`                                     | `"agent"` isoleert de normale harness-status per OpenClaw-agent. `"user"` is een expliciete opt-in die de native `$CODEX_HOME` of `~/.codex` deelt, native authenticatie gebruikt en threadbeheer uitsluitend voor de eigenaar inschakelt. Het gebruikersbereik ondersteunt lokale stdio of Unix-transport. Voor de afzonderlijke supervisieverbinding wordt een niet-ingestelde waarde omgezet naar `"user"` voor stdio of Unix en `"agent"` voor WebSocket. |
| `command`                            | beheerd Codex-binair bestand                           | Uitvoerbaar bestand voor stdio-transport. Laat dit niet ingesteld om het beheerde binaire bestand te gebruiken.                                                                                                                                                                                                                                                                                  |
| `args`                            | `["app-server", "--listen", "stdio://"]`                                     | Argumenten voor stdio-transport.                                                                                                                                                                                                                                                                                                                                                                |
| `url`                            | niet ingesteld                                         | WebSocket-App Server-URL of `unix://`-URL. Een expliciet leeg Unix-pad selecteert de canonieke besturingssocket in de thuismap van de gebruiker.                                                                                                                                                                                                                                         |
| `authToken`                            | niet ingesteld                                         | Bearer-token voor WebSocket-transport. Accepteert een letterlijke tekenreeks of SecretInput, zoals `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                           |
| `headers`                            | `{}`                                     | Extra WebSocket-headers. Headerwaarden accepteren letterlijke tekenreeksen of SecretInput-waarden, bijvoorbeeld `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                                                              |
| `clearEnv`                            | `[]`                                     | Namen van extra omgevingsvariabelen die uit het gestarte stdio-app-serverproces worden verwijderd nadat OpenClaw de overgenomen omgeving heeft opgebouwd.                                                                                                                                                                                                                                       |
| `remoteWorkspaceRoot`                            | niet ingesteld                                         | Hoofdmap van de externe Codex-app-serverwerkruimte. Wanneer deze is ingesteld, leidt OpenClaw de lokale hoofdmap van de werkruimte af van de herleide OpenClaw-werkruimte, behoudt het de huidige cwd-suffix onder deze externe hoofdmap en stuurt het alleen de uiteindelijke app-server-cwd naar Codex. Als de cwd buiten de herleide hoofdmap van de OpenClaw-werkruimte ligt, weigert OpenClaw veilig in plaats van een Gateway-lokaal pad naar de externe app-server te sturen. |
| `loopDetectionPreToolUseRelay`                            | `true`                                     | Installeer het Codex-`PreToolUse`-subproces dat uitsluitend wordt gebruikt voor OpenClaw-lusdetectie en de expliciete markering dat er geen beleid is. Stel `false` in om het aantal processen per tool te verminderen. Plugin-hooks vóór tools en beleid voor vertrouwde tools installeren nog steeds hun vereiste relay.                                                           |
| `requestTimeoutMs`                            | `60000`                                     | Time-out voor aanroepen van het besturingsvlak van de app-server.                                                                                                                                                                                                                                                                                                                               |
| `turnCompletionIdleTimeoutMs`                            | `60000`                                     | Stille periode nadat Codex een beurt accepteert of na een app-serververzoek binnen een beurt, terwijl OpenClaw wacht op `turn/completed`.                                                                                                                                                                                                                                                      |
| `turnAssistantCompletionIdleTimeoutMs`                            | `10000`                                     | Stille periode nadat een definitief/niet-commentaaritem van de assistent of een onbewerkte assistentvoltooiing vóór een tool de vrijgave van assistentuitvoer activeert, terwijl OpenClaw nog wacht op `turn/completed`. Een hogere waarde geeft Codex meer tijd om `turn/completed` uit te voeren voordat OpenClaw onderbreekt en de sessiebaan vrijgeeft.                                           |
| `postToolRawAssistantCompletionIdleTimeoutMs`                            | `300000`                                     | Bewaking voor inactiviteit na voltooiing en voortgang, gebruikt na een tooloverdracht, voltooiing van een native tool, onbewerkte assistentvoortgang na een tool, voltooiing van onbewerkte redenering of voortgang van redenering terwijl OpenClaw wacht op `turn/completed`. Gebruik dit voor vertrouwde of zware werklasten waarbij synthese na een tool legitiem langer stil kan blijven dan het vrijgavebudget voor de definitieve assistent. |
| `mode`                            | `"yolo"` tenzij lokale Codex-vereisten YOLO niet toestaan | Voorinstelling voor YOLO of door een guardian beoordeelde uitvoering.                                                                                                                                                                                                                                                                                                                           |
| `approvalPolicy`                            | `"never"` of een toegestaan guardian-goedkeuringsbeleid | Native Codex-goedkeuringsbeleid dat bij het starten en hervatten van een thread en bij een beurt wordt verzonden.                                                                                                                                                                                                                                                                                |
| `sandbox`                            | `"danger-full-access"` of een toegestane guardian-sandbox | Native Codex-sandboxmodus die bij het starten en hervatten van een thread wordt verzonden. Actieve OpenClaw-sandboxes beperken `danger-full-access`-beurten tot Codex `workspace-write`; de netwerkmarkering van de beurt volgt het uitgaande verkeer van de OpenClaw-sandbox.                                                                                                                        |
| `approvalsReviewer`                            | `"user"` of een toegestane guardian-beoordelaar | Gebruik `"auto_review"` om Codex native goedkeuringsprompts te laten beoordelen wanneer dit is toegestaan.                                                                                                                                                                                                                                                                                     |
| `defaultWorkspaceDir`                            | huidige procesmap                                      | Werkruimte die door `/codex bind` wordt gebruikt wanneer `--cwd` is weggelaten.                                                                                                                                                                                                                                                                                                  |
| `serviceTier`                            | niet ingesteld                                         | Optionele servicelaag van de Codex-app-server. `"priority"` schakelt routering in snelle modus in, `"flex"` vraagt flexibele verwerking aan en `null` wist de overschrijving. Verouderde `"fast"` wordt geaccepteerd als `"priority"`.                                                                                                                    |
| `networkProxy`                            | uitgeschakeld                                          | Meld je aan voor netwerken via het Codex-machtigingsprofiel voor app-serveropdrachten. OpenClaw definieert de geselecteerde `permissions.<profile>.network`-configuratie en selecteert deze met `default_permissions` in plaats van `sandbox` te verzenden.                                                                                                                                                   |
| `experimental.sandboxExecServer`              | `false`                                                | Opt-in voor de preview waarmee een door de OpenClaw-sandbox ondersteunde Codex-omgeving bij de ondersteunde Codex-appserver wordt geregistreerd, zodat native Codex-uitvoering binnen de actieve OpenClaw-sandbox kan plaatsvinden.                                                                                                                                                                                                            |

`appServer.networkProxy` is expliciet omdat dit het sandboxcontract van Codex
wijzigt. Wanneer dit is ingeschakeld, stelt OpenClaw ook `features.network_proxy.enabled` en
`default_permissions` in de Codex-threadconfiguratie in, zodat het gegenereerde
machtigingsprofiel door Codex beheerd netwerkverkeer kan starten. OpenClaw genereert
standaard een botsingsbestendige `openclaw-network-<fingerprint>`-profielnaam uit de
profielinhoud; gebruik `profileName` alleen wanneer een stabiele lokale naam
vereist is.

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

Als de normale app-serverruntime `danger-full-access` zou zijn, gebruikt het
inschakelen van `networkProxy` in plaats daarvan bestandssysteemtoegang in
werkruimtestijl voor het gegenereerde machtigingsprofiel. Door Codex beheerde
netwerkhandhaving is gesandboxte netwerktoegang, dus een profiel met volledige
toegang zou uitgaand verkeer niet beschermen.

De plugin blokkeert oudere, nieuwere maar niet-gevalideerde, prerelease-,
buildachtervoegsel- of niet-geversioneerde app-serverhandshakes. De Codex-app-server
moet een stabiele versie rapporteren vanaf `0.143.0` tot en met de
meegeleverde `0.145.0`.

OpenClaw beschouwt WebSocket-app-server-URL's die niet naar loopback verwijzen als
extern en vereist identiteitsdragende WebSocket-authenticatie via
`appServer.authToken` of een `Authorization`-header. `appServer.authToken` en elke
`appServer.headers.*`-waarde kunnen een SecretInput zijn; de secretsruntime lost
SecretRefs en env-afkortingen op voordat OpenClaw de startopties voor de
app-server samenstelt, en niet-opgeloste gestructureerde SecretRefs mislukken
voordat een token of header wordt verzonden. Wanneer native Codex-plugins zijn
geconfigureerd, gebruikt OpenClaw het pluginbeheer van de verbonden app-server
om die plugins te installeren of te vernieuwen en vernieuwt het daarna de
app-inventaris, zodat apps van plugins zichtbaar zijn voor de Codex-thread.
`app/list` blijft de gezaghebbende bron voor inventaris en metadata, maar
OpenClaw-beleid bepaalt of `thread/start` `config.apps[appId].enabled = true` verzendt voor een
vermelde toegankelijke app, zelfs als Codex deze momenteel als uitgeschakeld
markeert. Onbekende of ontbrekende app-id's blijven standaard geblokkeerd; dit
pad activeert alleen marketplace-plugins via `plugin/install` en vernieuwt de
inventaris. Verbind OpenClaw alleen met externe app-servers die je vertrouwt om
door OpenClaw beheerde plugininstallaties en vernieuwingen van de app-inventaris
te accepteren.

## Goedkeurings- en sandboxmodi

Lokale stdio-app-serversessies gebruiken standaard de YOLO-modus:
`approvalPolicy: "never"`, `approvalsReviewer: "user"` en
`sandbox: "danger-full-access"`. Met deze vertrouwde lokale operatorhouding kunnen
onbeheerde OpenClaw-beurten en Heartbeats voortgang boeken zonder native
goedkeuringsvragen die niemand kan beantwoorden.

Als het lokale systeemvereistenbestand van Codex impliciete YOLO-waarden voor
goedkeuring, reviewer of sandbox niet toestaat, behandelt OpenClaw de impliciete
standaardwaarde in plaats daarvan als guardian en selecteert het toegestane
guardian-machtigingen. `tools.exec.mode: "auto"` dwingt ook door guardian beoordeelde
Codex-goedkeuringen af en behoudt geen onveilige verouderde overschrijvingen van
`approvalPolicy: "never"` of `sandbox: "danger-full-access"`; stel `tools.exec.mode: "full"` in voor een
bewuste houding zonder goedkeuring. Vermeldingen in `[[remote_sandbox_config]]` die met de
hostnaam overeenkomen en in hetzelfde vereistenbestand staan, worden meegenomen
bij de beslissing over de standaardwaarde voor de sandbox.

Stel `appServer.mode: "guardian"` in voor door Codex guardian beoordeelde goedkeuringen:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

De voorinstelling `guardian` wordt uitgebreid naar
`approvalPolicy: "on-request"`, `approvalsReviewer: "auto_review"` en `sandbox: "workspace-write"` wanneer die
waarden zijn toegestaan. Afzonderlijke beleidsvelden overschrijven
`mode`. De oudere reviewerwaarde `guardian_subagent` wordt nog steeds
geaccepteerd als compatibiliteitsalias, maar nieuwe configuraties moeten
`auto_review` gebruiken.

Wanneer een OpenClaw-sandbox actief is, wordt het lokale Codex-app-serverproces
nog steeds op de Gateway-host uitgevoerd. Daarom schakelt OpenClaw voor die beurt
native Code Mode van Codex, MCP-servers van de gebruiker en door apps ondersteunde
pluginuitvoering uit, in plaats van sandboxing aan de Codex-hostzijde als
gelijkwaardig aan de OpenClaw-sandboxbackend te beschouwen. Shelltoegang wordt
beschikbaar gesteld via dynamische tools die door de OpenClaw-sandbox worden
ondersteund, zoals `sandbox_exec` en `sandbox_process`, wanneer de normale
exec-/procestools beschikbaar zijn.

<Note>
Op door Docker ondersteunde OpenClaw-sandboxhosts (`agents.defaults.sandbox.mode` ingesteld
op een Docker-backend) controleert `openclaw doctor` of de host de naamruimten
voor de onbevoorrechte gebruiker toestaat, en wanneer netwerkuitgaand verkeer
van de Docker-sandbox is uitgeschakeld, ook de netwerknaamruimten, die geneste
Codex `bwrap` nodig heeft voor `workspace-write`-shelluitvoering in
de sandboxcontainer. Een mislukte controle wordt op Ubuntu-/AppArmor-hosts
gewoonlijk weergegeven als `bwrap: setting up uid map: Permission denied` of
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. Herstel het gerapporteerde hostbeleid voor naamruimten voor
de OpenClaw-servicegebruiker en start de Gateway opnieuw; geef de voorkeur aan
een beperkt AppArmor-profiel voor het serviceproces boven de hostbrede
terugvaloptie `kernel.apparmor_restrict_unprivileged_userns=0`, en verleen geen ruimere
Docker-containerrechten alleen om aan geneste `bwrap` te voldoen.
</Note>

## Gesandboxte native uitvoering

De stabiele standaardinstelling is standaard blokkeren: actieve
OpenClaw-sandboxing schakelt native Codex-uitvoeringsoppervlakken uit die anders
vanaf de Codex-app-serverhost zouden worden uitgevoerd. Gebruik
`appServer.experimental.sandboxExecServer: true` alleen wanneer je ondersteuning voor externe omgevingen van
Codex met de sandboxbackend van OpenClaw wilt uitproberen. Dit previewpad werkt
met elke ondersteunde Codex-app-serverversie.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

Wanneer de vlag is ingeschakeld en de huidige OpenClaw-sessie is gesandboxt,
start OpenClaw een lokale loopback-exec-server die door de actieve sandbox wordt
ondersteund, registreert deze bij de Codex-app-server en start de Codex-thread en
-beurt met die omgeving van OpenClaw. Als de app-server de omgeving niet kan
registreren, wordt de uitvoering standaard geblokkeerd in plaats van stilzwijgend
terug te vallen op uitvoering op de host.

Dit previewpad is alleen lokaal beschikbaar. Een externe WebSocket-app-server
kan de loopback-exec-server niet bereiken tenzij deze op dezelfde host wordt
uitgevoerd, dus OpenClaw weigert die combinatie.

## Authenticatie en omgevingsisolatie

In de standaardhome per agent wordt authenticatie in deze volgorde geselecteerd:

1. Een expliciet OpenClaw Codex-authenticatieprofiel voor de agent.
2. Het bestaande account van de app-server in de Codex-home van die agent.
3. Alleen voor lokale stdio-starts van de app-server:
   `CODEX_API_KEY`, en daarna `OPENAI_API_KEY`, wanneer er geen
   app-serveraccount aanwezig is en OpenAI-authenticatie nog steeds vereist is.

Wanneer OpenClaw een Codex-authenticatieprofiel van het type ChatGPT-abonnement
detecteert (OAuth- of tokenreferentietype), verwijdert het `CODEX_API_KEY` en
`OPENAI_API_KEY` uit het gestarte Codex-subproces. Hierdoor blijven API-sleutels
op Gateway-niveau beschikbaar voor embeddings of directe OpenAI-modellen zonder
dat native beurten van de Codex-app-server per ongeluk via de API worden
gefactureerd.

Expliciete Codex-API-sleutelprofielen en lokale stdio-terugval op env-sleutels
gebruiken app-serveraanmelding in plaats van de overgenomen omgeving van het
subproces. WebSocket-app-serververbindingen ontvangen geen terugval op
Gateway-env-API-sleutels; gebruik een expliciet authenticatieprofiel of het eigen
account van de externe app-server.

Stdio-starts van de app-server nemen standaard de procesomgeving van OpenClaw
over. OpenClaw beheert de accountbrug van de Codex-app-server en stelt
`CODEX_HOME` in op een map per agent onder de OpenClaw-status van die agent.
Daardoor blijven Codex-configuratie, accounts, plugincache/-gegevens en
threadstatus beperkt tot de OpenClaw-agent, in plaats van binnen te lekken uit de
persoonlijke `~/.codex`-home van de operator.

Stel `appServer.homeScope: "user"` in om native Codex-status te delen met Codex Desktop en
de CLI. Deze lokale gebruikershomemodus ondersteunt beheerde stdio en expliciet
Unix-transport. De modus gebruikt `$CODEX_HOME` wanneer dit is ingesteld en
anders `~/.codex`, inclusief native authenticatie, configuratie, plugins
en threads. OpenClaw slaat voor de app-server zijn authenticatieprofielbrug over.
Geverifieerde beurten van de eigenaar kunnen `codex_threads` gebruiken om die
threads te vermelden, met een optioneel `search`-filter, te lezen, te
forken, te hernoemen, te archiveren en uit het archief te halen. Fork een thread
voordat je deze in OpenClaw voortzet; onafhankelijke Codex-processen coördineren
geen gelijktijdige schrijvers voor dezelfde thread.

Die opt-in van `homeScope` geldt voor gewone harness-sessies. Een Chat die
via Codex Sessions is gemaakt, gebruikt in plaats daarvan zijn privéverbinding
voor supervisie, waardoor de authenticatie- en providerconfiguratie van de native
verbinding voor de canonieke branch en toekomstige hervattingen behouden blijft.

In een modelvergrendelde Chat onder supervisie kan `codex_threads` geen andere
fork koppelen of de gekoppelde native thread van de Chat archiveren. Vermelden en
alleen metadata lezen blijven beschikbaar. Voor het lezen van onbewerkte
transcripten is `allowRawTranscripts` vereist; wanneer dit is uitgeschakeld, wordt
ook zoeken in de lijst geweigerd omdat native zoeken overeenkomsten kan vinden in
transcriptvoorbeelden. Voor hernoemen, uit het archief halen, een losgekoppelde
fork en het archiveren van een niet-gerelateerde thread die niet bij een andere
OpenClaw Chat hoort, is `allowWriteControls` vereist. Geen van beide opties omzeilt
een vergrendelde koppeling.

OpenClaw herschrijft `HOME` niet voor normale lokale starts van de
app-server. Door Codex uitgevoerde subprocessen zoals `openclaw`,
`gh`, `git`, cloud-CLI's en shellopdrachten zien de
normale proceshome en kunnen configuratie en tokens in de gebruikershome vinden.
Codex kan ook `$HOME/.agents/skills` en `$HOME/.agents/plugins/marketplace.json` ontdekken; die
`.agents`-detectie wordt bewust gedeeld met de operatorhome en staat los
van geïsoleerde `~/.codex`-status.

Binnen het standaardagentbereik blijven OpenClaw-plugins en momentopnamen van
OpenClaw-Skills via het eigen pluginregister en de Skills-loader van OpenClaw
lopen; persoonlijke Codex `~/.codex`-assets niet. Als je nuttige Codex
CLI-Skills of plugins uit een Codex-home hebt die onderdeel moeten worden van een
geïsoleerde OpenClaw-agent, inventariseer je deze expliciet:

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

Als een implementatie aanvullende omgevingsisolatie nodig heeft, voeg je die
variabelen toe aan `appServer.clearEnv`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` is alleen van invloed op het gestarte subproces van de
Codex-app-server. OpenClaw verwijdert `CODEX_HOME` en
`HOME` tijdens lokale startnormalisatie uit deze lijst:
`CODEX_HOME` blijft verwijzen naar het geselecteerde agent- of
gebruikersbereik en `HOME` blijft overgenomen, zodat subprocessen
normale status uit de gebruikershome kunnen gebruiken.

## Dynamische tools

Dynamische Codex-tools gebruiken standaard `searchable`-laden en worden
beschikbaar gesteld onder de naamruimte `openclaw` met
`deferLoading: true`. OpenClaw stelt dynamische tools die native
werkruimtebewerkingen van Codex of de eigen toolzoekfunctie van Codex dupliceren
normaal niet beschikbaar:

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

Wanneer een eindige runtime-toegestanelijst native Code Mode uitschakelt,
verzendt OpenClaw een lege selectie voor de uitvoeringsomgeving. In dat directe,
niet-gesandboxte geval behoudt OpenClaw de door beleid gefilterde tools
`exec` en `process` als shellterugval. Runtime-toegestanelijsten
en `codexDynamicToolsExclude` blijven van toepassing.

De meeste overige OpenClaw-integratietools, zoals berichten, media, cron,
browser, nodes, gateway, `heartbeat_respond` en `web_search`, zijn beschikbaar
via de Codex-toolzoekfunctie binnen die naamruimte. Hierdoor blijft de initiële
modelcontext kleiner. Een kleine groep tools blijft altijd rechtstreeks
aanroepbaar, ongeacht `codexDynamicToolsLoading`, omdat de Codex-toolzoekfunctie niet
beschikbaar kan zijn of uitsluitend connectors kan vinden: `agents_list`,
`sessions_spawn` en `sessions_yield`. Ontwikkelaarsinstructies sturen normale
Codex-subagents nog steeds naar de systeemeigen `spawn_agent` voor
Codex-eigen subagentwerk, terwijl `sessions_spawn` beschikbaar blijft voor
expliciete delegatie via OpenClaw of ACP. Bronantwoorden die alleen de
berichtentool gebruiken, blijven eveneens rechtstreeks beschikbaar, omdat dit
een contract voor beurtbesturing is.

Codex Code Mode geeft generieke resultaten van dynamische OpenClaw-tools als
tekst weer. Parse een JSON-resultaat voordat je velden leest. Geneste dynamische
aanroepen worden door de Codex-runtime geserialiseerd, zodat
`Promise.all` ze niet gelijktijdig indient; gebruik een begrensde
sequentiële startlus wanneer je onderliggende verzamelaars start.

Tools die zijn gemarkeerd met `catalogMode: "direct-only"`, waaronder de OpenClaw-tool
`computer`, worden gegroepeerd onder `openclaw_direct`. OpenClaw voegt
die naamruimte toe aan de lijst `code_mode.direct_only_tool_namespaces` van Codex zonder door de
operator opgegeven vermeldingen te vervangen. Codex stelt die tools daarom in
normale threads en threads die uitsluitend Code Mode gebruiken beschikbaar als
`DirectModelOnly`, in plaats van ze te routeren via geneste
Code Mode-aanroepen van `tools.*`. Deze grens is vereist voor
resultaten met afbeeldingen: geneste Code Mode-serialisatie zet
afbeeldingsuitvoer om in platte tekst, waardoor de schermafbeelding die nodig
is voor de volgende computeractie verloren zou gaan.

Stel `codexDynamicToolsLoading: "direct"` alleen in wanneer je verbinding maakt met een
aangepaste Codex-app-server die niet naar uitgestelde dynamische tools kan
zoeken, of wanneer je de volledige toolpayload debugt.

## Time-outs

Dynamische toolaanroepen die eigendom zijn van OpenClaw worden onafhankelijk
van `appServer.requestTimeoutMs` begrensd. Elk Codex-verzoek voor
`item/tool/call` gebruikt de eerste beschikbare time-out in deze volgorde:

- Een positief argument `timeoutMs` per aanroep.
- Voor `image_generate`: `agents.defaults.mediaModels.image.timeoutMs`.
- Voor `image_generate` zonder geconfigureerde time-out: de standaardwaarde
  van 120 seconden voor het genereren van afbeeldingen.
- Voor de tool `image` voor mediabegrip: de naar milliseconden
  geconverteerde `timeoutSeconds` van de geselecteerde afbeeldingsgeschikte
  vermelding `tools.media.models[]`, of de mediastandaard van 60 seconden. Voor
  afbeeldingsbegrip geldt dit voor het verzoek zelf en wordt dit niet verkort
  door eerder voorbereidend werk.
- Voor de tool `message`: een vast extern budget van 600 seconden dat
  Gateway-bezorging en begrensde reconciliatie met dezelfde sleutel omvat.
- De standaardwaarde van 90 seconden voor dynamische tools.

Deze bewaking is het externe dynamische budget voor `item/tool/call`.
Providerspecifieke time-outs voor verzoeken worden binnen die aanroep uitgevoerd
en behouden hun eigen time-outsemantiek. Budgetten voor dynamische tools zijn
beperkt tot 600000 ms. `agents_wait` voegt 30000 ms externe respijt voor
voltooiing toe en de app-serverclient staat 660000 ms toe, zodat het
gestructureerde wachtresultaat Codex kan bereiken. Bij een time-out breekt
OpenClaw waar ondersteund het toolsignaal af en retourneert het een mislukt
antwoord van de dynamische tool aan Codex, zodat de beurt kan doorgaan in
plaats van de sessie in `processing` achter te laten.

Nadat Codex een beurt heeft geaccepteerd en nadat OpenClaw heeft geantwoord op
een beurtgebonden app-serververzoek, verwacht het harnas dat Codex voortgang
boekt in de huidige beurt en de systeemeigen beurt uiteindelijk afrondt met
`turn/completed`. Als de app-server gedurende `appServer.turnCompletionIdleTimeoutMs` stilvalt,
onderbreekt OpenClaw de Codex-beurt naar beste vermogen, registreert het een
diagnostische time-out en geeft het de OpenClaw-sessiebaan vrij, zodat volgende
chatberichten niet achter een vastgelopen systeemeigen beurt in de wachtrij
blijven staan.

De meeste niet-terminale meldingen voor dezelfde beurt schakelen die korte
bewaking uit, omdat Codex heeft aangetoond dat de beurt nog actief is.
Tooloverdrachten gebruiken een langer inactiviteitsbudget na een tool: nadat
OpenClaw een antwoord voor `item/tool/call` retourneert, nadat systeemeigen
toolitems zoals `commandExecution` zijn voltooid, na onbewerkte voltooiingen
van `custom_tool_call_output`, en na onbewerkte assistentvoortgang na een tool,
voltooide onbewerkte redeneringen of voortgang van redeneringen. De beveiliging
gebruikt `appServer.postToolRawAssistantCompletionIdleTimeoutMs` wanneer dit is geconfigureerd en gebruikt anders
standaard vijf minuten. Datzelfde budget na een tool verlengt ook de
voortgangsbewaking voor het stille synthesevenster voordat Codex de volgende
gebeurtenis voor de huidige beurt uitstuurt. Voltooide redeneringen, voltooide
commentaaritems van `agentMessage` en onbewerkte redenerings- of
assistentvoortgang vóór een tool kunnen worden gevolgd door een automatisch
definitief antwoord, zodat ze de antwoordbewaking na voortgang gebruiken in
plaats van de sessiebaan onmiddellijk vrij te geven. Alleen voltooide
definitieve/niet-commentaaritems van `agentMessage` en voltooide onbewerkte
assistentuitvoer vóór een tool activeren de vrijgave na assistentuitvoer: als
Codex daarna stilvalt zonder `turn/completed`, onderbreekt OpenClaw de
systeemeigen beurt naar beste vermogen en geeft het de sessiebaan vrij.
Herhalingsveilige stdio-app-serverfouten, waaronder time-outs door inactiviteit
bij voltooiing van een beurt zonder bewijs van assistent-, tool-, actief-item-
of neveneffectactiviteit, worden eenmaal opnieuw geprobeerd met een nieuwe
app-serverpoging. Bij onveilige time-outs wordt de vastgelopen app-serverclient
nog steeds buiten gebruik gesteld en wordt de OpenClaw-sessiebaan vrijgegeven.
Ze wissen ook de verouderde systeemeigen threadkoppeling in plaats van
automatisch opnieuw te worden afgespeeld. Time-outs van de voltooiingsbewaking
tonen Codex-specifieke time-outtekst: in herhalingsveilige gevallen wordt
vermeld dat het antwoord mogelijk onvolledig is, terwijl onveilige gevallen de
gebruiker vragen de huidige toestand te controleren voordat deze het opnieuw
probeert. Openbare time-outdiagnostiek bevat structurele velden zoals de
methode van de laatste app-servermelding, de id/het type/de rol van het
onbewerkte assistentantwoorditem, aantallen actieve verzoeken/items en de
toestand van de geactiveerde bewaking. Wanneer de laatste melding een
onbewerkt assistentantwoorditem is, bevat de diagnostiek ook een begrensd
tekstvoorbeeld van de assistent. Ze bevat geen onbewerkte prompt- of
toolinhoud.

## Modeldetectie

Standaard vraagt de Codex-plugin de app-server naar beschikbare modellen. De
beschikbaarheid van modellen wordt beheerd door de Codex-app-server, zodat de
lijst kan veranderen wanneer OpenClaw de gebundelde versie
`@openai/codex` bijwerkt of wanneer een implementatie
`appServer.command` naar een ander Codex-binair bestand laat verwijzen.
Beschikbaarheid kan ook accountgebonden zijn. Gebruik `/codex models` op
een actieve gateway om de live catalogus voor dat harnas en account te
bekijken.

Als detectie mislukt of een time-out bereikt, gebruikt OpenClaw een gebundelde
fallbackcatalogus:

| Model-id       | Weergavenaam | Redeneerniveaus           |
| -------------- | ------------ | ------------------------- |
| `gpt-5.5`      | gpt-5.5      | laag, gemiddeld, hoog, xhoog |
| `gpt-5.4-mini` | GPT-5.4-Mini | laag, gemiddeld, hoog, xhoog |

<Note>
Het huidige gebundelde harnas is `@openai/codex` `0.145.0`. Een
`model/list`-probe op die gebundelde app-server retourneerde deze
openbare keuzelijstrijen:

| Model-id        | Invoermodaliteiten | Redeneerniveaus                         |
| --------------- | ------------------ | --------------------------------------- |
| `gpt-5.6-sol`   | tekst, afbeelding   | laag, gemiddeld, hoog, xhoog, max, ultra |
| `gpt-5.6-terra` | tekst, afbeelding   | laag, gemiddeld, hoog, xhoog, max, ultra |
| `gpt-5.6-luna`  | tekst, afbeelding   | laag, gemiddeld, hoog, xhoog, max        |
| `gpt-5.5`       | tekst, afbeelding   | laag, gemiddeld, hoog, xhoog             |
| `gpt-5.2`       | tekst, afbeelding   | laag, gemiddeld, hoog, xhoog             |

De app-servercatalogus kan `ultra` rapporteren; de
redeneerinstellingen van OpenClaw bieden momenteel niveaus tot en met
`max`.

Live keuzelijstrijen zijn accountgebonden en kunnen veranderen met het account,
de Codex-catalogus of de gebundelde versie; voer `/codex models` uit voor
de huidige lijst in plaats van te vertrouwen op een momentopnametabel.
Verborgen modellen kunnen ook in de app-servercatalogus verschijnen voor
interne of gespecialiseerde processen zonder normale keuzes in de
modelkeuzelijst te zijn.
</Note>

Stem detectie af onder `plugins.entries.codex.config.discovery`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

Schakel detectie uit wanneer je wilt voorkomen dat Codex bij het opstarten
wordt gepeild en uitsluitend de fallbackcatalogus wilt gebruiken:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## Bootstrapbestanden voor de werkruimte

Codex verwerkt `AGENTS.md` zelf via systeemeigen detectie van
projectdocumentatie. OpenClaw schrijft geen synthetische
Codex-projectdocumentatiebestanden en is voor personabestanden niet afhankelijk
van Codex-fallbackbestandsnamen, omdat Codex-fallbacks alleen van toepassing
zijn wanneer `AGENTS.md` ontbreekt.

Voor gelijkwaardigheid met de OpenClaw-werkruimte geeft het Codex-harnas de
andere bootstrapbestanden door als ontwikkelaarsinstructies, maar niet op
dezelfde manier:

- `TOOLS.md` wordt doorgestuurd als **overgenomen**
  Codex-ontwikkelaarsinstructies, zodat systeemeigen Codex-subagents die tijdens
  de beurt worden gestart deze ook zien.
- `SOUL.md`, `IDENTITY.md` en `USER.md`
  worden doorgestuurd als **beurtgebonden** samenwerkingsinstructies.
  Systeemeigen Codex-subagents nemen deze niet over, zodat subagentbeurten niet
  de persona en het gebruikersprofiel van de bovenliggende agent overnemen.
- De compacte lijst met geladen OpenClaw-Skills wordt ook doorgestuurd als
  beurtgebonden ontwikkelaarsinstructies voor samenwerking, zodat systeemeigen
  Codex-subagents deze evenmin overnemen.
- De inhoud van `HEARTBEAT.md` wordt niet geïnjecteerd;
  Heartbeat-beurten krijgen een verwijzing in samenwerkingsmodus om het bestand
  te lezen wanneer het bestaat en niet leeg is.
- De inhoud van `MEMORY.md` uit de geconfigureerde agentwerkruimte wordt
  niet in de invoer van de systeemeigen Codex-beurt geplakt wanneer
  geheugentools beschikbaar zijn voor die werkruimte; wanneer het bestand
  bestaat, voegt het harnas een kleine verwijzing naar het werkruimtegeheugen
  toe aan de beurtgebonden ontwikkelaarsinstructies voor samenwerking en moet
  Codex `memory_search` of `memory_get` gebruiken wanneer duurzaam
  geheugen relevant is. Als tools zijn uitgeschakeld, zoeken in het geheugen
  niet beschikbaar is of de actieve werkruimte afwijkt van de
  agentgeheugenwerkruimte, gebruikt `MEMORY.md` in plaats daarvan het
  normale begrensde pad voor beurtcontext.
- `BOOTSTRAP.md` wordt, wanneer aanwezig, doorgestuurd als
  referentiecontext voor OpenClaw-beurtinvoer.

## Omgevingsoverschrijvingen

Omgevingsoverschrijvingen blijven beschikbaar voor lokale tests:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_BIN` omzeilt het beheerde binaire bestand wanneer
`appServer.command` niet is ingesteld.

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` is verwijderd. Gebruik in plaats daarvan
`plugins.entries.codex.config.appServer.mode: "guardian"`, of `OPENCLAW_CODEX_APP_SERVER_MODE=guardian` voor eenmalige lokale tests.
Configuratie heeft de voorkeur voor herhaalbare implementaties, omdat het
plugingedrag daardoor in hetzelfde beoordeelde bestand blijft staan als de rest
van de Codex-harnasconfiguratie.

## Gerelateerd

- [Codex-harnas](/nl/plugins/codex-harness)
- [Codex-harnasruntime](/nl/plugins/codex-harness-runtime)
- [Codex-supervisie](/plugins/codex-supervision)
- [Systeemeigen Codex-plugins](/nl/plugins/codex-native-plugins)
- [Codex-computergebruik](/nl/plugins/codex-computer-use)
- [OpenAI-provider](/nl/providers/openai)
- [Configuratiereferentie](/nl/gateway/configuration-reference)
