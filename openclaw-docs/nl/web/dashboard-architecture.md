---
read_when:
    - De sessiedashboardfunctie (borden) implementeren of beoordelen
    - Widgethosting, de widgetbridge of bordopslag wijzigen
summary: 'Sessiedashboards: architectuur- en implementatieplan (technisch ontwerp, vóór algemene beschikbaarheid)'
title: Dashboardarchitectuur
x-i18n:
    generated_at: "2026-07-27T06:17:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
Technisch ontwerpdocument voor de sessiedashboardfunctie, geschreven vóór en
tijdens de implementatie. Het is de gezaghebbende bron voor de uitwerking. Wanneer de
functie wordt uitgebracht, wordt `/web/dashboard` de gebruikersgerichte pagina en blijft deze pagina
de architectuurreferentie.
</Note>

## Visie

Werken met een agent is tegenwoordig een tekststroom. Het dashboard maakt er een
werkbank van: de agent rendert live, interactieve widgets; de gebruiker zet ze vast op
een persistent oppervlak; de chat wordt aan de zijkant gedokt (of verborgen) en de hoofdinhoud is
het bord. Je gaat van "praten met de agent" naar "een bedieningspaneel bedienen dat de
agent voor je heeft gebouwd", zonder ooit de sessie te verlaten.

Principes:

- **Een bord is een weergave van een sessie, geen nieuw object.** Elke sessie (thread)
  heeft twee weergaven: het transcript en het bord. Een sessie zonder vastgezette widgets
  is een gewone chat. Zet één widget vast en het bord bestaat. Borden nemen de
  identiteit, het eigenaarschap door de agent, de naamgeving, het vastzetten en de levenscyclus van de sessie over. Er is
  geen `dashboard_create`, geen bordregister en geen afzonderlijk ACL-model.
- **Gelijkwaardigheid voor de agent.** Alles wat de gebruiker op een bord kan doen, kan de agent
  met tools doen: widgets toevoegen/bijwerken/verwijderen, ze rangschikken, tabbladen beheren, het
  zichtbare tabblad wisselen en de chat dokken of verbergen.
- **Native, niet ingebed.** Het bord bestaat uit Lit-componenten in de Control UI-shell
  (hetzelfde ontwerpsysteem als de rest van de app). Alleen de _inhoud_ van widgets wordt
  gesandboxed in iframes. Geen URL-balk, geen browserinterface.
- **Klein agentoppervlak.** Widgets worden aangesproken met een stabiele naam en ter
  plaatse bijgewerkt. De lay-out is een vloeiend, automatisch compacterend raster; de agent geeft afmetingen en
  ankers door, nooit pixels of coördinaten.
- **Mogelijkheden boven vertrouwen.** Widgetcode is willekeurige, door de agent geschreven HTML/JS
  in een streng afgeschermde sandbox. Toegang (gatewaygegevens, acties, netwerk) bestaat alleen via
  een gedeclareerd, door de beheerder toegekend mogelijkhedenmanifest.

## Concepten

| Concept             | Definitie                                                                                                                                                        |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sessie (thread)     | Bestaande gatewaysessie, geïdentificeerd door stabiele `sessionKey`. Eigendom van een agent.                                                                                        |
| Bord                | De widgetweergave van één sessie. Bestaat alleen als de sessie widgets/tabbladen heeft. Overleeft `/new`/`/reset` (gekoppeld aan `sessionKey`, niet aan het transcript).                 |
| Tabblad             | Een presentatiepagina van een bord: welke widgets, hun rangschikking en de dokstatus van de chat (`left`/`right`/`bottom`/`hidden`). Borden beginnen met één impliciet tabblad. |
| Widget              | Benoemd, gesandboxed HTML/JS-programma dat eigendom is van de sessie. Aangesproken als `sessionKey` + `name`. Ter plaatse bijgewerkt op naam.                                              |
| Mogelijkhedenmanifest | Declaratie per widget van de toegang: `data` (leesbindingen), `actions` (toegestane werkwoorden), `prompt` (naar sessie verzenden), `net` (toegestane origins).                      |
| Vastzetten (widget) | Een transcriptwidget naar het bord van de sessie verplaatsen (gebruikersbediening of argument voor een agenttool). Losmaken verwijdert de widget van het bord.                                         |
| Vastzetten (sessie) | Bestaande mogelijkheid om sessies in de zijbalk vast te zetten. Een vastgezette sessie met een bord opent in de bordweergave.                                                                      |

