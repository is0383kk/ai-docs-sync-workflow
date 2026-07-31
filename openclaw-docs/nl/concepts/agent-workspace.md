---
read_when:
    - Je moet de agentwerkruimte of de bestandsindeling ervan uitleggen
    - Je wilt een agentwerkruimte back-uppen of migreren
sidebarTitle: Agent workspace
summary: 'Agentwerkruimte: locatie, indeling en back-upstrategie'
title: Agentwerkruimte
x-i18n:
    generated_at: "2026-07-27T05:47:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b58ead9079c3dda4bcaec3253f8d55e67e7e554d5c5b87ccfec6b08ec4ba038f
    source_path: concepts/agent-workspace.md
    workflow: 16
---

De werkruimte is de thuisbasis van de agent: de werkmap die wordt gebruikt voor bestandstools
en werkruimtecontext. Houd deze privé en behandel deze als geheugen.

Dit staat los van `~/.openclaw/`, waarin configuratie, aanmeldgegevens en sessies worden opgeslagen.

<Warning>
De werkruimte is de **standaard-cwd**, geen harde sandbox. Tools lossen relatieve paden op ten opzichte van de werkruimte, maar absolute paden kunnen nog steeds andere locaties op de host bereiken, tenzij sandboxing is ingeschakeld. Gebruik [`agents.defaults.sandbox`](/nl/gateway/sandboxing) (en/of sandboxconfiguratie per agent) als je isolatie nodig hebt.

Wanneer sandboxing is ingeschakeld en `workspaceAccess` niet `"rw"` is, werken tools in een sandboxwerkruimte onder `~/.openclaw/sandboxes`, niet in je hostwerkruimte.
</Warning>

## Standaardlocatie

- Standaard: `~/.openclaw/workspace`
- Als `OPENCLAW_PROFILE` is ingesteld en niet `"default"` is, wordt de standaardwaarde `~/.openclaw/workspace-<profile>`.
- `OPENCLAW_WORKSPACE_DIR` overschrijft beide bovenstaande waarden wanneer deze is ingesteld.
- Niet-standaardagents (`agents.entries.*`) zonder expliciete werkruimte worden omgezet naar `<state-dir>/workspace-<agentId>`, niet naar de gedeelde standaardwerkruimte.

Overschrijven in `~/.openclaw/openclaw.json`:

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

Overschrijving per agent: `agents.entries.*.workspace`.

`openclaw onboard`, `openclaw configure` of `openclaw setup` maken de werkruimte aan en vullen de bootstrapbestanden als deze ontbreken.

<Note>
Bij het vullen van een sandbox worden alleen reguliere bestanden binnen de werkruimte gekopieerd; aliassen via symbolische of harde koppelingen die buiten de bronwerkruimte worden omgezet, worden genegeerd.
</Note>

Als je de werkruimtebestanden al zelf beheert, schakel je het aanmaken van bootstrapbestanden uit:

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Extra werkruimtemappen

Oudere installaties hebben mogelijk `~/openclaw` aangemaakt. Het behouden van meerdere werkruimtemappen kan verwarrende afwijkingen in authenticatie of status veroorzaken, omdat er slechts één werkruimte tegelijk actief is.

<Note>
**Aanbeveling:** behoud één actieve werkruimte. Als je de extra mappen niet meer gebruikt, archiveer ze dan of verplaats ze naar de prullenmand (bijvoorbeeld `trash ~/openclaw`). Als je bewust meerdere werkruimten behoudt, controleer dan of `agents.defaults.workspace` (of de sleutel `workspace` per agent) naar de actieve werkruimte verwijst.
</Note>

## Overzicht van werkruimtebestanden

Standaardbestanden die OpenClaw in de werkruimte verwacht:

