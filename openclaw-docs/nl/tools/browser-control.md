---
read_when:
    - De agentbrowser scripten of debuggen via de lokale besturings-API
    - Op zoek naar de `openclaw browser` CLI-referentie
    - Aangepaste browserautomatisering toevoegen met momentopnamen en referenties
summary: OpenClaw-API voor browserbesturing, CLI-referentie en scriptacties
title: API voor browserbesturing
x-i18n:
    generated_at: "2026-07-27T06:13:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

Voor installatie, configuratie en probleemoplossing, zie [Browser](/nl/tools/browser).
Deze pagina is de referentie voor de lokale HTTP-besturings-API, de `openclaw browser`
CLI en scriptpatronen (snapshots, refs, wachttijden, foutopsporingsflows).

## Besturings-API (optioneel)

Alleen voor lokale integraties stelt de Gateway een kleine loopback-HTTP-API beschikbaar.
Deze zelfstandige server is opt-in — stel de omgevingsvariabele
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` in de omgeving van de Gateway-service in
en start de Gateway opnieuw voordat de HTTP-eindpunten beschikbaar worden. Zonder
deze variabele blijft de runtime voor browserbesturing werken via de CLI en
agenttools, maar luistert niets op de loopback-besturingspoort.

- Status/starten/stoppen: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- Profielen: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- Tabbladen: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- Snapshot/schermafbeelding: `GET /snapshot`, `POST /screenshot`
- Acties: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Downloads: `POST /download`, `POST /wait/download`
- Machtigingen: `POST /permissions/grant`
- Foutopsporing: `GET /console`, `POST /pdf`
- Foutopsporing: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Netwerk: `POST /response/body`
- Status: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- Status: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Instellingen: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` is de gebundelde vorm die de CLI intern gebruikt voor
`browser tab`-subopdrachten (`{"action":"new"|"label"|"select"|"close"|"list", ...}`);
geef bij directe scripting de voorkeur aan de specifieke tabbladroutes hierboven.

Alle eindpunten accepteren `?profile=<name>`. `POST /start?headless=true` vraagt om een
eenmalige headless-start voor lokale beheerde profielen zonder de opgeslagen
browserconfiguratie te wijzigen; profielen die alleen koppelen, externe CDP-profielen en profielen met een bestaande sessie weigeren
die overschrijving omdat OpenClaw deze browserprocessen niet start.

Voor tabbladeindpunten is `targetId` de compatibiliteitsveldnaam. Geef bij voorkeur
`suggestedTargetId` door vanuit `GET /tabs` of `POST /tabs/open`; labels en `tabId`-
handles zoals `t1` worden ook geaccepteerd. Ruwe CDP-doel-id's en unieke ruwe
voorvoegsels van doel-id's werken nog steeds, maar zijn vluchtige diagnostische handles.

