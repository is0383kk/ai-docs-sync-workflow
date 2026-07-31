---
read_when:
    - Je stelt een CLAW.md-manifest op of valideert dit
    - Je wilt één agent uit een Claw bekijken of toevoegen
    - Je moet het eigenaarschap, de drift of het opschoongedrag van Claw controleren
summary: Experimentele Claw-agentpakketten maken, toevoegen, bijwerken en verwijderen
title: Klauwen
x-i18n:
    generated_at: "2026-07-27T04:59:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Een Claw is een geversioneerde configuratie voor één nieuwe OpenClaw-agent. Deze kan de
overdraagbare identiteit, werkruimtebestanden, Skills, plugins, MCP-servers en
cron-taken van de agent beschrijven. Harness-specifieke agentinstellingen kunnen worden opgenomen in een profiel waarnaar
vanuit het pakket wordt verwezen. Een Claw vervangt of wijzigt geen bestaande agent.

Claws zijn experimenteel. Hun schema, opdrachtuitvoer en levenscyclus kunnen veranderen.
Schakel het opdrachtoppervlak expliciet in:

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

De huidige CLI leest een lokale pakketmap, `CLAW.md` of een gegroepeerd JSON-manifest.
Het publiceren, zoeken en installeren van volledige Claws via ClawHub vormen een
afzonderlijk registertraject en maken nog geen deel uit van dit opdrachtoppervlak.

## Een Claw-pakket maken

Een pakket bevat `package.json`, een `CLAW.md`-manifest en eventuele profielen of
werkruimte-sidecars waarnaar dat manifest verwijst:

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` begint met YAML-frontmatter. De Markdown-body beschrijft de Claw
voor mensen en maakt geen deel uit van de agentconfiguratie:

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: Incidenttriage
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# Incidenttriage

Maakt één agent voor het beoordelen en routeren van incidenten.
```

`metadata` is een tekenreeks-naar-tekenreeks-toewijzing voor overdraagbare aanwijzingen voor consumenten. De
`openclaw.config`-sleutel van OpenClaw verwijst naar een optioneel, pakketrelatief YAML-profiel. De
geëxporteerde standaardwaarde is `profiles/openclaw.yml`; de verwijzing is normatief, zodat een
pakket een ander veilig relatief `.yml`- of `.yaml`-pad kan kiezen.

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

Dit profiel bestaat alleen binnen het Claw-pakket. OpenClaw valideert en gebruikt het
tijdens het inspecteren, toevoegen, bijwerken en exporteren van die Claw; het wordt niet gekopieerd
naar het normale OpenClaw-configuratiepad van de gebruiker. Andere harnesses kunnen
de metadata-sleutel met namespace negeren en de overdraagbare manifestvelden gebruiken.

Hetzelfde strikte versie 1-schema blijft gegroepeerde JSON-manifesten accepteren.
Gegroepeerde JSON gebruikt dezelfde `metadata.openclaw.config`-verwijzing in plaats van
een tweede kopie van het OpenClaw-profiel in te sluiten. De overige schemafragmenten
op deze pagina gebruiken JSON, met equivalente sleutels beschikbaar in `CLAW.md`-frontmatter.

Het OpenClaw-pakketprofiel kan elk ingebouwd toolprofiel selecteren dat door
de actieve OpenClaw-versie is geregistreerd en het vervolgens verfijnen met `alsoAllow`, `deny` en
`tools.fs.workspaceOnly: true`. Een Claw kan dat veld niet instellen op `false` en
de bestandssysteembeperking van de host verzwakken. `tools.allow` blijft beschikbaar als een
expliciete toestemmingslijst, maar kan niet worden gecombineerd met `alsoAllow`. Een Claw kan ook
`memory.search.enabled` instellen, de overdraagbare bronnen `memory` en `sessions` kiezen
en geheugen over gesprekken heen inschakelen met `rememberAcrossConversations`.
Het declareren van de bron `sessions` vereist die inschakeling.
Het hostbeleid blijft deze instellingen beperken en Claws bevatten geen aangepaste
profieldefinities, providers, aanmeldgegevens, bindingen of lokale geheugenpaden.
Het profiel waarnaar wordt verwezen is beperkt tot 256 KiB, moet JSON-compatibele YAML zijn, mag
geen aliassen, ankers, tags of merge-sleutels gebruiken en moet een normaal,
niet-gesymbolisch gekoppeld en niet-hardgekoppeld bestand binnen het pakket zijn.

Pakket- en werkruimtepaden moeten binnen de pakketroot blijven. Manifesten zijn
beperkt tot 1 MiB, pakketmetadata tot 256 KiB en werkruimtebronnen hanteren
afzonderlijke limieten per bestand en voor het totaal. Werkruimtebronnen weigeren ook ouders
die symbolische koppelingen zijn.

Werkruimtebestanden worden per pad gedeclareerd en uit pakket-sidecars gelezen. Bootstrapbestanden
zoals `SOUL.md` gebruiken benoemde vermeldingen; aanvullende bestanden gebruiken pakketrelatieve
bronnen en werkruimterelatieve doelen:

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Skills en plugins gebruiken exacte ClawHub-versies:

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

