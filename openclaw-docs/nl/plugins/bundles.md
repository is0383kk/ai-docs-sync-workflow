---
read_when:
    - Je wilt een bundel installeren die compatibel is met Codex, Claude of Cursor
    - Je moet begrijpen hoe OpenClaw bundelinhoud omzet in systeemeigen functies
    - Je debugt bundeldetectie of ontbrekende mogelijkheden
summary: Installeer en gebruik Codex-, Claude- en Cursor-bundels als OpenClaw-plugins
title: Pluginbundels
x-i18n:
    generated_at: "2026-07-27T05:39:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw kan plugins uit drie externe ecosystemen installeren: **Codex**, **Claude**
en **Cursor**. Deze worden **bundels** genoemd: pakketten met inhoud en metadata die
OpenClaw omzet in systeemeigen functies zoals Skills, hooks en MCP-tools.

<Info>
  Bundels zijn **niet** hetzelfde als systeemeigen OpenClaw-plugins. Systeemeigen plugins worden
  in het proces uitgevoerd en kunnen elke mogelijkheid registreren. Bundels zijn inhoudspakketten met
  selectieve functietoewijzing en een beperktere vertrouwensgrens.
</Info>

## Waarom bundels bestaan

Veel nuttige plugins worden gepubliceerd in de indeling van Codex, Claude of Cursor. In plaats
van auteurs te verplichten ze te herschrijven als systeemeigen OpenClaw-plugins, detecteert OpenClaw
deze indelingen en wijst de ondersteunde inhoud ervan toe aan de systeemeigen functieset.
Je kunt een Claude-opdrachtenpakket of een Codex-Skills-bundel installeren en deze
direct gebruiken.

## Een bundel installeren

<Steps>
  <Step title="Installeren vanuit een map, archief of marketplace">
    ```bash
    # Lokale map
    openclaw plugins install ./my-bundle

    # Archief
    openclaw plugins install ./my-bundle.tgz

    # Claude-marketplace
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` is een lokaal marketplace-pad of een lokale marketplace-repo, of een git-/GitHub-bron.

  </Step>

  <Step title="Detectie verifiëren">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    Bundels tonen `Format: bundle` plus een `Bundle format:`-waarde van `codex`,
    `claude` of `cursor`.

  </Step>

  <Step title="Opnieuw starten en gebruiken">
    ```bash
    openclaw gateway restart
    ```

    Toegewezen functies (Skills, hooks, MCP-tools, LSP-standaardwaarden) zijn beschikbaar in de volgende sessie.

  </Step>
</Steps>

## Wat OpenClaw uit bundels toewijst

Niet elke bundelfunctie wordt momenteel in OpenClaw uitgevoerd. Hieronder staat wat werkt en wat
wel wordt gedetecteerd, maar nog niet is aangesloten.

### Momenteel ondersteund

| Functie       | Toewijzing                                                                                       | Van toepassing op |
| ------------- | ------------------------------------------------------------------------------------------------- | ----------------- |
| Skills-inhoud | Hoofdmappen met bundel-Skills worden geladen als normale OpenClaw-Skills                          | Alle indelingen   |
| Opdrachten    | `commands/` en `.cursor/commands/` worden behandeld als hoofdmappen met Skills                    | Claude, Cursor    |
| Hookpakketten | OpenClaw-indelingen met `HOOK.md` + `handler.ts`                                         | Codex             |
| MCP-tools     | MCP-configuratie van de bundel wordt samengevoegd met ingesloten OpenClaw-instellingen; ondersteunde stdio- en HTTP-servers worden geladen | Alle indelingen |
| LSP-servers   | Claude-`.lsp.json` en in het manifest gedeclareerde `lspServers` worden samengevoegd met ingesloten OpenClaw-LSP-standaardwaarden | Claude |
| Instellingen  | Claude-`settings.json` wordt geïmporteerd als ingesloten OpenClaw-standaardwaarden                   | Claude            |

