---
read_when:
    - Je wilt dat agents wijzigingen in code of Markdown als diffs weergeven
    - Je wilt een viewer-URL die klaar is voor Canvas of een gerenderd diffbestand
    - Je hebt gecontroleerde, tijdelijke diffartefacten met veilige standaardinstellingen nodig
sidebarTitle: Diffs
summary: Alleen-lezen diffviewer en bestandsrenderer voor agents (optionele plugintool)
title: Verschillen
x-i18n:
    generated_at: "2026-07-27T05:52:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs` is een optionele gebundelde plugintool die voor-/natekst of een uniforme patch omzet in een alleen-lezen diff-artefact. De tool voegt ook korte instructies voor agents vooraan toe aan de systeemprompt en wordt geleverd met een bijbehorende skill voor uitgebreidere instructies.

Invoer: `before`- en `after`-tekst, of een uniforme `patch` (wederzijds uitsluitend).

Uitvoer: een Gateway-viewer-URL voor canvasweergave, een gerenderd PNG-/PDF-bestandspad voor berichtbezorging, of beide.

## Snel aan de slag

<Steps>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="Schakel de plugin in">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Kies een modus">
    <Tabs>
      <Tab title="view">
        Canvasgerichte flows: agents roepen `diffs` aan met `mode: "view"` en openen `details.viewerUrl` met `canvas present`.
      </Tab>
      <Tab title="file">
        Bestandsbezorging via chat: agents roepen `diffs` aan met `mode: "file"` en verzenden `details.filePath` met `message` via `path` of `filePath`.
      </Tab>
      <Tab title="both">
        Gecombineerd (standaard): agents roepen `diffs` aan met `mode: "both"` om beide artefacten in één aanroep te verkrijgen.
      </Tab>
    </Tabs>
  </Step>
</Steps>

## Ingebouwde systeeminstructies uitschakelen

Stel `plugins.entries.diffs.hooks.allowPromptInjection` in op `false` om de tool te behouden, maar de vooraan toegevoegde systeempromptinstructies weg te laten:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

Hiermee wordt de `before_prompt_build`-hook van de plugin geblokkeerd, terwijl de tool en skill beschikbaar blijven. Schakel in plaats daarvan de plugin uit om zowel de instructies als de tool uit te schakelen.

## Naslag voor toolinvoer

Alle velden zijn optioneel, tenzij anders vermeld.

<ParamField path="before" type="string">
  Oorspronkelijke tekst. Vereist met `after` wanneer `patch` is weggelaten.
</ParamField>
<ParamField path="after" type="string">
  Bijgewerkte tekst. Vereist met `before` wanneer `patch` is weggelaten.
</ParamField>
<ParamField path="patch" type="string">
  Uniforme diff-tekst. Wederzijds uitsluitend met `before` en `after`.
</ParamField>
<ParamField path="path" type="string">
  Weergavebestandsnaam voor de voor/na-modus.
</ParamField>
<ParamField path="lang" type="string">
  Hint om de taal voor de voor/na-modus te overschrijven. Onbekende waarden en talen buiten de standaardset van de viewer vallen terug op platte tekst, tenzij de plugin Diff Viewer Language Pack is geïnstalleerd.
</ParamField>
<ParamField path="title" type="string">
  Overschrijving van de viewertitel.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  Uitvoermodus. Standaard wordt de pluginstandaard `defaults.mode` (`both`) gebruikt. Verouderde alias: `"image"` gedraagt zich hetzelfde als `"file"`.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  Viewerthema. Standaard wordt de pluginstandaard `defaults.theme` gebruikt.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  Diff-indeling. Standaard wordt de pluginstandaard `defaults.layout` gebruikt.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  Vouw ongewijzigde secties uit wanneer de volledige context beschikbaar is. Alleen een optie per aanroep (geen standaardsleutel van de plugin).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  Gerenderde bestandsindeling. Standaard wordt de pluginstandaard `defaults.fileFormat` gebruikt.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  Kwaliteitsvoorinstelling voor PNG-/PDF-rendering.