## UX-stromen

- **Promotie:** agent roept `show_widget` aan in een chat → widget wordt inline
  in het transcript gerenderd, precies zoals nu → bij aanwijzen verschijnt **Vastzetten op dashboard** → widget
  verschijnt op het bord van de sessie. De agent kan `pin: true` doorgeven om hetzelfde te doen.
- **Bordweergave:** een sessie met een bord krijgt een weergaveschakelaar (Chat / Dashboard).
  Bordweergave = tabbladbalk (alleen bij >1 tabblad) + vloeiend raster + gedokt chatvenster.
  Het chatdok kan van grootte worden veranderd, worden verplaatst (links/rechts/onderaan) en worden ingeklapt, precies
  zoals de zijbalk. De dokstatus wordt per tabblad onthouden.
- **Slepen:** de gebruiker sleept widgets; het raster wordt automatisch gecompacteerd (widgets schuiven omhoog, aangrenzende
  widgets worden herschikt). Grootte wijzigen met een handgreep klikt vast op formaatstappen. Geen plaatsing op pixelniveau —
  voor niemand.
- **Waarschuwing bij resetten:** `/new` / `/reset` vraagt bij een sessie met een bord om
  bevestiging in de web-UI ("de context wordt gereset, het dashboard blijft bestaan") en behoudt
  het bord.
- **Zijbalk:** vastgezette sessies tonen hun bordweergave wanneer ze er een hebben.
  Het bord van de Home-sessie is het standaard-"agentdashboard".
- **Interacties** (drie niveaus, zie hieronder): stille statusgebeurtenissen, zichtbare
  promptverzendingen en automatiseringstriggers.

## Interactieniveaus

1. **Statusgebeurtenissen (standaard).** Interacties met de widget-UI waarvan het model op de hoogte moet zijn,
   maar waarop het niet moet reageren. `bridge.emitState({...})` voegt een gestructureerde
   sessiemelding toe (hetzelfde mechanisme als meldingen over groepsactiviteit). Er wordt geen agentbeurt
   gestart; het model ziet de verzamelde meldingen bij de volgende uitvoering.
2. **Prompts (expliciet praten).** `bridge.sendPrompt(text)` — vereist activering door de gebruiker;
   verzendt een zichtbaar gebruikersbericht naar de sessie (de gedokte chat
   toont het). Met snelheidsbeperking; elke verzending wordt door de gebruiker bevestigd, tenzij de widget
   de toegekende mogelijkheid `prompt` heeft.
3. **Automatisering.** `bridge.runAction(name, args)` — activeert een in het manifest gedeclareerde
   actie. Initiële set werkwoorden: `cron.trigger` (een bestaande crontaak nu uitvoeren) en
   `binding.refresh`. Crontaken worden al uitgevoerd in zichtbare, geïsoleerde uitvoeringssessies
   en kunnen een goedkoper model gebruiken: dat is het pad "klein model stuurt de widget aan".
   Nergens verborgen sessies.

## Widgetmodel en hosting

Widget-HTML/JS wordt geschreven door de agent (doorgaans via `show_widget`), verpakt
in de standaarddocumentshell (CSP-meta, formaatrapporteur, bridge-bootstrap) en
gerenderd in `<iframe sandbox="allow-scripts">` (nooit `allow-same-origin`).

