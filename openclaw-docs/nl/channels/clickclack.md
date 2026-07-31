---
read_when:
    - OpenClaw verbinden met een ClickClack-werkruimte
    - ClickClack-botidentiteiten testen
summary: Installatie van het ClickClack-kanaal met bottoken en doelsyntaxis
title: ClickClack
x-i18n:
    generated_at: "2026-07-27T05:00:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 761538cdd7a916415719131b9ff2f40bf3e3e0eab0f7bda450250886acde8a64
    source_path: channels/clickclack.md
    workflow: 16
---

ClickClack verbindt OpenClaw via volwaardige ClickClack-bottokens met een zelf gehoste ClickClack-werkruimte.

Gebruik dit wanneer je wilt dat een OpenClaw-agent als een ClickClack-botgebruiker verschijnt. ClickClack ondersteunt onafhankelijke servicebots en bots die eigendom zijn van gebruikers; bots die eigendom zijn van gebruikers behouden een `owner_user_id` en ontvangen alleen de tokenbereiken die je toekent.

## Snelle installatie

Open in ClickClack **Workspace settings → Integrations → OpenClaw**, maak een
bot met **Setup code (recommended)** en kopieer de gegenereerde opdracht:

```bash
openclaw channels add clickclack --code 'https://clickclack.example.com/#XXXX-XXXX-XXXX'
```

Voor afzonderlijke frontend- en API-origins of een API die onder een pad is gekoppeld, geeft ClickClack in plaats daarvan een
exact claimendpoint:

```bash
openclaw channels add clickclack --code 'https://api.example.com/services/clickclack/api/bot-setup-codes/claim#XXXX-XXXX-XXXX'
```

De installatiecode kan één keer worden gebruikt en verloopt na 10 minuten. OpenClaw claimt deze,
ontvangt het nieuw aangemaakte bottoken en de werkruimte-instellingen, slaat het account op,
verifieert de verbinding en meldt of de actieve Gateway deze heeft opgepikt.
Voor exacte endpoints met een versie valideert en bewaart OpenClaw de canonieke API-
basis die ClickClack retourneert, inclusief eventuele padprefix. De installatiecode zelf wordt
niet opgeslagen in de OpenClaw-configuratie.

Claims met een installatiecode gebruiken HTTPS voor openbare servers. Platte HTTP wordt ook ondersteund voor
lokale installaties op loopback-adressen zoals `localhost` en `127.0.0.1`.

Als OpenClaw al actief is, maakt ClickClack automatisch verbinding en is geen tweede
opdracht nodig. Start het anders met:

```bash
openclaw gateway
```

Je kunt de code ook afzonderlijk van de server-URL doorgeven:

```bash
openclaw channels add clickclack --code XXXX-XXXX-XXXX --base-url https://clickclack.example.com
```

Voer voor begeleide installatie het volgende uit:

```bash
openclaw onboard
```

Selecteer ClickClack en voer vervolgens de server-URL, het bottoken en de werkruimte in wanneer
daarom wordt gevraagd. De begeleide installatie controleert na het opslaan de server, het token en de werkruimte; een
mislukte controle verwijdert de configuratie niet.

### Alternatief: handmatig token

Kies **Manual token** in ClickClack wanneer je een niet-OpenClaw-client configureert of
wanneer je het token uitdrukkelijk zelf moet beheren:

```bash
openclaw channels add clickclack --base-url https://clickclack.example.com --token ccb_... --workspace default
```

`workspace` accepteert een werkruimte-id (`wsp_...`), slug of weergavenaam.
`--code` kan niet worden gecombineerd met `--token`, `--token-file` of `--use-env`.

### Alternatief: omgevingsvariabele voor token

Het standaardaccount kan `CLICKCLACK_BOT_TOKEN` lezen in plaats van een token
in de configuratie op te slaan:

```bash
export CLICKCLACK_BOT_TOKEN="ccb_..."
openclaw channels add clickclack --base-url https://clickclack.example.com --workspace default --use-env
openclaw gateway
```

Benoemde accounts moeten een geconfigureerd token of tokenbestand gebruiken; de gedeelde
omgevingsvariabele is bewust beperkt tot het standaardaccount.

### JSON5-referentie