De proefuitvoering gebruikt de bestaande preflightpaden voor Skills en plugins om het
exacte artefact, de integriteit en eventuele ClawHub-vertrouwenswaarschuwing vóór toestemming te bepalen. De
waarschuwing blijft zichtbaar in het aan integriteit gebonden plan. Toepassen installeert ontbrekende artefacten
of hergebruikt overeenkomende artefacten en registreert of de Claw elke hulpbron heeft geïntroduceerd of ernaar heeft verwezen.
Plugins blijven procesbrede OpenClaw-mogelijkheden in plaats van
installaties per agent.

Cron-taken declareren gepland werk voor de nieuwe agent:

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "Dagelijks incidentoverzicht",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "Vat actieve incidenten samen."
    }
  ]
}
```

Claws gebruiken de bestaande Gateway-planner en binden gemaakte taken aan de nieuwe
agent. Voorbeeldweergave, herkomst, status en verwijdering omvatten deze taken zonder
het gedrag van gewone cron-opdrachten te wijzigen. Bij verwijdering wordt de live taak
opnieuw via de Gateway gelezen en behouden wanneer de beheerde definitie na
de planning is gewijzigd.

MCP-declaraties gebruiken het bestaande configuratiemodel `mcp.servers`:

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

Omgevingsverwijzingen blijven verwijzingen; Claws sluiten geen opgeloste geheime
waarden in. Een declaratie zonder conflicten wordt beheerd, terwijl naar een exact bestaande
of gedeelde declaratie wordt verwezen. Voorbeeldweergave, herkomst, status, export en
verwijdering volgen hetzelfde eigendomsbeleid als andere Claw-hulpbronnen.

## Inspecteren en vooraf bekijken

Valideer de bron zonder lokale wijzigingen te plannen:

```bash
openclaw claws inspect ./incident-triage.claw.json
```

Bekijk alle voorgestelde levenscyclusacties vooraf:

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

Het plan rapporteert de afgeleide agent en werkruimte, elke voorgestelde actie,
vereisten, blokkades, afzonderlijke uitbreidingen van mogelijkheden en een `planIntegrity`-
digest. Mogelijkheidsrecords tonen het exacte effect op pakketten, MCP, gepland werk, sandbox,
tools of Heartbeat. Beoordeel het plan voordat je de agent maakt:

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

Alleen `--yes` is onvoldoende. OpenClaw bouwt het plan opnieuw op en weigert toestemming
wanneer de bron, bestemming of liveconfiguratie na de voorbeeldweergave is gewijzigd. Gebruik
`--agent-id` of `--workspace` tijdens zowel de voorbeeldweergave als het toepassen wanneer pakketstandaarden
botsen met de lokale status. Geef voor tijdelijke profielen en parallelle validatie
een expliciete `--workspace` door; `OPENCLAW_STATE_DIR` verplaatst de runtimestatus, maar
wijzigt de standaardlocatie van de werkruimte niet.

Het toevoegen van een Claw maakt de nieuwe agent en werkruimteconfiguratie, schrijft gedeclareerde
werkruimtebestanden, installeert of hergebruikt gedeclareerde Skill- en pluginartefacten en
registreert de herkomst van pakketten, MCP en Cron. Bestaande bestanden worden niet overschreven
en nieuwe pogingen worden veilig afgebroken wanneer beheerde inhoud is afgeweken.

## Geïnstalleerde status inspecteren

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` vergelijkt de geïnstalleerde agent en de geregistreerde herkomst van werkruimte, pakketten, MCP
en Cron met de huidige status. Het rapporteert onvolledige installaties, ontbrekende
hulpbronnen en afwijkingen zonder de lokale status te wijzigen. `openclaw doctor` voegt
Claw-specifieke diagnostiek toe voor onvolledige eigendomsrecords, onveilige beheerde
bestanden en cron-taken die niet met de live-inventaris van de Gateway kunnen worden bevestigd.

De herkomst van een Claw onderscheidt twee relaties:

- **Beheerd:** de Claw heeft de hulpbron geïntroduceerd en beheert deze momenteel. De hulpbron komt
  in aanmerking voor opschoning wanneer deze ongewijzigd is en er geen conflicterende eigenaar overblijft.
- **Naar verwezen:** de hulpbron bestond onafhankelijk of wordt gedeeld. Verwijdering
  maakt de verwijzing van deze Claw vrij en behoudt de hulpbron standaard.

Dit is geen referentietelling. Gewone opdrachten voor plugins, Skills en agents behouden
hun bestaande gedrag; Claws voegen daar herkomst en beveiligde levenscyclusbewerkingen
aan toe.

## Een geïnstalleerde Claw bijwerken

Standaard gebruikt de update de bron die is geregistreerd toen de Claw werd toegevoegd. Gebruik
`--from` wanneer die bron is verplaatst of wanneer je een andere pakketmap test:

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