<AccordionGroup>
  <Accordion title="AGENTS.md - bedieningsinstructies">
    Bedieningsinstructies voor de agent en hoe deze het geheugen moet gebruiken. Wordt aan het begin van elke sessie geladen. Een goede plek voor regels, prioriteiten en details over het gewenste gedrag.
  </Accordion>
  <Accordion title="SOUL.md - persona en toon">
    Persona, toon en grenzen. Wordt elke sessie geladen. Handleiding: [persoonlijkheidshandleiding voor SOUL.md](/nl/concepts/soul).
  </Accordion>
  <Accordion title="USER.md - wie de gebruiker is">
    Wie de gebruiker is en hoe deze moet worden aangesproken. Wordt elke sessie geladen.
  </Accordion>
  <Accordion title="IDENTITY.md - naam, uitstraling, emoji">
    De naam, uitstraling en emoji van de agent. Wordt tijdens het bootstrapritueel aangemaakt of bijgewerkt.
  </Accordion>
  <Accordion title="TOOLS.md - conventies voor lokale tools">
    Opmerkingen over je lokale tools en conventies. Bepaalt niet welke tools beschikbaar zijn; het dient alleen als richtlijn.
  </Accordion>
  <Accordion title="HEARTBEAT.md - Heartbeat-checklist">
    Optionele, kleine checklist voor Heartbeat-uitvoeringen. Houd deze kort om tokenverbruik te beperken.
  </Accordion>
  <Accordion title="BOOT.md - opstartchecklist">
    Optionele opstartchecklist die automatisch wordt uitgevoerd wanneer de Gateway opnieuw wordt gestart (als [interne hooks](/nl/automation/hooks) zijn ingeschakeld). Houd deze kort; gebruik de berichtentool voor uitgaande verzendingen.
  </Accordion>
  <Accordion title="BOOTSTRAP.md - ritueel voor de eerste uitvoering">
    Eenmalig ritueel voor de eerste uitvoering. Wordt alleen voor een gloednieuwe werkruimte aangemaakt. Verwijder het nadat het ritueel is voltooid.
  </Accordion>
  <Accordion title="memory/YYYY-MM-DD.md - dagelijks geheugenlogboek">
    Dagelijks geheugenlogboek (één bestand per dag). Het wordt aanbevolen om bij het starten van een sessie dat van vandaag en gisteren te lezen.
  </Accordion>
  <Accordion title="MEMORY.md - samengesteld langetermijngeheugen (optioneel)">
    Samengesteld langetermijngeheugen: duurzame feiten, voorkeuren, beslissingen en korte samenvattingen. Bewaar gedetailleerde logboeken in `memory/YYYY-MM-DD.md`, zodat geheugentools deze op verzoek kunnen ophalen zonder ze in elke prompt te injecteren. Laad `MEMORY.md` alleen in de persoonlijke hoofdsessie (niet in gedeelde of groepscontexten). Zie [Geheugen](/nl/concepts/memory) voor de workflow en het automatisch wegschrijven van het geheugen.
  </Accordion>
  <Accordion title="skills/ - werkruimte-Skills (optioneel)">
    Werkruimtespecifieke Skills. De locatie met de hoogste prioriteit voor Skills in die werkruimte, vóór projectagentskills, persoonlijke agentskills, beheerde Skills, meegeleverde Skills en `skills.load.extraDirs` wanneer namen conflicteren.
  </Accordion>
  <Accordion title="canvas/ - Canvas-UI-bestanden (optioneel)">
    Canvas-UI-bestanden voor Node-weergaven (bijvoorbeeld `canvas/index.html`).
  </Accordion>
</AccordionGroup>

<Note>
Als een bootstrapbestand ontbreekt, injecteert OpenClaw een markering voor een ontbrekend bestand in de sessie en gaat het verder. Grote bootstrapbestanden worden bij injectie afgekapt; pas de limieten aan met `agents.defaults.bootstrapMaxChars` (standaard: `20000`) en `agents.defaults.bootstrapTotalMaxChars` (standaard: `60000`). `openclaw setup` kan ontbrekende standaardbestanden opnieuw aanmaken zonder bestaande bestanden te overschrijven.
</Note>

## Wat NIET in de werkruimte staat

Deze bevinden zich onder `~/.openclaw/` en mogen NIET aan de werkruimterepository worden toegevoegd:

- `~/.openclaw/openclaw.json` (configuratie)
- `~/.openclaw/state/openclaw.sqlite` (gedeelde instellingsstatus en attestaties van de werkruimte)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (profielen voor modelauthenticatie: OAuth + API-sleutels)
- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` (sessierijen, transcripten en runtimestatus per agent)
- `~/.openclaw/agents/<agentId>/agent/codex-home/` (Codex-runtimeaccount, configuratie, Skills, plugins en systeemeigen threadstatus per agent)
- `~/.openclaw/credentials/` (kanaal-/providerstatus plus verouderde OAuth-importgegevens)
- `~/.openclaw/agents/<agentId>/sessions/` (verouderde migratiebronnen en archief-/ondersteuningsartefacten)
- `~/.openclaw/skills/` (beheerde Skills)

Als je sessies of configuratie moet migreren, kopieer deze dan afzonderlijk en houd ze buiten versiebeheer.

Oudere releases van OpenClaw schreven de werkruimte-sidecars `openclaw-workspace-state.json`,
`.openclaw/workspace-state.json` en `.attested`. De huidige
runtime gebruikt voor die status alleen de gedeelde SQLite-database. Als Doctor
een van deze bestanden meldt, voer dan `openclaw doctor --fix` uit; Doctor importeert geldige verouderde
status en verwijdert een bron pas nadat de databaserijen zijn geverifieerd.

## Git-back-up (aanbevolen, privé)

Behandel de werkruimte als privégeheugen. Plaats deze in een **privé**-gitrepository, zodat er een back-up van wordt gemaakt en herstel mogelijk is.

Voer deze stappen uit op de machine waarop de Gateway draait (daar bevindt de werkruimte zich).

<Steps>
  <Step title="De repository initialiseren">
    Als git is geïnstalleerd, worden gloednieuwe werkruimten automatisch geïnitialiseerd. Als deze werkruimte nog geen repository is, voer je het volgende uit:

    ```bash
    cd ~/.openclaw/workspace
    git init
    git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
    git commit -m "Add agent workspace"
    ```

  </Step>
  <Step title="Een privéremote toevoegen">
    <Tabs>
      <Tab title="GitHub-web-UI">
        1. Maak een nieuwe **privé**repository op GitHub.
        2. Initialiseer deze niet met een README (dit voorkomt samenvoegingsconflicten).
        3. Kopieer de HTTPS-remote-URL.
        4. Voeg de remote toe en push:

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
      <Tab title="GitHub CLI (gh)">
        ```bash
        gh auth login
        gh repo create openclaw-workspace --private --source . --remote origin --push
        ```
      </Tab>
      <Tab title="GitLab-web-UI">
        1. Maak een nieuwe **privé**repository op GitLab.
        2. Initialiseer deze niet met een README (dit voorkomt samenvoegingsconflicten).
        3. Kopieer de HTTPS-remote-URL.
        4. Voeg de remote toe en push:

        ```bash
        git branch -M main
        git remote add origin <https-url>
        git push -u origin main
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="Doorlopende updates">
    ```bash
    git status
    git add .
    git commit -m "Update memory"
    git push
    ```
  </Step>
</Steps>

## Leg geen geheimen vast

<Warning>
Vermijd zelfs in een privérepository het opslaan van geheimen in de werkruimte:

- API-sleutels, OAuth-tokens, wachtwoorden of privéaanmeldgegevens.
- Alles onder `~/.openclaw/`.
- Onbewerkte exports van chats of gevoelige bijlagen.

Als je gevoelige verwijzingen moet opslaan, gebruik dan plaatshouders en bewaar het echte geheim elders (wachtwoordbeheerder, omgevingsvariabelen of `~/.openclaw/`).
</Warning>

Voorgestelde basisinhoud voor `.gitignore`:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## De werkruimte naar een nieuwe machine verplaatsen

<Steps>
  <Step title="De repository klonen">
    Kloon de repository naar het gewenste pad (standaard `~/.openclaw/workspace`).
  </Step>
  <Step title="De configuratie bijwerken">
    Stel `agents.defaults.workspace` in op dat pad in `~/.openclaw/openclaw.json`.
  </Step>
  <Step title="Ontbrekende bestanden vullen">
    Voer `openclaw setup --workspace <path>` uit om ontbrekende bestanden te vullen.
  </Step>
  <Step title="Sessies kopiëren (optioneel)">
    Als je sessies nodig hebt, kopieer je `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
    afzonderlijk vanaf de oude machine. Kopieer `~/.openclaw/agents/<agentId>/sessions/`
    alleen als je ook verouderde migratie-invoer of archief-/ondersteuningsartefacten nodig hebt.
  </Step>
</Steps>

## Geavanceerde opmerkingen

- Routering met meerdere agents kan via `agents.entries.*.workspace` verschillende werkruimten per agent gebruiken. Zie [Kanaalroutering](/nl/channels/channel-routing) voor de routeringsconfiguratie.
- Als `agents.defaults.sandbox` is ingeschakeld, kunnen niet-hoofdsessies sandboxwerkruimten per sessie onder `agents.defaults.sandbox.workspaceRoot` gebruiken.

## Gerelateerd

- [Heartbeat](/nl/gateway/heartbeat) - HEARTBEAT.md-werkruimtebestand
- [Sandboxing](/nl/gateway/sandboxing) - toegang tot de werkruimte in sandboxomgevingen
- [Sessie](/nl/concepts/session) - opslagpaden voor sessies
- [Vaste instructies](/nl/automation/standing-orders) - permanente instructies in werkruimtebestanden