De equivalente configuratiestructuur is:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      defaultTo: "channel:general",
    },
  },
}
```

Een account geldt alleen als geconfigureerd wanneer `baseUrl`, een tokenbron en
`workspace` allemaal zijn ingesteld. Een tokenbron kan voor het standaardaccount `token`, `tokenFile` of
`CLICKCLACK_BOT_TOKEN` zijn. `workspace` accepteert een werkruimte-
id (`wsp_...`), slug of naam; de Gateway zet deze bij het opstarten om naar de id.

### Configuratiesleutels voor accounts

| Sleutel                  | Standaard           | Opmerkingen                                                                             |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `baseUrl`               | geen (vereist)      | Openbare ClickClack-URL die wordt gebruikt voor links voor de browser.                  |
| `apiBaseUrl`            | `baseUrl`           | Optioneel server-naar-server-endpoint voor REST- en realtime WebSocket-verkeer.         |
| `token`                 | geen                | Bottoken als platte tekenreeks of geheime verwijzing (`source: "env" \| "file" \| "exec"`).              |
| `tokenFile`             | geen                | Pad naar een bottokenbestand; heeft voorrang op `token`.                     |
| `workspace`             | geen (vereist)      | Werkruimte-id, slug of naam.                                                            |
| `replyMode`             | `"agent"`           | `"agent"` voert de volledige agentpijplijn uit; `"model"` verzendt korte rechtstreekse modelaanvullingen. |
| `defaultTo`             | `"channel:general"` | Doel dat wordt gebruikt wanneer een uitgaand pad geen doel opgeeft.                     |
| `allowFrom`             | `["*"]`             | Toelatingslijst met gebruikers-id's voor inkomende DM's en kanaalberichten.             |
| `botUserId`             | automatisch gedetecteerd | Bij het opstarten afgeleid van de identiteit van het bottoken.                      |
| `agentId`               | standaardroute      | Zet de inkomende berichten van dit account vast op één agent.                           |
| `toolsAllow`            | geen                | Toelatingslijst met tools voor agentantwoorden vanuit dit account.                      |
| `model`, `systemPrompt` | geen                | Gebruikt door `replyMode: "model"`-aanvullingen.                                          |
| `commandMenu`           | `true`              | Publiceer systeemeigen opdrachten in de automatische aanvulling van de ClickClack-composer. |
| `reconnectMs`           | `1500`              | Vertraging voor realtime opnieuw verbinden (100 tot 60000).                             |
| `discussions`           | uitgeschakeld       | Beheerde kanaalinstellingen per sessie; zie [Sessiediscussies](#session-discussions).   |

### Een door authenticatie afgeschermde openbare hostnaam behouden

Gebruik `apiBaseUrl` wanneer ClickClack en de OpenClaw-Gateway op dezelfde host draaien,
maar de openbare ClickClack-hostnaam wordt beschermd door een authenticatiegateway,
zoals Cloudflare Access:

```json5
{
  channels: {
    clickclack: {
      baseUrl: "https://clack.openclaw.ai",
      apiBaseUrl: "http://127.0.0.1:8484",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
    },
  },
}
```

De openbare hostnaam kan voor browsergebruikers volledig door authenticatie afgeschermd blijven. OpenClaw
gebruikt het loopback-endpoint voor REST-verzoeken, installatieverificatie en de
realtime WebSocket, terwijl `embedUrl`- en `openUrl`-links van discussies
de openbare `baseUrl` blijven gebruiken. Als `apiBaseUrl` wordt weggelaten, gebruikt al het verkeer
`baseUrl`, waardoor het bestaande gedrag behouden blijft.

Als `plugins.allow` een niet-lege beperkende lijst is, wordt bij het expliciet selecteren van
ClickClack tijdens de kanaalinstallatie of bij het uitvoeren van `openclaw plugins enable clickclack`
`clickclack` aan die lijst toegevoegd. Installatie via onboarding gebruikt hetzelfde
gedrag voor expliciete selectie. Deze paden overschrijven `plugins.deny` of een
globale `plugins.enabled: false`-instelling niet. Rechtstreeks
`openclaw plugins install @openclaw/clickclack` volgt het normale
beleid voor Plugin-installatie en registreert ClickClack ook in een bestaande toelatingslijst.

## Meerdere bots

Elk account opent zijn eigen realtime ClickClack-verbinding en gebruikt zijn eigen bottoken.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      defaultAccount: "service",
      accounts: {
        service: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SERVICE_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "channel:general",
          agentId: "service-bot",
        },
        support: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SUPPORT_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "dm:usr_...",
          agentId: "support-bot",
        },
      },
    },
  },
}
```

