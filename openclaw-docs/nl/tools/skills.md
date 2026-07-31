---
read_when:
    - Skills toevoegen of wijzigen
    - Gating, allowlists of laadregels voor Skills wijzigen
    - Inzicht in de prioriteit van Skills en het gedrag van snapshots
sidebarTitle: Skills
summary: Skills leren je agent hoe die tools gebruikt. Lees hoe ze worden geladen, hoe prioriteit werkt en hoe je toegangsbeperkingen, toelatingslijsten en omgevingsinjectie configureert.
title: Skills
x-i18n:
    generated_at: "2026-07-27T05:38:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills zijn Markdown-instructiebestanden die de agent leren hoe en wanneer
tools moeten worden gebruikt. Elke skill bevindt zich in een map met een `SKILL.md`-bestand met YAML-
frontmatter en een Markdown-body. OpenClaw laadt gebundelde skills plus eventuele lokale
overschrijvingen en filtert deze tijdens het laden op basis van de omgeving, configuratie en
aanwezigheid van binaire bestanden.

<CardGroup cols={2}>
  <Card title="Skills maken" href="/nl/tools/creating-skills" icon="hammer">
    Bouw en test een aangepaste skill vanaf nul.
  </Card>
  <Card title="Skill Workshop" href="/nl/tools/skill-workshop" icon="flask">
    Beoordeel en keur door de agent opgestelde skillvoorstellen goed.
  </Card>
  <Card title="Skillconfiguratie" href="/nl/tools/skills-config" icon="gear">
    Volledig `skills.*`-configuratieschema en agenttoelatingslijsten.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Bekijk en installeer skills van de community.
  </Card>
</CardGroup>

## Laadvolgorde

OpenClaw laadt uit deze bronnen, **met de hoogste prioriteit eerst**. Wanneer dezelfde
skillnaam op meerdere plaatsen voorkomt, wint de bron met de hoogste prioriteit.

| Prioriteit   | Bron                         | Pad                                     |
| ------------ | ---------------------------- | --------------------------------------- |
| 1 — hoogste  | Werkruimteskills             | `<workspace>/skills`                      |
| 2            | Projectagentskills           | `<workspace>/.agents/skills`                      |
| 3            | Persoonlijke agentskills     | `~/.agents/skills`                      |
| 4            | Beheerde / lokale skills     | `~/.openclaw/skills`                      |
| 5            | Gebundelde skills            | meegeleverd met de installatie          |
| 6 — laagste  | Extra mappen                 | `skills.load.extraDirs` + pluginskills       |

Skillhoofdmappen ondersteunen gegroepeerde indelingen. OpenClaw ontdekt een skill wanneer
`SKILL.md` ergens onder een geconfigureerde hoofdmap voorkomt (tot 6 niveaus diep):

```text
<workspace>/skills/research/SKILL.md          ✓ gevonden als "research"
<workspace>/skills/personal/research/SKILL.md ✓ ook gevonden als "research"
```

Het mappad dient alleen voor ordening. De naam en slashopdracht van de skill
komen uit het frontmatterveld `name` (of uit de mapnaam wanneer `name`
ontbreekt). Agenttoelatingslijsten (hieronder) vergelijken ook met deze `name`.

<Note>
  De ingebouwde map `$CODEX_HOME/skills` van Codex CLI is **geen** hoofdmap voor
  OpenClaw-skills. Gebruik `openclaw migrate plan codex` om die skills te inventariseren en vervolgens
  `openclaw migrate codex` om ze naar je OpenClaw-werkruimte te kopiëren.
</Note>

## Door Node gehoste skills

Een verbonden headless Node kan skills publiceren die in de actieve
OpenClaw-skillsmap zijn geïnstalleerd (standaard `~/.openclaw/skills`; overschrijvingen via
de profielomgeving zijn van toepassing). Ze verschijnen in de normale skillslijst van de agent zolang de Node
verbonden is en verdwijnen wanneer de verbinding wordt verbroken. Bij een naamconflict behoudt een lokale
of Gateway-skill zijn naam; de Node-skill krijgt een deterministische naam met een Node-voorvoegsel.
Voor door Node gehoste v1-skills moet de mapnaam overeenkomen met het frontmatterveld
`name` van de skill.