Als Gateway-authenticatie met een gedeeld geheim is geconfigureerd, vereisen de HTTP-browserroutes ook authenticatie:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` of HTTP Basic-authenticatie met dat wachtwoord

Opmerkingen:

- Deze zelfstandige loopback-browser-API gebruikt **geen** identiteitsheaders van vertrouwde proxy's of
  Tailscale Serve.
- Als `gateway.auth.mode` `none` of `trusted-proxy` is, nemen deze loopback-browserroutes
  die modi met identiteitsinformatie niet over; houd ze uitsluitend op loopback.

### Foutcontract van `/act`

`POST /act` gebruikt een gestructureerd foutantwoord voor validatie op routeniveau en
beleidsfouten:

```json
{ "error": "<message>", "code": "ACT_*" }
```

Huidige waarden voor `code`:

- `ACT_KIND_REQUIRED` (HTTP 400): `kind` ontbreekt of wordt niet herkend.
- `ACT_INVALID_REQUEST` (HTTP 400): normalisatie of validatie van de actiepayload is mislukt.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): `selector` is gebruikt met een niet-ondersteund actietype.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (of `wait --fn`) is uitgeschakeld door de configuratie.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): `targetId` op het hoogste niveau of in een batch conflicteert met het doel van de aanvraag.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): de actie wordt niet ondersteund voor profielen met een bestaande sessie.

Andere runtimefouten kunnen nog steeds `{ "error": "<message>" }` retourneren zonder een
`code`-veld.

### Playwright-vereiste

Sommige functies (navigeren/handelen/AI-snapshot/rolsnapshot, schermafbeeldingen van elementen,
PDF) vereisen Playwright. Als Playwright niet is geïnstalleerd, retourneren die eindpunten
een duidelijke 501-fout.

Wat nog steeds werkt zonder Playwright:

- ARIA-snapshots
- Toegankelijkheidssnapshots in rolstijl (`--interactive`, `--compact`,
  `--depth`, `--efficient`) wanneer een CDP-WebSocket per tabblad beschikbaar is. Dit is
  een terugvaloptie voor inspectie en het vinden van refs; Playwright blijft de primaire
  actie-engine.
- Paginaschermafbeeldingen voor de beheerde `openclaw`-browser wanneer een CDP-
  WebSocket per tabblad beschikbaar is
- Paginaschermafbeeldingen voor `existing-session`- / Chrome MCP-profielen
- Op `existing-session`-refs gebaseerde schermafbeeldingen (`--ref`) uit snapshotuitvoer

Waarvoor Playwright nog steeds nodig is:

- `navigate`
- `act`
- AI-snapshots die afhankelijk zijn van de native AI-snapshotindeling van Playwright
- Schermafbeeldingen van elementen via CSS-selectors (`--element`)
- Volledige browserexport naar PDF

Schermafbeeldingen van elementen weigeren ook `--full-page`; de route retourneert `fullPage is
not supported for element screenshots`.

Als je `Playwright is not available in this gateway build` ziet, ontbreekt in de verpakte
Gateway de kern-runtimeafhankelijkheid voor de browser. Installeer OpenClaw opnieuw of werk het bij
en start daarna de Gateway opnieuw. Installeer voor Docker ook de Chromium-
browserbinairen zoals hieronder weergegeven.

#### Playwright-installatie voor Docker

Als je Gateway in Docker draait, vermijd dan `npx playwright` (conflicten met npm-overschrijvingen).
Neem voor aangepaste images Chromium op in de image:

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

Installeer voor een bestaande image in plaats daarvan via de meegeleverde CLI:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Stel `PLAYWRIGHT_BROWSERS_PATH` in om browserdownloads te bewaren (bijvoorbeeld
`/home/node/.cache/ms-playwright`) en zorg dat `/home/node` behouden blijft via
`OPENCLAW_HOME_VOLUME` of een bind-mount. OpenClaw detecteert de opgeslagen
Chromium-installatie automatisch op Linux. Zie [Docker](/nl/install/docker).

## Werking (intern)

Een kleine loopback-besturingsserver accepteert HTTP-aanvragen en maakt via CDP verbinding met Chromium-gebaseerde browsers. Geavanceerde acties (klikken/typen/snapshot/PDF) verlopen via Playwright boven op CDP; wanneer Playwright ontbreekt, zijn alleen bewerkingen zonder Playwright beschikbaar. De agent ziet één stabiele interface, terwijl lokale/externe browsers en profielen daaronder vrij kunnen wisselen.

## Beknopt CLI-overzicht

Alle opdrachten accepteren `--browser-profile <name>` om een specifiek profiel te kiezen en `--json` voor machineleesbare uitvoer.

<AccordionGroup>

<Accordion title="Basis: status, tabbladen, openen/focussen/sluiten">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # voeg een live snapshotcontrole toe
openclaw browser start
openclaw browser start --headless # eenmalige lokale beheerde headless-start
openclaw browser stop            # wist ook emulatie bij alleen koppelen/externe CDP
openclaw browser reset-profile   # verplaatst de browsergegevens van het profiel naar de prullenmand
openclaw browser tabs
openclaw browser tab             # snelkoppeling voor het huidige tabblad
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="Profielen: weergeven, maken, verwijderen">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="Inspectie: schermafbeelding, snapshot, console, fouten, aanvragen">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # of --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="Acties: navigeren, klikken, typen, slepen, wachten, evalueren">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # of e12 voor rolrefs
openclaw browser click-coords 120 340        # viewportcoördinaten
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="Status: cookies, opslag, offline, headers, locatie, apparaat">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # gebruik --clear om te verwijderen
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

Opmerkingen:

- De agentgerichte `browser`-tool biedt `action=download` (vereiste `ref` en
  `path`) en `action=waitfordownload` (optionele `path`). Beide retourneren de opgeslagen
  download-URL, voorgestelde bestandsnaam en beveiligde lokale pad. Expliciete onderschepping van
  downloads is beschikbaar voor beheerde Playwright-profielen; profielen met bestaande sessies
  retourneren een fout voor een niet-ondersteunde bewerking.
- Geef de voorkeur aan atomische uploads via de bestandskiezer: geef de trigger `--ref` mee met de upload, zodat OpenClaw deze in één aanvraag activeert en aanklikt. `upload` met alleen paden blijft ondersteund wanneer een latere trigger de bedoeling is. Gebruik `--input-ref` of `--element` om rechtstreeks een bestandsinvoer in te stellen. `dialog` is een activeringsaanroep; voer deze uit vóór de klik/toetsaanslag die het dialoogvenster opent. Als een actie een modaal venster opent, bevat het actierespons `blockedByDialog` en `browserState.dialogs.pending`; geef die `dialogId` door om rechtstreeks te antwoorden. Dialoogvensters die buiten OpenClaw worden afgehandeld, verschijnen onder `browserState.dialogs.recent`.
- `click`/`type`/enz. vereisen een `ref` uit `snapshot` (numerieke `12`, rolreferentie `e12` of uitvoerbare ARIA-referentie `ax12`). CSS-selectors worden bewust niet ondersteund voor acties. Gebruik `click-coords` wanneer de zichtbare viewportpositie het enige betrouwbare doel is.
- Download- en tracepaden zijn beperkt tot tijdelijke OpenClaw-hoofdmappen: `/tmp/openclaw{,/downloads}` (terugvaloptie: `${os.tmpdir()}/openclaw/...`).
- `upload` accepteert bestanden uit de tijdelijke uploadhoofdmap van OpenClaw en
  door OpenClaw beheerde inkomende media. Naar beheerde inkomende media kan worden verwezen als
  `media://inbound/<id>`, sandbox-relatieve `media/inbound/<id>` of een herleid
  pad binnen de map voor beheerde inkomende media. Geneste mediareferenties,
  padtraversal, symbolische koppelingen, harde koppelingen en willekeurige lokale paden worden nog steeds geweigerd.