## Sessiediscussies

Schakel discussies in voor één ClickClack-account om elke OpenClaw-sessie een
speciaal ClickClack-kanaal te geven. Het accounttoken moet
`channels:write` bevatten (de bundel `bot:admin` bevat dit); het normale installatietoken `bot:write`
kan geen kanalen maken of synchroniseren.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      discussions: {
        enabled: true,
        workspace: "default",
        controlUrlBase: "https://team.openclaw.ai",
        section: "Sessions",
      },
    },
  },
}
```

`discussions.workspace` accepteert dezelfde werkruimte-id, slug of weergavenaam
als `workspace` op accountniveau en gebruikt standaard die waarde. `section` bepaalt
de sectie in de ClickClack-zijbalk en is standaard `Sessions`. Wanneer
`controlUrlBase` is ingesteld, linkt het beheerde kanaal terug naar de echte sessieroute
van de Control UI, `/chat?session=<encoded-session-key>`.

Schakel discussies in voor exact één ClickClack-account. De Gateway-provider heeft
geen accountselector, dus meerdere ingeschakelde discussieaccounts worden geweigerd
in plaats van er één te kiezen op basis van de configuratievolgorde.

Bij het openen van een discussie wordt een openbaar ClickClack-kanaal gemaakt dat als extern
beheerd is gemarkeerd. De Plugin houdt het sessielabel, de categorie en de archiefstatus
gesynchroniseerd. Het herstellen van een sessie herstelt het bijbehorende kanaal; het wissen van de sessiecategorie
verplaatst het kanaal terug naar de geconfigureerde standaardsectie. Bij het verwijderen van een
OpenClaw-sessie wordt het ClickClack-kanaal gearchiveerd in plaats van verwijderd, zodat de
geschiedenis beschikbaar blijft. De Plugin brengt koppelingen opnieuw in overeenstemming wanneer discussie-RPC's
worden gebruikt en ongeveer eenmaal per minuut zolang er koppelingen bestaan.

Inkomende berichten in een beheerd kanaal gebruiken een deterministische nevensessie onder
dezelfde agent-id als de gekoppelde hoofdsessie. De nevenagent krijgt te horen welke
hoofdsessie moet worden geobserveerd en kan `sessions_history` en `session_status` gebruiken
(`changesSince` is nuttig voor incrementele controles). Deze gebruikt `sessions_send` alleen
wanneer mensen in de discussie vragen om informatie door te geven aan of sturing te geven aan de hoofdsessie.
De koppeling, de verwijzing naar beheerd eigendom en de peeridentiteit van de nevensessie bevatten
de concrete OpenClaw-sessie-id samen met de vastgezette ClickClack-server en
het kanaal. Het opnieuw instellen van een herbruikbare sessiesleutel of het opnieuw richten van een account trekt het
oude kanaal lokaal in, archiveert het wanneer de oude referentie nog bruikbaar is en
kan het bijbehorende nevengesprek niet hergebruiken. Berichten die binnenkomen via een
gearchiveerde, opnieuw ingestelde, uitgeschakelde of opnieuw gerichte koppeling worden verwijderd in plaats van terug te
vallen op de normale kanaalroutering van het account. Vrijgegeven koppelingen laten een duurzame
markering voor een ingetrokken kanaal achter, zodat vertraagde realtime-gebeurtenissen gesloten blijven bij fouten. Eigendom
op afstand wordt bepaald door de ClickClack-server en kanaal-id, zodat het hernoemen van het lokale
account een beheerd kanaal niet in een gewoon kanaal kan veranderen.

Laat `tools.sessions.visibility` op de veiligere standaardwaarde `tree` staan. De Plugin
installeert uitsluitend een hostgebonden toekenning tussen elke nevensessie en de gekoppelde
hoofdsessie, plus een hook voor toolbeleid die sessiedetectie en
sessieoverschrijdende doelen blokkeert. Deze staat `sessions_history`, `session_status` en
`sessions_send` alleen toe voor de gekoppelde hoofdsessie en voorkomt dat de statusaanroep
het model van die sessie wijzigt. Die tools moeten nog steeds aanwezig zijn in de
effectieve tooltoelatingslijst van de agent. De systeemprompt is een richtlijn; de hosttoekenning
en hook vormen de autorisatiegrens.

De ClickClack-server moet velden voor beheerde kanalen (`external_managed`,
`external_ref`, `external_url` en `sidebar_section`) ondersteunen bij het maken en
bijwerken van kanalen en deze retourneren in kanaalresponsen. OpenClaw verifieert dat contract
voordat een binding wordt opgeslagen. Als een respons op het maken verloren gaat, neemt de volgende opening
het kanaal over op basis van de door de server afgedwongen `external_ref`, in plaats van nog een kanaal te maken.
Totdat die uitkomst is afgestemd, plaatst de openstaande reservering
anderszins niet-gebonden gebeurtenissen in de doelwerkruimte in quarantaine. De globale reconciler
neemt het kanaal over wanneer dezelfde sessie nog actief is of archiveert het na een
reset; de reservering wordt verwijderd wanneer er geen extern kanaal is gemaakt.
Die referentie bevat een duurzame naamruimte per OpenClaw-installatie plus een
hash van de sessiesleutel, de concrete sessie-id, de ClickClack-bestemming en de duurzame
bindinggeneratie. Afzonderlijke gateways kunnen elkaars kanalen niet overnemen,
geresette sessies kunnen geen oude kanaalgeschiedenis overnemen en een heen-en-terugwijziging
van account of werkruimte kan een eerder kanaal niet opnieuw overnemen. Bindingen zijn ook gekoppeld aan de
geconfigureerde ClickClack-server-URL en worden ongeldig gemaakt als het account
op een ander doel wordt ingesteld. Het wijzigen of verwijderen van `controlUrlBase` werkt de koppeling met het beheerde
kanaal bij of verwijdert deze tijdens de volgende afstemmingsronde. Het wijzigen van
`discussions.workspace` archiveert de oude binding en geeft deze vrij voordat een kanaal
in de nieuwe werkruimte kan worden geopend, wanneer de referentie voor de oude werkruimte geconfigureerd blijft.
Als het token is vervangen door een werkruimtegebonden referentie die
geen toegang tot de oude werkruimte heeft, registreert OpenClaw het oude kanaal als ingetrokken en
geeft het de binding vrij zonder het vervangende token te proberen; archiveer dat overgebleven
kanaal vanuit ClickClack.

De gekoppelde hoofdsessie krijgt ook een alleen-lezen `discussion`-tool. Deze leest
de nieuwste berichten en recente antwoorden in threads als één geëscapete record met bronvermelding
per bericht en heeft geen neveneffecten voor schrijven of de levenscyclus. Zoekacties voor kanaalhoofdberichten en threads
hebben vaste aanvraagbudgetten; het resultaat waarschuwt expliciet wanneer door die
veiligheidsgrens een oudere actieve thread kan worden weggelaten.

## Antwoordmodi

- `replyMode: "agent"` (standaard) verwerkt inkomende berichten via de normale agentpijplijn, inclusief sessieregistratie en toolbeleid.
- `replyMode: "model"` slaat de agentpijplijn over en gebruikt de `llm.complete` van de Plugin-runtime voor rechtstreekse botantwoorden, eventueel vormgegeven door `model` en `systemPrompt`. De geselecteerde provider en het geselecteerde model bepalen het voltooiingsbudget.

De modelmodus voert voltooiingen uit voor de opgeloste botagent-id, waarvoor
de expliciete vertrouwensbit `plugins.entries.clickclack.llm.allowAgentIdOverride: true`
vereist is:

```json5
{
  plugins: {
    entries: {
      clickclack: {
        llm: {
          allowAgentIdOverride: true,
        },
      },
    },
  },
}
```

Laat de vertrouwensbit uitgeschakeld als je alleen de standaardantwoordmodus `agent` gebruikt; deze is
daar niet nodig.

## Opdrachtmenu

Bij het starten van de Gateway publiceert elk geconfigureerd account de native
opdrachten van OpenClaw naar ClickClack. Ze verschijnen in de automatische aanvulling van het invoerveld, gelabeld met de
handle van de bot. De gepubliceerde set wordt bij elke start volledig vervangen,
waarbij ook een verouderd menu wordt verwijderd wanneer de catalogus met native opdrachten leeg is.

Synchronisatie van het opdrachtmenu is standaard ingeschakeld. Stel `commandMenu: false` in voor een account
om dit uit te schakelen:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      commandMenu: false,
    },
  },
}
```