</ParamField>
<ParamField path="fileScale" type="number">
  Overschrijving van de apparaatschaal (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  Maximale renderbreedte in CSS-pixels (`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  TTL van het artefact in seconden voor de viewer en zelfstandige bestandsuitvoer. Maximaal `21600`.
</ParamField>
<ParamField path="baseUrl" type="string">
  Overschrijving van de oorsprong van de viewer-URL. Overschrijft pluginwaarde `viewerBaseUrl`. Moet `http` of `https` zijn, zonder query/hash.
</ParamField>

<AccordionGroup>
  <Accordion title="Validatie en limieten">
    - `before`/`after`: elk maximaal 512 KiB.
    - `patch`: maximaal 2 MiB.
    - `path`: maximaal 2048 bytes.
    - `lang`: maximaal 128 bytes.
    - `title`: maximaal 1024 bytes.
    - Limiet voor patchcomplexiteit: maximaal 128 bestanden en in totaal 120000 regels.
    - `patch` samen met `before`/`after` wordt geweigerd.
    - Veiligheidslimieten voor gerenderde bestanden (PNG en PDF):
      - `fileQuality: "standard"`: maximaal 8 MP (8,000,000 gerenderde pixels).
      - `fileQuality: "hq"`: maximaal 14 MP.
      - `fileQuality: "print"`: maximaal 24 MP.
      - PDF is daarnaast beperkt tot 50 pagina's.

  </Accordion>
</AccordionGroup>

## Syntaxismarkering

Ingebouwde talen:

`javascript`, `typescript`, `tsx`, `jsx`, `json`, `markdown`, `yaml`, `css`, `html`, `sh`, `python`, `go`, `rust`, `java`, `c`, `cpp`, `csharp`, `php`, `sql`, `docker`, `ruby`, `swift`, `kotlin`, `r`, `dart`, `lua`, `powershell`, `xml` en `toml`.

Veelgebruikte aliassen (`js`, `ts`, `bash`, `md`, `yml`, `c++`, `dockerfile`, `rb`, `kt`, `ps1`, enzovoort) worden naar die talen genormaliseerd.

Installeer de plugin Diff Viewer Language Pack voor meer talen (Astro, Vue, Svelte, MDX, GraphQL, Terraform/HCL, Nix, Clojure, Elixir, Haskell, OCaml, Scala, Zig, Solidity, Verilog/VHDL, Fortran, MATLAB, LaTeX, Mermaid, Sass/Less/SCSS, Nginx, Apache, CSV, dotenv, INI, diff en meer):

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

Zonder het pakket worden niet-ondersteunde talen nog steeds als leesbare platte tekst gerenderd. Zie [plugin Diffs Language Pack](/nl/plugins/reference/diffs-language-pack) en [Shiki-talen](https://shiki.style/languages) voor de upstreamcatalogus.

## Contract voor uitvoerdetails

Alle geslaagde resultaten bevatten `changed`: identieke voor-/nainvoer retourneert `false` zonder een artefact te maken; gerenderde resultaten retourneren `true`.

<AccordionGroup>
  <Accordion title="Viewervelden (modi view en both)">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (`agentId`, `sessionId`, `messageChannel`, `agentAccountId` indien beschikbaar)

  </Accordion>
  <Accordion title="Bestandsvelden (modi file en both)">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (dezelfde waarde als `filePath`, voor compatibiliteit met de berichtentool)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| Modus    | Retourneert                                                                                     |
| -------- | ----------------------------------------------------------------------------------------------- |
| `"view"` | Alleen viewervelden.                                                                            |
| `"file"` | Alleen bestandsvelden, geen viewerartefact.                                                     |
| `"both"` | Viewervelden plus bestandsvelden. Als het renderen van het bestand mislukt, wordt de viewer nog steeds geretourneerd met `fileError`. |

### Samengevouwen ongewijzigde secties

De viewer toont rijen zoals `N unmodified lines`. Uitvouwbediening verschijnt alleen wanneer de gerenderde diff uitvouwbare contextgegevens bevat (gebruikelijk bij voor-/nainvoer). Bij veel uniforme patches ontbreken contextblokken in de hunks, waardoor de rij zonder uitvouwbediening kan verschijnen -- dit is verwacht en geen fout. `expandUnchanged` is alleen van toepassing wanneer uitvouwbare context bestaat.

### Navigatie tussen meerdere bestanden

Patches die meer dan één bestand wijzigen, beginnen met een overzichtskaart van gewijzigde bestanden: totale aantallen `+N` / `-N`, aantallen per bestand, badges voor toegevoegd/verwijderd/hernoemd en ankerlinks die naar elk bestand springen. Gerenderde PNG-/PDF-bestanden behouden de aantallen in de bestandskoppen, maar laten de interactieve weergaveschakelaars weg, omdat die bedieningselementen in een statisch bestand niet werken.

## Pluginstandaarden

Stel pluginbrede standaardwaarden in via `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

Ondersteunde `defaults`-sleutels: `fontFamily`, `fontSize`, `lineSpacing`, `layout`, `showLineNumbers`, `diffIndicators`, `wordWrap`, `background`, `theme`, `fileFormat`, `fileQuality`, `fileScale`, `fileMaxWidth`, `mode`, `ttlSeconds`. Expliciete parameters van toolaanroepen overschrijven deze.

### Configuratie voor permanente viewer-URL

<ParamField path="viewerBaseUrl" type="string">
  Door de plugin beheerde terugvalwaarde voor geretourneerde viewerlinks wanneer een toolaanroep geen `baseUrl` doorgeeft. Moet `http` of `https` zijn, zonder query/hash.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Beveiligingsconfiguratie

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: aanvragen naar viewerroutes die niet van loopback afkomstig zijn, worden geweigerd. `true`: externe viewers zijn toegestaan als het pad met token geldig is.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Levenscyclus en opslag van artefacten

- Viewer-HTML en metadata bevinden zich in de gedeelde `state/openclaw.sqlite`-database onder de blobnaamruimte van de Diffs-plugin. HTML wordt met gzip gecomprimeerd; SQLite slaat alleen een SHA-256-hash van het willekeurige URL-token op, niet het token zelf.
- Gerenderde PNG-/PDF-bestanden blijven tijdelijke materialisaties onder `$TMPDIR/openclaw-diffs`, omdat levering via een kanaal een bestandspad vereist. SQLite beheert de metadata over hun vervaldatum; er worden geen JSON-nevenbestanden geschreven.
- Standaard-TTL voor artefacten: 30 minuten. Maximaal geaccepteerde TTL: 6 uur.
- Opschoning wordt opportunistisch uitgevoerd na elke aanroep voor het maken van een artefact. Verlopen SQLite-rijen worden eerst verwijderd, gevolgd door de bijbehorende PNG-/PDF-map.
- Een fallback-scan verwijdert tijdelijke mappen zonder rij die ouder zijn dan 24 uur. Verouderde caches van `meta.json`, `file-meta.json` en `viewer.html` worden niet geïmporteerd of gelezen.

## Viewer-URL en netwerkgedrag

Viewer-route: `/plugins/diffs/view/{artifactId}/{token}`

Viewer-assets:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js` (alleen wanneer de diff een taal uit een taalpakket gebruikt)

Het viewer-document resolveert deze assets relatief ten opzichte van de viewer-URL, zodat een optioneel padvoorvoegsel `baseUrl` ook wordt toegepast op assetaanvragen.

Volgorde voor URL-resolutie: `baseUrl` van de toolaanroep (na strikte validatie) -> `viewerBaseUrl` van de plugin -> standaard loopback `127.0.0.1`. Als de bindmodus van de Gateway `custom` is en `gateway.customBindHost` is ingesteld, wordt die host gebruikt in plaats van loopback.

Regels voor `baseUrl`: moet `http://` of `https://` zijn; query en hash worden geweigerd; een origin met een optioneel basispad is toegestaan.

## Beveiligingsmodel

<AccordionGroup>
  <Accordion title="Viewer-beveiliging">
    - Standaard alleen via loopback.
    - Viewer-paden met tokens en strikte validatie van ID- en tokenpatronen.
    - CSP van de viewer-respons: `default-src 'none'`; scripts/assets alleen van dezelfde bron; geen uitgaande `connect-src`.
    - Beperking van externe mislukte pogingen wanneer externe toegang is ingeschakeld: 40 mislukte pogingen binnen 60 seconden activeren een blokkering van 60 seconden (`429 Too Many Requests`).

  </Accordion>
  <Accordion title="Beveiliging van bestandsrendering">
    - Routering van browseraanvragen voor schermafbeeldingen weigert standaard alles.
    - Alleen lokale viewer-assets van `http://127.0.0.1/plugins/diffs/assets/*` zijn toegestaan.
    - Externe netwerkaanvragen worden geblokkeerd.

  </Accordion>
</AccordionGroup>

## Browservereisten voor de bestandsmodus

`mode: "file"` en `mode: "both"` vereisen een Chromium-compatibele browser.

Resolutievolgorde:

<Steps>
  <Step title="Configuratie">
    `browser.executablePath` in de OpenClaw-configuratie.
  </Step>
  <Step title="Omgevingsvariabelen">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="Platformfallback">
    Gebruikelijke installatiepaden en `PATH`-zoekacties voor Chrome, Chromium, Edge en Brave.
  </Step>
</Steps>

Veelvoorkomende fouttekst: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`. Los dit op door Chrome, Chromium, Edge of Brave te installeren, of door een van de bovenstaande opties voor het pad naar het uitvoerbare bestand in te stellen.

## Probleemoplossing

<AccordionGroup>
  <Accordion title="Invoervalidatiefouten">
    - `Provide patch or both before and after text.` -- neem zowel `before` als `after` op, of geef `patch` op.
    - `Provide either patch or before/after input, not both.` -- combineer geen invoermodi.
    - `Invalid baseUrl: ...` -- gebruik een `http(s)`-origin met een optioneel pad, zonder query/hash.
    - `{field} exceeds maximum size (...)` -- verklein de payload.
    - Grote patch geweigerd -- verminder het aantal patchbestanden of het totale aantal regels.

  </Accordion>
  <Accordion title="Toegankelijkheid van de viewer">
    - De viewer-URL resolveert standaard naar `127.0.0.1`.
    - Stel voor externe toegang `viewerBaseUrl` van de plugin in, geef `baseUrl` per aanroep door, of gebruik `gateway.bind=custom` met `gateway.customBindHost`.
    - Als `gateway.trustedProxies` loopback bevat voor een proxy op dezelfde host (bijvoorbeeld Tailscale Serve), mislukken directe loopback-aanvragen aan de viewer zonder doorgestuurde headers met het client-IP standaard volgens het fail-closed-principe.
    - Geef voor die proxytopologie de voorkeur aan `mode: "file"`/`"both"` voor een bijlage, of schakel bewust `security.allowRemoteViewer` in plus `viewerBaseUrl` van de plugin/een `baseUrl` van de proxy voor een deelbare viewer-link.
    - Schakel `security.allowRemoteViewer` alleen in wanneer externe toegang tot de viewer bedoeld is.

  </Accordion>
  <Accordion title="Rij met ongewijzigde regels heeft geen uitvouwknop">
    Dit is te verwachten bij patchinvoer zonder uitvouwbare context; het is geen viewer-fout.
  </Accordion>
  <Accordion title="Artefact niet gevonden">
    - Artefact is vanwege de TTL verlopen.
    - Token of pad is gewijzigd.
    - De opschoning heeft verouderde gegevens verwijderd.

  </Accordion>
</AccordionGroup>

## Operationele richtlijnen

- Geef de voorkeur aan `mode: "view"` voor lokale interactieve reviews in canvas.
- Geef de voorkeur aan `mode: "file"` voor uitgaande chatkanalen die een bijlage nodig hebben.
- Houd `allowRemoteViewer` uitgeschakeld, tenzij je implementatie externe viewer-URL's vereist.
- Stel voor gevoelige diffs een expliciete korte `ttlSeconds` in.
- Vermijd het verzenden van geheimen in diffinvoer wanneer dat niet nodig is.
- Als je kanaal afbeeldingen sterk comprimeert (bijvoorbeeld Telegram of WhatsApp), geef dan de voorkeur aan PDF-uitvoer (`fileFormat: "pdf"`).

<Note>
Diff-renderingengine mogelijk gemaakt door [Diffs](https://diffs.com).
</Note>

## Gerelateerd

- [Browser](/nl/tools/browser)
- [Plugins](/nl/tools/plugin)
- [Overzicht van tools](/nl/tools)