De skillvermelding bevat de Node-locator. De bestanden, relatieve verwijzingen en
binaire bestanden bevinden zich op de Node; laad de skill daarom en voer deze uit met
`exec host=node node=<node-id>`. Start de Node-host opnieuw nadat je de skillbestanden hebt gewijzigd.
Zie [Nodes](/nl/nodes#node-hosted-skills) voor koppeling en uitschakelopties.

## Skills per agent versus gedeelde skills

In configuraties met meerdere agents heeft elke agent een eigen werkruimte. Gebruik het pad dat
overeenkomt met de gewenste zichtbaarheid:

| Bereik             | Pad                          | Zichtbaar voor                      |
| ------------------ | ---------------------------- | ----------------------------------- |
| Per agent          | `<workspace>/skills`           | Alleen die agent                    |
| Projectagent       | `<workspace>/.agents/skills`           | Alleen de agent van die werkruimte  |
| Persoonlijke agent | `~/.agents/skills`           | Alle agents op deze machine         |
| Gedeeld beheerd    | `~/.openclaw/skills`           | Alle agents op deze machine         |
| Extra mappen       | `skills.load.extraDirs`           | Alle agents op deze machine         |

## Agenttoelatingslijsten

De **locatie** van een skill (prioriteit) en de **zichtbaarheid** van een skill (welke agent deze kan gebruiken)
zijn afzonderlijke instellingen. Gebruik toelatingslijsten om te beperken welke skills een agent ziet,
ongeacht waaruit ze worden geladen.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // gedeelde basisset
    },
    list: [
      { id: "writer" }, // neemt github, weather over
      { id: "docs", skills: ["docs-search"] }, // vervangt de standaardwaarden volledig
      { id: "locked-down", skills: [] }, // geen skills
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="Regels voor toelatingslijsten">
    - Laat `agents.defaults.skills` weg om standaard geen beperkingen voor skills toe te passen.
    - Laat `agents.entries.*.skills` weg om `agents.defaults.skills` over te nemen.
    - Stel `agents.entries.*.skills: []` in om voor die agent geen skills beschikbaar te maken.
    - Een niet-lege lijst `agents.entries.*.skills` is de **definitieve** set — deze wordt niet
      samengevoegd met de standaardwaarden.
    - De effectieve toelatingslijst geldt voor het opbouwen van prompts, het ontdekken van
      slashopdrachten, sandboxsynchronisatie en skillsnapshots.
    - Dit is geen autorisatiegrens voor de hostshell. Als dezelfde agent
      `exec` kan gebruiken, beperk die shell dan afzonderlijk met sandboxing, isolatie
      per OS-gebruiker, uitvoeringsblokkeer-/toelatingslijsten en referenties per resource.
  </Accordion>
</AccordionGroup>

## Plugins en skills

Plugins kunnen hun eigen skills meeleveren door `skills`-mappen op te nemen in
`openclaw.plugin.json` (paden relatief ten opzichte van de hoofdmap van de plugin). Pluginskills worden geladen
wanneer de plugin is ingeschakeld — de browserplugin levert bijvoorbeeld een
`browser-automation`-skill voor browserbesturing in meerdere stappen.

Mappen met pluginskills worden samengevoegd op hetzelfde lage prioriteitsniveau als
`skills.load.extraDirs`, waardoor een gelijknamige gebundelde, beheerde, agent- of werkruimteskill
deze overschrijft. Bepaal de geschiktheid van een pluginskill zelf via
`metadata.openclaw.requires` in de frontmatter, net als bij elke andere skill.

Zie [Plugins](/nl/tools/plugin) en [Tools](/nl/tools) voor het volledige pluginsysteem.

## Skill Workshop

[Skill Workshop](/nl/tools/skill-workshop) is een wachtrij voor voorstellen tussen de agent
en je actieve skillbestanden. Wanneer de agent herbruikbaar werk herkent, stelt deze een
voorstel op in plaats van rechtstreeks naar `SKILL.md` te schrijven. Je beoordeelt het en keurt het goed
voordat er iets verandert.

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Zie [Skill Workshop](/nl/tools/skill-workshop) voor de volledige levenscyclus, CLI-
referentie en configuratie.

## Installeren vanuit ClawHub

[ClawHub](https://clawhub.ai) is het openbare register voor skills. Gebruik
`openclaw skills`-opdrachten voor installatie en updates, of de CLI `clawhub` voor
publicatie en synchronisatie.

| Actie                                      | Opdracht                                               |
| ------------------------------------------ | ------------------------------------------------------ |
| Een skill in de werkruimte installeren     | `openclaw skills install @owner/<slug>`                                     |
| Vanuit een Git-repository installeren      | `openclaw skills install git:owner/repo@ref`                                     |
| Een lokale skillmap installeren            | `openclaw skills install ./path/to/skill --as my-tool`                                     |
| Voor alle lokale agents installeren        | `openclaw skills install @owner/<slug> --global`                                     |
| Alle werkruimteskills bijwerken            | `openclaw skills update --all`                                     |
| Een gedeelde beheerde skill bijwerken      | `openclaw skills update @owner/<slug> --global`                                     |
| Alle gedeelde beheerde skills bijwerken    | `openclaw skills update --all --global`                                     |
| De vertrouwensgrenzen van een skill verifiëren | `openclaw skills verify @owner/<slug>`                                 |
| De gegenereerde Skill Card afdrukken       | `openclaw skills verify @owner/<slug> --card`                                     |
| Publiceren / synchroniseren via ClawHub CLI | `clawhub sync --all`                                    |

<AccordionGroup>
  <Accordion title="Installatiedetails">
    `openclaw skills install` installeert standaard in de map `skills/`
    van de actieve werkruimte. Voeg `--global` toe om in de gedeelde map
    `~/.openclaw/skills` te installeren, die zichtbaar is voor alle lokale agents tenzij
    agenttoelatingslijsten dit beperken.

    Bij Git- en lokale installaties wordt `SKILL.md` in de hoofdmap van de bron verwacht. De slug komt,
    indien geldig, uit `name` in de `SKILL.md`-frontmatter en valt vervolgens terug op de
    map- of repositorynaam. Gebruik `--as <slug>` om dit te overschrijven.
    `openclaw skills update` houdt alleen ClawHub-installaties bij — installeer Git- of
    lokale bronnen opnieuw om ze te vernieuwen.

  </Accordion>
  <Accordion title="Verificatie en beveiligingsscans">
    `openclaw skills verify @owner/<slug>` vraagt ClawHub om de
    `clawhub.skill.verify.v1`-vertrouwensgrenzen van de skill. Geïnstalleerde ClawHub-skills worden geverifieerd
    aan de hand van de versie en het register die in `.clawhub/origin.json` zijn vastgelegd.
    Kale slugs blijven geaccepteerd voor bestaande geïnstalleerde of eenduidige skills, maar
    verwijzingen met een eigenaar voorkomen onduidelijkheid over de uitgever.

    ClawHub-skillpagina's tonen vóór de installatie de status van de nieuwste beveiligingsscan,
    met detailpagina's voor VirusTotal, ClawScan en statische analyse. De
    opdracht sluit af met een niet-nulstatus wanneer ClawHub de verificatie als mislukt markeert. Uitgevers
    kunnen fout-positieve resultaten herstellen via het ClawHub-dashboard of
    `clawhub skill rescan @owner/<slug>`.

  </Accordion>
  <Accordion title="Installaties uit privéarchieven">
    Gateway-clients die levering buiten ClawHub nodig hebben, kunnen een ziparchief met een skill klaarzetten
    met `skills.upload.begin`, `skills.upload.chunk` en `skills.upload.commit`,
    en dit vervolgens installeren met `skills.install({ source: "upload", ... })`. Dit pad is
    standaard uitgeschakeld en vereist `skills.install.allowUploadedArchives: true` in
    `openclaw.json`. Normale ClawHub-installaties hebben die instelling nooit nodig.
  </Accordion>
</AccordionGroup>

## Beveiliging

<Warning>
  Behandel skills van derden als **niet-vertrouwde code**. Lees ze voordat je ze inschakelt.
  Geef voor niet-vertrouwde invoer en risicovolle tools de voorkeur aan uitvoeringen in een sandbox. Zie
  [Sandboxing](/nl/gateway/sandboxing) voor instellingen aan de agentzijde.
</Warning>

<AccordionGroup>
  <Accordion title="Padbegrenzing">
    Bij het ontdekken van werkruimte-, projectagent- en extra-mapskills worden alleen
    skillhoofdmappen geaccepteerd waarvan het opgeloste realpath binnen de geconfigureerde hoofdmap blijft, tenzij
    `skills.load.allowSymlinkTargets` een doelhoofdmap expliciet vertrouwt.
    Skill Workshop schrijft alleen via deze vertrouwde doelen wanneer
    `skills.workshop.allowSymlinkTargetWrites` is ingeschakeld.
    Beheerde `~/.openclaw/skills` en persoonlijke `~/.agents/skills` mogen
    skillmappen met symbolische koppelingen bevatten, maar elk realpath van `SKILL.md` moet nog steeds
    binnen de opgeloste skillmap blijven.
  </Accordion>
  <Accordion title="Installatiebeleid voor operators">
    Configureer `security.installPolicy` om een vertrouwde lokale beleidsopdracht uit te voeren
    voordat skillinstallaties doorgaan. Het beleid ontvangt metagegevens en het klaargezette
    bronpad, is van toepassing op ClawHub-, upload-, Git-, lokale, update- en
    afhankelijkheidsinstallatiepaden, en sluit uit veiligheidsoverwegingen af wanneer de opdracht geen
    geldige beslissing kan retourneren.
  </Accordion>
  <Accordion title="Bereik van geheiminjectie">
    `skills.entries.*.env` en `skills.entries.*.apiKey` injecteren geheimen alleen voor die
    agentbeurt in het **hostproces** — niet in de sandbox. Houd
    geheimen uit prompts en logboeken.
  </Accordion>
</AccordionGroup>

Zie [Beveiliging](/nl/gateway/security) voor het bredere dreigingsmodel en
beveiligingschecklists.

## SKILL.md-indeling

Elke skill heeft minimaal een `name` en `description` in de frontmatter nodig:

```markdown
---
name: image-lab
description: Afbeeldingen genereren of bewerken via een door een provider ondersteunde afbeeldingsworkflow
---

Wanneer de gebruiker vraagt om een afbeelding te genereren, gebruik je de tool `image_generate`...
```

<Note>
  OpenClaw volgt de specificatie [AgentSkills](https://agentskills.io). Frontmatter
  wordt eerst als YAML geparseerd; als dat mislukt, wordt teruggevallen op een parser die
  alleen één regel ondersteunt. Geneste `metadata`-blokken (waaronder YAML-toewijzingen van meerdere regels) worden
  afgevlakt tot een JSON-tekenreeks en opnieuw als JSON5 geparseerd, waardoor de blokvorm die
  onder [Poortvoorwaarden](#gating) wordt weergegeven werkt. Gebruik `{baseDir}` in de body om naar het
  pad van de skillmap te verwijzen.
</Note>

### Optionele frontmattersleutels

<ParamField path="homepage" type="string">
  URL die als "Website" wordt weergegeven in de macOS-gebruikersinterface voor Skills. Wordt ook ondersteund via
  `metadata.openclaw.homepage`.
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  Wanneer `true`, wordt de skill beschikbaar gesteld als een door de gebruiker aanroepbare slash-opdracht.
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  Wanneer `true`, houdt OpenClaw de instructies van de skill buiten de normale
  prompt van de agent. De skill blijft beschikbaar als slash-opdracht wanneer `user-invocable`
  ook `true` is.
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  Wanneer dit is ingesteld op `tool`, omzeilt de slash-opdracht het model en wordt deze
  rechtstreeks naar een geregistreerde tool doorgestuurd.
</ParamField>

<ParamField path="command-tool" type="string">
  Naam van de tool die moet worden aangeroepen wanneer `command-dispatch: tool` is ingesteld.
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  Voor toolroutering wordt de onbewerkte argumenttekenreeks zonder
  kernverwerking naar de tool doorgestuurd. De tool ontvangt
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.
</ParamField>

## Toegangsvoorwaarden

OpenClaw filtert skills tijdens het laden met `metadata.openclaw` (een JSON5-object
dat in de frontmatter is ingesloten; zie de opmerking over parsering hierboven). Een skill zonder
`metadata.openclaw`-blok komt altijd in aanmerking, tenzij deze expliciet is uitgeschakeld.

```markdown
---
name: image-lab
description: Afbeeldingen genereren of bewerken via een door een provider ondersteunde afbeeldingsworkflow
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  Wanneer `true`, wordt de skill altijd opgenomen en worden alle andere voorwaarden overgeslagen.
</ParamField>

<ParamField path="emoji" type="string">
  Optionele emoji die in de macOS-interface voor Skills wordt weergegeven.
</ParamField>

<ParamField path="homepage" type="string">
  Optionele URL die als "Website" in de macOS-interface voor Skills wordt weergegeven.
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  Platformfilter. Wanneer dit is ingesteld, komt de skill alleen in aanmerking op een vermeld besturingssysteem.
</ParamField>

<ParamField path="requires.bins" type="string[]">
  Elk binair bestand moet bestaan op `PATH`.
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  Ten minste één binair bestand moet bestaan op `PATH`.
</ParamField>

<ParamField path="requires.env" type="string[]">
  Elke omgevingsvariabele moet in het proces bestaan of via de configuratie worden opgegeven.
</ParamField>

<ParamField path="requires.config" type="string[]">
  Elk `openclaw.json`-pad moet waarheidsgetrouw zijn.
</ParamField>

<ParamField path="primaryEnv" type="string">
  Naam van de omgevingsvariabele die aan `skills.entries.<name>.apiKey` is gekoppeld.
</ParamField>

<ParamField path="install" type="object[]">
  Optionele installatiespecificaties die worden gebruikt door de macOS-interface voor Skills (brew / node / go / uv / download).
</ParamField>

<Note>
  Verouderde `metadata.clawdbot`-blokken worden nog steeds geaccepteerd wanneer
  `metadata.openclaw` ontbreekt, zodat oudere geïnstalleerde skills hun
  afhankelijkheidsvoorwaarden en installatietips behouden. Nieuwe skills moeten
  `metadata.openclaw` gebruiken.
</Note>

### Installatiespecificaties

Installatiespecificaties vertellen de macOS-interface voor Skills hoe een afhankelijkheid moet worden geïnstalleerd:

```markdown
---
name: gemini
description: Gebruik Gemini CLI voor hulp bij programmeren en zoekopdrachten via Google.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Gemini CLI installeren (brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="Regels voor installatieselectie">
    - Wanneer meerdere installatieprogramma's worden vermeld, kiest de Gateway één voorkeursoptie
      (brew indien beschikbaar, anders node).
    - Als alle installatieprogramma's `download` zijn, vermeldt OpenClaw elk item zodat je
      alle beschikbare artefacten kunt zien.
    - Specificaties kunnen `os: ["darwin"|"linux"|"win32"]` bevatten om op platform te filteren.
    - Node-installaties respecteren `skills.install.nodeManager` in `openclaw.json`
      (standaard: npm; opties: npm / pnpm / yarn / bun). Dit is alleen van invloed op
      skillinstallaties; de Gateway-runtime moet nog steeds Node zijn.
    - Installatievoorkeur van de Gateway: Homebrew → uv → geconfigureerde nodebeheerder →
      go → download.
  </Accordion>
  <Accordion title="Details per installatieprogramma">
    - **Homebrew:** OpenClaw installeert Homebrew niet automatisch en vertaalt brew-
      formules niet naar pakketopdrachten van het systeem. In Linux-containers zonder
      `brew` worden installatieprogramma's die alleen brew ondersteunen verborgen; gebruik een aangepaste image of installeer
      de afhankelijkheid handmatig.
    - **Go:** OpenClaw vereist Go 1.21 of nieuwer voor automatische skillinstallaties.
      Als `go` ontbreekt en Homebrew beschikbaar is, installeert OpenClaw eerst Go via
      Homebrew; op Linux zonder Homebrew kan het in plaats daarvan `apt-get`
      als root of via wachtwoordloze `sudo` gebruiken wanneer de vernieuwde `golang-go`-
      kandidaat aan de minimumversie voldoet. De daadwerkelijke `go install` voor de
      afhankelijkheid is altijd gericht op een speciale, door OpenClaw beheerde map voor binaire bestanden
      (`bin` van Homebrew bij een nieuwe installatie, anders `~/.local/bin`) en niet op
      je geconfigureerde `GOBIN` — je eigen omgevingsvariabelen `GOBIN`, `GOPATH` en `GOTOOLCHAIN`
      worden gelezen, maar nooit overschreven.
    - **Download:** `url` (vereist), `archive` (`tar.gz` | `tar.bz2` | `zip`),
      `extract` (standaard: automatisch wanneer een archief wordt gedetecteerd), `stripComponents`,
      `targetDir` (standaard: `~/.openclaw/tools/<skillKey>`).
  </Accordion>
  <Accordion title="Opmerkingen over sandboxing">
    `requires.bins` wordt tijdens het laden van de skill op de **host** gecontroleerd. Als een agent
    in een sandbox wordt uitgevoerd, moet het binaire bestand ook **in de container** bestaan.
    Installeer het via `agents.defaults.sandbox.docker.setupCommand` of een aangepaste
    image. `setupCommand` wordt eenmaal na het maken van de container uitgevoerd en vereist
    uitgaand netwerkverkeer, een beschrijfbaar rootbestandssysteem en een rootgebruiker in de sandbox.
  </Accordion>
</AccordionGroup>

## Configuratieoverschrijvingen

Schakel gebundelde of beheerde skills in of uit en configureer ze onder `skills.entries` in
`~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` schakelt de skill uit, zelfs wanneer deze is gebundeld of geïnstalleerd. De gebundelde skill
  `coding-agent` is opt-in — stel `skills.entries.coding-agent.enabled: true` in
  en zorg dat een van `claude`, `codex`, `opencode` of een andere ondersteunde CLI
  is geïnstalleerd en geauthenticeerd.
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  Gemaksveld voor skills die `metadata.openclaw.primaryEnv` declareren.
  Ondersteunt een tekenreeks met platte tekst of een SecretRef-object.
</ParamField>

<ParamField path="env" type="Record<string, string>">
  Omgevingsvariabelen die voor de agentuitvoering worden geïnjecteerd. Worden alleen geïnjecteerd wanneer de
  variabele nog niet in het proces is ingesteld.
</ParamField>

<ParamField path="config" type="object">
  Optionele verzameling aangepaste configuratievelden per skill.
</ParamField>

<ParamField path="allowBundled" type="string[]">
  Optionele toelatingslijst, uitsluitend voor **gebundelde** skills. Wanneer deze is ingesteld, komen alleen gebundelde skills
  in de lijst in aanmerking. Beheerde skills en werkruimteskills worden niet beïnvloed.
</ParamField>

<Note>
  Configuratiesleutels komen standaard overeen met de **skillnaam**. Als een skill
  `metadata.openclaw.skillKey` definieert, gebruik je in plaats daarvan die sleutel onder `skills.entries`.
  Zet namen met koppeltekens tussen aanhalingstekens: JSON5 staat sleutels tussen aanhalingstekens toe.
</Note>

## Omgevingsinjectie

Wanneer een agentuitvoering start, voert OpenClaw het volgende uit:

<Steps>
  <Step title="Leest skillmetadata">
    OpenClaw bepaalt de effectieve skilllijst voor de agent en past daarbij toegangsvoorwaarden,
    toelatingslijsten en configuratieoverschrijvingen toe.
  </Step>
  <Step title="Injecteert omgevingsvariabelen en API-sleutels">
    `skills.entries.<key>.env` en `skills.entries.<key>.apiKey` worden gedurende
    de uitvoering toegepast op `process.env`.
  </Step>
  <Step title="Bouwt de systeemprompt">
    In aanmerking komende skills worden gecompileerd tot een compact XML-blok en in de
    systeemprompt geïnjecteerd.
  </Step>
  <Step title="Herstelt de omgeving">
    Nadat de uitvoering is beëindigd, wordt de oorspronkelijke omgeving hersteld.
  </Step>
</Steps>

<Warning>
  Omgevingsinjectie is beperkt tot de agentuitvoering op de **host**, niet tot de sandbox. Binnen een
  sandbox hebben `env` en `apiKey` geen effect. Zie
  [Skills-configuratie](/nl/tools/skills-config#sandboxed-skills-and-env-vars) voor informatie over het
  doorgeven van geheimen aan uitvoeringen in een sandbox.
</Warning>

Voor de gebundelde `claude-cli`-backend maakt OpenClaw dezelfde
momentopname van in aanmerking komende skills ook beschikbaar als een tijdelijke Claude Code-plugin en geeft deze door via
`--plugin-dir`. Andere CLI-backends gebruiken alleen de promptcatalogus.

## Momentopnamen en vernieuwen

OpenClaw maakt **wanneer een sessie start** een momentopname van in aanmerking komende skills en hergebruikt die
lijst voor alle volgende beurten in de sessie. Wijzigingen aan skills of configuratie worden
van kracht bij de volgende nieuwe sessie.

Skills worden tijdens een sessie in twee gevallen vernieuwd:

- De skillwatcher detecteert een wijziging in `SKILL.md`.
- Een nieuwe in aanmerking komende externe node maakt verbinding.

De vernieuwde lijst wordt bij de volgende agentbeurt opgehaald. Als de effectieve
toelatingslijst van de agent verandert, vernieuwt OpenClaw de momentopname om de zichtbare skills
daarmee in overeenstemming te houden.

<AccordionGroup>
  <Accordion title="Skillwatcher">
    Standaard bewaakt OpenClaw skillmappen en werkt het de momentopname bij wanneer
    `SKILL.md`-bestanden veranderen. Configureer dit onder `skills.load`:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // standaard
        },
      },
    }
    ```

    Watchergebeurtenissen gebruiken een ingebouwde debounce van 250 ms. Gebruik `allowSymlinkTargets`
    voor opzettelijke indelingen met symbolische koppelingen waarbij een symbolische koppeling van een skillhoofdmap buiten de geconfigureerde hoofdmap wijst, bijvoorbeeld
    `<workspace>/skills/manager -> ~/Projects/manager/skills`.
    Schakel `skills.workshop.allowSymlinkTargetWrites` alleen in wanneer Skill Workshop
    voorstellen ook via die vertrouwde paden met symbolische koppelingen moet toepassen.

  </Accordion>
  <Accordion title="Externe macOS-nodes (Linux-gateway)">
    Als de Gateway op Linux wordt uitgevoerd, maar een **macOS-node** is verbonden waarop
    `system.run` is toegestaan, kan OpenClaw uitsluitend voor macOS bestemde skills als geschikt beschouwen wanneer
    de vereiste binaire bestanden op die node aanwezig zijn. De agent moet die
    skills uitvoeren via de tool `exec` met `host=node`.

    Offline nodes maken uitsluitend externe skills **niet** zichtbaar. Als een node niet meer
    reageert op controles naar binaire bestanden, wist OpenClaw de overeenkomsten met binaire bestanden uit de cache.

  </Accordion>
</AccordionGroup>

## Tokenimpact

Wanneer skills in aanmerking komen, injecteert OpenClaw een compact XML-blok in de
systeemprompt. De kosten zijn deterministisch en nemen lineair toe per skill:

- **Basisoverhead** (alleen wanneer 1 of meer skills in aanmerking komen): een vast blok inleidende
  tekst plus de `<available_skills>`-wrapper.
- **Per skill:** ~97 tekens + de veldlengtes van je `name`, `description` en `location`.
- XML-escaping zet `& < > " '` om in entiteiten, waardoor per
  voorkomen enkele tekens worden toegevoegd.
- Bij ~4 tekens/token zijn 97 tekens ≈ 24 tokens per skill, vóór de veldlengtes.

Als het gerenderde blok het geconfigureerde promptbudget
(`skills.limits.maxSkillsPromptChars`) zou overschrijden, behoudt OpenClaw eerst zoveel mogelijk
skillidentiteiten (naam, locatie en versie) als in de compacte indeling zonder
beschrijvingen passen. Vervolgens wordt het resterende budget gebruikt voor
ingekorte beschrijvingen. Als er geen budget voor beschrijvingen overblijft,
worden beschrijvingen weggelaten. De prompt bevat een opmerking die verwijst
naar `openclaw skills check` wanneer compacte opmaak of inkorting van de lijst
nodig is.

Houd beschrijvingen kort en duidelijk om de promptoverhead te beperken.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Skills maken" href="/nl/tools/creating-skills" icon="hammer">
    Stapsgewijze handleiding voor het maken van een aangepaste skill.
  </Card>
  <Card title="Skillworkshop" href="/nl/tools/skill-workshop" icon="flask">
    Wachtrij met voorstellen voor door agents opgestelde skills.
  </Card>
  <Card title="Skills-configuratie" href="/nl/tools/skills-config" icon="gear">
    Volledig configuratieschema voor `skills.*` en toelatingslijsten voor agents.
  </Card>
  <Card title="Slashcommando's" href="/nl/tools/slash-commands" icon="terminal">
    Hoe slashcommando's voor skills worden geregistreerd en gerouteerd.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Bekijk en publiceer skills in het openbare register.
  </Card>
  <Card title="Plugins" href="/nl/tools/plugin" icon="plug">
    Plugins kunnen skills meeleveren naast de tools die ze documenteren.
  </Card>
</CardGroup>