Het token heeft `commands:write` nodig. De huidige ClickClack-bundels `bot:write` en
`bot:admin` bevatten dat bereik en het kan ook afzonderlijk worden toegekend.
Voor tokens die zijn gemaakt voordat opdrachtmenu's werden geïntroduceerd, moet het
bereik mogelijk worden toegevoegd of is een vervangend token nodig.

Synchronisatie gebeurt naar beste vermogen en wordt eenmaal per start van de Gateway uitgevoerd. Bij een ontbrekend bereik of een
netwerkfout wordt een waarschuwing gelogd; bij een oudere ClickClack-server zonder het eindpunt wordt dit op
debugniveau gelogd. Geen van deze fouten blokkeert het realtime opstarten. Menu's blijven
beschikbaar terwijl de agent offline is en worden verwijderd wanneer de bot de
werkruimte verlaat.

Deze release publiceert alleen native opdrachtspecificaties. Aliassen en
catalogi met Skills, Plugins of aangepaste opdrachten worden niet aan het menu toegevoegd. Als een
naam ook als HTTP-slashopdracht is geregistreerd, verwerkt ClickClack die
registratie eerst; andere menuopdrachten blijven via de normale
berichtbezorging lopen.

Gebruik de modus `agent` voor bewijs van correlatie tussen services. Voor een gezaghebbende
ClickClack-bericht-id in de canonieke vorm `msg_<ulid>` leidt het kanaal
de deterministische OpenClaw-uitvoerings-id `clickclack:<message-id>` af. Elke modelaanroep is
vervolgens in diagnostiek zichtbaar als `clickclack:<message-id>:model:<n>`; wanneer die
beurt ClawRouter gebruikt, wordt dezelfde modelaanroep-id verzonden als `X-Request-ID`.
De modus `model` omzeilt de normale diagnostiek voor agentuitvoeringen en sessies en is daarom
niet geschikt voor dit bewijsproces.