- **Inlinewidgets (transcript)** behouden de huidige canvas-documentpijplijn:
  geschreven onder de statusmap, aangeboden door de Gateway, opgeschoond per bereik, geen
  goedkeuring (ze hebben per definitie geen mogelijkheden — promptverzendingen worden door de gebruiker bevestigd).
- **Bordwidgets** zijn sessiestatus: bytes bevinden zich in de SQLite-database van de eigenaar-agent
  (`board_widgets`), aangeboden via een kernroute van de Gateway
  (`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`) die de database leest.
  Bij het vastzetten van een transcriptwidget worden de bytes gekopieerd. Limieten: 256 KB per widget,
  48 widgets per bord.
- **Ter plaatse bijwerken:** een widget opnieuw uitsturen met dezelfde `name` vervangt de
  bytes, verhoogt `revision`, zendt `board.changed` uit en actieve weergaven herladen
  alleen dat iframe.
- **Bytes vastzetten:** toegekende mogelijkheden worden gekoppeld aan de sha256 van de widgetbytes.
  Bij wijziging van de bytes blijven toekenningen voor `data`/`net`/`actions` alleen behouden als de nieuwe
  revisie een subset van het toegekende manifest declareert; bij een uitgebreid manifest
  wordt de beheerder opnieuw om toestemming gevraagd.

### Widgets hosten inhoud; MCP-apps zijn één inhoudstype

De **widget is de primitieve bouwsteen van OpenClaw**: de benoemde, vastgezette, van een formaat voorziene,
sessiegebonden bordcel met een toekenningsrecord. Wat erin wordt gerenderd, is een
inhoudstype:

- `html` — door de agent geschreven via `show_widget`, bytes in bordopslag.
- `mcp-app` — een MCP-appweergave van derden (`ui://`-resource van een geconfigureerde
  server) die in de widgetcel wordt gehost.

MCP-apps bepalen het widgetmodel niet; widgets hebben de mogelijkheid gekregen om
ze te hosten. Identiteit, plaatsing, vastzetten, toekenningen en de API voor auteurs blijven
van OpenClaw — zodat `show_widget`-code even kort blijft als tegenwoordig en nooit
hoeft te weten dat de MCP Apps-specificatie bestaat.

Gedeelde onderliggende infrastructuur (hier vindt de vereenvoudiging plaats):

- **Eén sandboxhost.** `html`-widgets worden gerenderd via dezelfde geharde
  pijplijn waarmee MCP-apps zijn uitgebracht (dubbel iframe op de speciale sandbox-origin,
  per widget gedeclareerde CSP die met gesloten standaard wordt gedecodeerd) in plaats van een tweede
  op maat gemaakte iframehost. De proxy ontvangt HTML als waarde, waardoor lokale inhoud
  het natuurlijke geval is.
- **Eén autorisatiemodel.** De toegang van een widget is een toegekende toelatingslijst,
  ongeacht het type: voor `html`-widgets hosttools; voor `mcp-app`-widgets
  de voor de app zichtbare tools van de server (via het bestaande `allowedAppToolNames`-
  mechanisme, duurzaam gemaakt per widget in plaats van per aanmaakuitvoering).