#### Skills-inhoud

- Hoofdmappen met bundel-Skills worden geladen als normale hoofdmappen met OpenClaw-Skills.
- Claude-`commands/`-hoofdmappen worden behandeld als aanvullende hoofdmappen met Skills.
- Cursor-`.cursor/commands/`-hoofdmappen worden behandeld als aanvullende hoofdmappen met Skills.

Markdown-opdrachtbestanden van Claude en Markdown-opdrachten van Cursor werken beide via de
normale OpenClaw-Skills-lader.

#### Hookpakketten

Hoofdmappen met bundelhooks werken **alleen** wanneer ze de normale indeling voor
OpenClaw-hookpakketten gebruiken: `HOOK.md` plus `handler.ts` of `handler.js`. Momenteel is dit voornamelijk
het Codex-compatibele geval.

#### MCP voor ingesloten OpenClaw

- Ingeschakelde bundels kunnen MCP-serverconfiguratie aanleveren.
- OpenClaw voegt de MCP-configuratie van de bundel als `mcpServers` samen met de effectieve
  instellingen van ingesloten OpenClaw.
- OpenClaw stelt ondersteunde MCP-tools uit bundels beschikbaar tijdens agentbeurten
  van ingesloten OpenClaw door stdio-servers te starten of verbinding te maken met HTTP-servers.
- De toolprofielen `coding` en `messaging` bevatten standaard MCP-tools uit bundels;
  gebruik `tools.deny: ["bundle-mcp"]` om dit voor een agent of Gateway uit te schakelen.
- Projectlokale instellingen voor ingesloten agents blijven van toepassing na de standaardwaarden van de bundel,
  zodat werkruimte-instellingen indien nodig MCP-vermeldingen van de bundel kunnen overschrijven.
- MCP-toolcatalogi van bundels worden vóór registratie deterministisch gesorteerd, zodat
  wijzigingen in de volgorde van upstream-`listTools()` geen voortdurende wijzigingen in toolblokken van de promptcache veroorzaken.

##### Transporten

MCP-servers kunnen stdio- of HTTP-transport gebruiken.