Wanneer een realtime gebeurtenis een gevalideerde `payload.correlation_id` bevat,
neemt het kanaal deze mee als `X-Correlation-ID` bij het ophalen van het gezaghebbende bericht en
de resulterende ClickClack-antwoordverzoeken. Waarden gebruiken ClickClacks veilige
set van 128 tekens (`A-Z`, `a-z`, `0-9`, `.`, `_`, `:` en `-`); ongeldige waarden
worden weggelaten. Deze koppelingen bevatten alleen identificatoren, nooit berichtinhoud,
prompts, voltooiingen, referenties of tooluitvoer.

## Duurzame medialevering

Agentantwoorden met media gebruiken verplichte duurzame levering. OpenClaw wijst
vóór de eerste ClickClack-schrijfactie stabiele bericht- en uploadnonces per onderdeel toe, zodat
bij een nieuwe poging dezelfde upload en hetzelfde bericht worden hergebruikt in plaats van opslagquotum te verbruiken
of duplicaten te publiceren. Als een upload na een herstart al bestaat,
leest OpenClaw het oorspronkelijke lokale pad of de externe media-URL niet opnieuw.

Dit herstelcontract vereist een ClickClack-server die het volgende ondersteunt:

- `GET /api/uploads/by-nonce` met
  `X-ClickClack-Upload-Nonce: supported` voor gevonden en ontbrekende resultaten.
- `GET /api/messages/by-nonce` met
  `X-ClickClack-Message-Nonce: supported` voor gevonden en ontbrekende resultaten.
- Idempotent maken van berichten en koppelen van bijlagen voor dezelfde
  eigenaarsgebonden nonce en upload.

Een generieke 404 van een oudere server wordt niet beschouwd als bewijs dat een verzending ontbreekt.
OpenClaw laat de levering onopgelost in plaats van het risico op een duplicaat te nemen; werk
ClickClack bij voordat je agentantwoorden inschakelt die media produceren.

## Rijen met agentactiviteit

