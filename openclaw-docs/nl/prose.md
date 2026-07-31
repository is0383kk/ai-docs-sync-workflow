---
read_when:
    - Je wilt .prose-workflowbestanden uitvoeren of schrijven
    - Je wilt de OpenProse-plugin inschakelen
    - Je moet begrijpen hoe OpenProse wordt gekoppeld aan OpenClaw-primitieven
sidebarTitle: OpenProse
summary: OpenProse is een workflowindeling met Markdown als uitgangspunt voor AI-sessies met meerdere agents. In OpenClaw wordt het geleverd als een Plugin met een `/prose`-slashcommando en een Skills-pakket.
title: OpenProse
x-i18n:
    generated_at: "2026-07-27T05:18:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b04eb23bf827fbec6db11c1e95993e7f6c617451c5f4fda771ad078674c12bc
    source_path: prose.md
    workflow: 16
---

OpenProse is een overdraagbare, op Markdown gerichte workflowindeling voor het orkestreren van AI-
sessies. In OpenClaw wordt het geleverd als een plugin die een OpenProse-Skills-
pakket en een `/prose`-slashopdracht installeert. Programma's staan in `.prose`-bestanden en kunnen
meerdere subagents starten met een expliciete besturingsstroom.

<CardGroup cols={3}>
  <Card title="Installeren" icon="download" href="#install">
    Schakel de OpenProse-plugin in en herstart de Gateway.
  </Card>
  <Card title="Een programma uitvoeren" icon="play" href="#slash-command">
    Gebruik `/prose run` om een `.prose`-bestand of extern programma uit te voeren.
  </Card>
  <Card title="Programma's schrijven" icon="pencil" href="#example-parallel-research-and-synthesis">
    Schrijf multi-agentworkflows met parallelle en opeenvolgende stappen.
  </Card>
</CardGroup>

## Installeren

<Steps>
  <Step title="De plugin inschakelen">
    OpenProse wordt meegeleverd, maar is standaard uitgeschakeld. Schakel de plugin in:

    ```bash
    openclaw plugins enable open-prose
    ```

  </Step>
  <Step title="De Gateway herstarten">
    ```bash
    openclaw gateway restart
    ```
  </Step>
  <Step title="Verifiëren">
    ```bash
    openclaw plugins list | grep prose
    ```

    Je zou `open-prose` als ingeschakeld moeten zien. De Skills-opdracht `/prose` is nu
    beschikbaar in de chat.

  </Step>
</Steps>

Vanuit een uitgecheckte repository kun je de plugin rechtstreeks installeren:
`openclaw plugins install ./extensions/open-prose`

## Slashopdracht

OpenProse registreert `/prose` als een door de gebruiker aanroepbare Skills-opdracht:

```text
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

`/prose run <handle/slug>` wordt omgezet in `https://p.prose.md/<handle>/<slug>`.
Rechtstreekse URL's worden ongewijzigd opgehaald met de tool `web_fetch`.

Externe uitvoeringen op het hoogste niveau zijn expliciet. Externe imports binnen een `.prose`-programma zijn
transitieve codeafhankelijkheden: voordat OpenProse een extern `use`-doel ophaalt,
toont het de omgezette importlijst en moet de operator voor die uitvoering exact
`approve remote prose imports` antwoorden.

## Mogelijkheden

- Multi-agentonderzoek en -synthese met expliciet parallellisme.
- Herhaalbare, goedkeuringsveilige workflows (codebeoordeling, incidenttriage, contentpijplijnen).
- Herbruikbare `.prose`-programma's die je in ondersteunde agentruntimes kunt uitvoeren.

## Voorbeeld: parallel onderzoek en synthese

```prose
# Onderzoek + synthese met twee agents die parallel worden uitgevoerd.

input topic: "Wat moeten we onderzoeken?"

agent researcher:
  model: sonnet
  prompt: "Je doet grondig onderzoek en vermeldt bronnen."

agent writer:
  model: opus
  prompt: "Je schrijft een beknopte samenvatting."

parallel:
  findings = session: researcher
    prompt: "Onderzoek {topic}."
  draft = session: writer
    prompt: "Vat {topic} samen."

session "Voeg de bevindingen + het concept samen tot een definitief antwoord."
  context: { findings, draft }
```