Het plan vergelijkt de huidige herkomst en live-status met het doelmanifest.
Het rapporteert wijzigingen in agent, werkruimte, pakketten, MCP, Cron en eigendom,
inclusief uitbreidingen van mogelijkheden en blokkades. Uitbreidingen van mogelijkheden hebben
afzonderlijke machineleesbare records en `!`-regels met exact geredigeerde effecten in
voor mensen leesbare uitvoer. Opgeloste pakketintegriteit, installatie-identiteit en eventuele
vertrouwenswaarschuwingen worden opgenomen. Het verwijderen van een pakketdeclaratie maakt de koppeling van deze Claw vrij
zonder het artefact tijdens de update te verwijderen. De uiteindelijke
exacte bevestiging `planIntegrity` bindt zowel die bekendgemaakte set als gewone
inhoudswijzigingen. Hosts kunnen dezelfde records gebruiken voor een afzonderlijk dialoogvenster of een
geaggregeerde beoordeling van meerdere agents. Pas het exact beoordeelde plan toe met expliciete
toestemming:

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw bouwt het plan opnieuw op en voert vóór elke mutatie een compare-and-swap uit op de beheerde status.
Verwijderde pakketdeclaraties maken afhankelijkheidskoppelingen vrij zonder
artefacten te verwijderen. Bij Cron-wijzigingen wordt de definitie van de liveplanner opnieuw gelezen en
wordt gestopt bij afwijkingen door de operator. Pakketinstallatieprogramma's, schrijvers van bronconfiguraties en de Gateway-planner
vormen niet één transactie. Als compensatie na een externe
mutatie niet kan worden bewezen, rapporteert OpenClaw foutcode `update_partial` met gestructureerde
`status: partial`, behoudt het onzekere herkomstgegevens
en stopt het. Inspecteer `claws status`, de betrokken hulpbron en `openclaw doctor`;
bekijk daarna opnieuw een voorbeeld voordat je het opnieuw probeert of iets verwijdert.

## Een geïnstalleerde Claw verwijderen

Bekijk de verwijdering vooraf voordat je opschoning selecteert:

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

Standaard wordt in aanmerking komende beheerde status verwijderd en status waarnaar wordt verwezen vrijgegeven.
Gewijzigde bestanden en hulpbronnen met een andere huidige eigenaar worden behouden of
geblokkeerd. Opschoningskeuzes maken deel uit van de plandigest; `--yes` breidt
deze nooit uit. Wereldwijd geïnstalleerde plugins worden behouden terwijl de verwijzing van deze Claw wordt
vrijgegeven; gebruik de gewone pluginlevenscyclus afzonderlijk wanneer je een
procesbrede plugin wilt verwijderen.

Om ongewijzigde, door de Claw geïntroduceerde verwijzingen te verwijderen die geen andere huidige
eigenaar hebben, neem je `--remove-unused` op in zowel de voorbeeldweergave als het toepassen. Om in plaats daarvan exacte
hulpbronnen waarnaar wordt verwezen te selecteren, herhaal je `--remove-referenced`:

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

Gebruik `--force-referenced` alleen nadat je de weergegeven afhankelijken,
onafhankelijke eigenaren en reeds bestaande oorsprong hebt beoordeeld. Hiermee is geselecteerde opschoning ondanks
die conflicten toegestaan; toestemming op basis van planintegriteit wordt hiermee niet overgeslagen.

## Een geïnstalleerde agent exporteren

Export maakt een nieuwe pakketmap aan en mislukt als de bestemming bestaat of
de beheerde status is afgeweken:

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

Het resultaat bevat `package.json`, canonieke `CLAW.md` en sidecars van de beheerde
werkruimte. Het is een overdraagbaar Claw-pakket, geen back-up van de volledige instantie: niet-gerelateerde
agents, referenties, sessies en lokale status zonder eigenaar worden uitgesloten.

## Opdrachtenoverzicht

| Opdracht                            | Doel                                                |
| ----------------------------------- | --------------------------------------------------- |
| `claws inspect <source>`            | Valideer een pakketmap of gegroepeerd manifest.     |
| `claws add <source>`                | Bekijk een voorbeeld of maak één nieuwe agent en werkruimte aan. |
| `claws status [claw-or-agent]`      | Rapporteer geïnstalleerde status, eigendom en afwijkingen. |
| `claws update <claw-or-agent>`      | Bekijk een voorbeeld of pas wijzigingen uit de geselecteerde bron toe. |
| `claws remove <claw-or-agent>`      | Bekijk een voorbeeld of verwijder de agent en in aanmerking komende resources. |
| `claws export <agent> --out <path>` | Maak een overdraagbaar pakket van een geïnstalleerde agent. |

Gebruik `--json` voor experimentele machineleesbare uitvoer.

## Zie ook

- [Agents](/nl/cli/agents)
- [Skills](/nl/tools/skills)
- [Plugins](/nl/tools/plugin)
- [Cron-taken](/nl/automation/cron-jobs)
- [MCP-configuratie](/nl/gateway/configuration-reference#mcp)