Standaard toont een ClickClack-kanaal niets terwijl een agentbeurt wordt uitgevoerd; alleen het uiteindelijke antwoord wordt geplaatst. Stel `agentActivity: true` in voor een account om duurzame berichtrijen voor `agent_commentary` en `agent_tool` te publiceren terwijl de beurt wordt uitgevoerd:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      agentActivity: true,
    },
  },
}
```

Vereisten en gedrag:

- **Standaard uitgeschakeld.** Standaardconfiguraties en oudere ClickClack-servers blijven ongewijzigd.
- **Vereist het tokenbereik `agent_activity:write`.** Dit bereik staat los van `bot:write` en wordt er niet door overgenomen; maak het bottoken met `--scopes bot:write,agent_activity:write` (of ken het bereik toe aan een bestaand token) voordat je de optie inschakelt.
- **Degradatie naar beste vermogen.** Als het token `agent_activity:write` niet heeft of de server schrijfacties voor activiteit weigert, worden fouten gelogd en wordt het uiteindelijke antwoord nog steeds normaal geleverd; er verschijnen geen activiteitsrijen.
- Rijen worden per beurt gegroepeerd (`turn_id`) en samengevoegd zodat één logische stap één rij vormt. Toolrijen gebruiken dezelfde voortgangsopmaak als Discord/Slack/Telegram (toolnaam plus opdrachtdetails).
- **Toeschrijvingsmetadata.** Door agents geschreven berichten (activiteitsrijen en het uiteindelijke antwoord) bevatten de velden `author_model` en `author_thinking`, bepaald op basis van het model dat daadwerkelijk voor de beurt is gebruikt (ook na een fallback). Servers die deze kolommen niet definiëren, negeren de onbekende JSON-velden; servers die ze opslaan, kunnen per bericht antwoord geven op de vraag "welk model zei deze regel, op welk denkniveau".

## Doelen

- `channel:<name-or-id>` verzendt naar een werkruimtekanaal. Kale doelen gebruiken standaard `channel:`.
- `dm:<user_id>` maakt of hergebruikt een rechtstreeks gesprek met die gebruiker.
- `thread:<message_id>` antwoordt in de thread die bij dat bericht begint.

Expliciete uitgaande doelen kunnen ook het providerprefix `clickclack:` of `cc:` bevatten.

Uitgaande media gebruiken ClickClacks upload-API en koppelen vervolgens de duurzame upload
aan het gemaakte kanaalbericht, het antwoord in de thread of het privébericht. Lokale bestanden en ondersteunde
externe media-URL's volgen het normale mediatoegangsbeleid van OpenClaw, met een limiet van 64 MiB
per bestand. Duurzame verzendingen in de wachtrij gebruiken afzonderlijke eigenaarsgebonden nonces voor elke
upload en elk berichtonderdeel en proberen vervolgens opnieuw de bijlage aan dezelfde
objecten te koppelen. Zie [Duurzame medialevering](#durable-media-delivery) voor het servercontract
en het herstelgedrag.

Voorbeelden:

```bash
openclaw message send --channel clickclack --target channel:general --message "hello"
openclaw message send --channel clickclack --target dm:usr_123 --message "hello"
openclaw message send --channel clickclack --target thread:msg_123 --message "following up"
```

## Machtigingen

ClickClack-tokenbereiken worden afgedwongen door de ClickClack-API.

- `bot:read`: werkruimte-, kanaal-, bericht-, thread-, privébericht-, realtime- en profielgegevens lezen.
- `bot:write`: `bot:read` plus kanaalberichten, antwoorden in threads, privéberichten, uploads en publicatie van opdrachtmenu's.
- `bot:admin`: `bot:write` plus het maken van kanalen.
- `commands:write`: het opdrachtmenu van de bot publiceren. Opgenomen in de huidige bundels `bot:write` en `bot:admin` en afzonderlijk toekenbaar.
- `agent_activity:write`: duurzame rijen met agentactiviteit (`agent_commentary` / `agent_tool`). Wordt niet overgenomen door `bot:write` of `bot:admin`; alleen vereist wanneer `agentActivity: true` is ingesteld.

OpenClaw heeft alleen het huidige `bot:write` nodig voor normale agentchat en synchronisatie van het opdrachtmenu. Voeg `agent_activity:write` toe wanneer je [rijen met agentactiviteit](#agent-activity-rows) inschakelt.

## Probleemoplossing

- `ClickClack is not configured for account "<id>"`: stel voor dat account `baseUrl`, `token` (bijvoorbeeld via `CLICKCLACK_BOT_TOKEN`) en `workspace` in.
- `ClickClack workspace not found: <value>`: stel `workspace` in op de werkruimte-id, slug of naam die ClickClack retourneert.
- Geen inkomende antwoorden: controleer of het token realtime leestoegang heeft en houd er rekening mee dat de bot zijn eigen berichten en berichten van andere bots negeert.
- Verzending naar kanalen mislukt: controleer of de bot lid is van de werkruimte en `bot:write` heeft.
- Geen opdrachtmenu: controleer of `commandMenu` niet `false` is, of de ClickClack-server `PUT /api/bots/self/commands` ondersteunt en of het token `commands:write` heeft.