**Stdio** start een onderliggend proces:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** maakt verbinding met een actieve MCP-server en gebruikt standaard `sse`, tenzij
`streamable-http` wordt aangevraagd:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` accepteert `"streamable-http"` of `"sse"`; bij weglating is de standaardwaarde `sse`.
- `type: "http"` is een CLI-eigen downstream-vorm; gebruik `transport: "streamable-http"` in de OpenClaw-configuratie. `openclaw mcp set` en `openclaw doctor --fix` normaliseren de algemene alias.
- Alleen de URL-schema's `http:` en `https:` zijn toegestaan.
- `headers`-waarden ondersteunen `${ENV_VAR}`-interpolatie.
- Een serververmelding met zowel `command` als `url` wordt geweigerd.
- URL-aanmeldgegevens (gebruikersinformatie en queryparameters) worden afgeschermd in toolbeschrijvingen
  en logboeken.
- `connectionTimeoutMs` overschrijft de standaardverbindingstime-out van 30 seconden voor
  zowel stdio- als HTTP-transporten. De time-out voor aanvragen is standaard 60 seconden en
  kan worden overschreven met `requestTimeoutMs`.

##### Toolnamen

OpenClaw registreert MCP-tools uit bundels met providerveilige namen in de vorm
`serverName__toolName`. Een server met de sleutel `"vigil-harbor"` die bijvoorbeeld een
`memory_search`-tool beschikbaar stelt, wordt geregistreerd als `vigil-harbor__memory_search`.

- Tekens buiten `A-Za-z0-9_-` worden vervangen door `-`.
- Fragmenten die met een niet-letter zouden beginnen, krijgen een letter als voorvoegsel, zodat numerieke
  serversleutels zoals `12306` providerveilige toolvoorvoegsels worden.
- Servervoorvoegsels zijn beperkt tot 30 tekens.
- Volledige toolnamen zijn beperkt tot 64 tekens.
- Lege servernamen vallen terug op `mcp`.
- Botsende opgeschoonde namen worden onderscheiden met numerieke achtervoegsels.
- De uiteindelijke volgorde van beschikbaar gestelde tools is deterministisch op basis van de veilige naam, waardoor herhaalde
  beurten van ingesloten agents cachestabiel blijven.
- Profielfiltering behandelt elke tool van één MCP-server uit een bundel als
  eigendom van de plugin `bundle-mcp`, zodat lijsten met toegestane/geweigerde profielen kunnen verwijzen naar
  afzonderlijke beschikbaar gestelde toolnamen of de pluginsleutel `bundle-mcp`.

#### Instellingen voor ingesloten OpenClaw

Claude-`settings.json` wordt geïmporteerd als standaardinstellingen voor ingesloten OpenClaw wanneer
de bundel is ingeschakeld. OpenClaw schoont sleutels voor shelloverschrijvingen op voordat
ze worden toegepast:

- `shellPath`
- `shellCommandPrefix`

#### LSP voor ingesloten OpenClaw

- Ingeschakelde Claude-bundels kunnen LSP-serverconfiguratie aanleveren.
- OpenClaw laadt `.lsp.json` plus alle in het manifest gedeclareerde `lspServers`-paden.
- De LSP-configuratie van de bundel wordt samengevoegd met de effectieve LSP-standaardwaarden
  van ingesloten OpenClaw.
- Alleen ondersteunde, op stdio gebaseerde LSP-servers kunnen momenteel worden uitgevoerd; niet-ondersteunde
  transporten worden nog steeds weergegeven in `openclaw plugins inspect <id>`.

### Gedetecteerd maar niet uitgevoerd

Deze worden herkend en weergegeven in diagnostische gegevens, maar OpenClaw voert ze niet uit:

- Claude-`agents`, `hooks/hooks.json`-automatisering, `outputStyles`
- Cursor-`.cursor/agents`, `.cursor/hooks.json`, `.cursor/rules`
- Codex-`.app.json`-metadata naast mogelijkhedenrapportage

## Bundelindelingen

<AccordionGroup>
  <Accordion title="Codex-bundels">
    Markeringen: `.codex-plugin/plugin.json`

    Optionele inhoud: `skills/`, `hooks/`, `.mcp.json`, `.app.json`

    Codex-bundels sluiten het beste aan op OpenClaw wanneer ze hoofdmappen met Skills en
    OpenClaw-achtige mappen voor hookpakketten gebruiken (`HOOK.md` + `handler.ts`).

  </Accordion>

  <Accordion title="Claude-bundels">
    Twee detectiemodi:

    - **Op basis van een manifest:** `.claude-plugin/plugin.json`
    - **Zonder manifest:** standaardindeling van Claude (`skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`, `settings.json`)

    Claude-specifiek gedrag:

    - `commands/` wordt behandeld als Skills-inhoud
    - `settings.json` wordt geïmporteerd in de instellingen van ingesloten OpenClaw (sleutels voor shelloverschrijvingen worden opgeschoond)
    - `.mcp.json` stelt ondersteunde stdio-tools beschikbaar aan ingesloten OpenClaw
    - `.lsp.json` plus in het manifest gedeclareerde `lspServers`-paden worden geladen in de LSP-standaardwaarden van ingesloten OpenClaw
    - `hooks/hooks.json` wordt gedetecteerd maar niet uitgevoerd
    - Aangepaste componentpaden in het manifest zijn aanvullend; ze breiden de standaardwaarden uit en vervangen ze niet

  </Accordion>

  <Accordion title="Cursor-bundels">
    Markeringen: `.cursor-plugin/plugin.json`

    Optionele inhoud: `skills/`, `.cursor/commands/`, `.cursor/agents/`, `.cursor/rules/`, `.cursor/hooks.json`, `.mcp.json`

    - `.cursor/commands/` wordt behandeld als Skills-inhoud
    - `.cursor/rules/`, `.cursor/agents/` en `.cursor/hooks.json` worden alleen gedetecteerd

  </Accordion>
</AccordionGroup>

## Detectievolgorde

OpenClaw controleert eerst op de indeling voor systeemeigen plugins:

1. `openclaw.plugin.json` of een geldige `package.json` met `openclaw.extensions` - wordt behandeld als een **systeemeigen plugin**
2. Bundelmarkeringen (`.codex-plugin/`, `.claude-plugin/` of de standaardindeling van Claude/Cursor) - wordt behandeld als een **bundel**

Als een map beide bevat, gebruikt OpenClaw het systeemeigen pad. Dit voorkomt
dat pakketten met twee indelingen gedeeltelijk als bundels worden geïnstalleerd.

## Runtime-afhankelijkheden en opschoning

- Compatibele bundels van derden krijgen bij het opstarten geen `npm install`-reparatie. Ze
  moeten via `openclaw plugins install` worden geïnstalleerd en alles wat
  ze nodig hebben in de geïnstalleerde pluginmap meeleveren.
- Gebundelde plugins die eigendom zijn van OpenClaw worden ofwel lichtgewicht in de kern meegeleverd, of
  kunnen via het installatieprogramma voor plugins worden gedownload. Bij het opstarten van de Gateway wordt hiervoor nooit een
  pakketbeheerder uitgevoerd.
- `openclaw doctor --fix` verwijdert verouderde lokale installatierecords van gebundelde plugins
  en kan downloadbare plugins herstellen die ontbreken in de lokale pluginindex
  wanneer de configuratie er nog naar verwijst.

## Beveiliging

Bundels hebben een beperktere vertrouwensgrens dan systeemeigen plugins:

- OpenClaw laadt **geen** willekeurige runtime-modules van bundels in het proces.
- Paden voor Skills en hookpakketten moeten binnen de hoofdmap van de plugin blijven (grenzen worden gecontroleerd).
- Instellingsbestanden worden met dezelfde grenscontroles gelezen.
- Ondersteunde stdio-MCP-servers kunnen als subprocessen worden gestart.

Hierdoor zijn bundels standaard veiliger, maar je moet bundels van derden nog steeds
als vertrouwde inhoud behandelen voor de functies die ze beschikbaar stellen.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Bundel wordt gedetecteerd, maar mogelijkheden worden niet uitgevoerd">
    Voer `openclaw plugins inspect <id>` uit. Als een mogelijkheid wordt vermeld maar is gemarkeerd als
    niet gekoppeld, is dat een productbeperking en geen defecte installatie.
  </Accordion>

  <Accordion title="Claude-opdrachtbestanden worden niet weergegeven">
    Zorg dat de bundel is ingeschakeld en dat de Markdown-bestanden zich in een gedetecteerde
    `commands/`- of `skills/`-hoofdmap bevinden.
  </Accordion>

  <Accordion title="Claude-instellingen worden niet toegepast">
    Alleen ingesloten OpenClaw-instellingen uit `settings.json` worden ondersteund. OpenClaw
    behandelt bundelinstellingen niet als onbewerkte configuratiepatches.
  </Accordion>

  <Accordion title="Claude-hooks worden niet uitgevoerd">
    `hooks/hooks.json` is alleen voor detectie. Als je uitvoerbare hooks nodig hebt, gebruik dan de
    OpenClaw-hookpakketindeling of lever een native plugin.
  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Plugins installeren en configureren](/nl/tools/plugin)
- [Plugins bouwen](/nl/plugins/building-plugins) - maak een native plugin
- [Pluginmanifest](/nl/plugins/manifest) - schema voor een native manifest