## Toewijzing aan de OpenClaw-runtime

OpenProse-programma's worden toegewezen aan OpenClaw-primitieven:

| OpenProse-concept          | OpenClaw-tool                                   |
| ------------------------- | ----------------------------------------------- |
| Sessie starten / Task-tool | `sessions_spawn`                                |
| Bestand lezen / schrijven | `read` / `write`                                |
| Web ophalen               | `web_fetch` (`exec` + curl wanneer POST nodig is) |

<Warning>
  Als je lijst met toegestane tools `sessions_spawn`, `read`, `write` of
  `web_fetch` blokkeert, mislukken OpenProse-programma's. Controleer je
  [configuratie voor de lijst met toegestane tools](/nl/gateway/config-tools).
</Warning>

## Bestandslocaties

OpenProse bewaart de status onder `.prose/` in je werkruimte:

```text
.prose/
├── .env                      # configuratie (sleutel=waarde), bijvoorbeeld OPENPROSE_POSTGRES_URL
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose     # kopie van het uitgevoerde programma
│       ├── state.md          # uitvoeringsstatus
│       ├── bindings/
│       ├── imports/          # geneste uitvoeringen van externe programma's
│       └── agents/
└── agents/                   # permanente agents binnen het projectbereik
```

Permanente agents op gebruikersniveau (gedeeld tussen projecten) staan in:

```text
~/.prose/agents/
```

## Statusbackends

<AccordionGroup>
  <Accordion title="bestandssysteem (standaard)">
    De status wordt in de werkruimte naar `.prose/runs/...` geschreven. Geen extra
    afhankelijkheden vereist.
  </Accordion>
  <Accordion title="in context">
    Tijdelijke status die in het contextvenster wordt bewaard; selecteer met `--in-context`.
    Geschikt voor kleine, kortlopende programma's.
  </Accordion>
  <Accordion title="sqlite (experimenteel)">
    Selecteer met `--state=sqlite`. Vereist het binaire bestand `sqlite3` op `PATH`
    (valt terug op het bestandssysteem wanneer dit ontbreekt); de status wordt opgeslagen in
    `.prose/runs/{id}/state.db`.
  </Accordion>
  <Accordion title="postgres (experimenteel)">
    Selecteer met `--state=postgres`. Vereist `psql` en een verbindingsreeks in
    `OPENPROSE_POSTGRES_URL` (stel deze in `.prose/.env` in).

    <Warning>
      Postgres-referenties komen in de logboeken van subagents terecht. Gebruik een afzonderlijke
      database met minimale bevoegdheden.
    </Warning>

  </Accordion>
</AccordionGroup>

## Beveiliging

Behandel `.prose`-bestanden als code. Controleer ze vóór uitvoering, inclusief externe
`use`-imports. `/prose run https://...`-verzoeken op het hoogste niveau zijn expliciet, maar
voor transitieve externe imports is per uitvoering goedkeuring vereist voordat ze worden opgehaald of
uitgevoerd. Gebruik OpenClaw-lijsten met toegestane tools en goedkeuringspoorten om neveneffecten
te beheersen. Vergelijk voor deterministische workflows met verplichte goedkeuring met
[Lobster](/nl/tools/lobster).

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Skills-referentie" href="/nl/tools/skills" icon="puzzle-piece">
    Hoe het Skills-pakket van OpenProse wordt geladen en welke poorten van toepassing zijn.
  </Card>
  <Card title="Subagents" href="/nl/tools/subagents" icon="users">
    De ingebouwde multi-agentcoördinatielaag van OpenClaw.
  </Card>
  <Card title="Tekst-naar-spraak" href="/nl/tools/tts" icon="volume-high">
    Voeg audio-uitvoer toe aan je workflows.
  </Card>
  <Card title="Slashopdrachten" href="/nl/tools/slash-commands" icon="terminal">
    Alle beschikbare chatopdrachten, inclusief /prose.
  </Card>
</CardGroup>

Officiële website: [https://www.prose.md](https://www.prose.md)