- **Hosttools voor `html`-widgets** (beschikbaar via de widgetbridge, gecontroleerd
  aan de hand van de toekenning):
  - `openclaw.prompt.send` — niveau 2; gerouteerd via het zichtbare invoerveld,
    door de gebruiker bevestigd tenzij toegekend
  - `openclaw.state.emit` — sessiemeldingen van niveau 1 (samengevoegd, met maximale grootte)
  - `openclaw.data.read` — geparameteriseerde alleen-lezenbindingen (bestaande
    toegestane set lees-RPC's), opgelost aan de Gateway-zijde
  - `openclaw.cron.trigger` — automatisering van niveau 3
- **`net` = CSP.** Netwerktoegang gebruikt de reeds uitgebrachte CSP-declaratie
  per widget (`connect-src`-origins) — de zichzelf bijwerkende weerwidget
  haalt de API rechtstreeks op vanuit de sandbox, zonder tussenkomst van de Gateway.
- **Toekenningen.** Een widget die niets declareert, wordt onmiddellijk gerenderd (gesandboxed,
  `default-src 'none'`, promptverzendingen afzonderlijk bevestigd) — hetzelfde vertrouwensniveau als
  de huidige inlinechatwidgets. Gedeclareerde tools/origins plaatsen de widget in
  `pending` op het bord: een tijdelijke kaart vermeldt ze in voor mensen leesbare vorm met
  één tik op **Toestaan**/**Afwijzen**. Toekenningen gelden per widgetnaam; voor `html`-widgets
  worden ze aan bytes gebonden (sha256) en gewijzigde bytes behouden de toekenning alleen als de
  declaratie is ingeperkt.
- **Shim voor auteurs.** De documentwrapper injecteert `window.openclaw.prompt`,
  `window.openclaw.state`, `window.openclaw.data` en `window.openclaw.cron`
  als de stabiele API voor auteurs. Dashboardaanroepen delen één aan een weergaveticket gebonden
  aanvraagkanaal; formaatrapportage en thematokens blijven afzonderlijke
  hostmeldingen.

### Declaraties van pluginmogelijkheden

Ingeschakelde plugins kunnen de widgethost uitbreiden via `dashboard.dataBindings`
en `dashboard.actionVerbs` in `openclaw.plugin.json`. Pluginlokale id's worden
toekenningsnamen met het plugin-id als voorvoegsel, zoals `workboard.cards.list` en
`workboard.dispatch`; `%` en `.` in het plugin-id-segment worden geëscapet, zodat een
andere splitsing tussen plugin en lokale id niet dezelfde persistente toekenning kan overnemen. Tijdens
de registratie van de plugin verifieert OpenClaw dat elke binding verwijst naar een RPC
die door dezelfde plugin is geregistreerd met `operator.read` en elke actie naar een
met `operator.write`; ongeldige declaraties laten het laden van de plugin mislukken. Het gevalideerde
register wordt alleen opnieuw opgebouwd bij wijzigingen in de levenscyclus van plugins, terwijl widgettoekenningen
per widget en aan bytes en revisies gebonden blijven.

### Gemodelleerd restrisico: WebRTC-datakanalen

De sandbox-CSP genereert de voorgestelde `webrtc 'block'`-directive, maar
[de huidige set CSP-directives van Chromium](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)
implementeert deze niet. Scriptbare widgets kunnen daarom in de huidige Chromium
WebRTC-datakanalen gebruiken voor uitgaand verkeer. Hetzelfde restrisico is al aanwezig
voor inlinechatwidgets en de MCP Apps-host op `main`.

**Geaccepteerde afweging:** OpenClaw blokkeert scriptbare widgets niet op basis van dit
restrisico. Widgetinhoud krijgt alleen toegang tot gevoelige OpenClaw-gegevens via
een door de operator verleende, byte-bevroren `data:read`-capability, en het sandbox-
Permissions Policy blokkeert toegang tot de camera en microfoon. Een DOM-API-beveiliging is
best-effort defense-in-depth, geen beveiligingsgrens, en hoort thuis in
vervolghardening.

### Transcriptweergave: één widgetkaart

Inlineweergave wordt verenigd op basis van de widgetprimitive. Wanneer een toolresultaat UI bevat —
`show_widget`-uitvoer of een MCP-toolresultaat met een app-resource — materialiseert het systeem
een **tijdelijke widget met automatisch toegewezen naam** (sessiegebonden, opgeschoond) en
geeft het transcript één widgetkaart weer die routeert op basis van het inhoudstype.
Automatische weergave van MCP-apps blijft precies zoals de specificatie verwacht (nul extra modelwerk);
onderliggend _is_ het gewoon een widget. Dit verwijdert de parallelle speciale gevallen voor
`mcpApp` in de chatweergave (oppervlaktebeperking, afzonderlijke deduplicatie), geeft elke
inline-UI dezelfde vastzetmogelijkheid en maakt het widgetregister het primaire
pad voor opnieuw openen (reconstructie door het transcript te scannen blijft een fallback voor nooit-vastgezette
geschiedenis). De alleen-lezen, ticketgebonden zelfstandige host overlapt met boards als een
persistent oppervlak voor opnieuw openen — een consolidatiekandidaat om in T6 te evalueren, niet
als uitgangspunt.

Samenstelling: v1 gebruikt aangrenzende rasters (agent-chrome-widget naast een app-widget op
één tabblad). v2 voegt **door de host beheerde app-slots** toe — HTML van agentwidgets declareert een
slotregio en de host stelt de echte appweergave samen als een naastgelegen sandbox.
De app wordt nooit binnen het iframe van de agent weergegeven: nesten zou de bridge-
identiteit verbreken en overlay/clickjacking van verleende app-UI mogelijk maken, dus het slot is een
lay-outcontract, geen insluiting.

### Servergestuurde widgets (vastgezette MCP-apps)

Met de verenigde host is het vastzetten van een MCP-app van derden gewoon een widget waarvan
de inhoud van de server wordt opgehaald in plaats van opgeslagen: `board_widgets` bewaart de
descriptor (`serverName`, `toolName`, `uiResourceUri`, oorspronkelijke
`toolCallId` + `sessionKey`) in plaats van HTML-bytes, en het board maakt de
weergavelease opnieuw aan na de TTL van 10 minuten van de chatbeurt (waarbij de resource
`ui://` opnieuw wordt opgehaald wanneer die verouderd is). Inline-MCP-appweergaven in de chat krijgen dezelfde mogelijkheid
**Vastzetten op dashboard** als agentwidgets. Opnieuw geopende weergaven zijn tegenwoordig bewust
alleen-lezen; vastgezette apps die interactief moeten blijven, krijgen een duurzame grant voor de
voor apps zichtbare tools van de server (expliciete allowlist die bij het vastzetten aan de operator wordt getoond), losgekoppeld
van de run die de lease aanmaakt. Vastgezette apps zonder grant blijven alleen-lezen — nog steeds nuttig voor weergave-
dashboards. v1 zet vast op het board van de oorspronkelijke sessie; vastzetten tussen sessies
vereist een leasebroker en moet wachten. Coördineer met open PR #109807 (`ui/message`-
composerroutering, doorgifte van thema/grootte).

### WorkBoard-integratie

Het WorkBoard-integratieprogramma houdt kaarten en boards in eigendom van de Plugin, terwijl verzonden kaarten via de bestaande `sessionKey` en `runId` weer aan hun sessieboards worden gekoppeld, WorkBoard-feeds en verzending via door de Plugin gedeclareerde bindingen en acties beschikbaar worden gesteld, en die resultaten worden samengesteld met de bestaande widgettypen `html` en `mcp-app` in plaats van een WorkBoard-specifiek widgettype te introduceren.

## Lay-out: vloeiend raster

12 kolommen, vaste rijhoogte, **automatisch compact** (zwaartekracht omhoog, opzij duwen bij
slepen — gridstack-semantiek, native geïmplementeerd; rasterberekeningen blijven puur en
DOM-vrij). Widgetlay-outstatus per tabblad: `{ name, w (1-12), h (rows) }` plus
volgorde. Agentvocabulaire:

- `size`: `sm` (3×3) · `md` (6×4) · `lg` (8×6) · `xl` (12×8) · `full`
  (tabblad met één widget)
- `after: <widgetName>` optioneel volgordeanker; weggelaten = achteraan toevoegen
- De gebruiker sleept en past de grootte vrij aan; hetzelfde model voor volgorde+grootte maakt een volledige roundtrip.

## Gegevensmodel (DB per agent)

Nieuwe tabellen in `agents/<agentId>/agent/openclaw-agent.sqlite`
(**vereist een verhoging van de schemaversie van de agent-DB — goedkeuring van de operator vereist
voordat dit wordt geland**):

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- slug
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- stable widget name
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- wrapped document source
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- order within tab (auto-compact input)
  manifest     TEXT NOT NULL DEFAULT '{}',  -- capability manifest JSON
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- byte-frozen grant
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

Een board bestaat zodra er rijen zijn voor de `sessionKey`. Als een sessie wordt verwijderd, worden de bijbehorende
boardrijen verwijderd. `/new`/`/reset` raakt ze niet aan.

## Protocoloppervlak

RPC's (kerntabel met methoden, typebox-schema's in `gateway-protocol`):

- `board.get { sessionKey }` → tabbladen + widgetmetadata (geen bytes) — `operator.read`
- `board.update { sessionKey, ops[] }` — CRUD/herordening van tabbladen, widget verplaatsen/grootte wijzigen/
  verwijderen/losmaken, dockstatus, tabblad focussen — `operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }` —
  `operator.write` (pad van agenttool en vastzetpad)
- `board.widget.grant { sessionKey, name, decision }` — `operator.approvals`
- `board.event { ticket, payload }` — invoer van ticketgebonden tier-1-statusgebeurtenissen;
  de verouderde vorm `{ sessionKey, widget, payload }` voor vertrouwde hosts blijft bestaan —
  `operator.write`
- `board.prompt.authorize { ticket }` — retourneert of een zichtbare promptverzending
  nog steeds bevestiging per klik vereist — `operator.read`
- `board.data.read { ticket, bindingId, params? }` — door de Gateway uitgevoerde resolutie van een
  core-leesbinding of leesbinding van een actieve Plugin uit een allowlist — `operator.read`
- `board.action { ticket, action, ... }` — automatiseringsverzending met exacte grant
  via het bestaande pad om een cron-run direct uit te voeren of een gevalideerd actiewerkwoord
  van een actieve Plugin — `operator.write`

Gebeurtenissen (in `EVENT_SCOPE_GUARDS`, leesbereik):

- `board.changed { sessionKey, revision, widget? }` — persistente status gewijzigd;
  UI haalt opnieuw op (en herlaadt één iframe wanneer `widget` aanwezig is).
- `board.command { sessionKey, command }` — tijdelijke UI-aansturing (agent wisselt
  van zichtbaar tabblad, schakelt chatdock) — het `ui.command`-patroon.

Widgetbytes worden aangeboden via het geauthenticeerde HTTP-oppervlak, niet via de socket.

## Agenttools

In totaal drie tools (core, altijd geregistreerd; weergave beperkt op basis van de
`inline-widgets`-clientcapability zoals nu):

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }` — maken/bijwerken op naam; `pin` plaatst deze op het board.
  Zonder `name`/`pin` gedraagt deze zich precies zoals nu (inline, tijdelijk).
- `dashboard { action, ... }` — beheerwerkwoorden voor boards: `read`, `tab_create`,
  `tab_update`, `tab_delete`, `tabs_reorder`, `widget_move`, `widget_remove`,
  `unpin`, `focus_tab`, `set_chat_dock`.
- Bestaande `cron`-tools dekken de automatiseringstier; geen nieuwe tool nodig.

Toolbeschrijvingen leggen de vocabulaire voor grootte/anker en het tiermodel uit. De
agent wordt via sessiemeldingen geïnformeerd over tier-1-gebeurtenissen van gebruikers, bijvoorbeeld
`[dashboard] user clicked "Refresh" on widget weather (tab main)`.

## Wat dit vervangt

- **`extensions/workspaces` wordt verwijderd.** Experimenteel, `enabledByDefault:
false`, nooit in een stabiele release opgenomen (verscheen voor het eerst in bèta's van 2026.7.2). Geen
  migratie; een doctor-regel verwijdert verouderde `<stateDir>/workspaces/` indien aanwezig.
  Overgenomen ideeën: pure rasterberekeningen, bridgebeveiligingsmodel (poortbootstrap,
  beperking van bindingen, frequentielimieten), byte-bevroren goedkeuring.
- **Widgethosting verhuist van `extensions/canvas` naar core.** De canvasdocument-
  opslag, documentwrapper, HTTP-aanbieding en de tool `show_widget` worden onderdeel van core
  (`src/canvas/`); de Plugin behoudt de node-canvas-besturingstool (`canvas`) en
  A2UI. De advertentie `pluginSurfaceUrls["canvas"]` en de
  paden `/__openclaw__/canvas` zijn uitgebrachte native-clientcontracten en blijven
  stabiel. Discord-sessies behouden de variant `show_widget` die eigendom is van Discord.

## Niet-doelen (dit programma)

- Boarddeling/ACL's voor meerdere gebruikers (toekomstig; komt via sessiedeling).
- Native boardweergave voor macOS/iOS (zij krijgen die overal waar ze de
  Control UI insluiten; het pad voor inline-widgets blijft ongewijzigd).
- Ingebouwde gegevenswidgets (kaarten voor sessies/gebruik/cron) — de capabilitybridge plus
  door agents gemaakte widgets volstaan voor v1; een ingebouwd typeregister kan later komen.

## Implementatieplan

Onafhankelijke worktrees, gebouwd met Codex, review+landing achtereenvolgens. Eerst landen, dan repareren.

| #   | Branch                               | Bereik                                                                                                                                                                              | Afhankelijk van                       |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | Workspaces-Plugin + UI + documentatie + i18n-sleutels verwijderen; opschoonregel voor doctor                                                                                                              | —                                |
| T2  | `claude/dashboard-canvas-core`       | Widgethosting + `show_widget` promoveren naar core; canvas-Plugin behoudt node-tool; geen gedragswijziging                                                                                | —                                |
| T3  | `claude/dashboard-domain`            | Agent-DB-tabellen (schemaverhoging), `board.*`-RPC's + gebeurtenissen, tool `dashboard`, argumenten voor vastzetten/naam/manifest van `show_widget`, tier-1-meldingen, reset-behoudt-board                                  | T2                               |
| T4  | `claude/dashboard-ui`                | Boardweergave + tabbladstrook + vloeiend automatisch compact raster + chatdock (links/rechts/onder/verborgen) + vastzetmogelijkheid in transcript + boardweergave in zijbalk + resetbevestiging                           | T3 (eerst mocks via ontwikkelfixtures) |
| T5  | `claude/dashboard-capabilities`      | Grantopslag/UI + bytebevriezing; widgets van `html` verplaatsen naar de gedeelde sandboxhost; hosttools (`openclaw.prompt.send/state.emit/data.read/cron.trigger`); `net`-CSP; auteurscompatibiliteitslaag | T3, T4                           |
| T7  | `claude/dashboard-mcp-apps`          | Inhoudstype `mcp-app`: vastzetmogelijkheid voor inline-appweergaven, descriptoropslag, lease opnieuw aanmaken/verversen, duurzame grants voor servertools (hergebruikt uitgebrachte MCP Apps-host)                   | T3, T4                           |
| T6  | afwerking                               | Live-E2E op een tijdelijke Gateway (echte sleutels), schermafbeeldingen, reparaties, gebruikersgerichte herschrijving van `/web/dashboard`, beoordeling voor standaardinschakeling                                                     | alles                              |

Validatie volgens de reporegels: gerichte vitest lokaal, volledige controles op
Crabbox/Testbox, `$autoreview` vóór elke landing, livebewijs voor T6.