- `upload` kan bestandsinvoer ook rechtstreeks instellen via `--input-ref` of `--element`.

Stabiele tabblad-ID's en labels blijven behouden wanneer Chromium een onbewerkt doel vervangt en OpenClaw
het vervangende tabblad kan vaststellen, bijvoorbeeld bij een uniek oud/nieuw-paar voor dezelfde URL of
wanneer één oud tabblad na formulierverzending één nieuw tabblad wordt. Ambigue
vervangingen met dubbele URL's krijgen nieuwe handles. Onbewerkte doel-ID's blijven
vluchtig; geef in scripts de voorkeur aan `suggestedTargetId` uit `tabs`.

Momentopnamevlaggen in één oogopslag:

- `--format ai` (standaard met Playwright): AI-momentopname met numerieke referenties (`aria-ref="<n>"`).
- `--format aria`: toegankelijkheidsstructuur met `axN`-referenties. Wanneer Playwright beschikbaar is, koppelt OpenClaw referenties met backend-DOM-ID's aan de livepagina, zodat vervolgacties ze kunnen gebruiken; behandel de uitvoer anders alleen als inspectie.
- `--efficient` (of `--mode efficient`): compacte voorinstelling voor rolmomentopnamen. Stel `browser.snapshotDefaults.mode: "efficient"` in om dit de standaard te maken (zie [Gateway-configuratie](/nl/gateway/configuration-reference#browser)).
- `--interactive`, `--compact`, `--depth`, `--selector` dwingen een rolmomentopname met `ref=e12`-referenties af. `--frame "<iframe>"` beperkt rolmomentopnamen tot een iframe.
- Met Playwright voegt `--labels` een schermafbeelding toe met overliggende referentielabels
  (toont `MEDIA:<path>`), plus een `annotations`-array met het begrenzingsvak
  van elke referentie. Bij `screenshot` werken door Playwright ondersteunde labels met `--full-page`,
  `--ref` en `--element`; bij `snapshot` blijft de bijbehorende schermafbeelding
  beperkt tot de viewport. Profielen met bestaande sessies/chrome-mcp renderen overliggende labels op
  paginaschermafbeeldingen, maar retourneren geen `annotations` en gebruiken niet de
  Playwright-projectiehelper voor volledige pagina's/referenties/elementen. Zonder Playwright of chrome-mcp
  zijn gelabelde schermafbeeldingen niet beschikbaar.
- `--urls` voegt gevonden linkbestemmingen toe aan AI-momentopnamen.

## Momentopnamen en referenties

OpenClaw ondersteunt twee stijlen voor 'momentopnamen':

- **AI-momentopname (numerieke referenties)**: `openclaw browser snapshot` (standaard; `--format ai`)
  - Uitvoer: een tekstuele momentopname met numerieke referenties.
  - Acties: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Intern wordt de referentie herleid via Playwrights `aria-ref`.

- **Rolmomentopname (rolreferenties zoals `e12`)**: `openclaw browser snapshot --interactive` (of `--compact`, `--depth`, `--selector`, `--frame`)
  - Uitvoer: een op rollen gebaseerde lijst/structuur met `[ref=e12]` (en optioneel `[nth=1]`).
  - Acties: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Intern wordt de referentie herleid via `getByRole(...)` (plus `nth()` voor duplicaten).
  - Voeg `--labels` toe om een schermafbeelding met overliggende `e12`-labels op te nemen. Bij
    door Playwright ondersteunde profielen retourneert dit ook metadata van begrenzingsvakken per referentie
    (`annotations[]`).
  - Voeg `--urls` toe wanneer linktekst ambigu is en de agent concrete
    navigatiedoelen nodig heeft.

- **ARIA-momentopname (ARIA-referenties zoals `ax12`)**: `openclaw browser snapshot --format aria`
  - Uitvoer: de toegankelijkheidsstructuur als gestructureerde knooppunten.
  - Acties: `openclaw browser click ax12` werkt wanneer het momentopnamepad
    de referentie via Playwright en backend-DOM-ID's van Chrome kan koppelen.
- Als Playwright niet beschikbaar is, kunnen ARIA-momentopnamen nog steeds nuttig zijn voor
  inspectie, maar zijn referenties mogelijk niet uitvoerbaar. Maak opnieuw een momentopname met `--format ai`
  of `--interactive` wanneer je actiereferenties nodig hebt.
- Docker-bewijs voor het onbewerkte CDP-terugvalpad: `pnpm test:docker:browser-cdp-snapshot`
  start Chromium met CDP, voert `browser doctor --deep` uit en verifieert dat rolmomentopnamen
  link-URL's, door de cursor tot klikbaar gepromoveerde elementen en iframe-metadata bevatten.

Referentiegedrag:

- Referenties zijn **niet stabiel tussen navigaties**; voer bij een fout `snapshot` opnieuw uit en gebruik een nieuwe referentie.
- `/act` retourneert de huidige onbewerkte `targetId` na een door een actie geactiveerde vervanging
  wanneer het vervangende tabblad kan worden vastgesteld. Blijf stabiele tabblad-ID's/labels gebruiken voor
  vervolgopdrachten.
- Als de rolmomentopname met `--frame` is gemaakt, zijn rolreferenties tot de volgende rolmomentopname beperkt tot dat iframe.
- Onbekende of verouderde `axN`-referenties mislukken onmiddellijk in plaats van terug te vallen op
  Playwrights `aria-ref`-selector. Maak op hetzelfde tabblad een nieuwe momentopname wanneer
  dat gebeurt.

## CLI voor browserbatches

`openclaw browser batch` voert een array met geneste `/act`-acties uit in één `/act`-
aanroep (dezelfde `kind="batch"`-runtime die via de agenttool wordt bereikt), zodat CLI-
gebruikers en scripts acties zoals `wait`, `click`, `type` en
`evaluate` kunnen combineren tot één herhaalbaar plan zonder heen-en-weer-aanvragen per actie. Elke
vermelding in `actions[]` is een `BrowserActRequest` — de gesloten union die de `/act`-
route accepteert (`click`, `clickCoords`, `type`, `press`, `hover`,
`scrollIntoView`, `drag`, `select`, `fill`, `resize`, `wait`, `evaluate`,
`close`, `batch`) — geen willekeurige `openclaw browser`-subopdrachten. `batch` wordt
niet ondersteund op `profile="user"` en andere profielen met bestaande sessies (chrome-mcp);
verzend acties daar afzonderlijk.

- CLI: `openclaw browser batch --actions '<json>'`, `openclaw browser batch
--actions-file plan.json` of `openclaw browser batch --actions-file -` om
  de JSON-array uit stdin te lezen. `--continue` stelt `stopOnError=false` in; standaard
  wordt bij de eerste fout gestopt. `--target-id` beperkt de hele batch tot
  één tabblad.
- Levenscyclus van referenties: referenties komen uit een `snapshot`-uitvoering vóór de batch (momentopname is
  geen geneste actie). Een geneste actie die de paginastatus wijzigt — zoals een
  `click` die navigatie activeert, of een `evaluate` die het DOM wijzigt — kan
  eerdere referenties voor de rest van de batch ongeldig maken. Plaats statuswijzigende acties
  eerst, of splits ze op in een vervolgbatch nadat je opnieuw een momentopname hebt gemaakt. Navigatie en
  het opnieuw maken van momentopnamen vinden buiten de batch plaats (`openclaw browser navigate` /
  `snapshot`), omdat `open`, `navigate` en `snapshot` geen `/act`-soorten zijn.
- Conflicten met doel-ID's: een geneste actie mag `targetId` weglaten of
  `targetId` op aanvraagniveau herhalen; een expliciete geneste `targetId` die naar een
  ander tabblad wordt herleid, wordt met `ACT_TARGET_ID_MISMATCH` geweigerd voordat een actie
  wordt uitgevoerd. Gebatchte acties delen per ontwerp het tabblad van de aanvraag.
- Foutoverzicht: het antwoord is `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }`, met één vermelding per actie in volgorde. Wanneer
  `stopOnError` de standaard is, eindigt de array bij de eerste fout; met
  `--continue` omvat deze elke actie. Elke mislukte vermelding zorgt ervoor dat de CLI
  met een niet-nulstatus afsluit; geef `--json` mee om het volledige geordende antwoord voor scripts te behouden.

## Uitgebreide wachtmogelijkheden

Je kunt op meer wachten dan alleen tijd/tekst:

- Wachten op URL (globs worden door Playwright ondersteund):
  - `openclaw browser wait --url "**/dash"`
- Wachten op laadstatus:
  - `openclaw browser wait --load networkidle`
  - Ondersteund op beheerde `openclaw`- en onbewerkte/externe CDP-profielen. Profielen die het `existing-session`-stuurprogramma gebruiken (waaronder het standaardprofiel `user`) weigeren `networkidle`; gebruik daar `--url`, `--text`, een selector of `--fn`-wachtbewerkingen.
- Wachten op een JS-predicaat:
  - `openclaw browser wait --fn "window.ready===true"`
- Wachten tot een selector zichtbaar wordt:
  - `openclaw browser wait "#main"`

Deze kunnen worden gecombineerd:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Werkstromen voor foutopsporing

Wanneer een actie mislukt (bijvoorbeeld 'niet zichtbaar', 'schending van strikte modus', 'bedekt'):

1. `openclaw browser snapshot --interactive`
2. Gebruik `click <ref>` / `type <ref>` (geef in interactieve modus de voorkeur aan rolreferenties)
3. Als het nog steeds mislukt: `openclaw browser highlight <ref>` om te zien waarop Playwright zich richt
4. Als de pagina zich vreemd gedraagt:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. Voor grondige foutopsporing: neem een trace op:
   - `openclaw browser trace start`
   - reproduceer het probleem
   - `openclaw browser trace stop` (toont `TRACE:<path>`)

## JSON-uitvoer

`--json` is bedoeld voor scripts en gestructureerde tools.

Voorbeelden:

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

Rolmomentopnamen in JSON bevatten `refs` plus een klein `stats`-blok (regels/tekens/referenties/interactief), zodat tools kunnen redeneren over de grootte en dichtheid van de payload.

## Status- en omgevingsinstellingen

Deze zijn nuttig voor werkstromen waarin 'de site zich als X moet gedragen':

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Opslag: `storage local|session get|set|clear`
- Offline: `set offline on|off`
- Headers: `set headers --headers-json '{"X-Debug":"1"}'` (of de positionele vorm `set headers '{"X-Debug":"1"}'`)
- HTTP-basisauthenticatie: `set credentials user pass` (of `--clear`)
- Geolocatie: `set geo <lat> <lon> --origin "https://example.com"` (of `--clear`)
- Media: `set media dark|light|no-preference|none`
- Tijdzone / landinstelling: `set timezone ...`, `set locale ...`
- Apparaat / viewport:
  - `set device "iPhone 14"` (Playwright-apparaatvoorinstellingen)
  - `set viewport 1280 720`

## Beveiliging en privacy

- Het openclaw-browserprofiel kan aangemelde sessies bevatten; behandel het als gevoelige informatie.
- `browser act kind=evaluate` / `openclaw browser evaluate` en `wait --fn`
  voeren willekeurige JavaScript-code uit in de paginacontext. Promptinjectie kan dit
  sturen. Schakel dit uit met `browser.evaluateEnabled=false` als je het niet nodig hebt.
- `openclaw browser evaluate --fn` accepteert de broncode van een functie, een expressie of
  de hoofdtekst van een instructie. Hoofdteksten van instructies worden verpakt als asynchrone functies, dus gebruik
  `return` voor de waarde die je terug wilt krijgen. Gebruik `--timeout-ms <ms>` wanneer de
  functie aan de paginazijde mogelijk langer nodig heeft dan de standaardtime-out voor evaluatie.
- Zie [Browseraanmelding + posten op X/Twitter](/nl/tools/browser-login) voor opmerkingen over aanmeldingen en antibotmaatregelen (X/Twitter enzovoort).
- Houd de Gateway-/Node-host privé (alleen loopback of tailnet).
- Externe CDP-eindpunten zijn krachtig; tunnel ze en beveilig ze.

Voorbeeld van strikte modus (privé-/interne bestemmingen standaard blokkeren):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optionele exacte toestemming
    },
  },
}
```

## Gerelateerd

- [Browser](/nl/tools/browser) - overzicht, configuratie, profielen, beveiliging
- [Browseraanmelding](/nl/tools/browser-login) - aanmelden bij websites
- [Problemen met Browser op Linux oplossen](/nl/tools/browser-linux-troubleshooting)
- [Problemen met Browser via externe CDP op WSL2 oplossen](/nl/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
